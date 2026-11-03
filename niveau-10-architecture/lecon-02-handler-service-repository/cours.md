# Leçon 2 — handler / service / repository

> Presque tous les backends Go professionnels ont la même forme : un **handler** qui parle HTTP, un
> **service** qui exécute les cas d'usage, un **repository** qui parle à la base. Le découpage est simple ;
> ce qui demande du discernement, c'est de savoir **où** mettre chaque décision : la validation,
> l'autorisation, la transaction, la traduction d'erreur, la conversion des types. Mal placées, elles se
> dupliquent ou se perdent. Cette leçon donne un emplacement à chacune.

## Objectifs

1. Connaître le rôle exact de chaque couche, et ce qu'elle ne doit pas faire.
2. Distinguer validation **syntaxique** (handler), **métier** (domaine) et **d'intégrité** (base).
3. Distinguer **authentification** (handler) et **autorisation** (service).
4. Placer la frontière de **transaction** dans le service, sans qu'il connaisse `database/sql`.
5. Séparer les types d'API (DTO), du domaine et du stockage.
6. Choisir entre organisation par couche et par fonctionnalité.

## Explication

### Qui fait quoi

| Décision | Handler | Service | Domaine | Repository | Base |
|---|---|---|---|---|---|
| Décoder le JSON, limiter la taille, refuser les champs inconnus | ✔ | | | | |
| Champs présents, bien typés (validation **syntaxique**) | ✔ | | | | |
| Établir **qui** appelle (authentification) | ✔ (ou middleware) | | | | |
| Décider si cet appelant **peut** faire cette action (autorisation) | | ✔ | | | |
| Ouvrir et fermer la **transaction** | | ✔ | | | |
| Les règles (« 3 emprunts au plus », « exemplaire libre ») | | | ✔ | | |
| Requêtes SQL, *mapping* des lignes, traduction des erreurs SQL | | | | ✔ | |
| Unicité, clés étrangères, `CHECK` (validation **d'intégrité**) | | | | | ✔ |
| Erreur du domaine → statut HTTP | ✔ | | | | |
| Domaine → DTO → JSON | ✔ | | | | |

Trois remarques :

- **La validation est à trois étages.** Le handler vérifie la **forme** (`copy_id` présent, du JSON
  valide) et répond 400 ; le domaine vérifie les **règles** et répond par une erreur métier (409, 422) ;
  la base garantit l'**intégrité** même si tout le reste se trompe (N8 L4, projet 4 S4). Ce n'est
  pas de la duplication : chaque étage vérifie une chose différente.
- **L'autorisation est une règle métier.** « Un adhérent n'emprunte que pour lui-même, un bibliothécaire
  pour n'importe qui » change avec le métier, pas avec le transport. Le handler établit **qui** appelle
  (`Actor`) ; le service **décide**. Une autorisation dans le handler disparaît le jour où une CLI ou un
  consommateur de messages appelle le même service.
- **La transaction suit le cas d'usage.** Un emprunt modifie l'adhérent **et** l'exemplaire : tout ou
  rien. Le service ouvre la transaction, parce que c'est lui qui sait ce qui doit être atomique — mais il
  le fait à travers une interface (`Store.WithinTx`, N8 L7), sans importer `database/sql`.

### Un cas d'usage, une méthode

Chaque méthode publique du service est un **cas d'usage** nommé par le métier (`Borrow`, `Return`,
`Renew`), pas une opération technique (`UpdateMember`). Elle prend les données d'entrée **et** l'acteur,
rend un résultat du domaine ou une erreur du domaine. Elle est l'endroit naturel pour :

- l'autorisation ;
- la transaction ;
- les appels à d'autres services, les événements à publier (leçon 8), l'idempotence (leçon 9) ;
- la journalisation et les métriques du cas d'usage (leçon 10).

### Trois familles de types

