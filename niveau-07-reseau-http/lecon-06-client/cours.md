# Leçon 6 — Client HTTP

> Un service appelle presque toujours d'autres services. Le client HTTP de Go tient en une ligne —
> `http.Get(url)` — et cette ligne contient quatre bugs de production : **aucun délai**, un statut
> d'erreur qui n'est **pas** une erreur, un corps qui doit être **fermé**, et aucune politique de
> **réessai**. Cette leçon écrit un client correct, et explique ce que fait le `Transport` en dessous.

## Objectifs

1. Configurer `http.Client` avec des délais, et comprendre le rôle du `Transport`.
2. Envoyer des requêtes avec contexte, et toujours fermer le corps.
3. Traiter les statuts non 2xx comme des erreurs, explicitement.
4. Réessayer seulement ce qui est réessayable, avec *backoff* et *jitter*.

## Explication

### `http.Get` et le client par défaut

`http.Get` utilise `http.DefaultClient`, qui n'a **aucun délai**. Un serveur distant qui accepte la
connexion puis ne répond jamais bloque la goroutine appelante **pour toujours** — et dans un service,
chaque requête entrante qui fait cet appel ajoute une goroutine bloquée (une fuite, N6 L10). Règle : ne
jamais utiliser `http.DefaultClient` (ni `http.Get`, `http.Post`) dans du code de production.

```go
client := &http.Client{Timeout: 10 * time.Second}
```

`Client.Timeout` borne **toute** la tentative : connexion, envoi, attente des en-têtes, et **lecture
du corps**. Un délai plus fin par appel passe par le **contexte** de la requête (leçon 7).

### Un client, réutilisé

Un `http.Client` est **sûr pour un usage concurrent** et doit être **réutilisé** : il porte un
`Transport` qui maintient un **pool de connexions** (*keep-alive*). Créer un client par requête, c'est
ouvrir une connexion TCP (et une poignée de main TLS) à chaque appel — des dizaines de millisecondes
perdues, et des milliers de sockets en `TIME_WAIT` sous charge.

