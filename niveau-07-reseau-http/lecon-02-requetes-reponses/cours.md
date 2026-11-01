# Leçon 2 — Requêtes et réponses

> Un handler fait toujours la même chose : **lire** une requête (méthode, chemin, paramètres,
> en-têtes, corps), **décider**, puis **écrire** une réponse (statut, en-têtes, corps). Chacune de
> ces étapes a ses pièges — un corps non borné, un en-tête écrit trop tard, un statut perdu, une
> réponse qui ne part qu'à la fin. Cette leçon les passe en revue, avec le JSON comme format d'échange.

## Objectifs

1. Lire paramètres de chemin, de requête, en-têtes et corps — en bornant le corps.
2. Écrire une réponse dans le bon ordre : en-têtes, statut, corps.
3. Choisir les bons statuts.
4. Streamer une réponse avec `http.ResponseController`.

## Explication

### Lire la requête

| Donnée | Accès |
|---|---|
| méthode | `r.Method` (souvent déjà filtrée par le motif, leçon 1) |
| paramètre de chemin | `r.PathValue("id")` |
| paramètres de requête (`?page=2&tri=nom`) | `r.URL.Query().Get("page")` — `""` si absent |
| plusieurs valeurs (`?tag=a&tag=b`) | `r.URL.Query()["tag"]` |
| en-tête | `r.Header.Get("Content-Type")` — insensible à la casse |
| corps | `r.Body`, un `io.ReadCloser` |
| formulaire | `r.ParseForm()` puis `r.FormValue("nom")` |
| contexte (annulation, valeurs) | `r.Context()` (leçon 7) |

Tout ce qui vient de la requête est **contrôlé par le client** : chaque valeur doit être validée
(type, plage, longueur) avant usage.

### Le corps : toujours borné

`r.Body` est un flux d'une taille inconnue — un client peut envoyer des gigaoctets. On le borne avec
`http.MaxBytesReader`, conçu pour le HTTP :

```go
r.Body = http.MaxBytesReader(w, r.Body, 1<<20)   // 1 Mio
```

Au-delà, la lecture échoue avec une `*http.MaxBytesError` (testable avec `errors.As`, N2 L6) et le
serveur ferme la connexion après la réponse. Mieux que `io.LimitReader` ici : `LimitReader` couperait
silencieusement, alors que `MaxBytesReader` rend une erreur et permet de répondre **413 Content Too Large**.

Pour du JSON, on combine avec ce qu'on a vu au niveau 4 (leçon 6) : `json.NewDecoder(r.Body)`,
`DisallowUnknownFields()`, puis une **validation métier** séparée du décodage.

Le serveur ferme lui-même `r.Body` à la fin du handler : pas besoin de `defer r.Body.Close()` côté
serveur (c'est différent côté **client**, leçon 6).

### Écrire la réponse : l'ordre compte

```go
w.Header().Set("Content-Type", "application/json")   // 1. les en-têtes
w.WriteHeader(http.StatusCreated)                      // 2. le statut — envoie les en-têtes
json.NewEncoder(w).Encode(v)                           // 3. le corps
```

Le **premier** `Write` ou `WriteHeader` envoie la ligne de statut et les en-têtes sur le réseau. Après :

- un `w.Header().Set(…)` n'a **plus aucun effet** ;
- un second `WriteHeader` est ignoré, avec un avertissement dans le journal du serveur :
  `http: superfluous response.WriteHeader call from …` ;
- un `Write` sans `WriteHeader` préalable envoie implicitement **200**.

Conséquence pratique : une fois le corps commencé, il est **trop tard** pour signaler une erreur par
le statut. D'où l'habitude de tout préparer (valider, calculer, encoder dans un buffer si nécessaire)
**avant** d'écrire.

Si `Content-Type` n'est pas fixé, le serveur le **devine** à partir des 512 premiers octets
(`http.DetectContentType`) : un JSON peut partir en `text/plain`. Toujours le fixer.

### Les statuts à connaître

| Code | Nom | Quand |
|---|---|---|
| 200 | OK | succès avec corps |
| 201 | Created | ressource créée (avec `Location`) |
| 204 | No Content | succès sans corps (suppression) |
| 301 / 302 / 307 / 308 | redirections | 307/308 conservent la méthode et le corps |
| 304 | Not Modified | cache valide (`ETag`, `If-None-Match`) |
| 400 | Bad Request | requête mal formée (JSON illisible, paramètre invalide) |
| 401 | Unauthorized | **non authentifié** (identité inconnue) |
| 403 | Forbidden | authentifié, mais **pas autorisé** |
| 404 | Not Found | ressource inexistante |
| 405 | Method Not Allowed | méthode non acceptée (automatique, leçon 1) |
| 409 | Conflict | conflit d'état (doublon, version obsolète) |
| 413 | Content Too Large | corps trop gros |
| 415 | Unsupported Media Type | `Content-Type` non accepté |
| 422 | Unprocessable Content | bien formé mais invalide selon les règles métier |
| 429 | Too Many Requests | limite de débit (N6 L11) |
| 500 | Internal Server Error | bug ou panne côté serveur |
| 503 | Service Unavailable | surcharge, maintenance (N6 L11) |
| 504 | Gateway Timeout | une dépendance n'a pas répondu à temps |

Les constantes `http.StatusXxx` évitent les nombres magiques. La leçon 5 construira la correspondance
systématique entre erreurs du domaine et statuts.

### Streamer : `http.ResponseController`

Par défaut, la réponse est **bufferisée** : le client ne reçoit rien tant que le buffer (4 Kio) n'est
pas plein ou que le handler n'a pas fini. Pour envoyer au fil de l'eau (progression, événements,
gros export) :

```go
rc := http.NewResponseController(w)    // Go 1.20
fmt.Fprintln(w, "étape 1")
rc.Flush()                             // envoyer maintenant
```

`ResponseController` remplace l'ancienne assertion `w.(http.Flusher)` (l'interface optionnelle du
défi de N2 L4) et fonctionne même quand `w` est enveloppé par un middleware. Il permet aussi de
modifier les délais de **cette** requête (`SetWriteDeadline`) — utile pour un long téléchargement.

