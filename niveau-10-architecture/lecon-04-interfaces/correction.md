# Leçon 4 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Casser la conformité

Avec la ligne de vérification, le compilateur signale **deux** endroits :

```
./main.go:87:20: cannot use (*Store)(nil) (value of type *Store) as overdueLister value in variable declaration: *Store does not implement overdueLister (missing method Overdue)
./main.go:123:21: cannot use store (variable of type *Store) as overdueLister value in argument to RemindOverdue: *Store does not implement overdueLister (missing method Overdue)
```

Sans elle, seul l'appel de `RemindOverdue` est signalé. Ici, la ligne est donc redondante : l'usage réel
vérifiait déjà la conformité. Elle devient utile quand **aucun** usage dans le même programme ne la
vérifie : un type destiné à être utilisé par un autre module, un plug-in, un type enregistré dans un
registre par réflexion, ou une implémentation dont le seul appelant est dans un paquet de test.

## E2 — Le faux partiel

```
--- FAIL: TestReportPartial (0.00s)
panic: runtime error: invalid memory address or nil pointer dereference [recovered, repanicked]
…
ex.(*partialFake).CountByMember(…)
```

Le message parle d'un pointeur nil, pas d'une méthode « non simulée » : il faut lire la pile pour
comprendre que le faux n'implémentait pas `CountByMember`. La correction :

```go
type countFunc func(context.Context) (map[string]int, error)

func (f countFunc) CountByMember(ctx context.Context) (map[string]int, error) { return f(ctx) }

r, err := Report(ctx, countFunc(func(context.Context) (map[string]int, error) {
	return map[string]int{"ada": 2}, nil
}))
```

Possible parce que `Report` ne demande que `loanCounter` : le faux implémente exactement ce qui est utilisé.

## E3 — `io` comme modèle

```go
func CountLines(r io.Reader) (int, error) {
	sc := bufio.NewScanner(r)
	n := 0
	for sc.Scan() {
		n++
	}
	return n, sc.Err()
}
```

`*os.File`, `*strings.Reader`, le corps d'une réponse HTTP (un type interne de `net/http`),
`*io.LimitedReader`… et tout type futur qui aura une méthode `Read`. La fonction accepte des types qui
n'existaient pas quand elle a été écrite, et qu'elle ne nomme nulle part. C'est l'effet d'une interface
d'**une** méthode, déclarée par le consommateur (ici, par la bibliothèque standard, une fois pour toutes).

## E4 — Rendre une interface

L'appelant qui a besoin de `Remove` ne peut plus l'appeler : `StoreInterface` ne l'a pas. Il doit
soit faire une assertion de type (`s.(*Store).Remove(…)`, fragile), soit obtenir qu'on ajoute `Remove` à
l'interface. Et ajouter une méthode à une interface **exportée** casse toutes les implémentations tierces
(un faux de test, un cache, une autre équipe) : elles ne la satisfont plus. Rendre `*Store` laisse chaque
consommateur choisir sa petite interface, et le fournisseur libre d'ajouter des méthodes.

## E5 — Pollution

1. Rendre `*UserService` (struct exportée) depuis le constructeur ; supprimer l'interface de 14 méthodes.
2. Dans chaque paquet consommateur, déclarer l'interface d'une ou deux méthodes qu'il utilise.
3. Supprimer le *mock* généré ; écrire dans chaque test un faux de quelques lignes (adaptateur fonction ou
   petite struct).
4. Les tests deviennent plus courts, plus lisibles (le faux montre ce qui compte pour le test), et
   n'échouent plus quand on ajoute une méthode au service.
5. Si un besoin d'interface « globale » réapparaît (plusieurs implémentations réelles), l'extraire à ce
   moment-là, avec seulement les méthodes réellement partagées.

## Exercice intermédiaire — Découvrir les interfaces du service de prêts

```go
// Fichier : loan/loan.go
// Package loan : les types partagés du domaine.
package loan

import "time"

type Loan struct {
	Member, Book string
	Due          time.Time
}

type EventKind string

const (
	Borrowed EventKind = "emprunt"
	Returned EventKind = "retour"
)

type Event struct {
	Kind EventKind
	At   time.Time
}
```