| Types | Où | Exemple |
|---|---|---|
| **DTO** (*data transfer objects*) de l'API | handler | `borrowRequest`, `borrowResponse` avec balises `json` |
| **Domaine** | domaine, service | `Member`, `Copy`, sans balise |
| **Lignes** de stockage | repository | une struct avec balises `db`, ou des `Scan` directs (N8 L2) |

Utiliser le même type partout est tentant et ça marche… jusqu'au jour où l'on ajoute un champ interne
(un hachage de mot de passe, un compteur technique) qui part dans le JSON, ou un renommage d'API qui
casse la base. Les conversions entre familles sont du code ennuyeux, mais c'est lui qui découple
l'API, le domaine et le schéma.

### Organiser les paquets

Deux organisations :

- **par couche** : `handlers/`, `services/`, `repositories/`. Pour lire ou modifier « l'emprunt », il faut
  ouvrir trois dossiers ; tout doit être exporté pour traverser les paquets ;
- **par fonctionnalité** (ou par domaine) : `loan/` contient son handler, son service, son stockage ;
  `member/`, `catalog/` de même. Ce qui change ensemble est rangé ensemble ; les détails restent non
  exportés.

La communauté Go privilégie nettement la seconde. Dans une fonctionnalité qui grossit, on sépare ensuite
en sous-paquets (`loan/`, `loan/pgstore`, `loan/httpapi`) comme à la leçon 1. L'exemple de cette leçon
tient en un seul paquet `loan`, avec un fichier par couche.

### Quand ne pas le faire

- **Un CRUD sans règles** : un service qui ne fait que transmettre au repository ajoute une couche vide.
  Le handler peut appeler le repository directement ; on intercalera un service le jour où une règle
  apparaît.
- **Des DTO identiques au domaine**, champ pour champ, pour une API interne qui ne changera pas
  indépendamment : une seule struct suffit, tant que rien d'interne n'y est ajouté.
- **Une méthode de service par requête SQL** (`GetMemberByID`, `GetMemberByEmail`…) : si le service
  n'ajoute rien, il n'est qu'un miroir du repository.

## Exemple

Un emprunt qui modifie deux objets dans une transaction, avec authentification, validation et
autorisation chacune à sa place.

```go
// Fichier : loan/domain.go
// Domaine : types et règles. Aucune notion de requête, d'utilisateur connecté ni de transaction.
package loan

import (
	"errors"
	"time"
)

var (
	ErrTooManyLoans = errors.New("nombre maximal d'emprunts atteint")
	ErrUnavailable  = errors.New("exemplaire déjà emprunté")
	ErrNotFound     = errors.New("introuvable")
	ErrForbidden    = errors.New("action non autorisée")
)

type Member struct {
	ID    string
	Loans int
}

type Copy struct {
	ID         string
	BorrowedBy string // "" si disponible
	Due        time.Time
}

const MaxLoans = 3

// Lend : règle portée par les deux objets ; le service garantit qu'ils sont modifiés ensemble.
func Lend(m *Member, c *Copy, now time.Time) error {
	if c.BorrowedBy != "" {
		return ErrUnavailable
	}
	if m.Loans >= MaxLoans {
		return ErrTooManyLoans
	}
	m.Loans++
	c.BorrowedBy, c.Due = m.ID, now.Add(21*24*time.Hour)
	return nil
}
```

