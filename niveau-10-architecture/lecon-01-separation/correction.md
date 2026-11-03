# Leçon 1 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Le corrigé complète le module de l'exemple : les fichiers modifiés (`loan/loan.go`, `loan/service.go`,
`loan/loan_test.go`, `httpapi/httpapi.go`) sont donnés en entier, les autres sont ceux du cours.

## E1 — Tester `avant`

```go
// Fichier : avant/handler_test.go
package avant

import (
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"
)

// E1 : pour simuler un retard, il faut écrire DANS la variable globale. Le test dépend de
// l'état laissé par les autres, et ne peut pas s'exécuter en parallèle.
func TestOverdue(t *testing.T) {
	mu.Lock()
	loans["bob"] = []time.Time{time.Now().Add(-time.Hour)} // un retour prévu dans le passé
	mu.Unlock()
	t.Cleanup(func() { mu.Lock(); delete(loans, "bob"); mu.Unlock() })

	mux := http.NewServeMux()
	mux.HandleFunc("POST /members/{id}/loans", Borrow)
	rec := httptest.NewRecorder()
	mux.ServeHTTP(rec, httptest.NewRequest("POST", "/members/bob/loans", strings.NewReader(`{"book_id":"y"}`)))
	if rec.Code != http.StatusConflict {
		t.Fatalf("statut %d", rec.Code)
	}
}
```

Pour simuler un retard, il faut **écrire dans la map globale** une date passée, puis la nettoyer : le test
connaît la représentation interne du handler. Il fait autant de lignes que `TestBorrowRules`, mais n'en
teste qu'un cas, et chaque cas supplémentaire demande la même mise en scène. Surtout, deux tests de ce
paquet ne peuvent pas s'exécuter en **parallèle** : ils partagent `loans`, et l'un verrait les emprunts de
l'autre. Et la règle « à l'heure exactement » est impossible à tester : `time.Now()` est lu à l'intérieur.

## E2 — Lire les imports

Avec `func Status(err error) int` dans `loan`, `go list` montre `exemple/loan → … net/http` : compiler
(et tester) le domaine demande désormais tout `net/http`, et le domaine connaît le transport. Importer
`memstore` depuis `loan` est refusé par le compilateur :

```
package exemple/loan
	imports exemple/memstore from service.go
	imports exemple/loan from memstore.go: import cycle not allowed
```

`memstore` importe déjà `loan` (pour `loan.Member`). Go interdit les cycles d'imports : c'est une
protection gratuite de l'architecture, à condition que la dépendance « normale » (adaptateur → domaine)
existe déjà.

## E3 — Une deuxième porte d'entrée

```go
// Fichier : cmd/emprunt/main.go
// E3 : une deuxième porte d'entrée. Aucune ligne de loan n'a changé.
package main

import (
	"context"
	"errors"
	"flag"
	"fmt"
	"os"
	"time"

	"exemple/loan"
	"exemple/memstore"
)

func main() {
	member := flag.String("member", "", "adhérent")
	book := flag.String("book", "", "livre")
	flag.Parse()
	svc := loan.NewService(memstore.New("ada", "bob"), time.Now)
	l, err := svc.Borrow(context.Background(), *member, *book)
	if err != nil {
		fmt.Fprintln(os.Stderr, "emprunt refusé :", err)
		code := 1
		if errors.Is(err, loan.ErrInvalid) {
			code = 2 // erreur d'utilisation
		}
		os.Exit(code)
	}
	fmt.Printf("%s à rendre le %s\n", l.BookID, l.Due.Format(time.DateOnly))
}
```

```
$ go run ./cmd/emprunt -member ada -book a
a à rendre le 2026-11-24
$ go run ./cmd/emprunt -member zoe -book a
emprunt refusé : adhérent introuvable
exit status 1
```

**Aucune** ligne de `loan` n'a changé : la traduction erreur → code de sortie (`ErrInvalid` → 2) joue pour
la CLI le rôle de `httpapi.status` pour HTTP. C'est la preuve concrète que la séparation a payé.

## E4 — Le retour

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

var ErrNotBorrowed = errors.New("ce livre n'est pas emprunté par cet adhérent")