Le `Transport` se règle pour les cas exigeants (`MaxIdleConnsPerHost`, défaut 2, souvent trop bas pour
un service qui appelle beaucoup le même hôte ; délais de connexion, de TLS, d'attente des en-têtes).
On crée **un** client par service distant, configuré une fois.

### Requête avec contexte

```go
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
req.Header.Set("Accept", "application/json")
resp, err := client.Do(req)
```

Le contexte permet d'annuler l'appel quand l'appelant n'en a plus besoin (le client de **notre**
service s'est déconnecté, leçon 7). Les paramètres d'URL s'encodent avec `url.QueryEscape` ou
`url.Values` — jamais par concaténation brute (`"Saint-Étienne"` contient un espace potentiel et des
caractères non ASCII).

### Le statut n'est pas une erreur

`client.Do` rend une erreur **seulement** pour un problème de transport (réseau, délai, annulation).
Un **404** ou un **500** est une réponse **réussie** du point de vue du client HTTP : `err == nil`.
Il faut tester `resp.StatusCode` soi-même, et le transformer en erreur typée (`*StatusError`) que
l'appelant pourra examiner avec `errors.As`.

### Fermer le corps — et ce qui a changé en Go 1.27

```go
resp, err := client.Do(req)
if err != nil {
	return err
}
defer resp.Body.Close()
```

**Toujours** fermer `resp.Body`, même si on ne le lit pas, même pour une réponse d'erreur. Un corps non
fermé garde la connexion occupée : elle ne retourne jamais dans le pool. Mesuré sur 20 requêtes :

| Traitement du corps | Connexions TCP ouvertes |
|---|---|
| lu entièrement puis fermé | 1 |
| fermé sans être lu (Go 1.27) | 1 |
| ni lu ni fermé | **20** |

Pendant des années, la consigne était de **lire tout le corps** avant de le fermer (`io.Copy(io.Discard,
resp.Body)`), sans quoi la connexion n'était pas réutilisée. Depuis Go 1.27, `Close` **draine le corps
lui-même**, de façon asynchrone, jusqu'à une limite raisonnable : la documentation précise qu'une
lecture complète manuelle n'est plus nécessaire dans la plupart des cas. On rencontrera encore
beaucoup de `io.Copy(io.Discard, …)` dans le code existant ; il reste utile pour un corps qu'on sait
volumineux. Ce qui ne change pas : sans `Close`, rien ne se passe.

Et toujours **borner** la lecture d'un corps distant (`io.LimitReader`) : un service distant défaillant
ou malveillant peut renvoyer des gigaoctets.

### Réessayer, mais pas n'importe quoi

| Situation | Réessayer ? |
|---|---|
| erreur réseau (connexion refusée, coupée) | oui, si la méthode est **idempotente** |
| 502, 503, 504 | oui |
| 429 Too Many Requests | oui, en respectant `Retry-After` |
| 500 | souvent (c'est peut-être transitoire), avec un plafond |
| 4xx (400, 401, 404, 422…) | **non** : la même requête échouera de la même façon |
| contexte annulé ou expiré | **non** : l'appelant a abandonné |
| `POST` non idempotent | **non**, sauf avec une clé d'idempotence (leçon 4) |

Entre deux tentatives : une attente **exponentielle** (100 ms, 200 ms, 400 ms…) **avec jitter** (une
part aléatoire). Sans jitter, mille clients qui ont échoué en même temps réessaient tous en même temps
et recréent le pic qui a fait tomber le service (la tempête de réessais, N6 L11). L'attente elle-même
doit écouter le contexte. Et le nombre de tentatives est **plafonné** : trois ou quatre, pas plus.

Le niveau 10 (leçon 9) généralisera : budgets de réessais, disjoncteurs.

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : weather/client.go
// Package weather est un client de l'API (fictive) de météo.
package weather

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"math/rand/v2"
	"net/http"
	"net/url"
	"time"
)

// StatusError : une réponse HTTP non 2xx. Le client HTTP de Go NE la considère PAS comme une erreur.
type StatusError struct {
	Code int
	Body string // début du corps, pour le diagnostic
}

func (e *StatusError) Error() string { return fmt.Sprintf("statut %d : %s", e.Code, e.Body) }

type Forecast struct {
	City  string  `json:"city"`
	TempC float64 `json:"temp_c"`
}

type Client struct {
	base    string
	http    *http.Client
	retries int
	sleep   func(context.Context, time.Duration) error // injectable : pas de vraie attente en test
}

func New(base string) *Client {
	return &Client{
		base: base,
		http: &http.Client{
			Timeout: 10 * time.Second, // délai TOTAL d'une tentative : connexion + en-têtes + corps
		},
		retries: 3,
		sleep:   sleepCtx,
	}
}

func sleepCtx(ctx context.Context, d time.Duration) error {
	t := time.NewTimer(d)
	defer t.Stop()
	select {
	case <-t.C:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

// retryable : erreurs réseau et statuts transitoires seulement. Jamais un 4xx (sauf 429).
func retryable(err error) bool {
	var se *StatusError
	if errors.As(err, &se) {
		return se.Code == http.StatusTooManyRequests || se.Code >= 500
	}
	return err != nil && !errors.Is(err, context.Canceled) && !errors.Is(err, context.DeadlineExceeded)
}

// Forecast est un GET : idempotent, donc on peut réessayer sans risque.
func (c *Client) Forecast(ctx context.Context, city string) (Forecast, error) {
	u := c.base + "/v1/forecast?city=" + url.QueryEscape(city)
	var lastErr error
	for attempt := range c.retries + 1 {
		if attempt > 0 {
			// backoff exponentiel avec jitter : 100 ms, 200 ms, 400 ms… ± aléa (N6 L10 : livelock)
			d := (100 * time.Millisecond << (attempt - 1)) / 2
			d += rand.N(d)
			if err := c.sleep(ctx, d); err != nil {
				return Forecast{}, err
			}
		}
		f, err := c.once(ctx, u)
		if err == nil || !retryable(err) {
			return f, err
		}
		lastErr = err
	}
	return Forecast{}, fmt.Errorf("prévision pour %q après %d tentatives : %w", city, c.retries+1, lastErr)
}

func (c *Client) once(ctx context.Context, u string) (Forecast, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, u, nil)
	if err != nil {
		return Forecast{}, err
	}
	req.Header.Set("Accept", "application/json")
	resp, err := c.http.Do(req)
	if err != nil {
		return Forecast{}, err // erreur réseau, délai, annulation
	}
	defer resp.Body.Close() // TOUJOURS : sinon la connexion n'est jamais réutilisée

	if resp.StatusCode != http.StatusOK {
		b, _ := io.ReadAll(io.LimitReader(resp.Body, 200))
		return Forecast{}, &StatusError{Code: resp.StatusCode, Body: string(b)}
	}
	var f Forecast
	if err := json.NewDecoder(io.LimitReader(resp.Body, 1<<20)).Decode(&f); err != nil {
		return Forecast{}, fmt.Errorf("réponse illisible : %w", err)
	}
	return f, nil
}
```

```go
// Fichier : weather/client_test.go
package weather

import (
	"context"
	"errors"
	"fmt"
	"net/http"
	"net/http/httptest"
	"sync/atomic"
	"testing"
	"time"
)

// flaky répond `failures` fois avec `code`, puis correctement.
func flaky(failures int32, code int) (*httptest.Server, *atomic.Int32) {
	var calls atomic.Int32
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		n := calls.Add(1)
		if n <= failures {
			http.Error(w, "indisponible", code)
			return
		}
		fmt.Fprintf(w, `{"city":%q,"temp_c":21.5}`, r.URL.Query().Get("city"))
	}))
	return srv, &calls
}

func newTestClient(base string) (*Client, *[]time.Duration) {
	c := New(base)
	var waits []time.Duration
	c.sleep = func(ctx context.Context, d time.Duration) error { waits = append(waits, d); return ctx.Err() }
	return c, &waits
}

