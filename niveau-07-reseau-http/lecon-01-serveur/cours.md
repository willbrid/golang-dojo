# Leçon 1 — `net/http` : le serveur

> Beaucoup de langages ont besoin d'un framework pour servir du HTTP. En Go, la bibliothèque
> standard suffit pour des services de production : le serveur de `net/http` est concurrent, gère
> HTTP/1.1 et HTTP/2, et depuis Go 1.22 son routeur comprend les méthodes et les paramètres de chemin.
> Toute l'architecture repose sur **une** interface d'**une** méthode — c'est la plus belle
> illustration de la leçon sur les petites interfaces (N2 L4).

## Objectifs

1. Comprendre `http.Handler`, `http.HandlerFunc` et `http.ServeMux`.
2. Router avec les motifs Go 1.22+ : méthode, jokers `{id}`, `{chemin...}`, priorité.
3. Démarrer un serveur, et savoir que chaque requête est servie dans sa propre goroutine.
4. Tester des handlers avec `net/http/httptest`, sans ouvrir de port.

## Explication

### Une interface

```go
type Handler interface {
	ServeHTTP(w http.ResponseWriter, r *http.Request)
}
```

Tout ce qui sait répondre à une requête est un `Handler` : un routeur, un fichier statique, un
middleware (leçon 3), votre propre type. `r` décrit la requête (méthode, URL, en-têtes, corps) ; `w`
est l'endroit où l'on écrit la réponse — c'est un `io.Writer` (N4 L5), avec en plus les en-têtes et
le statut.

Écrire un type pour chaque route serait lourd. `http.HandlerFunc` adapte une **fonction** :

```go
type HandlerFunc func(http.ResponseWriter, *http.Request)

func (f HandlerFunc) ServeHTTP(w http.ResponseWriter, r *http.Request) { f(w, r) }
```

Un type **fonction** avec une méthode (N1 L11 + N2 L3) : une fonction de la bonne signature devient
un `Handler` par simple conversion. Ce petit adaptateur est au cœur de tout l'écosystème HTTP de Go.