// Return retire l'emprunt et rend le nombre de jours ENTAMÉS de retard (0 si à l'heure).
func (m *Member) Return(bookID string, now time.Time) (int, error) {
	for i, l := range m.Loans {
		if l.BookID != bookID {
			continue
		}
		m.Loans = append(m.Loans[:i], m.Loans[i+1:]...)
		if !now.After(l.Due) {
			return 0, nil
		}
		late := now.Sub(l.Due)
		days := int(late / (24 * time.Hour))
		if late%(24*time.Hour) != 0 {
			days++
		}
		return days, nil
	}
	return 0, ErrNotBorrowed
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

func (s *Service) Return(ctx context.Context, memberID, bookID string) (int, error) {
	m, err := s.members.Get(ctx, memberID)
	if err != nil {
		return 0, err
	}
	days, err := m.Return(bookID, s.now())
	if err != nil {
		return 0, err
	}
	return days, s.members.Save(ctx, m)
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

func TestReturn(t *testing.T) {
	due := day0.Add(LoanPeriod)
	cases := []struct {
		at   time.Time
		want int
	}{
		{due, 0},                              // le jour même, à l'heure exacte : à l'heure
		{due.Add(time.Minute), 1},             // une minute de retard : un jour entamé
		{due.Add(24 * time.Hour), 1},          // exactement un jour
		{due.Add(24*time.Hour + time.Second), 2},
	}
	for _, c := range cases {
		m := Member{ID: "ada"}
		m.Borrow("a", day0)
		got, err := m.Return("a", c.at)
		if err != nil || got != c.want || len(m.Loans) != 0 {
			t.Errorf("retour à %v : %d, %v ; attendu %d", c.at.Sub(due), got, err, c.want)
		}
	}
	m := Member{ID: "ada"}
	if _, err := m.Return("inconnu", day0); !errors.Is(err, ErrNotBorrowed) {
		t.Fatalf("err = %v", err)
	}
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
	mux.HandleFunc("DELETE /members/{id}/loans/{book}", func(w http.ResponseWriter, r *http.Request) {
		days, err := svc.Return(r.Context(), r.PathValue("id"), r.PathValue("book"))
		if err != nil {
			http.Error(w, err.Error(), status(err))
			return
		}
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]int{"late_days": days})
	})
	return mux
}

// status : la seule traduction erreur du domaine → statut HTTP (N7 L5).
func status(err error) int {
	switch {
	case errors.Is(err, loan.ErrInvalid):
		return http.StatusUnprocessableEntity
	case errors.Is(err, loan.ErrNotFound), errors.Is(err, loan.ErrNotBorrowed):
		return http.StatusNotFound
	case errors.Is(err, loan.ErrTooManyLoans), errors.Is(err, loan.ErrOverdue):
		return http.StatusConflict
	default:
		return http.StatusInternalServerError
	}
}
```

La décision « un jour **entamé** compte comme un jour » est une règle métier : elle est écrite et testée
dans le domaine, aux limites exactes (à l'heure pile, une minute après, exactement 24 h, 24 h et une
seconde). Rendre un livre non emprunté est une erreur du domaine (`ErrNotBorrowed`) traduite en 404.

## E5 — Quand ne pas séparer

- **(a) Proxy** : non. Aucune règle métier ; un seul paquet, quelques handlers et un `httputil.ReverseProxy`
  (N7 L9).
- **(b) Export CSV nocturne** : non. Une requête et un écrivain CSV ; découper en couches ajouterait des
  types intermédiaires sans rien à tester de plus.
- **(c) Pénalités de retard** : **oui**. Tarifs dégressifs, exceptions, cas limites de dates : exactement
  ce qui doit se tester seul, en grand nombre, sans base.
- **(d) Webhooks de paiement** : **oui**, modérément. La règle (quelles transitions de statut sont
  permises, idempotence d'un webhook reçu deux fois, leçon 9) mérite un domaine ; le reste est de la
  traduction.

## Exercice intermédiaire — Refactoriser sans casser

```go
// Fichier : caracterisation/suite.go
// Package caracterisation : le comportement HTTP figé, exécuté contre l'ancienne ET la nouvelle version.
package caracterisation

