# Leçon 1 — Séparation des responsabilités

> Un handler HTTP de quarante lignes qui lit le JSON, vérifie les règles métier, consulte l'horloge,
> modifie une map globale et écrit la réponse fonctionne parfaitement. Puis on veut tester la règle
> « pas plus de trois emprunts » : il faut un serveur HTTP. Puis l'appeler depuis une tâche de nuit : il
> faut fabriquer une fausse requête. Puis remplacer la map par PostgreSQL : il faut tout réécrire. Cette
> leçon sépare ce qui change pour des raisons différentes, et apprend à **lire** l'architecture d'un
> programme Go dans ses imports.

## Objectifs

1. Reconnaître les trois responsabilités d'un service : **transport** (HTTP, CLI, messages), **règles
   métier**, **stockage et services externes**.
2. Isoler les règles dans un paquet de domaine qui n'importe aucune technologie.
3. Diriger les dépendances **vers** le domaine.
4. Rendre les règles testables : horloge et stockage passés en paramètre ou injectés.
5. Savoir quand une séparation ne vaut pas son coût.

## Explication

### Ce qui change pour des raisons différentes

Dans le handler « tout-en-un » de l'exemple (paquet `avant`), trois choses cohabitent :

| Responsabilité | Change quand… |
|---|---|
| Lire le JSON, choisir un statut HTTP | l'API évolue (version, format), ou on ajoute une CLI, une file de messages |
| « 3 emprunts au plus, aucun en retard, prêt de 21 jours » | la bibliothèque change ses règles |
| La map protégée par un mutex | on passe à PostgreSQL, on ajoute un cache |

Mélangées, chaque modification touche les trois, et chaque test doit les mettre en place toutes les trois.
Le principe est ancien et simple : **regrouper ce qui change ensemble, séparer ce qui change pour des
raisons différentes**.

### Le domaine au centre

Les règles vont dans un paquet de **domaine** (`loan`) qui :

- ne connaît **ni** `net/http` **ni** `database/sql` : il exprime les règles avec des types Go ordinaires
  et des erreurs métier (`ErrTooManyLoans`) ;
- reçoit tout ce qui est « extérieur » : la date courante en **paramètre** (`Borrow(bookID, now)`), le
  stockage par une **interface** qu'il déclare lui-même (`Members`, côté consommateur, N2 L4) ;
- se teste en quelques microsecondes, sans serveur ni base, à n'importe quelle date simulée.

Autour, des paquets **d'adaptation** traduisent : `httpapi` (HTTP ↔ domaine, erreurs → statuts, N7 L5),
`memstore` (domaine ↔ stockage). Demain, `pgstore` (N8 L7) ou `cli` s'ajouteront sans toucher `loan`.

### Les dépendances pointent vers le domaine

