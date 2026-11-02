# Leçon 4 — Conception d'API REST

> Une API publique est un **contrat** : une fois des clients branchés dessus, chaque nom de champ,
> chaque statut, chaque URL devient très coûteux à changer. Cette leçon ne parle presque pas de Go —
> elle parle des décisions qui rendent une API prévisible, évolutive et agréable à consommer. Le code
> Go, lui, reste ce qu'on a vu aux leçons 1 à 3.

## Objectifs

1. Modéliser des **ressources** et leur associer les bons verbes et statuts.
2. Comprendre l'**idempotence** et ses conséquences.
3. Paginer correctement (par curseur plutôt que par décalage).
4. Faire évoluer une API sans casser ses clients (versionnement).

## Explication

### Des ressources, pas des actions

Une API REST expose des **ressources** (des noms) et utilise les **méthodes HTTP** comme verbes :

| Requête | Sens | Succès |
|---|---|---|
| `GET /v1/tasks` | lister | 200 |
| `POST /v1/tasks` | créer | 201 + `Location` + la ressource créée |
| `GET /v1/tasks/42` | lire | 200 (404 si absente) |
| `PUT /v1/tasks/42` | **remplacer** entièrement | 200 ou 204 |
| `PATCH /v1/tasks/42` | modifier **partiellement** | 200 + la ressource modifiée |
| `DELETE /v1/tasks/42` | supprimer | 204 |

Les URL sont des **noms au pluriel**, en minuscules, avec des tirets (`/v1/user-groups`) ; jamais de
verbe (`/v1/getTasks`, `/v1/deleteTask?id=42`). Les sous-ressources s'imbriquent sur un niveau
(`/v1/projects/7/tasks`), rarement plus.

Une action qui n'entre pas dans ce moule (« archiver », « envoyer ») se modélise soit comme un
changement d'état (`PATCH /v1/tasks/42` avec `{"archived": true}`), soit comme une sous-ressource
créée (`POST /v1/invoices/9/sendings`). C'est un choix à faire **une fois** et à appliquer partout.

### Méthodes sûres et idempotentes

| Méthode | Sûre (ne modifie rien) | Idempotente (N appels = 1 appel) |
|---|---|---|
| GET, HEAD | oui | oui |
| PUT, DELETE | non | **oui** |
| POST | non | **non** |
| PATCH | non | pas forcément |

L'idempotence compte parce que **le réseau perd des réponses** : un client qui n'a pas reçu la réponse
à son `DELETE` peut le renvoyer sans risque ; un second `DELETE` de la même ressource doit donc réussir
aussi (204, pas 404). Un `POST` renvoyé crée **deux** ressources — d'où, pour les opérations sensibles
(paiement), une **clé d'idempotence** fournie par le client (`Idempotency-Key`), que le serveur mémorise
pour rejouer la même réponse (N5 L3, exercice du paiement ; niveau 10).

### PATCH : distinguer « absent » de « zéro »

`{"done": false}` et `{}` ne veulent pas dire la même chose : le premier change `done`, le second ne
le touche pas. Décodé dans un `bool`, les deux donnent `false`. En Go, on utilise des **pointeurs**
dans la struct de décodage : `nil` = champ absent, pointeur vers `false` = valeur fournie.

### Le format des réponses

- **JSON**, `Content-Type: application/json`, une convention de nommage unique (`snake_case` ou
  `camelCase`, mais une seule) ;
- les collections dans un objet (`{"items": [...]}`) plutôt qu'un tableau nu : on pourra ajouter
  `next_cursor`, `total`… sans casser les clients ;
- des tableaux **jamais `null`** (N4 L6) ;
- des dates en **RFC 3339 UTC** (N4 L4) ; des identifiants `int64` en chaîne si des clients JavaScript
  sont visés (N4 L3) ; des montants en **entiers de centimes** (N4 L3) ;
- des erreurs dans un format unique (leçon 5).

### Pagination : décalage contre curseur

**Décalage** (`?offset=40&limit=20`) : simple, mais lent sur de grandes tables (la base parcourt les 40
premières lignes pour les jeter, niveau 8), et **instable** — si une ligne est insérée ou supprimée
entre deux pages, le client voit un élément en double ou en saute un.

