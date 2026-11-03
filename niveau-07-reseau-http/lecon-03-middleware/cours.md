# Leçon 3 — Middleware

> Journaliser chaque requête, attribuer un identifiant, récupérer les paniques, vérifier
> l'authentification, mesurer la latence : ces besoins concernent **toutes** les routes. Les écrire
> dans chaque handler serait absurde. Un **middleware** est un handler qui en enveloppe un autre —
> c'est le décorateur de la leçon 5 du niveau 2, appliqué à `http.Handler`. En Go, il n'y a ni
> framework ni annotation : une fonction `Handler → Handler` suffit.

## Objectifs

1. Écrire un middleware et composer plusieurs middlewares dans le bon ordre.
2. Capturer le statut et la taille d'une réponse sans casser `http.ResponseController`.
3. Propager une donnée de requête (identifiant) par le contexte.
4. Transformer une panique en réponse 500 propre.

## Explication

### La forme

```go
func Logging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)                           // appeler le handler enveloppé
		log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
	})
}
```

Un middleware reçoit le handler **suivant** et rend un nouveau handler. Il peut agir **avant** l'appel
(lire, refuser, enrichir la requête), **après** (mesurer, journaliser), ou **ne pas appeler** `next`
(refuser une requête non authentifiée, leçon 8). Il ne connaît rien des routes.

Un middleware **configurable** est une fonction qui **rend** un middleware :

```go
func Logging(out io.Writer) func(http.Handler) http.Handler { … }
```

C'est la même idée que les options fonctionnelles (N1 L11) : la configuration est capturée par la
closure.

### Composer, et l'ordre

```go
h := A(B(C(mux)))
```

La requête traverse A, puis B, puis C, puis le routeur ; la réponse remonte C, B, A. A est le **plus
extérieur**. Une fonction `Chain(h, A, B, C)` rend cette composition lisible dans l'ordre de traversée.

L'ordre a des conséquences précises :

| Si… | alors… |
|---|---|
| `Recover` est **à l'intérieur** de `Logging` | la panique devient un 500 **avant** que `Logging` mesure : le journal affiche 500 |
| `Recover` est **à l'extérieur** de `Logging` | `Logging` est interrompu par la panique et ne journalise rien |
| `RequestID` est **à l'extérieur** de `Logging` | le journal contient l'identifiant |
| l'authentification est **avant** la journalisation | les requêtes refusées ne sont pas journalisées — voulu ? |

Règle habituelle, de l'extérieur vers l'intérieur : identifiant → journalisation/métriques → récupération
→ délais et limites → authentification → routeur.

### Capturer le statut

`http.ResponseWriter` n'a aucune méthode pour **lire** le statut écrit. Pour le journaliser, on
**enveloppe** le `ResponseWriter` dans un type qui intercepte `WriteHeader` et `Write` (en embarquant
l'original, N2 L5) :

```go
type statusRecorder struct {
	http.ResponseWriter
	status int
}

func (s *statusRecorder) WriteHeader(code int) { s.status = code; s.ResponseWriter.WriteHeader(code) }
```

Deux pièges :

- un handler qui écrit sans appeler `WriteHeader` envoie un **200 implicite** : il faut l'intercepter
  dans `Write` aussi ;
- l'enveloppe **masque** les interfaces optionnelles du `ResponseWriter` d'origine (`http.Flusher`,
  `http.Hijacker`…) — le problème du décorateur vu en N4 L5 (défi). Depuis Go 1.20, il suffit d'ajouter
  une méthode **`Unwrap() http.ResponseWriter`** : `http.ResponseController` (leçon 2) sait alors
  descendre jusqu'au vrai `ResponseWriter`.

### Transmettre une donnée : le contexte

Un middleware ne peut pas ajouter de paramètre au handler. Il transmet ce qu'il sait (identifiant de
requête, utilisateur authentifié) par le **contexte** de la requête, avec une clé de type non exporté
et des accesseurs typés (N6 L8) :

```go
next.ServeHTTP(w, r.WithContext(context.WithValue(r.Context(), requestIDKey{}, id)))
```

`r.WithContext` rend une **copie** superficielle de la requête avec le nouveau contexte. C'est un des
rares usages **légitimes** des valeurs de contexte : une donnée de requête qui traverse toutes les
couches.

### Récupérer une panique

Le serveur `net/http` récupère déjà les paniques (leçon 1, E4)… mais en **coupant la connexion** sans
réponse. Un middleware de récupération répond un vrai **500**, journalise la pile avec l'identifiant de
requête, et alimente une métrique. Une exception : `http.ErrAbortHandler`, une panique **volontaire**
qui signifie « abandonner cette réponse » ; on la relance telle quelle.