```go
// Fichier : loan/service.go
package loan

import (
	"context"
	"fmt"
	"time"
)

// Repos : ce que le service attend du stockage, DANS une transaction.
type Repos interface {
	Member(ctx context.Context, id string) (Member, error)
	Copy(ctx context.Context, id string) (Copy, error)
	SaveMember(ctx context.Context, m Member) error
	SaveCopy(ctx context.Context, c Copy) error
}

// Store ouvre une transaction et donne les repositories qui y travaillent (N8 L7).
type Store interface {
	WithinTx(ctx context.Context, fn func(Repos) error) error
}

// Actor : QUI agit. Le handler l'établit (authentification) ; le service décide (autorisation).
type Actor struct {
	ID        string
	Librarian bool
}

type Service struct {
	store Store
	now   func() time.Time
}

func NewService(s Store, now func() time.Time) *Service { return &Service{store: s, now: now} }

// Borrow : un cas d'usage = une méthode = une transaction.
func (s *Service) Borrow(ctx context.Context, who Actor, memberID, copyID string) (Copy, error) {
	// Autorisation : une règle métier (« un adhérent n'emprunte que pour lui-même »).
	if who.ID != memberID && !who.Librarian {
		return Copy{}, fmt.Errorf("%s pour %s : %w", who.ID, memberID, ErrForbidden)
	}
	var lent Copy
	err := s.store.WithinTx(ctx, func(r Repos) error {
		m, err := r.Member(ctx, memberID)
		if err != nil {
			return err
		}
		c, err := r.Copy(ctx, copyID)
		if err != nil {
			return err
		}
		if err := Lend(&m, &c, s.now()); err != nil {
			return err
		}
		if err := r.SaveMember(ctx, m); err != nil {
			return err
		}
		lent = c
		return r.SaveCopy(ctx, c)
	})
	return lent, err
}
```

```go
// Fichier : loan/memstore.go
package loan

import (
	"context"
	"fmt"
	"maps"
	"sync"
)

// MemStore : stockage en mémoire, transactions par copie (un verrou, des brouillons).
type MemStore struct {
	mu      sync.Mutex
	members map[string]Member
	copies  map[string]Copy
	FailOn  string // pour les tests : faire échouer SaveCopy de cet exemplaire
}

func NewMemStore(members []Member, copies []Copy) *MemStore {
	s := &MemStore{members: map[string]Member{}, copies: map[string]Copy{}}
	for _, m := range members {
		s.members[m.ID] = m
	}
	for _, c := range copies {
		s.copies[c.ID] = c
	}
	return s
}

type memTx struct {
	members map[string]Member
	copies  map[string]Copy
	failOn  string
}

func (t *memTx) Member(_ context.Context, id string) (Member, error) {
	m, ok := t.members[id]
	if !ok {
		return Member{}, fmt.Errorf("adhérent %s : %w", id, ErrNotFound)
	}
	return m, nil
}

func (t *memTx) Copy(_ context.Context, id string) (Copy, error) {
	c, ok := t.copies[id]
	if !ok {
		return Copy{}, fmt.Errorf("exemplaire %s : %w", id, ErrNotFound)
	}
	return c, nil
}

func (t *memTx) SaveMember(_ context.Context, m Member) error { t.members[m.ID] = m; return nil }

func (t *memTx) SaveCopy(_ context.Context, c Copy) error {
	if c.ID == t.failOn {
		return fmt.Errorf("panne simulée en écrivant %s", c.ID)
	}
	t.copies[c.ID] = c
	return nil
}

func (s *MemStore) WithinTx(_ context.Context, fn func(Repos) error) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	tx := &memTx{maps.Clone(s.members), maps.Clone(s.copies), s.FailOn}
	if err := fn(tx); err != nil {
		return err // brouillons abandonnés : rollback
	}
	s.members, s.copies = tx.members, tx.copies // commit
	return nil
}

func (s *MemStore) Snapshot() (map[string]Member, map[string]Copy) {
	s.mu.Lock()
	defer s.mu.Unlock()
	return maps.Clone(s.members), maps.Clone(s.copies)
}
```