**Curseur** (`?cursor=…&limit=20`) : le serveur rend avec chaque page un jeton opaque qui désigne
« après le dernier élément vu » (ici l'identifiant, souvent encodé en base64 pour rester opaque). La
page suivante commence **après** ce point : stable sous insertion, et efficace en base (un `WHERE id >
?` indexé). C'est le choix par défaut pour une API publique.

Toujours **borner** `limit` (ici 1 à 100) : sans borne, un client demande un million d'éléments.

### Filtrer, trier, choisir les champs

Par paramètres de requête, sur une liste **blanche** : `?done=false`, `?sort=-created_at`. Ne jamais
passer un paramètre de tri ou de filtre tel quel à une requête SQL (injection, niveau 8).

### Versionner

Une API publique change. Les changements **compatibles** (ajouter un champ dans une réponse, un
paramètre facultatif, une route) ne cassent personne — à condition que les clients ignorent les champs
inconnus. Les changements **incompatibles** (renommer ou supprimer un champ, changer un type ou un
statut) exigent une **nouvelle version** :

- par le chemin : `/v1/tasks`, `/v2/tasks` — le plus simple et le plus visible ;
- par un en-tête (`Accept: application/vnd.exemple.v2+json`) — plus « pur », plus difficile à tester
  et à déboguer.

On garde l'ancienne version le temps que les clients migrent, en l'annonçant dépréciée (en-têtes
`Deprecation` et `Sunset`). Le niveau 13 (leçon 6) traite la compatibilité dans la durée.

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : tasks/api.go
// Package tasks expose une ressource REST /v1/tasks.
package tasks

import (
	"cmp"
	"encoding/json"
	"fmt"
	"net/http"
	"slices"
	"strconv"
	"sync"
)

type Task struct {
	ID    int    `json:"id"`
	Title string `json:"title"`
	Done  bool   `json:"done"`
}

// Page : une liste paginée par CURSEUR. next_cursor est vide sur la dernière page.
type Page struct {
	Items      []Task `json:"items"`
	NextCursor string `json:"next_cursor,omitempty"`
}

type API struct {
	mu    sync.Mutex
	tasks map[int]Task // protégé par mu
	next  int
	mux   *http.ServeMux
}

func New() *API {
	a := &API{tasks: map[int]Task{}, next: 1, mux: http.NewServeMux()}
	a.mux.HandleFunc("GET /v1/tasks", a.list)
	a.mux.HandleFunc("POST /v1/tasks", a.create)
	a.mux.HandleFunc("GET /v1/tasks/{id}", a.get)
	a.mux.HandleFunc("PATCH /v1/tasks/{id}", a.patch)
	a.mux.HandleFunc("DELETE /v1/tasks/{id}", a.delete)
	return a
}

func (a *API) ServeHTTP(w http.ResponseWriter, r *http.Request) { a.mux.ServeHTTP(w, r) }

func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(v)
}

func decode(w http.ResponseWriter, r *http.Request, v any) bool {
	dec := json.NewDecoder(http.MaxBytesReader(w, r.Body, 1<<16))
	dec.DisallowUnknownFields()
	if err := dec.Decode(v); err != nil {
		http.Error(w, "JSON invalide : "+err.Error(), http.StatusBadRequest)
		return false
	}
	return true
}

func pathID(w http.ResponseWriter, r *http.Request) (int, bool) {
	id, err := strconv.Atoi(r.PathValue("id"))
	if err != nil || id < 1 {
		http.Error(w, "identifiant invalide", http.StatusBadRequest)
		return 0, false
	}
	return id, true
}