### Bibliothèques

L'écosystème offre des middlewares prêts à l'emploi (compression, CORS, limites de débit, traçage avec
OpenTelemetry, niveau 12). Tous ont la même forme `func(http.Handler) http.Handler` et se composent avec
les vôtres. Écrire les trois de cette leçon soi-même permet de comprendre — et de déboguer — ceux qu'on
importera.

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : mw/mw.go
// Package mw rassemble des middlewares HTTP : des fonctions Handler → Handler.
package mw

import (
	"context"
	"crypto/rand"
	"fmt"
	"io"
	"net/http"
	"runtime/debug"
	"time"
)

// Middleware enveloppe un handler pour lui ajouter un comportement.
type Middleware func(http.Handler) http.Handler

// Chain applique les middlewares dans l'ordre de lecture : Chain(h, A, B) = A(B(h)).
// A est donc le plus EXTÉRIEUR : il voit la requête en premier et la réponse en dernier.
func Chain(h http.Handler, mws ...Middleware) http.Handler {
	for i := len(mws) - 1; i >= 0; i-- {
		h = mws[i](h)
	}
	return h
}

// ---- Identifiant de requête, propagé par le contexte (N6 L8). ----

type requestIDKey struct{}

func RequestID(ctx context.Context) string {
	id, _ := ctx.Value(requestIDKey{}).(string)
	return id
}

// WithRequestID réutilise l'en-tête X-Request-ID s'il est fourni (par un proxy), sinon en crée un.
func WithRequestID(newID func() string) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			id := r.Header.Get("X-Request-ID")
			if id == "" || len(id) > 64 {
				id = newID()
			}
			w.Header().Set("X-Request-ID", id)
			next.ServeHTTP(w, r.WithContext(context.WithValue(r.Context(), requestIDKey{}, id)))
		})
	}
}

// RandomID : 128 bits d'aléa cryptographique (N4 L3).
func RandomID() string { return rand.Text() }

// ---- Journalisation : il faut CAPTURER le statut, que http.ResponseWriter ne rend pas. ----

type statusRecorder struct {
	http.ResponseWriter
	status int
	bytes  int
}

func (s *statusRecorder) WriteHeader(code int) {
	if s.status == 0 {
		s.status = code
	}
	s.ResponseWriter.WriteHeader(code)
}

func (s *statusRecorder) Write(p []byte) (int, error) {
	if s.status == 0 {
		s.status = http.StatusOK // un Write sans WriteHeader vaut 200 (leçon 2)
	}
	n, err := s.ResponseWriter.Write(p)
	s.bytes += n
	return n, err
}

// Unwrap permet à http.ResponseController (Flush, délais) de traverser l'enveloppe.
func (s *statusRecorder) Unwrap() http.ResponseWriter { return s.ResponseWriter }

func Logging(out io.Writer, now func() time.Time) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := now()
			rec := &statusRecorder{ResponseWriter: w}
			next.ServeHTTP(rec, r)
			fmt.Fprintf(out, "%s %s %s %d %dB %v\n",
				RequestID(r.Context()), r.Method, r.URL.Path, rec.status, rec.bytes, now().Sub(start))
		})
	}
}

// ---- Récupération : une panique devient un 500 propre, avec la pile journalisée (N6 L12). ----

func Recover(out io.Writer) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			defer func() {
				if v := recover(); v != nil {
					if v == http.ErrAbortHandler { // signal volontaire d'abandon : à relancer tel quel
						panic(v)
					}
					fmt.Fprintf(out, "%s panique : %v\n%s", RequestID(r.Context()), v, debug.Stack())
					http.Error(w, "erreur interne", http.StatusInternalServerError)
				}
			}()
			next.ServeHTTP(w, r)
		})
	}
}
```

```go
// Fichier : mw/mw_test.go
package mw

import (
	"bytes"
	"fmt"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"
)