```go
// Fichier : loan/handler.go
package loan

import (
	"encoding/json"
	"errors"
	"net/http"
	"time"
)

// Les DTO : la forme de l'API, distincte des types du domaine (N7 L4, projet 3 D8).
type borrowRequest struct {
	CopyID string `json:"copy_id"`
}

type borrowResponse struct {
	CopyID string    `json:"copy_id"`
	Due    time.Time `json:"due"`
}

func Routes(svc *Service) http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("POST /members/{id}/loans", func(w http.ResponseWriter, r *http.Request) {
		// 1. Authentification (simulée ici) : établir QUI. Pas de décision métier.
		who := Actor{ID: r.Header.Get("X-User"), Librarian: r.Header.Get("X-Role") == "librarian"}
		if who.ID == "" {
			http.Error(w, "non authentifié", http.StatusUnauthorized)
			return
		}
		// 2. Validation SYNTAXIQUE : le corps est-il bien formé ?
		var in borrowRequest
		dec := json.NewDecoder(http.MaxBytesReader(w, r.Body, 1<<10))
		dec.DisallowUnknownFields()
		if err := dec.Decode(&in); err != nil || in.CopyID == "" {
			http.Error(w, "corps invalide : {\"copy_id\": \"…\"} attendu", http.StatusBadRequest)
			return
		}
		// 3. Le cas d'usage.
		c, err := svc.Borrow(r.Context(), who, r.PathValue("id"), in.CopyID)
		if err != nil {
			http.Error(w, err.Error(), status(err))
			return
		}
		// 4. Domaine → DTO → JSON.
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusCreated)
		json.NewEncoder(w).Encode(borrowResponse{CopyID: c.ID, Due: c.Due})
	})
	return mux
}

func status(err error) int {
	switch {
	case errors.Is(err, ErrForbidden):
		return http.StatusForbidden
	case errors.Is(err, ErrNotFound):
		return http.StatusNotFound
	case errors.Is(err, ErrTooManyLoans), errors.Is(err, ErrUnavailable):
		return http.StatusConflict
	default:
		return http.StatusInternalServerError
	}
}
```