Un handler qui streame doit écouter `r.Context().Done()` : il est annulé quand le client se
déconnecte (leçon 7).

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : api/api.go
// Package api montre la lecture d'une requête et l'écriture d'une réponse.
package api

import (
	"encoding/json"
	"errors"
	"fmt"
	"net/http"
	"strconv"
	"strings"
	"time"
)

type Order struct {
	Item     string `json:"item"`
	Quantity int    `json:"quantity"`
}

type Receipt struct {
	ID       int    `json:"id"`
	Item     string `json:"item"`
	Quantity int    `json:"quantity"`
	Currency string `json:"currency"`
}

const maxBody = 1 << 20 // 1 Mio

// writeJSON : statut, en-tête et corps, dans CET ordre.
func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(v) // l'erreur d'écriture vers le client n'a plus de recours ici
}

// CreateOrder lit un corps JSON borné et strict, un paramètre de requête et un en-tête.
func CreateOrder(w http.ResponseWriter, r *http.Request) {
	if ct := r.Header.Get("Content-Type"); !strings.HasPrefix(ct, "application/json") {
		http.Error(w, "Content-Type application/json attendu", http.StatusUnsupportedMediaType)
		return
	}
	r.Body = http.MaxBytesReader(w, r.Body, maxBody)
	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields()
	var o Order
	if err := dec.Decode(&o); err != nil {
		var tooBig *http.MaxBytesError
		if errors.As(err, &tooBig) {
			http.Error(w, fmt.Sprintf("corps limité à %d octets", tooBig.Limit), http.StatusRequestEntityTooLarge)
			return
		}
		http.Error(w, "JSON invalide : "+err.Error(), http.StatusBadRequest)
		return
	}
	if o.Item == "" || o.Quantity < 1 {
		http.Error(w, "item requis et quantity ≥ 1", http.StatusUnprocessableEntity)
		return
	}
	currency := r.URL.Query().Get("currency") // paramètre facultatif : ?currency=EUR
	if currency == "" {
		currency = "EUR"
	}
	writeJSON(w, http.StatusCreated, Receipt{ID: 1, Item: o.Item, Quantity: o.Quantity, Currency: currency})
}

// Countdown diffuse une ligne par seconde (streaming) ; s'arrête si le client part.
func Countdown(w http.ResponseWriter, r *http.Request) {
	n, err := strconv.Atoi(r.URL.Query().Get("from"))
	if err != nil || n < 0 || n > 10 {
		http.Error(w, "from doit être un entier entre 0 et 10", http.StatusBadRequest)
		return
	}
	w.Header().Set("Content-Type", "text/plain; charset=utf-8")
	rc := http.NewResponseController(w)
	for i := n; i >= 0; i-- {
		fmt.Fprintln(w, i)
		if err := rc.Flush(); err != nil { // envoyer MAINTENANT, sans attendre la fin du handler
			return
		}
		if i == 0 {
			return
		}
		select {
		case <-time.After(time.Second):
		case <-r.Context().Done(): // le client s'est déconnecté
			return
		}
	}
}
```

```go
// Fichier : api/api_test.go
package api

import (
	"net/http/httptest"
	"strings"
	"testing"
	"testing/synctest"
	"time"
)