func TestForecastRetries(t *testing.T) {
	tests := []struct {
		name      string
		failures  int32
		code      int
		wantCalls int32
		wantErr   bool
	}{
		{"succès direct", 0, 0, 1, false},
		{"deux 503 puis succès", 2, 503, 3, false},
		{"toujours 503", 10, 503, 4, true},
		{"404 : pas de réessai", 10, 404, 1, true},
		{"429 : réessai", 1, 429, 2, false},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			srv, calls := flaky(tt.failures, tt.code)
			defer srv.Close()
			c, waits := newTestClient(srv.URL)
			f, err := c.Forecast(t.Context(), "Saint-Étienne")
			if (err != nil) != tt.wantErr || calls.Load() != tt.wantCalls {
				t.Fatalf("err = %v, appels = %d ; attendu erreur=%v, %d appels", err, calls.Load(), tt.wantErr, tt.wantCalls)
			}
			if !tt.wantErr && f.City != "Saint-Étienne" {
				t.Errorf("ville = %q (encodage de l'URL ?)", f.City)
			}
			for i, d := range *waits { // chaque attente est dans [base/2, base)
				base := 100 * time.Millisecond << i
				if d < base/2 || d >= base {
					t.Errorf("attente %d = %v hors de [%v, %v)", i, d, base/2, base)
				}
			}
			var se *StatusError
			if tt.wantErr && !errors.As(err, &se) {
				t.Errorf("erreur %v : StatusError attendue", err)
			}
		})
	}
}

func TestForecastCanceled(t *testing.T) {
	srv, calls := flaky(10, 503)
	defer srv.Close()
	ctx, cancel := context.WithCancel(t.Context())
	c, _ := newTestClient(srv.URL)
	c.sleep = func(context.Context, time.Duration) error { cancel(); return context.Canceled }
	if _, err := c.Forecast(ctx, "Lyon"); !errors.Is(err, context.Canceled) || calls.Load() != 1 {
		t.Errorf("err = %v, appels = %d", err, calls.Load())
	}
}
```

`go test -race ./weather` passe : six scénarios, sans aucune vraie attente — la fonction d'attente est
injectée (N5 L3) et le test vérifie que chaque délai tombe dans l'intervalle attendu.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `&http.Client{Timeout: 10 * time.Second}` | Un délai total par tentative ; jamais `DefaultClient`. |
| Un seul `http.Client` dans `Client` | Créé une fois, réutilisé : son pool de connexions sert à tous les appels. |
| `url.QueryEscape(city)` | « Saint-Étienne » arrive intact côté serveur — le test le vérifie. |
| `NewRequestWithContext` | L'annulation de l'appelant interrompt l'appel en cours. |
| `defer resp.Body.Close()` juste après l'erreur | Sur tous les chemins, y compris les statuts d'erreur. |
| `StatusError` | Le statut devient une erreur typée ; l'appelant peut tester `se.Code`. |
| `io.LimitReader` sur le corps | Jamais plus de 1 Mio lu depuis un service distant. |
| `retryable` | Réseau, 429 et 5xx oui ; 4xx et annulation non. |
| Attente `[d/2, d)` doublée à chaque fois | *Backoff* exponentiel avec *jitter* ; vérifié par le test. |
| `sleep` injecté | Le test des réessais dure 10 ms au lieu de plusieurs secondes. |
| `TestForecastCanceled` | Une annulation pendant l'attente arrête tout : un seul appel, `context.Canceled`. |

## Erreurs fréquentes

1. **`http.Get` / `http.DefaultClient`** sans délai.
2. **Un `http.Client` créé par requête** : plus de pool de connexions.
3. **Corps non fermé** : connexions perdues, puis épuisement des sockets.
4. **`err == nil` ⇒ succès** : un 500 passe pour une réponse valide.
5. **Lire un corps distant sans limite.**
6. **Réessayer un 4xx**, ou un `POST` sans clé d'idempotence.
7. **Réessayer sans attente, ou sans jitter**, ou sans plafond.
8. **URL construite par concaténation** sans encodage.

## Bonnes pratiques Go

- Un `http.Client` par dépendance distante, avec `Timeout`, créé une fois.
- `NewRequestWithContext` toujours ; contexte propagé depuis l'appelant.
- `defer resp.Body.Close()` immédiatement après la vérification de l'erreur.
- Statuts non 2xx → erreur typée.
- Lectures bornées ; décodage strict.
- Réessais : idempotents seulement, transitoires seulement, *backoff* + *jitter* + plafond + contexte.
- Tests avec `httptest.NewServer` simulant les pannes.

## Ce que je dois retenir

- `http.DefaultClient` n'a aucun délai : ne pas l'utiliser.
- Réutiliser le client (pool de connexions du `Transport`).
- `client.Do` ne rend une erreur que pour le transport ; **tester le statut**.
- **Toujours `Close`** le corps ; depuis Go 1.27, `Close` draine lui-même, la lecture complète manuelle n'est plus nécessaire en général.
- Réessayer ce qui est idempotent et transitoire, avec *backoff* exponentiel, *jitter* et plafond.

---

➡️ [Exercices](exercices.md)