// GET /v1/tasks?limit=2&cursor=3&done=false
func (a *API) list(w http.ResponseWriter, r *http.Request) {
	q := r.URL.Query()
	limit := 20
	if s := q.Get("limit"); s != "" {
		n, err := strconv.Atoi(s)
		if err != nil || n < 1 || n > 100 {
			http.Error(w, "limit : entier de 1 à 100", http.StatusBadRequest)
			return
		}
		limit = n
	}
	after := 0 // le curseur est l'id de la dernière tâche de la page précédente
	if s := q.Get("cursor"); s != "" {
		n, err := strconv.Atoi(s)
		if err != nil || n < 0 {
			http.Error(w, "cursor invalide", http.StatusBadRequest)
			return
		}
		after = n
	}
	filter := q.Get("done") // "", "true" ou "false"

	a.mu.Lock()
	var all []Task
	for _, t := range a.tasks {
		if t.ID > after && (filter == "" || strconv.FormatBool(t.Done) == filter) {
			all = append(all, t)
		}
	}
	a.mu.Unlock()
	slices.SortFunc(all, func(x, y Task) int { return cmp.Compare(x.ID, y.ID) })

	page := Page{Items: []Task{}} // jamais null (N4 L6)
	if len(all) > limit {
		page.Items = all[:limit]
		page.NextCursor = strconv.Itoa(all[limit-1].ID)
	} else if all != nil {
		page.Items = all
	}
	writeJSON(w, http.StatusOK, page)
}

func (a *API) create(w http.ResponseWriter, r *http.Request) {
	var in struct {
		Title string `json:"title"`
	}
	if !decode(w, r, &in) {
		return
	}
	if in.Title == "" {
		http.Error(w, "title requis", http.StatusUnprocessableEntity)
		return
	}
	a.mu.Lock()
	t := Task{ID: a.next, Title: in.Title}
	a.tasks[t.ID] = t
	a.next++
	a.mu.Unlock()
	w.Header().Set("Location", fmt.Sprintf("/v1/tasks/%d", t.ID))
	writeJSON(w, http.StatusCreated, t)
}

func (a *API) get(w http.ResponseWriter, r *http.Request) {
	id, ok := pathID(w, r)
	if !ok {
		return
	}
	a.mu.Lock()
	t, found := a.tasks[id]
	a.mu.Unlock()
	if !found {
		http.Error(w, "tâche introuvable", http.StatusNotFound)
		return
	}
	writeJSON(w, http.StatusOK, t)
}

// PATCH : mise à jour PARTIELLE. Des pointeurs distinguent « absent » de « valeur zéro ».
func (a *API) patch(w http.ResponseWriter, r *http.Request) {
	id, ok := pathID(w, r)
	if !ok {
		return
	}
	var in struct {
		Title *string `json:"title"`
		Done  *bool   `json:"done"`
	}
	if !decode(w, r, &in) {
		return
	}
	if in.Title != nil && *in.Title == "" {
		http.Error(w, "title ne peut pas être vide", http.StatusUnprocessableEntity)
		return
	}
	a.mu.Lock()
	defer a.mu.Unlock()
	t, found := a.tasks[id]
	if !found {
		http.Error(w, "tâche introuvable", http.StatusNotFound)
		return
	}
	if in.Title != nil {
		t.Title = *in.Title
	}
	if in.Done != nil {
		t.Done = *in.Done
	}
	a.tasks[id] = t
	writeJSON(w, http.StatusOK, t)
}

// DELETE est idempotent : supprimer une tâche déjà supprimée rend aussi 204.
func (a *API) delete(w http.ResponseWriter, r *http.Request) {
	id, ok := pathID(w, r)
	if !ok {
		return
	}
	a.mu.Lock()
	delete(a.tasks, id)
	a.mu.Unlock()
	w.WriteHeader(http.StatusNoContent)
}
```

```go
// Fichier : tasks/api_test.go
package tasks