func TestChain(t *testing.T) {
	var logs, panics bytes.Buffer
	clock := time.Date(2026, 11, 1, 10, 0, 0, 0, time.UTC)
	now := func() time.Time { clock = clock.Add(5 * time.Millisecond); return clock } // chaque lecture avance de 5 ms

	mux := http.NewServeMux()
	mux.HandleFunc("GET /hello", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "bonjour (%s)", RequestID(r.Context()))
	})
	mux.HandleFunc("GET /boom", func(w http.ResponseWriter, r *http.Request) {
		var m map[string]int
		m["x"]++ // panique
	})

	h := Chain(mux,
		WithRequestID(func() string { return "req-1" }),
		Logging(&logs, now),
		Recover(&panics),
	)

	for _, tc := range []struct {
		path, reqID, wantBody string
		wantStatus            int
	}{
		{"/hello", "", "bonjour (req-1)", 200},
		{"/hello", "proxy-42", "bonjour (proxy-42)", 200},
		{"/boom", "", "erreur interne\n", 500},
		{"/absent", "", "404 page not found\n", 404},
	} {
		req := httptest.NewRequest("GET", tc.path, nil)
		if tc.reqID != "" {
			req.Header.Set("X-Request-ID", tc.reqID)
		}
		rec := httptest.NewRecorder()
		h.ServeHTTP(rec, req)
		if rec.Code != tc.wantStatus || rec.Body.String() != tc.wantBody {
			t.Errorf("%s = %d %q, attendu %d %q", tc.path, rec.Code, rec.Body.String(), tc.wantStatus, tc.wantBody)
		}
	}

	want := "req-1 GET /hello 200 15B 5ms\n" +
		"proxy-42 GET /hello 200 18B 5ms\n" +
		"req-1 GET /boom 500 15B 5ms\n" +
		"req-1 GET /absent 404 19B 5ms\n"
	if logs.String() != want {
		t.Errorf("journal :\n%s\nattendu :\n%s", logs.String(), want)
	}
	if !strings.Contains(panics.String(), "assignment to entry in nil map") {
		t.Errorf("la panique n'a pas été journalisée : %q", panics.String())
	}
}
```

`go test -race ./mw` passe : le journal contient exactement

```
req-1 GET /hello 200 15B 5ms
proxy-42 GET /hello 200 18B 5ms
req-1 GET /boom 500 15B 5ms
req-1 GET /absent 404 19B 5ms
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `type Middleware func(http.Handler) http.Handler` | Un type fonction nommé (N1 L11) : les signatures restent lisibles. |
| `Chain` qui parcourt à l'envers | Pour que `Chain(h, A, B, C)` donne `A(B(C(h)))` : on lit les middlewares dans l'ordre de traversée. |
| `WithRequestID(newID)` | Le générateur est **injecté** : le test obtient `req-1` de façon déterministe ; en production, `RandomID` (`crypto/rand`). |
| Réutiliser `X-Request-ID` reçu (≤ 64 caractères) | Un proxy ou un service appelant peut fournir l'identifiant : la même requête garde le même identifiant à travers plusieurs services. La longueur est bornée : c'est une entrée client. |
| `statusRecorder` avec `Write` et `WriteHeader` | Capture le 200 implicite **et** les statuts explicites. |
| `Unwrap()` | Sans elle, un handler qui streame (leçon 2) perdrait la capacité de `Flush` derrière ce middleware. |
| Horloge injectée dans `Logging` | Le journal du test est **exact** (5 ms), pas « à peu près ». |
| `Recover` à l'intérieur de `Logging` | La panique de `/boom` est journalisée comme un 500, avec l'identifiant. |
| `http.ErrAbortHandler` relancé | On ne transforme pas un abandon volontaire en 500. |
| `/absent` → 404 journalisé | Les middlewares voient **toutes** les requêtes, y compris celles que le routeur rejette. |

## Erreurs fréquentes

1. **Mauvais ordre** : `Recover` à l'extérieur de `Logging`, identifiant ajouté après le journal.
2. **Oublier le 200 implicite** dans le `statusRecorder` : statut journalisé à 0.
3. **Pas d'`Unwrap`** : `Flush` et les délais par requête cessent de fonctionner.
4. **Données passées par variables globales** au lieu du contexte.
5. **Clé de contexte de type `string`** (collisions, N6 L8).
6. **Faire confiance à un `X-Request-ID` client** sans en borner la longueur.
7. **Récupérer les paniques sans journaliser la pile.**
8. **Un middleware qui lit le corps** (`io.ReadAll(r.Body)`) sans le remettre : le handler reçoit un corps vide.

## Bonnes pratiques Go

- `func(http.Handler) http.Handler` ; configuration par closure.
- Une fonction `Chain` et un ordre documenté.
- Enveloppes de `ResponseWriter` avec `Unwrap`.
- Contexte pour les données de requête, avec accesseurs typés.
- Dépendances (horloge, sortie, générateur) injectées : chaque middleware se teste seul.

## Ce que je dois retenir

- Middleware = `Handler → Handler` ; agit avant, après, ou à la place de `next`.
- `A(B(C(h)))` : A voit la requête en premier et la réponse en dernier ; l'ordre change le comportement.
- Pour lire le statut, envelopper le `ResponseWriter` (et intercepter le 200 implicite) ; ajouter `Unwrap`.
- Données de requête par `r.WithContext(context.WithValue(…))`.
- Récupération : 500 + pile journalisée ; relancer `http.ErrAbortHandler`.

---

➡️ [Exercices](exercices.md)