```go
// Fichier : memstore/memstore.go
// Package memstore : un stockage qui ne sait rien de ses consommateurs.
package memstore

import (
	"context"
	"sync"
	"time"

	"exemple/loan"
)

type Store struct {
	mu     sync.Mutex
	loans  []loan.Loan
	events []loan.Event
}

func New() *Store { return &Store{} }

func (s *Store) Add(_ context.Context, l loan.Loan, at time.Time) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.loans = append(s.loans, l)
	s.events = append(s.events, loan.Event{Kind: loan.Borrowed, At: at})
}

func (s *Store) Overdue(_ context.Context, now time.Time) ([]loan.Loan, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	var out []loan.Loan
	for _, l := range s.loans {
		if now.After(l.Due) {
			out = append(out, l)
		}
	}
	return out, nil
}

func (s *Store) Events(_ context.Context, from, to time.Time) ([]loan.Event, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	var out []loan.Event
	for _, e := range s.events {
		if !e.At.Before(from) && e.At.Before(to) {
			out = append(out, e)
		}
	}
	return out, nil
}
```

```go
// Fichier : jobs/reminders.go
// Package jobs : la tâche de nuit. Elle déclare ses besoins, et n'importe pas memstore.
package jobs

import (
	"context"
	"fmt"
	"time"

	"exemple/loan"
)

type overdueSource interface {
	Overdue(ctx context.Context, now time.Time) ([]loan.Loan, error)
}

// SendReminders envoie UN rappel par adhérent, listant ses livres en retard.
func SendReminders(ctx context.Context, src overdueSource, now time.Time, send func(to, msg string) error) (int, error) {
	loans, err := src.Overdue(ctx, now)
	if err != nil {
		return 0, err
	}
	byMember := map[string][]string{}
	var order []string
	for _, l := range loans {
		if _, seen := byMember[l.Member]; !seen {
			order = append(order, l.Member)
		}
		byMember[l.Member] = append(byMember[l.Member], l.Book)
	}
	for _, m := range order {
		if err := send(m, fmt.Sprintf("en retard : %v", byMember[m])); err != nil {
			return 0, fmt.Errorf("rappel à %s : %w", m, err)
		}
	}
	return len(order), nil
}
```

```go
// Fichier : jobs/reminders_test.go
package jobs

import (
	"context"
	"testing"
	"time"

	"exemple/loan"
)

type overdueFunc func(context.Context, time.Time) ([]loan.Loan, error)

func (f overdueFunc) Overdue(ctx context.Context, now time.Time) ([]loan.Loan, error) {
	return f(ctx, now)
}

func TestOneReminderPerMember(t *testing.T) {
	src := overdueFunc(func(context.Context, time.Time) ([]loan.Loan, error) {
		return []loan.Loan{{Member: "ada", Book: "a"}, {Member: "bob", Book: "b"}, {Member: "ada", Book: "c"}}, nil
	})
	var sent []string
	n, err := SendReminders(t.Context(), src, time.Now(), func(to, msg string) error {
		sent = append(sent, to+" "+msg)
		return nil
	})
	if err != nil || n != 2 || sent[0] != "ada en retard : [a c]" || sent[1] != "bob en retard : [b]" {
		t.Fatalf("%d %v %v", n, sent, err)
	}
}
```

```go
// Fichier : reports/monthly.go
// Package reports : le rapport mensuel. Lui aussi déclare ses propres besoins.
package reports

import (
	"context"
	"time"

	"exemple/loan"
)

type eventSource interface {
	Events(ctx context.Context, from, to time.Time) ([]loan.Event, error)
}

type Day struct{ Borrowed, Returned int }

// Monthly rend, pour chaque jour du mois (1 à 31), le nombre d'emprunts et de retours.
func Monthly(ctx context.Context, src eventSource, year int, month time.Month) (map[int]Day, error) {
	from := time.Date(year, month, 1, 0, 0, 0, 0, time.UTC)
	events, err := src.Events(ctx, from, from.AddDate(0, 1, 0))
	if err != nil {
		return nil, err
	}
	out := map[int]Day{}
	for _, e := range events {
		d := out[e.At.Day()]
		switch e.Kind {
		case loan.Borrowed:
			d.Borrowed++
		case loan.Returned:
			d.Returned++
		}
		out[e.At.Day()] = d
	}
	return out, nil
}
```