import (
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func call(t *testing.T, h http.Handler, method, target, body string) (int, string) {
	t.Helper()
	rec := httptest.NewRecorder()
	h.ServeHTTP(rec, httptest.NewRequest(method, target, strings.NewReader(body)))
	return rec.Code, rec.Body.String()
}

func TestTasksAPI(t *testing.T) {
	a := New()
	for _, title := range []string{"lire", "écrire", "relire"} {
		if code, _ := call(t, a, "POST", "/v1/tasks", `{"title":"`+title+`"}`); code != 201 {
			t.Fatalf("POST %s : %d", title, code)
		}
	}
	steps := []struct {
		method, target, body string
		wantCode             int
		wantBody             string
	}{
		{"GET", "/v1/tasks?limit=2", "", 200,
			`{"items":[{"id":1,"title":"lire","done":false},{"id":2,"title":"écrire","done":false}],"next_cursor":"2"}`},
		{"GET", "/v1/tasks?limit=2&cursor=2", "", 200, `{"items":[{"id":3,"title":"relire","done":false}]}`},
		{"PATCH", "/v1/tasks/2", `{"done":true}`, 200, `{"id":2,"title":"écrire","done":true}`},
		{"GET", "/v1/tasks?done=true", "", 200, `{"items":[{"id":2,"title":"écrire","done":true}]}`},
		{"PATCH", "/v1/tasks/2", `{"title":""}`, 422, "title ne peut pas être vide"},
		{"GET", "/v1/tasks?limit=0", "", 400, "limit : entier de 1 à 100"},
		{"DELETE", "/v1/tasks/1", "", 204, ""},
		{"DELETE", "/v1/tasks/1", "", 204, ""},
		{"GET", "/v1/tasks/1", "", 404, "tâche introuvable"},
		{"GET", "/v1/tasks?cursor=99", "", 200, `{"items":[]}`},
	}
	for _, s := range steps {
		code, body := call(t, a, s.method, s.target, s.body)
		if code != s.wantCode || strings.TrimSpace(body) != s.wantBody {
			t.Errorf("%s %s = %d %s\nattendu %d %s", s.method, s.target, code, body, s.wantCode, s.wantBody)
		}
	}
}
```

`go test -race ./tasks` passe : le test est une **conversation** avec l'API, étape par étape, qui sert
aussi de documentation de son comportement.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `/v1/` dans toutes les routes | Version dès le premier jour : ajouter `/v2/` plus tard ne casse rien. |
| `POST` → 201 + `Location` + la tâche | Le client connaît immédiatement l'identifiant et l'URL de ce qu'il a créé. |
| `Page{Items: []Task{}}` | La liste vide s'encode `[]`, jamais `null`. |
| Curseur = dernier `id` de la page | `next_cursor` absent (`omitempty`) sur la dernière page : le client sait qu'il a fini. |
| Tri par `id` avant de couper | Sans ordre stable, la pagination n'a pas de sens (les maps sont non ordonnées, N1 L7). |
| `limit` borné à 100 | Un client ne peut pas demander tout d'un coup. |
| `PATCH` avec `*string` et `*bool` | `{"done":true}` ne touche pas au titre ; `{"title":""}` est **refusé** (422), alors qu'avec un `string` on ne saurait pas s'il a été fourni. |
| `DELETE` deux fois → 204 deux fois | Idempotence : le client peut réessayer sans crainte. |
| `decode` avec `MaxBytesReader` et `DisallowUnknownFields` | Chaque route d'écriture lit son corps de la même façon (leçon 2). |

## Erreurs fréquentes

1. **Des verbes dans les URL** (`/getTasks`, `/tasks/delete`).
2. **`POST` pour tout**, y compris la lecture.
3. **`200` pour une création**, ou sans `Location`.
4. **`DELETE` qui rend 404** au second appel : les réessais échouent.
5. **`PATCH` avec des types valeur** : impossible de distinguer « absent » de « zéro ».
6. **Pagination par décalage** sur de grandes collections, ou **sans borne** sur `limit`.
7. **Tableau JSON nu** en réponse de liste : impossible d'ajouter des métadonnées.
8. **Renommer un champ** dans une version publiée.
9. **Pas de version** dans l'URL de départ.

## Bonnes pratiques Go

- Une route par couple méthode + ressource, avec les motifs 1.22+.
- Une struct d'entrée par opération (création, modification), distincte de la struct de sortie.
- Des fonctions communes (`writeJSON`, `decode`, `pathID`) pour un comportement uniforme.
- Un test « conversation » par ressource, qui documente le contrat.
- Écrire la spécification (OpenAPI) ou au minimum un tableau des routes, statuts et formats.

## Ce que je dois retenir

- Ressources au pluriel, méthodes HTTP comme verbes ; pas de verbe dans l'URL.
- GET/PUT/DELETE idempotents ; POST non (clé d'idempotence si nécessaire).
- 201 + `Location` à la création, 204 à la suppression, 404 / 409 / 422 selon le cas.
- PATCH : pointeurs pour distinguer absent et zéro.
- Pagination par **curseur**, `limit` borné, réponses enveloppées (`{"items": …}`).
- Versionner (`/v1/`) ; n'ajouter que des changements compatibles dans une version.

---

➡️ [Exercices](exercices.md)