func TestCreateOrder(t *testing.T) {
	big := `{"item":"` + strings.Repeat("x", 2<<20) + `","quantity":1}`
	tests := []struct {
		name, target, ctype, body string
		wantStatus                int
		wantBody                  string
	}{
		{"ok", "/orders", "application/json", `{"item":"clavier","quantity":2}`, 201,
			`{"id":1,"item":"clavier","quantity":2,"currency":"EUR"}` + "\n"},
		{"devise", "/orders?currency=XOF", "application/json", `{"item":"clavier","quantity":1}`, 201,
			`{"id":1,"item":"clavier","quantity":1,"currency":"XOF"}` + "\n"},
		{"mauvais type", "/orders", "text/plain", `{}`, 415, "Content-Type application/json attendu\n"},
		{"champ inconnu", "/orders", "application/json", `{"item":"a","quantity":1,"prix":3}`, 400,
			"JSON invalide : json: unknown field \"prix\"\n"},
		{"quantité nulle", "/orders", "application/json", `{"item":"a","quantity":0}`, 422,
			"item requis et quantity ≥ 1\n"},
		{"trop gros", "/orders", "application/json", big, 413, "corps limité à 1048576 octets\n"},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			req := httptest.NewRequest("POST", tt.target, strings.NewReader(tt.body))
			req.Header.Set("Content-Type", tt.ctype)
			rec := httptest.NewRecorder()
			CreateOrder(rec, req)
			if rec.Code != tt.wantStatus || rec.Body.String() != tt.wantBody {
				t.Errorf("= %d %q\nattendu %d %q", rec.Code, rec.Body.String(), tt.wantStatus, tt.wantBody)
			}
		})
	}
}

// Pas de vrai serveur ici : synctest ne sait pas virtualiser le réseau.
// On appelle le handler directement ; le Recorder implémente Flush.
func TestCountdown(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		rec := httptest.NewRecorder()
		start := time.Now()
		Countdown(rec, httptest.NewRequest("GET", "/countdown?from=3", nil))
		if got := rec.Body.String(); got != "3\n2\n1\n0\n" {
			t.Errorf("corps = %q", got)
		}
		if !rec.Flushed {
			t.Error("la réponse n'a jamais été vidée (Flush)")
		}
		if d := time.Since(start); d != 3*time.Second {
			t.Errorf("durée = %v, attendu 3s (virtuelles)", d)
		}
	})
}
```

`go test -race -v ./api` : les six cas de `TestCreateOrder` et `TestCountdown` passent ; ce dernier
vérifie, en **temps virtuel**, que le compte à rebours dure exactement 3 secondes et a bien été vidé.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| Vérification du `Content-Type` en premier | 415 avant même de lire le corps. |
| `http.MaxBytesReader` + `errors.As(err, &tooBig)` | Un corps de 2 Mio produit 413, avec la limite dans le message. |
| `DisallowUnknownFields` | `"prix"` est rejeté : une faute de frappe du client ne passe pas inaperçue (N4 L6). |
| 400 contre 422 | JSON illisible ou inattendu → 400 ; JSON correct mais quantité nulle → 422. |
| `r.URL.Query().Get("currency")` | Paramètre facultatif, avec une valeur par défaut. |
| `writeJSON` | L'ordre en-têtes → statut → corps, écrit **une fois** dans une fonction utilitaire. |
| `http.NewResponseController(w).Flush()` | Chaque ligne part immédiatement. |
| `select` sur `r.Context().Done()` | Le compte à rebours s'arrête si le client se déconnecte. |
| `TestCountdown` sans serveur | `synctest` ne virtualise pas le réseau (une bulle autour d'un `httptest.NewServer` reste bloquée) : on appelle le handler directement, avec un `Recorder` qui sait faire `Flush`. |

## Erreurs fréquentes

1. **Corps non borné** : `io.ReadAll(r.Body)` sans limite.
2. **En-tête fixé après le premier `Write`** : ignoré sans erreur.
3. **Deux `WriteHeader`** (une erreur détectée après un succès partiel) : `superfluous response.WriteHeader call`.
4. **`Content-Type` oublié**.
5. **Oublier `return`** après `http.Error` : le handler continue et écrit une seconde réponse.
6. **400 pour tout** : distinguer 400, 401, 403, 404, 409, 422.
7. **Faire confiance au client** : paramètres non validés.
8. **Un streaming sans `Flush`** : le client reçoit tout à la fin.

## Bonnes pratiques Go

- Valider tôt, répondre tôt, `return` après chaque réponse d'erreur.
- `MaxBytesReader` systématique sur les corps.
- Une fonction `writeJSON` (et une `writeError`, leçon 5) pour tout le service.
- Fixer `Content-Type` explicitement.
- `http.ResponseController` pour le streaming et les délais par requête.
- Tester chaque statut, y compris les erreurs, avec `httptest`.

## Ce que je dois retenir

- Lire : `PathValue`, `URL.Query()`, `Header.Get`, `Body` (borné par `MaxBytesReader`).
- Écrire : en-têtes, **puis** `WriteHeader`, **puis** corps ; après le premier octet, trop tard.
- Un `Write` sans statut = 200 ; un second `WriteHeader` est ignoré.
- 400 mal formé, 422 invalide, 401 non authentifié, 403 interdit, 409 conflit, 413 trop gros.
- `http.NewResponseController(w).Flush()` pour streamer.

---

➡️ [Exercices](exercices.md)