```go
// Fichier : loan/loan_test.go
package loan

import (
	"errors"
	"net/http/httptest"
	"strings"
	"sync"
	"testing"
	"time"
)

var now = time.Date(2026, 3, 1, 10, 0, 0, 0, time.UTC)

func newSvc() (*Service, *MemStore) {
	st := NewMemStore([]Member{{ID: "ada"}, {ID: "bob"}}, []Copy{{ID: "c1"}, {ID: "c2"}})
	return NewService(st, func() time.Time { return now }), st
}

func TestServiceAuthorization(t *testing.T) {
	svc, _ := newSvc()
	if _, err := svc.Borrow(t.Context(), Actor{ID: "bob"}, "ada", "c1"); !errors.Is(err, ErrForbidden) {
		t.Fatalf("bob pour ada : %v", err)
	}
	if _, err := svc.Borrow(t.Context(), Actor{ID: "lib", Librarian: true}, "ada", "c1"); err != nil {
		t.Fatalf("bibliothécaire : %v", err)
	}
}

func TestServiceAtomicity(t *testing.T) {
	svc, st := newSvc()
	st.FailOn = "c1" // SaveCopy échoue APRÈS SaveMember
	if _, err := svc.Borrow(t.Context(), Actor{ID: "ada"}, "ada", "c1"); err == nil {
		t.Fatal("erreur attendue")
	}
	members, _ := st.Snapshot()
	if members["ada"].Loans != 0 {
		t.Fatalf("compteur d'emprunts incrémenté malgré l'échec : %d", members["ada"].Loans)
	}
}

func TestServiceConcurrentSameCopy(t *testing.T) {
	svc, _ := newSvc()
	var wg sync.WaitGroup
	var mu sync.Mutex
	ok := 0
	for _, who := range []string{"ada", "bob"} {
		wg.Go(func() {
			if _, err := svc.Borrow(t.Context(), Actor{ID: who}, who, "c2"); err == nil {
				mu.Lock()
				ok++
				mu.Unlock()
			}
		})
	}
	wg.Wait()
	if ok != 1 {
		t.Fatalf("%d emprunts réussis du même exemplaire", ok)
	}
}

func TestHandler(t *testing.T) {
	svc, _ := newSvc()
	h := Routes(svc)
	cases := []struct {
		user, role, member, body string
		want                     int
	}{
		{"", "", "ada", `{"copy_id":"c1"}`, 401},
		{"ada", "", "ada", `{"copy":"c1"}`, 400},
		{"bob", "", "ada", `{"copy_id":"c1"}`, 403},
		{"ada", "", "ada", `{"copy_id":"zz"}`, 404},
		{"ada", "", "ada", `{"copy_id":"c1"}`, 201},
		{"bob", "", "bob", `{"copy_id":"c1"}`, 409},
	}
	for _, c := range cases {
		req := httptest.NewRequest("POST", "/members/"+c.member+"/loans", strings.NewReader(c.body))
		req.Header.Set("X-User", c.user)
		req.Header.Set("X-Role", c.role)
		rec := httptest.NewRecorder()
		h.ServeHTTP(rec, req)
		if rec.Code != c.want {
			t.Errorf("%+v → %d %s", c, rec.Code, rec.Body)
		}
	}
}
```
`go test -race ./...` passe.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `Lend(m, c, now)` | La règle touche deux objets ; le domaine ne sait pas comment ils sont stockés ni protégés. |
| `Repos` et `Store.WithinTx` | Le service décide de l'atomicité sans connaître la technologie (N8 L7). |
| `Actor` | Le handler dit **qui** ; le service décide si c'est **permis** (`ErrForbidden` → 403). |
| Autorisation **avant** la transaction | On n'ouvre pas de transaction pour une demande qu'on refusera de toute façon. |
| `MemStore` par brouillons | Le même contrat transactionnel qu'une base, pour des tests rapides. |
| `TestServiceAtomicity` | `SaveCopy` échoue après `SaveMember` : le compteur d'emprunts n'est pas modifié. |
| `TestServiceConcurrentSameCopy` | Deux emprunts simultanés du même exemplaire : un seul réussit. |
| `borrowRequest` / `borrowResponse` | La forme de l'API, découplée des types du domaine. |
| `DisallowUnknownFields` | `{"copy":"c1"}` (faute de frappe) → 400, au lieu d'un `copy_id` vide traité plus loin. |
| `status(err)` | Une seule traduction erreur → statut (N7 L5). |
| Un seul paquet `loan` | Organisation par fonctionnalité : handler, service, domaine et stockage côte à côte. |

## Erreurs fréquentes

1. **L'autorisation dans le handler** : contournée par toute autre porte d'entrée.
2. **La transaction dans le handler** (qui importe alors `database/sql`) ou **dans le repository** (qui ne
   peut pas composer deux opérations).
3. **Valider une seule fois** : soit seulement dans le handler (les règles fuient), soit seulement en base
   (messages d'erreur incompréhensibles).
4. **Le même type** pour l'API, le domaine et la base.
5. **Des services miroirs** du repository, sans rien ajouter.
6. **Organiser par couche technique** dès le premier jour.
7. **Un service qui reçoit `*http.Request`** ou rend un statut HTTP.

## Bonnes pratiques Go

- Handler : décoder, valider la forme, établir l'acteur, appeler, traduire, encoder.
- Service : un cas d'usage par méthode, autorisation, transaction, orchestration.
- Domaine : les règles, sans technologie. Repository : le SQL, rien d'autre.
- Contraintes en base comme dernier filet.
- DTO pour l'API dès qu'un champ diffère ou risque de différer.
- Paquets par fonctionnalité.

## Ce que je dois retenir

- Validation syntaxique au handler, règles au domaine, intégrité en base.
- Authentification au handler, autorisation au service.
- La transaction appartient au cas d'usage, donc au service, à travers une interface.
- DTO, domaine et lignes de stockage sont trois familles de types.
- Organiser par fonctionnalité ; supprimer les couches qui n'ajoutent rien.

---

➡️ [Exercices](exercices.md)