```go
// Fichier : reports/monthly_test.go
package reports

import (
	"context"
	"testing"
	"time"

	"exemple/loan"
)

type eventsFunc func(context.Context, time.Time, time.Time) ([]loan.Event, error)

func (f eventsFunc) Events(ctx context.Context, from, to time.Time) ([]loan.Event, error) {
	return f(ctx, from, to)
}

func TestMonthly(t *testing.T) {
	at := func(d int) time.Time { return time.Date(2026, 3, d, 9, 0, 0, 0, time.UTC) }
	src := eventsFunc(func(_ context.Context, from, to time.Time) ([]loan.Event, error) {
		if from.Month() != time.March || to.Month() != time.April {
			t.Errorf("intervalle %v → %v", from, to)
		}
		return []loan.Event{{Kind: loan.Borrowed, At: at(1)}, {Kind: loan.Borrowed, At: at(1)}, {Kind: loan.Returned, At: at(31)}}, nil
	})
	got, err := Monthly(t.Context(), src, 2026, time.March)
	if err != nil || got[1] != (Day{2, 0}) || got[31] != (Day{0, 1}) {
		t.Fatalf("%v %v", got, err)
	}
}
```

```go
// Fichier : wiring_test.go
// Le vrai stockage satisfait les interfaces des deux consommateurs, sans les connaître.
package wiring

import (
	"testing"
	"time"

	"exemple/jobs"
	"exemple/loan"
	"exemple/memstore"
	"exemple/reports"
)

func TestRealStoreFitsBoth(t *testing.T) {
	s := memstore.New()
	day := time.Date(2026, 3, 2, 10, 0, 0, 0, time.UTC)
	s.Add(t.Context(), loan.Loan{Member: "ada", Book: "a", Due: day}, day.Add(-21*24*time.Hour))
	n, err := jobs.SendReminders(t.Context(), s, day.Add(time.Hour), func(string, string) error { return nil })
	if err != nil || n != 1 {
		t.Fatal(n, err)
	}
	if _, err := reports.Monthly(t.Context(), s, 2026, time.February); err != nil {
		t.Fatal(err)
	}
}
```

`go list` le confirme :

```
exemple/jobs → context exemple/loan fmt time
exemple/reports → context exemple/loan time
```

Ni la tâche ni le rapport n'importent `memstore` (ni un futur `pgstore`) : ils ne dépendent que du domaine.
Le test `TestRealStoreFitsBoth` montre que le vrai stockage satisfait les deux interfaces sans les
connaître.

Bilan : **deux** interfaces d'**une** méthode chacune ; deux faux d'une ligne. Une unique interface
`LoanStore` couvrant tous les besoins aurait eu toutes les méthodes du stockage (une dizaine), et chaque
test aurait dû soit toutes les implémenter, soit tomber dans le piège du faux partiel.

## Défi — Décorer par composition

```go
// Fichier : deco/deco.go
// Package deco : des décorateurs qui satisfont la MÊME petite interface qu'ils enveloppent.
package deco

import (
	"context"
	"sync"
	"sync/atomic"
	"time"

	"exemple/loan"
)

type OverdueLister interface {
	Overdue(ctx context.Context, now time.Time) ([]loan.Loan, error)
}

// Cached garde le résultat ttl (horloge injectée).
type Cached struct {
	next  OverdueLister
	ttl   time.Duration
	clock func() time.Time
	mu    sync.Mutex
	at    time.Time
	value []loan.Loan
	ok    bool
}

func NewCached(next OverdueLister, ttl time.Duration, clock func() time.Time) *Cached {
	return &Cached{next: next, ttl: ttl, clock: clock}
}

func (c *Cached) Overdue(ctx context.Context, now time.Time) ([]loan.Loan, error) {
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.ok && c.clock().Sub(c.at) < c.ttl {
		return c.value, nil
	}
	v, err := c.next.Overdue(ctx, now)
	if err != nil {
		return nil, err // une erreur n'est jamais mise en cache
	}
	c.value, c.at, c.ok = v, c.clock(), true
	return v, nil
}

// Counted compte les appels et les erreurs.
type Counted struct {
	next          OverdueLister
	Calls, Errors atomic.Int64
}

func NewCounted(next OverdueLister) *Counted { return &Counted{next: next} }

func (c *Counted) Overdue(ctx context.Context, now time.Time) ([]loan.Loan, error) {
	c.Calls.Add(1)
	v, err := c.next.Overdue(ctx, now)
	if err != nil {
		c.Errors.Add(1)
	}
	return v, err
}
```