import (
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

type step struct {
	member, body string
	wantCode     int
	wantKeys     []string // clés JSON attendues dans une réponse 201
}

var steps = []step{
	{"ada", `{"book_id":"a"}`, 201, []string{"book_id", "due"}},
	{"ada", `{"book_id":"b"}`, 201, nil},
	{"ada", `{"book_id":"c"}`, 201, nil},
	{"ada", `{"book_id":"d"}`, 409, nil},
	{"ada", `{`, 400, nil},
}

// Run exécute la suite ; newHandler rend un handler NEUF (état vierge) avec l'adhérent "ada".
func Run(t *testing.T, newHandler func(t *testing.T) http.Handler) {
	h := newHandler(t)
	for i, s := range steps {
		rec := httptest.NewRecorder()
		h.ServeHTTP(rec, httptest.NewRequest("POST", "/members/"+s.member+"/loans", strings.NewReader(s.body)))
		if rec.Code != s.wantCode {
			t.Fatalf("étape %d : statut %d, attendu %d (%s)", i, rec.Code, s.wantCode, rec.Body)
		}
		if s.wantKeys != nil {
			var got map[string]any
			if err := json.Unmarshal(rec.Body.Bytes(), &got); err != nil {
				t.Fatal(err)
			}
			for _, k := range s.wantKeys {
				if _, ok := got[k]; !ok {
					t.Fatalf("étape %d : clé %q absente de %s", i, k, rec.Body)
				}
			}
		}
	}
}
```

```go
// Fichier : caracterisation/suite_test.go
package caracterisation

import (
	"net/http"
	"testing"
	"time"

	"exemple/avant"
	"exemple/httpapi"
	"exemple/loan"
	"exemple/memstore"
)

func TestAvant(t *testing.T) {
	Run(t, func(t *testing.T) http.Handler {
		mux := http.NewServeMux()
		mux.HandleFunc("POST /members/{id}/loans", avant.Borrow)
		return mux
	})
}

func TestApres(t *testing.T) {
	Run(t, func(t *testing.T) http.Handler {
		return httpapi.Routes(loan.NewService(memstore.New("ada"), time.Now))
	})
}
```

La même suite passe contre `avant` et contre la nouvelle version : le comportement **observable** est
conservé. Points à noter :

- **Ce qui est figé** : les statuts, et les **clés** du JSON de succès. Pas le texte des erreurs
  (`trop d'emprunts` contre `nombre maximal d'emprunts atteint`) : ce changement a été **décidé** — les
  clients ne devraient pas dépendre d'un texte d'erreur (N7 L5) — et doit figurer dans les notes de
  version.
- **Ce qui a changé volontairement** : une demande sans livre (`{"book_id":""}`) donnait 400 dans
  `avant`, elle donne 422 dans la nouvelle version (entrée bien formée mais invalide). La suite de
  caractérisation ne contient donc pas ce cas ; on l'a retiré **en le notant**, au lieu de le laisser
  échouer ou de l'effacer en silence.
- **La méthode** : tests de caractérisation d'abord, puis petites étapes (extraire la règle, puis le
  stockage, puis l'horloge), tests verts après chacune. Un grand remplacement d'un coup ne permet pas de
  savoir quelle étape a cassé quoi.
- **Mesures** : `avant` ≈ 40 lignes, un test possible seulement à travers HTTP ; après, ≈ 150 lignes
  réparties, et des tests du domaine qui s'exécutent en microsecondes à n'importe quelle date. Trois fois
  plus de code pour une règle aussi simple : c'est la limite discutée en E5 — la séparation paie quand
  les règles grossissent.

## Défi — Une règle qui dépend de plusieurs agrégats

1. **Où placer la règle.** « Un exemplaire n'a qu'un emprunteur » concerne le **livre**, pas l'adhérent :
   `Member` ne connaît pas les emprunts des autres. Deux versions :
   - dans le **service**, qui consulte un `Books` (« ce livre est-il libre ? ») avant d'appeler
     `Member.Borrow` : simple, mais la règle sort du domaine et se teste avec des faux ;
   - un type `Book` (exemplaire) avec `Book.Lend(memberID, now)` qui vérifie disponibilité et
     réservation, et que le service appelle **en plus** de `Member.Borrow` : les deux règles restent
     dans le domaine, testables seules ; le service orchestre deux objets.
   La seconde est plus testable ; c'est aussi celle que la leçon 6 formalise.
2. **Concurrence.** En mémoire, le service doit tenir un verrou couvrant la lecture **et** l'écriture des
   deux objets (sinon « vérifier puis agir », N6 L6) : un verrou du stockage par objet ne suffit pas, il
   faut une opération atomique sur les deux. Avec PostgreSQL : une transaction qui verrouille la ligne de
   l'exemplaire (`SELECT … FOR UPDATE`) et l'adhérent, dans un ordre fixe (N8 L4), ou une contrainte
   `UNIQUE` sur « emprunt en cours par exemplaire » (un index partiel `WHERE returned_at IS NULL`), qui
   rend la double attribution impossible quelle que soit la logique Go.
3. **Domaine ou stockage ?** La **règle** (un seul emprunteur) est du domaine ; la **garantie** sous
   concurrence est fournie par le stockage (verrou, transaction, contrainte). La leçon 6 en donne le
   vocabulaire : chaque **agrégat** (adhérent, exemplaire) est une frontière de cohérence ; une règle qui
   en couvre deux demande soit une transaction qui les englobe, soit d'accepter une cohérence à terme.

## Quiz

1. Transport, règles métier, stockage/services externes ; elles changent pour des raisons différentes.
2. Une technologie de transport ou de stockage (`net/http`, `database/sql`), ou un paquet d'adaptation.
3. Vers le domaine : les adaptateurs l'importent, il n'importe rien du projet.
4. `go list -f '{{.ImportPath}} → {{join .Imports " "}}' ./...`.
5. Pour tester la règle à n'importe quelle date, sans dépendre de l'horloge.
6. Dans le paquet qui l'utilise (le domaine) : c'est lui qui sait ce dont il a besoin.
7. Il orchestre un cas d'usage (charger, appliquer, enregistrer) ; il ne contient ni règle ni détail de
   transport.
8. Pour que l'appelant ne modifie pas l'état stocké sans passer par `Save`.
9. Un proxy, un export sans règle, un prototype jetable.
10. Un test qui fige le comportement existant d'un code avant de le modifier ; à écrire avant toute
    refactorisation d'un code non testé.