Le sens des imports **est** l'architecture. `go list` l'affiche (mesuré sur l'exemple) :

```
exemple/avant    → encoding/json net/http sync time
exemple/httpapi  → encoding/json errors exemple/loan net/http
exemple/loan     → context errors fmt time
exemple/memstore → context exemple/loan slices sync
```

`httpapi` et `memstore` importent `loan` ; `loan` n'importe rien du projet. Si un jour `loan` importe
`net/http` ou `memstore`, la frontière est franchie : le domaine dépend d'un détail technique, et le
compilateur Go refusera même certains retours en arrière (les cycles d'imports sont interdits). La leçon
7 vérifie ces règles automatiquement par un test.

### Le « service » : orchestrer sans décider

Entre le transport et le domaine, `loan.Service` **orchestre** un cas d'usage : charger l'adhérent,
appeler la règle avec l'heure courante, enregistrer. Il ne contient aucune règle (elles sont dans
`Member.Borrow`) et aucun détail HTTP. La leçon 2 précise ce qui va dans chaque couche.

### Quand ne pas le faire

Séparer a un coût : plus de paquets, plus de fichiers, des traductions entre types. Ce coût ne se justifie
pas pour :

- un **script** ou un outil interne de 200 lignes qu'on jettera ;
- un service **sans règles** : un proxy, un exportateur de métriques, un CRUD qui ne fait que recopier des
  champs entre HTTP et la base. Il n'y a pas de domaine à isoler ; trois couches qui se contentent de
  passer les valeurs à la suivante sont du bruit ;
- un **prototype** dont on ne sait pas encore s'il vivra.

Le bon moment pour séparer est souvent celui où la première règle métier apparaît, ou celui où l'on doit
la tester. Commencer simple (un paquet), séparer quand la douleur est réelle, et le faire **avant** que le
paquet ne fasse 3 000 lignes.

## Exemple

Le même comportement deux fois : tout dans un handler (`avant`), puis séparé en domaine, service,
stockage et HTTP.

```go
// Fichier : avant/handler.go
// Package avant : TOUT dans un handler. Ça marche — et rien ne se teste ni ne se réutilise.
package avant

import (
	"encoding/json"
	"net/http"
	"sync"
	"time"
)

var (
	mu    sync.Mutex
	loans = map[string][]time.Time{} // adhérent → dates de retour prévues
)

func Borrow(w http.ResponseWriter, r *http.Request) {
	var in struct {
		BookID string `json:"book_id"`
	}
	if err := json.NewDecoder(r.Body).Decode(&in); err != nil || in.BookID == "" {
		http.Error(w, "requête invalide", http.StatusBadRequest)
		return
	}
	member := r.PathValue("id")
	mu.Lock()
	defer mu.Unlock()
	if len(loans[member]) >= 3 { // règle métier, enfouie dans du HTTP
		http.Error(w, "trop d'emprunts", http.StatusConflict)
		return
	}
	for _, due := range loans[member] {
		if time.Now().After(due) { // horloge non injectable : la règle est intestable
			http.Error(w, "emprunt en retard", http.StatusConflict)
			return
		}
	}
	due := time.Now().Add(21 * 24 * time.Hour)
	loans[member] = append(loans[member], due)
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(map[string]any{"book_id": in.BookID, "due": due})
}
```

```go
// Fichier : loan/loan.go
// Package loan est le DOMAINE : les règles du prêt. Il n'importe ni net/http ni database/sql.
package loan

import (
	"errors"
	"fmt"
	"time"
)

const (
	MaxLoans   = 3
	LoanPeriod = 21 * 24 * time.Hour
)

var (
	ErrTooManyLoans = errors.New("nombre maximal d'emprunts atteint")
	ErrOverdue      = errors.New("un emprunt est en retard")
	ErrNotFound     = errors.New("adhérent introuvable")
	ErrInvalid      = errors.New("demande invalide")
)

type Loan struct {
	BookID string
	Due    time.Time
}

type Member struct {
	ID    string
	Loans []Loan
}

// Borrow applique les règles et enregistre l'emprunt. « Maintenant » est un paramètre :
// la règle se teste à n'importe quelle date, sans attendre trois semaines.
func (m *Member) Borrow(bookID string, now time.Time) (Loan, error) {
	if bookID == "" {
		return Loan{}, fmt.Errorf("%w : livre manquant", ErrInvalid)
	}
	if len(m.Loans) >= MaxLoans {
		return Loan{}, ErrTooManyLoans
	}
	for _, l := range m.Loans {
		if now.After(l.Due) {
			return Loan{}, fmt.Errorf("%w : %s attendu le %s", ErrOverdue, l.BookID, l.Due.Format(time.DateOnly))
		}
	}
	l := Loan{BookID: bookID, Due: now.Add(LoanPeriod)}
	m.Loans = append(m.Loans, l)
	return l, nil
}
```

```go
// Fichier : loan/service.go
package loan

import (
	"context"
	"time"
)

// Members est ce dont le service a besoin du stockage — déclaré ici, côté consommateur.
type Members interface {
	Get(ctx context.Context, id string) (Member, error)
	Save(ctx context.Context, m Member) error
}

// Service orchestre : charger, appliquer la règle, enregistrer.
type Service struct {
	members Members
	now     func() time.Time
}

func NewService(members Members, now func() time.Time) *Service {
	return &Service{members: members, now: now}
}

func (s *Service) Borrow(ctx context.Context, memberID, bookID string) (Loan, error) {
	m, err := s.members.Get(ctx, memberID)
	if err != nil {
		return Loan{}, err
	}
	l, err := m.Borrow(bookID, s.now())
	if err != nil {
		return Loan{}, err
	}
	return l, s.members.Save(ctx, m)
}
```

```go
// Fichier : loan/loan_test.go
package loan

import (
	"errors"
	"testing"
	"time"
)

var day0 = time.Date(2026, 3, 1, 10, 0, 0, 0, time.UTC)

func TestBorrowRules(t *testing.T) {
	m := Member{ID: "ada"}
	for _, b := range []string{"a", "b", "c"} {
		if _, err := m.Borrow(b, day0); err != nil {
			t.Fatal(err)
		}
	}
	if _, err := m.Borrow("d", day0); !errors.Is(err, ErrTooManyLoans) {
		t.Fatalf("4e emprunt : %v", err)
	}
	late := Member{ID: "bob", Loans: []Loan{{BookID: "x", Due: day0}}}
	if _, err := late.Borrow("y", day0.Add(time.Hour)); !errors.Is(err, ErrOverdue) {
		t.Fatalf("retard : %v", err)
	}
	if _, err := late.Borrow("y", day0.Add(-time.Hour)); err != nil {
		t.Fatalf("pas encore en retard : %v", err)
	}
}
```

```go
// Fichier : memstore/memstore.go
// Package memstore : un stockage en mémoire qui satisfait loan.Members. Il importe loan
// (pour ses types et ses erreurs) ; loan, lui, ne sait pas que memstore existe.
package memstore

import (
	"context"
	"slices"
	"sync"

	"exemple/loan"
)

type Store struct {
	mu      sync.Mutex
	members map[string]loan.Member
}

func New(ids ...string) *Store {
	s := &Store{members: map[string]loan.Member{}}
	for _, id := range ids {
		s.members[id] = loan.Member{ID: id}
	}
	return s
}

func (s *Store) Get(_ context.Context, id string) (loan.Member, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	m, ok := s.members[id]
	if !ok {
		return loan.Member{}, loan.ErrNotFound
	}
	m.Loans = slices.Clone(m.Loans) // une copie : l'appelant ne modifie pas le stockage par accident
	return m, nil
}

func (s *Store) Save(_ context.Context, m loan.Member) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.members[m.ID] = m
	return nil
}
```

```go
// Fichier : httpapi/httpapi.go
// Package httpapi : la traduction HTTP ↔ domaine, et rien d'autre.
package httpapi

import (
	"encoding/json"
	"errors"
	"net/http"

	"exemple/loan"
)

func Routes(svc *loan.Service) http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("POST /members/{id}/loans", func(w http.ResponseWriter, r *http.Request) {
		var in struct {
			BookID string `json:"book_id"`
		}
		if err := json.NewDecoder(http.MaxBytesReader(w, r.Body, 1<<10)).Decode(&in); err != nil {
			http.Error(w, "JSON invalide", http.StatusBadRequest)
			return
		}
		l, err := svc.Borrow(r.Context(), r.PathValue("id"), in.BookID)
		if err != nil {
			http.Error(w, err.Error(), status(err))
			return
		}
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusCreated)
		json.NewEncoder(w).Encode(map[string]any{"book_id": l.BookID, "due": l.Due})
	})
	return mux
}

// status : la seule traduction erreur du domaine → statut HTTP (N7 L5).
func status(err error) int {
	switch {
	case errors.Is(err, loan.ErrInvalid):
		return http.StatusUnprocessableEntity
	case errors.Is(err, loan.ErrNotFound):
		return http.StatusNotFound
	case errors.Is(err, loan.ErrTooManyLoans), errors.Is(err, loan.ErrOverdue):
		return http.StatusConflict
	default:
		return http.StatusInternalServerError
	}
}
```

```go
// Fichier : httpapi/httpapi_test.go
package httpapi

import (
	"net/http/httptest"
	"strings"
	"testing"
	"time"

	"exemple/loan"
	"exemple/memstore"
)

func TestBorrowHTTP(t *testing.T) {
	now := time.Date(2026, 3, 1, 10, 0, 0, 0, time.UTC)
	h := Routes(loan.NewService(memstore.New("ada"), func() time.Time { return now }))
	codes := []int{}
	for _, body := range []string{`{"book_id":"a"}`, `{"book_id":"b"}`, `{"book_id":"c"}`, `{"book_id":"d"}`, `{"book_id":""}`, `{`} {
		rec := httptest.NewRecorder()
		h.ServeHTTP(rec, httptest.NewRequest("POST", "/members/ada/loans", strings.NewReader(body)))
		codes = append(codes, rec.Code)
	}
	rec := httptest.NewRecorder()
	h.ServeHTTP(rec, httptest.NewRequest("POST", "/members/inconnu/loans", strings.NewReader(`{"book_id":"a"}`)))
	codes = append(codes, rec.Code)
	want := []int{201, 201, 201, 409, 422, 400, 404}
	for i := range want {
		if codes[i] != want[i] {
			t.Fatalf("statuts = %v, attendu %v", codes, want)
		}
	}
}
```
`go test -race ./...` passe : `loan` sans serveur ni stockage, `httpapi` avec le vrai domaine et le
stockage en mémoire. Le paquet `avant`, lui, n'a pas de test : pour en écrire un, il faudrait un serveur,
manipuler la map globale, et attendre trois semaines pour tester le retard (`time.Now()` en dur).

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `avant.Borrow` | Lecture JSON, règles, horloge, stockage global et réponse dans une seule fonction. |
| `var loans = map…` global | État partagé par tous les tests, impossible à isoler. |
| `loan.Member.Borrow(bookID, now)` | La règle, pure : elle ne lit ni l'horloge ni le stockage. |
| Erreurs du domaine (`ErrTooManyLoans`…) | Le domaine dit **ce qui** s'est passé ; `httpapi.status` décide du statut. |
| `loan.Members` déclarée dans `loan` | L'interface appartient au consommateur ; `memstore` s'y conforme. |
| `Service.now func() time.Time` | L'horloge est une dépendance comme une autre : injectée. |
| `memstore.Get` rend une **copie** des emprunts | L'appelant ne modifie pas l'état stocké sans `Save`. |
| `TestBorrowRules` | Trois règles testées à des dates arbitraires, en microsecondes. |
| `TestBorrowHTTP` | Le transport testé avec le vrai domaine : statuts 201, 409, 422, 400, 404. |

## Erreurs fréquentes

1. **Des règles métier dans les handlers**, ou dans les requêtes SQL.
2. **Un domaine qui importe `net/http`** (pour un statut) ou un paquet de stockage.
3. **`time.Now()` au milieu d'une règle** : la règle devient intestable aux dates qui comptent.
4. **Un état global** (`var store = …`) au lieu d'une dépendance passée au constructeur.
5. **Des couches qui ne font que recopier** les valeurs à la couche suivante, sans aucune règle.
6. **Séparer par type technique** (`models/`, `controllers/`, `utils/`) au lieu de séparer par
   responsabilité et par domaine.
7. **Attendre que le fichier fasse 3 000 lignes** pour séparer.

## Bonnes pratiques Go

- Un paquet de domaine par concept métier, nommé d'après le métier (`loan`), sans import technique.
- Les adaptateurs (`httpapi`, `pgstore`, `cli`) importent le domaine, jamais l'inverse.
- Horloge, aléa, identifiants, stockage : passés en paramètre ou injectés.
- Lire régulièrement `go list -f '{{.ImportPath}} → {{join .Imports " "}}' ./...`.
- Commencer simple, séparer quand la première règle mérite un test.

## Ce que je dois retenir

- Trois responsabilités : transport, règles, stockage/services externes ; elles changent pour des raisons
  différentes.
- Le domaine au centre, sans technologie ; les dépendances pointent vers lui.
- Les imports **sont** l'architecture.
- Une règle qui reçoit « maintenant » en paramètre se teste à n'importe quelle date.
- Ne pas séparer ce qui n'a pas de règles ; séparer dès qu'une règle mérite d'être testée seule.

---

➡️ [Exercices](exercices.md)