### Le routeur : `http.ServeMux`

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /notes/{id}", getNote)
mux.HandleFunc("POST /notes", createNote)
mux.Handle("/static/", http.FileServer(http.Dir("public")))
```

Depuis **Go 1.22**, un motif peut contenir :

| Élément | Exemple | Effet |
|---|---|---|
| une **méthode** | `"POST /notes"` | ne correspond qu'à cette méthode (`GET` accepte aussi `HEAD`) |
| un **joker** | `"/notes/{id}"` | un segment, lu avec `r.PathValue("id")` |
| un joker **final** | `"/files/{path...}"` | tout le reste du chemin |
| `{$}` | `"/{$}"` | uniquement la racine exacte (sinon `/` correspond à **tout**) |
| un **hôte** | `"api.example.com/"` | ne correspond qu'à cet hôte |

Deux comportements automatiques :

- une route qui existe pour une **autre** méthode rend **405 Method Not Allowed**, avec un en-tête
  `Allow` qui liste les méthodes acceptées ;
- une route inconnue rend **404**.

**Priorité** : quand deux motifs correspondent, le plus **spécifique** gagne (`/notes/new` l'emporte
sur `/notes/{id}`), quel que soit l'ordre d'enregistrement. Deux motifs aussi spécifiques l'un que
l'autre et qui se chevauchent font **paniquer** l'enregistrement — au démarrage, ce qui est voulu.

Ce routeur couvre la grande majorité des besoins. Les routeurs tiers (`chi`, `gorilla/mux`) restent
utiles pour des cas plus riches, mais ne sont plus nécessaires par défaut.

### Démarrer

```go
log.Fatal(http.ListenAndServe(":8080", mux))
```

Pour la production, on construit un `http.Server` avec des **délais** (leçon 9) — la ligne ci-dessus
n'en a aucun. Et on l'arrête proprement avec `Shutdown` (leçon 7).

### Une goroutine par requête

Le serveur sert **chaque connexion dans sa propre goroutine** (niveau 6, leçon 1). Conséquences :

- les handlers s'exécutent **en parallèle** : tout état partagé (une map, un compteur) doit être
  protégé (N6 L6) — l'exemple utilise un `Mutex` ;
- un handler lent ne bloque pas les autres ;
- une panique dans un handler est récupérée par le serveur, qui ferme **cette** connexion et
  journalise la pile (N6 L12) — le programme continue.

### Tester sans réseau : `httptest`

```go
req := httptest.NewRequest("GET", "/notes/1", nil)   // une requête fabriquée
rec := httptest.NewRecorder()                         // un ResponseWriter qui enregistre
handler.ServeHTTP(rec, req)
resp := rec.Result()                                  // statut, en-têtes, corps
```

Un handler est une fonction : on l'appelle directement, sans ouvrir de port, en quelques
microsecondes. Pour un test de bout en bout à travers le vrai réseau local, `httptest.NewServer(h)`
démarre un serveur sur un port libre et fournit son URL (`srv.URL`).

### Le handler comme méthode d'un type

L'exemple attache l'état (les notes, le verrou) à un type `Server` et enregistre ses **méthodes**
comme handlers (`s.mux.HandleFunc("GET /notes", s.list)` — une valeur de méthode, N2 L3). C'est la
forme la plus courante : pas de variable globale, et chaque test crée son propre `Server` neuf.

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : notes/server.go
// Package notes est un petit service HTTP de notes en mémoire.
package notes

import (
	"fmt"
	"io"
	"net/http"
	"strconv"
	"strings"
	"sync"
)

// Server garde les notes et sait se router lui-même.
type Server struct {
	mu    sync.Mutex
	notes map[int]string // protégé par mu
	next  int            // protégé par mu
	mux   *http.ServeMux
}

func NewServer() *Server {
	s := &Server{notes: map[int]string{}, next: 1, mux: http.NewServeMux()}
	// Motifs Go 1.22+ : méthode, chemin, jokers nommés.
	s.mux.HandleFunc("GET /health", s.health)
	s.mux.HandleFunc("GET /notes", s.list)
	s.mux.HandleFunc("POST /notes", s.create)
	s.mux.HandleFunc("GET /notes/{id}", s.get)
	s.mux.HandleFunc("DELETE /notes/{id}", s.delete)
	return s
}

// ServeHTTP fait de *Server un http.Handler : il délègue au routeur.
func (s *Server) ServeHTTP(w http.ResponseWriter, r *http.Request) { s.mux.ServeHTTP(w, r) }

func (s *Server) health(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "ok")
}

func (s *Server) list(w http.ResponseWriter, r *http.Request) {
	s.mu.Lock()
	defer s.mu.Unlock()
	for id := 1; id < s.next; id++ {
		if n, ok := s.notes[id]; ok {
			fmt.Fprintf(w, "%d\t%s\n", id, n)
		}
	}
}

func (s *Server) create(w http.ResponseWriter, r *http.Request) {
	b, err := io.ReadAll(io.LimitReader(r.Body, 1<<16)) // corps borné (N4 L5) ; la leçon 2 détaille
	text := strings.TrimSpace(string(b))
	if err != nil || text == "" {
		http.Error(w, "corps vide ou illisible", http.StatusBadRequest)
		return
	}
	s.mu.Lock()
	id := s.next
	s.notes[id] = text
	s.next++
	s.mu.Unlock()
	w.Header().Set("Location", fmt.Sprintf("/notes/%d", id))
	w.WriteHeader(http.StatusCreated)
	fmt.Fprintln(w, id)
}

func (s *Server) get(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.Atoi(r.PathValue("id"))
	if err != nil {
		http.Error(w, "identifiant invalide", http.StatusBadRequest)
		return
	}
	s.mu.Lock()
	n, ok := s.notes[id]
	s.mu.Unlock()
	if !ok {
		http.NotFound(w, r)
		return
	}
	fmt.Fprintln(w, n)
}

func (s *Server) delete(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.Atoi(r.PathValue("id"))
	if err != nil {
		http.Error(w, "identifiant invalide", http.StatusBadRequest)
		return
	}
	s.mu.Lock()
	delete(s.notes, id)
	s.mu.Unlock()
	w.WriteHeader(http.StatusNoContent)
}
```

```go
// Fichier : notes/server_test.go
package notes

import (
	"io"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func do(t *testing.T, h http.Handler, method, target, body string) *http.Response {
	t.Helper()
	req := httptest.NewRequest(method, target, strings.NewReader(body))
	rec := httptest.NewRecorder()
	h.ServeHTTP(rec, req)
	return rec.Result()
}

func bodyOf(t *testing.T, resp *http.Response) string {
	t.Helper()
	b, err := io.ReadAll(resp.Body)
	if err != nil {
		t.Fatal(err)
	}
	return string(b)
}

func TestNotes(t *testing.T) {
	s := NewServer()

	resp := do(t, s, "POST", "/notes", "acheter-du-pain")
	if resp.StatusCode != http.StatusCreated || resp.Header.Get("Location") != "/notes/1" {
		t.Fatalf("POST : %d %q", resp.StatusCode, resp.Header.Get("Location"))
	}

	tests := []struct {
		name, method, target string
		wantStatus           int
		wantBody             string
	}{
		{"lire", "GET", "/notes/1", 200, "acheter-du-pain\n"},
		{"absente", "GET", "/notes/99", 404, "404 page not found\n"},
		{"id invalide", "GET", "/notes/abc", 400, "identifiant invalide\n"},
		{"liste", "GET", "/notes", 200, "1\tacheter-du-pain\n"},
		{"mauvaise méthode", "PUT", "/notes/1", 405, "Method Not Allowed\n"},
		{"route inconnue", "GET", "/inconnu", 404, "404 page not found\n"},
		{"santé", "GET", "/health", 200, "ok\n"},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			resp := do(t, s, tt.method, tt.target, "")
			if got := bodyOf(t, resp); resp.StatusCode != tt.wantStatus || got != tt.wantBody {
				t.Errorf("%s %s = %d %q, attendu %d %q", tt.method, tt.target, resp.StatusCode, got, tt.wantStatus, tt.wantBody)
			}
		})
	}

	resp = do(t, s, "PUT", "/notes/1", "")
	if allow := resp.Header.Get("Allow"); allow != "DELETE, GET, HEAD" {
		t.Errorf("Allow = %q", allow)
	}
	if resp := do(t, s, "DELETE", "/notes/1", ""); resp.StatusCode != http.StatusNoContent {
		t.Errorf("DELETE : %d", resp.StatusCode)
	}
}

// Avec un vrai serveur sur un port local : le même handler, à travers le réseau.
func TestRealServer(t *testing.T) {
	srv := httptest.NewServer(NewServer())
	defer srv.Close()
	resp, err := http.Get(srv.URL + "/health")
	if err != nil {
		t.Fatal(err)
	}
	defer resp.Body.Close()
	if got := bodyOf(t, resp); got != "ok\n" {
		t.Errorf("GET /health = %q", got)
	}
}
```

