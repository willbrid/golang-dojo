# Leçon 6 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Contourner

- `ada.loans = nil` : refusé, `ada.loans undefined (cannot refer to unexported field loans)`.
- `lending.CopyID{"x"}` : refusé, `implicit assignment to unexported field v in struct literal of type lending.CopyID`.
- `lending.CopyID{}` : **compile**. La valeur zéro d'un type est toujours constructible hors de son paquet,
  et elle a échappé à `ParseCopyID`. Mesuré : `ada.Borrow(lending.CopyID{}, now)` rendait `nil` — un
  emprunt sans cote. C'est une vraie faille, et la limite de la technique.

Parade : dans chaque méthode qui reçoit un `CopyID`, refuser la valeur zéro
(`if c == (CopyID{}) { return …ErrInvalid }`, ajouté à `Borrow` ci-dessous). Autre approche : faire en sorte
que la valeur zéro soit **valide et inoffensive** par conception, quand c'est possible (ce n'est pas le
cas pour une cote).

## E2 — Le langage

Mots du métier : *adhérent*, *rendre en retard*, *moins d'un mois*, *réserver*, *nouveauté*. Signature :
`func (m *Member) Reserve(w Work, now time.Time) error`, erreur `ErrRecentLateReturn` (« retour en retard
récent »). La règle appartient à l'agrégat `Member` (c'est un droit de l'adhérent), mais il lui manque
deux informations : la **date du dernier retour en retard** (à mémoriser dans `Return`) et le fait qu'une
œuvre soit une **nouveauté** (une notion du catalogue, passée en paramètre ou par une valeur
`IsNew bool` fournie par le service, pas en important le catalogue).

## E3 — Payer

`PayFines` est dans `member.go` ci-dessous ; ses tests dans `corr_test.go`.

## E4 — L'égalité

`lending.Cents(100) == lending.Cents(100)` est **vrai** : deux valeurs identiques sont interchangeables.
Deux `*Member` créés séparément avec le même identifiant sont **différents** pour `==` (deux pointeurs).
C'est exactement la distinction : une valeur n'a pas d'identité, seule sa valeur compte ; une entité a
une identité, et deux objets en mémoire ne sont « la même entité » que si leurs **identifiants** sont
égaux (comparer `a.ID() == b.ID()`, jamais les pointeurs ni les structs entières).

## E5 — La taille de l'agrégat