```go
// Fichier : deco/deco_test.go
package deco

import (
	"context"
	"errors"
	"testing"
	"time"

	"exemple/loan"
)

type overdueFunc func(context.Context, time.Time) ([]loan.Loan, error)

func (f overdueFunc) Overdue(ctx context.Context, now time.Time) ([]loan.Loan, error) {
	return f(ctx, now)
}

func TestStacking(t *testing.T) {
	clock := time.Date(2026, 3, 1, 0, 0, 0, 0, time.UTC)
	tick := func() time.Time { return clock }
	store := overdueFunc(func(context.Context, time.Time) ([]loan.Loan, error) { return []loan.Loan{{Book: "a"}}, nil })

	// Ordre 1 : compteur → cache → stockage : compte les appels REÇUS.
	received := NewCounted(NewCached(store, time.Minute, tick))
	// Ordre 2 : cache → compteur → stockage : compte les appels qui atteignent le STOCKAGE.
	atStore := NewCounted(store)
	cached := NewCached(atStore, time.Minute, tick)

	for range 5 {
		received.Overdue(t.Context(), clock)
		cached.Overdue(t.Context(), clock)
	}
	clock = clock.Add(2 * time.Minute) // le cache expire
	received.Overdue(t.Context(), clock)
	cached.Overdue(t.Context(), clock)

	if received.Calls.Load() != 6 || atStore.Calls.Load() != 2 {
		t.Fatalf("reçus=%d au stockage=%d", received.Calls.Load(), atStore.Calls.Load())
	}
}

func TestErrorsNotCached(t *testing.T) {
	fail := true
	src := NewCounted(overdueFunc(func(context.Context, time.Time) ([]loan.Loan, error) {
		if fail {
			return nil, errors.New("panne")
		}
		return nil, nil
	}))
	c := NewCached(src, time.Hour, time.Now)
	c.Overdue(t.Context(), time.Now())
	fail = false
	if _, err := c.Overdue(t.Context(), time.Now()); err != nil {
		t.Fatal("l'erreur a été mise en cache")
	}
	if src.Errors.Load() != 1 || src.Calls.Load() != 2 {
		t.Fatalf("erreurs=%d appels=%d", src.Errors.Load(), src.Calls.Load())
	}
}
```

- **Ordre d'empilement** : `Counted(Cached(stockage))` compte les appels **reçus** (6 sur 6) ;
  `Cached(Counted(stockage))` compte les appels qui **atteignent le stockage** (2 : le premier, puis
  après expiration du cache). Les deux mesures sont utiles (charge reçue, charge épargnée au stockage) ;
  un décorateur mesure ce qui passe **à travers lui**.
- **Les erreurs ne sont pas mises en cache** : une panne passagère ne doit pas être servie pendant une
  heure (`TestErrorsNotCached`).
- **Pourquoi c'est simple** : avec une interface d'une méthode, chaque décorateur implémente une méthode
  et délègue. Avec une interface de quinze, chaque décorateur doit en implémenter quinze, dont quatorze
  qui ne font que déléguer — ou embarquer l'interface et retomber dans le piège du faux partiel (une
  méthode oubliée appelle silencieusement l'objet enveloppé sans passer par le cache ni le compteur).

## Quiz

1. Le consommateur ; c'est possible parce que la satisfaction d'une interface est implicite.
2. Une à trois méthodes : facile à implémenter, à simuler, à décorer, à comprendre.
3. Les paramètres sont des interfaces (flexibilité pour l'appelant), les retours des structs (toutes les
   méthodes disponibles, fournisseur libre d'évoluer).
4. `error`, une fabrique qui choisit vraiment entre plusieurs implémentations, ou pour cacher un type non
   exporté.
5. Quand un besoin réel apparaît : un test, une seconde implémentation, un découplage de paquets.
6. Une interface par struct, `IFoo`, déclarée chez le fournisseur, beaucoup de méthodes, *mocks* générés
   énormes.
7. Un type fonction qui implémente une interface d'une méthode en s'appelant lui-même ;
   `http.HandlerFunc`.
8. L'interface embarquée est nil : toute méthode non redéfinie panique, loin de la cause.
9. À vérifier à la compilation qu'un type satisfait une interface quand aucun usage ne le vérifie.
10. Quand la dépendance n'est qu'une seule opération (`send func(to, msg string)`).