Sortie de `go test -race -v ./notes` (extrait) :

```
--- PASS: TestNotes (0.00s)
    --- PASS: TestNotes/lire (0.00s)
    --- PASS: TestNotes/absente (0.00s)
    --- PASS: TestNotes/id_invalide (0.00s)
    --- PASS: TestNotes/liste (0.00s)
    --- PASS: TestNotes/mauvaise_méthode (0.00s)
    --- PASS: TestNotes/route_inconnue (0.00s)
    --- PASS: TestNotes/santé (0.00s)
--- PASS: TestRealServer (0.00s)
```

Pour l'essayer à la main, un `main` de trois lignes suffit :
`http.ListenAndServe("127.0.0.1:8080", notes.NewServer())`, puis
`curl -i -X POST --data 'test' localhost:8080/notes`.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `*Server` a une méthode `ServeHTTP` | Il **est** un `http.Handler` : on peut le passer à `ListenAndServe`, à `httptest`, à un middleware. |
| `"GET /notes/{id}"` | Méthode et joker dans le motif ; `r.PathValue("id")` lit le segment. |
| `strconv.Atoi` sur l'identifiant | Le joker accepte n'importe quel texte : la validation reste à faire, d'où le cas « id invalide ». |
| `io.LimitReader(r.Body, 1<<16)` | Jamais de lecture non bornée d'un corps (N4 L5). |
| `w.Header().Set(…)` **avant** `w.WriteHeader(201)` | Les en-têtes doivent être fixés avant le statut (leçon 2). |
| `http.Error`, `http.NotFound` | Réponses d'erreur texte standard. La leçon 5 construira un modèle d'erreur JSON. |
| Le `Mutex` | Les handlers tournent en parallèle ; `-race` le vérifie dans les tests. |
| `PUT` → 405 et `Allow: DELETE, GET, HEAD` | Comportement automatique du routeur 1.22+ ; `GET` implique `HEAD`. |
| `httptest.NewRecorder` | Le test appelle le handler directement : aucun port, aucune goroutine réseau. |
| `httptest.NewServer` | Le même handler derrière un vrai socket local, pour un test de bout en bout. |

## Erreurs fréquentes

1. **État global** partagé par les handlers (`var notes = map…`) : non testable et souvent non protégé.
2. **Accès concurrent non protégé** à l'état dans les handlers.
3. **`"/"` qui capture tout** alors qu'on voulait la racine seule : `"/{$}"`.
4. **Oublier de valider** un paramètre de chemin.
5. **`http.ListenAndServe` sans délais** en production (leçon 9).
6. **Tester en démarrant le vrai `main`** au lieu d'appeler le handler.
7. **Écrire le statut après le corps** (leçon 2).
8. **Un routeur tiers par habitude** alors que `ServeMux` suffit.

## Bonnes pratiques Go

- Un type qui porte l'état et les dépendances ; ses méthodes sont les handlers ; un constructeur qui
  enregistre les routes.
- Motifs avec méthode (`"GET /…"`) et jokers ; `r.PathValue` + validation.
- Un test `httptest` par handler, en table.
- Aucun état global ; un `Server` neuf par test.
- `http.Server` avec délais et `Shutdown` en production (leçons 7 et 9).

## Ce que je dois retenir

- `http.Handler` : `ServeHTTP(w, r)` ; `http.HandlerFunc` adapte une fonction.
- `ServeMux` 1.22+ : `"MÉTHODE /chemin/{joker}"`, `{reste...}`, `{$}` ; 405 et `Allow` automatiques ;
  le plus spécifique gagne.
- Une goroutine par requête : protéger l'état partagé.
- `httptest.NewRecorder` pour tester un handler, `httptest.NewServer` pour un test de bout en bout.

---

➡️ [Exercices](exercices.md)