| Règle | Où |
|---|---|
| (a) 3 emprunts au plus | dans `Member` : invariant immédiat d'un seul agrégat |
| (b) un exemplaire, un emprunteur | dans un agrégat `Copy` (qui garantit qu'il n'est pas prêté deux fois) ; l'emprunt relie `Member` et `Copy` : transaction commune (N8 L4) ou, à grande échelle, cohérence à terme avec compensation |
| (c) notification à J-3 | hors des agrégats : une tâche planifiée qui lit les échéances et publie des rappels (cohérence à terme) |
| (d) 1 000 exemplaires prêtés au plus | un compteur global : un agrégat dédié (`LendingQuota`) très disputé, ou une règle approximative en cohérence à terme ; ne surtout pas faire de `Library` un agrégat géant |
| (e) réservé → seulement pour le réservataire | dans `Copy` (la réservation est un état de l'exemplaire) |

## Exercice intermédiaire — Persister un agrégat

```go
// Fichier : lending/member.go
package lending

import (
	"errors"
	"fmt"
	"time"
)

var (
	ErrTooManyLoans  = errors.New("nombre maximal d'emprunts atteint")
	ErrUnpaidFines   = errors.New("pénalités impayées au-delà du seuil")
	ErrNotBorrowed   = errors.New("exemplaire non emprunté par cet adhérent")
	ErrRenewalDenied = errors.New("prolongation refusée")
)

const (
	MaxLoans    = 3
	LoanPeriod  = 21 * 24 * time.Hour
	MaxRenewals = 1
)

var (
	FinePerDay = Cents(20)  // 0,20 € par jour entamé
	FineLimit  = Cents(500) // au-delà de 5 €, plus d'emprunt
)

// loan : ENTITÉ interne à l'agrégat (identité : la cote de l'exemplaire). Invisible de l'extérieur.
type loan struct {
	copy     CopyID
	due      time.Time
	renewals int
}

// Event : ce qui s'est passé, dans le langage du métier. Collecté par l'agrégat (leçon 8).
type Event struct {
	Name string
	At   time.Time
	Data string
}

// Member : RACINE de l'agrégat. Tous les changements passent par ses méthodes,
// qui garantissent les invariants. Les champs non exportés empêchent tout contournement.
type Member struct {
	id     string
	loans  []loan
	fines  Money
	events []Event
}

func NewMember(id string) *Member { return &Member{id: id} }

func (m *Member) ID() string     { return m.id }
func (m *Member) LoanCount() int { return len(m.loans) }
func (m *Member) Fines() Money   { return m.fines }

// PullEvents rend les événements enregistrés et vide la liste.
func (m *Member) PullEvents() []Event {
	ev := m.events
	m.events = nil
	return ev
}

func (m *Member) record(name string, at time.Time, data string) {
	m.events = append(m.events, Event{name, at, data})
}

// Borrow : invariants « 3 au plus » et « pénalités sous le seuil ».
func (m *Member) Borrow(c CopyID, now time.Time) error {
	if c == (CopyID{}) { // la valeur zéro échappe à ParseCopyID : on la refuse ici
		return fmt.Errorf("%w : cote vide", ErrInvalid)
	}
	if len(m.loans) >= MaxLoans {
		return ErrTooManyLoans
	}
	if m.fines.GreaterThan(FineLimit) {
		return fmt.Errorf("%w (%s)", ErrUnpaidFines, m.fines)
	}
	m.loans = append(m.loans, loan{copy: c, due: now.Add(LoanPeriod)})
	m.record("ExemplaireEmprunté", now, c.String())
	return nil
}

// Return : calcule et ajoute la pénalité de retard.
func (m *Member) Return(c CopyID, now time.Time) (Money, error) {
	for i, l := range m.loans {
		if l.copy != c {
			continue
		}
		m.loans = append(m.loans[:i], m.loans[i+1:]...)
		fine := Cents(0)
		if late := now.Sub(l.due); late > 0 {
			days := int64((late + 24*time.Hour - 1) / (24 * time.Hour))
			fine = Money{days * FinePerDay.cents}
			m.fines = m.fines.Add(fine)
		}
		m.record("ExemplaireRendu", now, fmt.Sprintf("%s pénalité=%s", c, fine))
		return fine, nil
	}
	return Cents(0), ErrNotBorrowed
}

var ErrBadPayment = errors.New("paiement invalide")

// PayFines : ni montant négatif ou nul, ni plus que ce qui est dû.
func (m *Member) PayFines(amount Money, now time.Time) (Money, error) {
	switch {
	case !amount.GreaterThan(Cents(0)):
		return m.fines, fmt.Errorf("%w : montant %s", ErrBadPayment, amount)
	case amount.GreaterThan(m.fines):
		return m.fines, fmt.Errorf("%w : %s payé pour %s dû", ErrBadPayment, amount, m.fines)
	}
	m.fines = Money{m.fines.cents - amount.cents}
	m.record("PénalitéPayée", now, amount.String())
	return m.fines, nil
}

// Renew : une prolongation au plus, et jamais d'un emprunt déjà en retard.
func (m *Member) Renew(c CopyID, now time.Time) error {
	for i := range m.loans {
		l := &m.loans[i]
		if l.copy != c {
			continue
		}
		switch {
		case l.renewals >= MaxRenewals:
			return fmt.Errorf("%w : déjà prolongé", ErrRenewalDenied)
		case now.After(l.due):
			return fmt.Errorf("%w : emprunt en retard", ErrRenewalDenied)
		}
		l.renewals++
		l.due = l.due.Add(LoanPeriod)
		m.record("EmpruntProlongé", now, c.String())
		return nil
	}
	return ErrNotBorrowed
}
```

```go
// Fichier : lending/state.go
// Persistance d'un agrégat aux champs non exportés : un état exporté, et une reconstruction
// qui REVÉRIFIE les invariants.
package lending

import (
	"errors"
	"fmt"
	"time"
)

type LoanState struct {
	Copy     string
	Due      time.Time
	Renewals int
}

type MemberState struct {
	ID         string
	Loans      []LoanState
	FinesCents int64
	Version    int // verrou optimiste : incrémenté à chaque enregistrement
}

// Snapshot rend une COPIE de l'état : la modifier ne modifie pas l'agrégat.
func (m *Member) Snapshot() MemberState {
	st := MemberState{ID: m.id, FinesCents: m.fines.cents}
	for _, l := range m.loans {
		st.Loans = append(st.Loans, LoanState{l.copy.v, l.due, l.renewals})
	}
	return st
}

var ErrCorruptState = errors.New("état d'adhérent invalide")

func RestoreMember(st MemberState) (*Member, error) {
	if st.ID == "" || st.FinesCents < 0 || len(st.Loans) > MaxLoans {
		return nil, fmt.Errorf("%w : %+v", ErrCorruptState, st)
	}
	m := &Member{id: st.ID, fines: Cents(st.FinesCents)}
	for _, l := range st.Loans {
		c, err := ParseCopyID(l.Copy)
		if err != nil || l.Renewals < 0 || l.Renewals > MaxRenewals {
			return nil, fmt.Errorf("%w : emprunt %+v", ErrCorruptState, l)
		}
		m.loans = append(m.loans, loan{c, l.Due, l.Renewals})
	}
	return m, nil
}
```

```go
// Fichier : lending/repo.go
package lending

import (
	"context"
	"errors"
	"slices"
	"sync"
)

var (
	ErrConcurrentModification = errors.New("modification concurrente : recharger et réessayer")
	ErrNotFound               = errors.New("adhérent introuvable")
)

// MemRepo stocke des ÉTATS (copies), jamais des *Member partagés : chaque chargement
// rend un agrégat indépendant, comme le ferait une base.
type MemRepo struct {
	mu   sync.Mutex
	data map[string]MemberState
}

func NewMemRepo() *MemRepo { return &MemRepo{data: map[string]MemberState{}} }

func (r *MemRepo) Load(_ context.Context, id string) (*Member, int, error) {
	r.mu.Lock()
	st, ok := r.data[id]
	r.mu.Unlock()
	if !ok {
		return nil, 0, ErrNotFound
	}
	st.Loans = slices.Clone(st.Loans)
	m, err := RestoreMember(st)
	return m, st.Version, err
}

// Save échoue si la version stockée n'est plus celle qu'on a chargée.
func (r *MemRepo) Save(_ context.Context, m *Member, loadedVersion int) error {
	r.mu.Lock()
	defer r.mu.Unlock()
	if cur, ok := r.data[m.id]; ok && cur.Version != loadedVersion {
		return ErrConcurrentModification
	}
	st := m.Snapshot()
	st.Version = loadedVersion + 1
	r.data[m.id] = st
	return nil
}
```

```go
// Fichier : lending/service.go
package lending

import (
	"context"
	"sync"
	"time"
)

type Service struct {
	repo   *MemRepo
	now    func() time.Time
	mu     sync.Mutex
	events []Event // en vrai : écrits dans une outbox, dans la transaction (leçon 8)
}

func (s *Service) Events() []Event {
	s.mu.Lock()
	defer s.mu.Unlock()
	return append([]Event(nil), s.events...)
}

func NewService(r *MemRepo, now func() time.Time) *Service { return &Service{repo: r, now: now} }

func (s *Service) Borrow(ctx context.Context, memberID string, c CopyID) error {
	m, version, err := s.repo.Load(ctx, memberID)
	if err != nil {
		return err
	}
	if err := m.Borrow(c, s.now()); err != nil {
		return err
	}
	if err := s.repo.Save(ctx, m, version); err != nil {
		return err
	}
	s.mu.Lock()
	s.events = append(s.events, m.PullEvents()...)
	s.mu.Unlock()
	return nil
}
```

```go
// Fichier : lending/corr_test.go
package lending_test

import (
	"errors"
	"sync"
	"testing"
	"time"

	"exemple/lending"
)

var now = time.Date(2026, 3, 1, 10, 0, 0, 0, time.UTC)

func cp(s string) lending.CopyID { c, _ := lending.ParseCopyID(s); return c }

func TestZeroCopyIDRefused(t *testing.T) {
	if err := lending.NewMember("ada").Borrow(lending.CopyID{}, now); !errors.Is(err, lending.ErrInvalid) {
		t.Fatalf("err = %v", err)
	}
}

func TestPayFines(t *testing.T) {
	ada := lending.NewMember("ada")
	ada.Borrow(cp("EX-1"), now)
	ada.Return(cp("EX-1"), now.AddDate(0, 0, 31)) // 10 jours de retard : 2,00 €
	for _, bad := range []lending.Money{lending.Cents(0), lending.Cents(-5), lending.Cents(201)} {
		if _, err := ada.PayFines(bad, now); !errors.Is(err, lending.ErrBadPayment) {
			t.Errorf("paiement %s : %v", bad, err)
		}
	}
	rest, err := ada.PayFines(lending.Cents(150), now)
	if err != nil || rest != lending.Cents(50) {
		t.Fatalf("reste %s, %v", rest, err)
	}
	ev := ada.PullEvents()
	if last := ev[len(ev)-1]; last.Name != "PénalitéPayée" || last.Data != "1,50 €" {
		t.Fatalf("événement : %+v", last)
	}
}

func TestEquality(t *testing.T) {
	if lending.Cents(100) != lending.Cents(100) {
		t.Fatal("deux valeurs égales doivent être ==")
	}
	if lending.NewMember("ada") == lending.NewMember("ada") {
		t.Fatal("deux pointeurs vers deux entités distinctes ne sont pas ==, même identifiant")
	}
}

func TestRestoreRejectsCorruptState(t *testing.T) {
	for _, st := range []lending.MemberState{
		{ID: ""},
		{ID: "ada", FinesCents: -1},
		{ID: "ada", Loans: make([]lending.LoanState, 4)},
		{ID: "ada", Loans: []lending.LoanState{{Copy: "42"}}},
		{ID: "ada", Loans: []lending.LoanState{{Copy: "EX-1", Renewals: 2}}},
	} {
		if _, err := lending.RestoreMember(st); !errors.Is(err, lending.ErrCorruptState) {
			t.Errorf("%+v accepté : %v", st, err)
		}
	}
}

func TestSnapshotRestoreRoundTrip(t *testing.T) {
	ada := lending.NewMember("ada")
	ada.Borrow(cp("EX-1"), now)
	ada.Renew(cp("EX-1"), now)
	back, err := lending.RestoreMember(ada.Snapshot())
	if err != nil {
		t.Fatal(err)
	}
	// L'invariant « une prolongation au plus » a survécu à l'aller-retour.
	if err := back.Renew(cp("EX-1"), now); !errors.Is(err, lending.ErrRenewalDenied) {
		t.Fatalf("err = %v", err)
	}
}

func TestOptimisticLocking(t *testing.T) {
	repo := lending.NewMemRepo()
	repo.Save(t.Context(), lending.NewMember("ada"), 0)

	// Deux chargements de la même version, puis deux enregistrements.
	a, va, _ := repo.Load(t.Context(), "ada")
	b, vb, _ := repo.Load(t.Context(), "ada")
	a.Borrow(cp("EX-1"), now)
	b.Borrow(cp("EX-2"), now)
	if err := repo.Save(t.Context(), a, va); err != nil {
		t.Fatal(err)
	}
	if err := repo.Save(t.Context(), b, vb); !errors.Is(err, lending.ErrConcurrentModification) {
		t.Fatalf("second enregistrement : %v", err)
	}
}

func TestServiceConcurrentBorrowsWithRetry(t *testing.T) {
	repo := lending.NewMemRepo()
	repo.Save(t.Context(), lending.NewMember("ada"), 0)
	svc := lending.NewService(repo, func() time.Time { return now })
	var wg sync.WaitGroup
	var mu sync.Mutex
	results := map[string]int{}
	for i := range 5 {
		wg.Go(func() {
			var err error
			for range 50 { // réessayer sur conflit de version : recharger, réappliquer
				err = svc.Borrow(t.Context(), "ada", cp("EX-"+string(rune('1'+i))))
				if !errors.Is(err, lending.ErrConcurrentModification) {
					break
				}
				mu.Lock()
				results["conflits"]++
				mu.Unlock()
			}
			mu.Lock()
			if err == nil {
				results["ok"]++
			} else if errors.Is(err, lending.ErrTooManyLoans) {
				results["refus"]++
			}
			mu.Unlock()
		})
	}
	wg.Wait()
	if results["ok"] != 3 || results["refus"] != 2 || len(svc.Events()) != 3 {
		t.Fatalf("%v, %d événements : l'invariant des 3 emprunts doit tenir", results, len(svc.Events()))
	}
	t.Logf("%v", results)
}
```

`go test -race ./...` passe.

1. **Trois solutions.** (a) Le *repository* dans `lending` : simple, mais le paquet du domaine importe
   alors `database/sql` — on perd la séparation de la leçon 1. (b) Un état exporté + `RestoreMember` qui
   revérifie : un autre paquet peut fabriquer un état, mais **pas** un `Member` invalide, puisque la
   reconstruction valide. (c) Champs exportés : plus aucune protection. La (b) est le compromis courant.
2. **`RestoreMember` revérifie** (identifiant, pénalités, nombre d'emprunts, cotes, prolongations) : une
   ligne corrompue en base ne produit pas un agrégat incohérent, mais une erreur claire. Le test
   d'aller-retour montre que l'invariant « une prolongation au plus » survit à la persistance.
3. **Le dépôt stocke des états**, pas des `*Member` : partager le pointeur entre deux chargements
   ferait que deux requêtes modifient le **même** objet en mémoire, sans transaction ni conflit détecté —
   un comportement qu'aucune base n'a, donc des tests qui mentent.
4. **Verrou optimiste** : `Save` échoue si la version a changé depuis le chargement. Mesuré : cinq emprunts
   concurrents du même adhérent produisent de 0 à 3 conflits selon les exécutions ; chaque conflit est
   résolu en rechargeant et en réappliquant la règle ; à la fin, toujours 3 emprunts réussis, 2 refusés
   (`ErrTooManyLoans`) et 3 événements. Sans version, les cinq auraient pu réussir à partir du même état
   initial (la mise à jour perdue de N8 L4). En PostgreSQL : `UPDATE … SET version = version + 1 WHERE id = $1
   AND version = $2` puis `RowsAffected() == 0` → conflit.

## Défi — Deux contextes bornés

```go
// Fichier : catalog/catalog.go
// Le contexte « catalogue » : des ŒUVRES. Il ignore tout des exemplaires et des emprunts.
package catalog

import "errors"

type Work struct {
	ISBN, Title string
	Authors     []string
	Withdrawn   bool
}

type Catalog struct{ works map[string]*Work }

func New(works ...Work) *Catalog {
	c := &Catalog{works: map[string]*Work{}}
	for i := range works {
		c.works[works[i].ISBN] = &works[i]
	}
	return c
}

var ErrUnknownWork = errors.New("œuvre inconnue")

func (c *Catalog) Title(isbn string) (string, error) {
	w, ok := c.works[isbn]
	if !ok {
		return "", ErrUnknownWork
	}
	return w.Title, nil
}

// WorkWithdrawn : l'événement publié par le catalogue (le prêt y réagit, à terme).
type WorkWithdrawn struct{ ISBN string }

func (c *Catalog) Withdraw(isbn string) (WorkWithdrawn, error) {
	w, ok := c.works[isbn]
	if !ok {
		return WorkWithdrawn{}, ErrUnknownWork
	}
	w.Withdrawn = true
	return WorkWithdrawn{ISBN: isbn}, nil
}
```

```go
// Fichier : display/display.go
// Lecture : assemble les deux contextes pour l'affichage. Seul paquet qui importe les deux.
package display

import (
	"fmt"

	"exemple/catalog"
)

// LoanLine reçoit du contexte prêt seulement un adhérent et un ISBN.
func LoanLine(cat *catalog.Catalog, member, isbn string) string {
	title, err := cat.Title(isbn)
	if err != nil {
		title = "œuvre inconnue (" + isbn + ")"
	}
	return fmt.Sprintf("%s a emprunté « %s »", member, title)
}
```

```go
// Fichier : arch_test.go
// Test d'architecture : les deux contextes bornés ne s'importent pas l'un l'autre.
package arch

import (
	"os/exec"
	"strings"
	"testing"
)

func imports(t *testing.T, pkg string) []string {
	t.Helper()
	out, err := exec.Command("go", "list", "-deps", "-f", "{{.ImportPath}}", pkg).Output()
	if err != nil {
		t.Fatal(err)
	}
	return strings.Fields(string(out))
}

func TestBoundedContextsIndependent(t *testing.T) {
	for _, c := range []struct{ pkg, forbidden string }{
		{"./lending", "exemple/catalog"},
		{"./catalog", "exemple/lending"},
	} {
		for _, dep := range imports(t, c.pkg) {
			if dep == c.forbidden {
				t.Errorf("%s dépend de %s : les contextes bornés doivent rester indépendants", c.pkg, dep)
			}
		}
	}
}
```

- Ni `lending` ni `catalog` n'importent l'autre ; le test d'architecture le **vérifie** à chaque
  `go test` (il échoue si quelqu'un ajoute l'import).
- L'affichage « Ada a emprunté *La Peste* » est assemblé par `display`, côté lecture, qui importe le
  catalogue et reçoit du prêt seulement un adhérent et un ISBN.
- **Retrait d'une œuvre** : le catalogue publie `WorkWithdrawn{ISBN}`. Le contexte prêt y réagit **à
  terme** : il marque ses exemplaires de cette œuvre « à retirer » (plus de nouvel emprunt), laisse les
  emprunts en cours se terminer normalement, et retire l'exemplaire à son retour. Il n'y a aucune raison
  d'annuler un emprunt en cours parce qu'une notice de catalogue a changé : la réaction est une décision
  du contexte prêt, pas du catalogue. La leçon 8 montre comment acheminer cet événement de façon fiable.

## Quiz

1. Les mots des experts du métier, utilisés un pour un dans le code ; ils rendent les règles vérifiables
   par ceux qui les connaissent.
2. Une valeur n'a pas d'identité et est immuable ; une entité a une identité qui persiste.
3. Un champ non exporté et un constructeur qui valide (en gardant un œil sur la valeur zéro).
4. Un groupe d'objets cohérent ; la racine est la seule entrée, et garantit les invariants.
5. Parce que ses invariants doivent être vrais à chaque enregistrement : on le charge et l'enregistre
   entier, atomiquement.
6. Par la visibilité du paquet : champs non exportés, modifications seulement par les méthodes.
7. Pour ne pas charger ni verrouiller l'autre agrégat, et garder des frontières de transaction petites.
8. Un fait du métier, au passé ; l'agrégat l'enregistre, le service le publie.
9. Une frontière où un modèle et ses mots ont un sens unique ; catalogue (œuvres) et prêt (exemplaires).
10. Pour un domaine pauvre en règles (CRUD, intégration technique).
