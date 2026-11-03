# Leçon 5 — Hexagonale et clean

> « Architecture hexagonale », « ports et adaptateurs », « clean architecture », « architecture en
> oignon » : plusieurs noms pour une même idée, que les leçons 1 à 4 ont déjà appliquée sans la nommer —
> **le domaine au centre, la technologie autour, les dépendances vers l'intérieur**. Cette leçon donne le
> vocabulaire, montre ce que ces architectures ajoutent (des ports d'**entrée**, plusieurs adaptateurs
> pour un même cœur) et, surtout, mesure leur coût : appliquées à la lettre en Go, elles produisent
> souvent plus de cérémonie que de valeur.

## Objectifs

1. Connaître le vocabulaire : cœur, ports d'entrée et de sortie, adaptateurs, règle de dépendance.
2. Brancher un même cœur sur plusieurs adaptateurs (HTTP, CLI ; mémoire, console).
3. Tester chaque partie seule : le cœur avec des faux des ports de sortie, un adaptateur d'entrée avec un
   faux du port d'entrée.
4. Mesurer le coût du découpage, et savoir quand il ne vaut pas.

## Explication

### Le vocabulaire

```
          adaptateurs d'ENTRÉE                         adaptateurs de SORTIE
     (pilotent le cœur : driving)                   (pilotés par le cœur : driven)

   HTTP ──┐                                               ┌── mémoire / PostgreSQL
          ├──► port d'entrée ──► CŒUR ──► ports de sortie ─┤
   CLI  ──┘     (Lending)       (règles)  (Members,        └── console / courriel
                                           Notifier)
```

- Le **cœur** contient les règles et les cas d'usage. Il ne dépend de rien d'extérieur.
- Un **port** est une interface à la frontière du cœur :
  - port **d'entrée** (*driving*) : ce que le cœur **offre** (`Lending.Borrow`) ;
  - port **de sortie** (*driven*) : ce dont le cœur **a besoin** (`Members`, `Notifier`).
- Un **adaptateur** branche une technologie sur un port : HTTP et CLI appellent le port d'entrée ; la
  mémoire, PostgreSQL, la console, un serveur de courriel implémentent les ports de sortie.
- **Règle de dépendance** : les dépendances pointent toujours vers le cœur. Exactement la leçon 1.

La *clean architecture* dit la même chose avec des cercles concentriques (entités, cas d'usage,
adaptateurs d'interface, frameworks) et la même règle. Les différences entre ces écoles sont surtout de
vocabulaire.

### Ce qui est propre à Go

- Les ports de sortie sont des **interfaces déclarées par le cœur** (côté consommateur, leçon 4). Les
  adaptateurs de sortie les satisfont **implicitement** : dans l'exemple, `memory` et `console`
  n'importent même pas `core` (mesuré avec `go list`). Dans d'autres langages, ils devraient déclarer
  `implements Members`.
- Le port d'**entrée** est la seule interface que le cœur exporte « pour les autres ». Elle est utile
  quand on veut tester un adaptateur d'entrée **sans** le cœur (un faux de `Lending` dans le test HTTP)
  ou brancher plusieurs implémentations du cas d'usage. Sinon, l'adaptateur d'entrée peut très bien
  dépendre directement de la struct de service (leçon 2) : c'est l'écart le plus courant, et le plus
  raisonnable, avec la version « pure » de l'architecture.

### Ce que ça coûte

Mesuré sur l'exemple, sans compter les tests : **6 fichiers dans 6 paquets, 156 lignes**, pour une règle
(« 3 emprunts au plus ») et une notification. La version de la leçon 1 faisait la même chose avec trois
paquets. Les coûts réels d'une architecture hexagonale appliquée à la lettre :

| Coût | Manifestation |
|---|---|
| Indirection | Pour suivre un appel, on traverse adaptateur → port → cœur → port → adaptateur ; l'éditeur saute sur des interfaces, pas sur le code. |
| Conversions | Chaque frontière a ses types (DTO, domaine, lignes) ; leur conversion est du code sans valeur métier. |
| Paquets et dossiers | Un paquet par adaptateur et par port multiplie les importations et les noms (`adapters/httpin`, `ports/out`…). |
| Abstractions prématurées | Des ports pour des dépendances qui n'auront jamais qu'une implémentation. |
| Courbe d'apprentissage | Le vocabulaire et les conventions de l'école choisie, à transmettre à chaque arrivant. |

Et les bénéfices réels :

| Bénéfice | Quand il se matérialise |
|---|---|
| Cœur testable sans infrastructure | Dès qu'il y a des règles non triviales. |
| Plusieurs portes d'entrée | HTTP + CLI + consommateur de messages + tâche planifiée sur le même cas d'usage. |
| Changer une technologie | Remplacer un fournisseur de courriel, migrer de base : un adaptateur à réécrire. |
| Frontières claires | Grandes équipes, longue durée de vie. |

### Quand ne pas le faire

- **Un service CRUD** sans règles : les ports ne protègent rien ; tout le code est de la conversion.
- **Une seule porte d'entrée et une seule technologie de sortie, pour toujours** : la séparation de la
  leçon 1 (domaine + adaptateurs, sans port d'entrée) suffit.
- **Une petite équipe sur un service jeune** : commencer simple ; les frontières se tracent quand on
  sait où elles passent.
- **Recopier une arborescence type** (`domain/`, `application/`, `infrastructure/`, `ports/in`, `ports/out`,
  `adapters/…`) trouvée ailleurs : en Go, on organise par fonctionnalité (leçon 2), et on extrait un
  paquet quand il a une raison d'exister.

Le bon usage en Go est presque toujours **partiel** : un domaine sans dépendance technique, des
interfaces de sortie déclarées par le domaine (là où un test ou une seconde implémentation les
justifient), des adaptateurs qui dépendent du service concret. C'est l'essentiel du bénéfice pour une
fraction du coût.

## Exemple

Un cœur, deux adaptateurs d'entrée (CLI, HTTP), deux de sortie (mémoire, console). La fonction d'exemple
de test assemble le tout et pilote le **même** cœur par les deux entrées ; sa sortie est vérifiée par
`go test`.

```go
// Fichier : core/ports.go
// Package core : le domaine et ses PORTS. Il ne dépend d'aucun adaptateur.
package core

import (
	"context"
	"time"
)

// Port d'ENTRÉE (driving) : ce que le cœur offre au monde extérieur.
type Lending interface {
	Borrow(ctx context.Context, member, book string) (time.Time, error)
}

// Ports de SORTIE (driven) : ce dont le cœur a besoin du monde extérieur.
type Members interface {
	Loans(ctx context.Context, member string) (int, error)
	AddLoan(ctx context.Context, member, book string, due time.Time) error
}

type Notifier interface {
	Notify(ctx context.Context, member, msg string) error
}
```

```go
// Fichier : core/service.go
package core

import (
	"context"
	"errors"
	"fmt"
	"time"
)

var ErrTooManyLoans = errors.New("nombre maximal d'emprunts atteint")

const MaxLoans = 3

type service struct {
	members Members
	notify  Notifier
	now     func() time.Time
}

// New rend le cœur, vu à travers son port d'entrée.
func New(m Members, n Notifier, now func() time.Time) Lending {
	return &service{members: m, notify: n, now: now}
}

func (s *service) Borrow(ctx context.Context, member, book string) (time.Time, error) {
	n, err := s.members.Loans(ctx, member)
	if err != nil {
		return time.Time{}, err
	}
	if n >= MaxLoans {
		return time.Time{}, ErrTooManyLoans
	}
	due := s.now().Add(21 * 24 * time.Hour)
	if err := s.members.AddLoan(ctx, member, book, due); err != nil {
		return time.Time{}, err
	}
	// La notification est un effet secondaire : son échec ne remet pas l'emprunt en cause.
	_ = s.notify.Notify(ctx, member, fmt.Sprintf("« %s » à rendre le %s", book, due.Format(time.DateOnly)))
	return due, nil
}
```

```go
// Fichier : core/service_test.go
package core

import (
	"context"
	"errors"
	"testing"
	"time"
)

type fakeMembers map[string]int

func (f fakeMembers) Loans(_ context.Context, m string) (int, error) { return f[m], nil }
func (f fakeMembers) AddLoan(_ context.Context, m, _ string, _ time.Time) error {
	f[m]++
	return nil
}

type notifyFunc func(context.Context, string, string) error

func (f notifyFunc) Notify(ctx context.Context, m, msg string) error { return f(ctx, m, msg) }

func TestBorrow(t *testing.T) {
	now := time.Date(2026, 3, 1, 0, 0, 0, 0, time.UTC)
	var sent []string
	l := New(fakeMembers{"bob": 3}, notifyFunc(func(_ context.Context, m, msg string) error {
		sent = append(sent, m+" : "+msg)
		return errors.New("serveur de courriel en panne") // ne doit pas faire échouer l'emprunt
	}), func() time.Time { return now })
	due, err := l.Borrow(t.Context(), "ada", "La Peste")
	if err != nil || !due.Equal(now.Add(21*24*time.Hour)) || len(sent) != 1 {
		t.Fatalf("%v %v %v", due, err, sent)
	}
	if _, err := l.Borrow(t.Context(), "bob", "x"); !errors.Is(err, ErrTooManyLoans) {
		t.Fatalf("bob : %v", err)
	}
}
```

```go
// Fichier : adapters/memory/members.go
// Adaptateur de SORTIE : le port Members, en mémoire.
package memory

import (
	"context"
	"sync"
	"time"
)

type Members struct {
	mu    sync.Mutex
	loans map[string]int
}

func NewMembers() *Members { return &Members{loans: map[string]int{}} }

func (m *Members) Loans(_ context.Context, member string) (int, error) {
	m.mu.Lock()
	defer m.mu.Unlock()
	return m.loans[member], nil
}

func (m *Members) AddLoan(_ context.Context, member, _ string, _ time.Time) error {
	m.mu.Lock()
	defer m.mu.Unlock()
	m.loans[member]++
	return nil
}
```

```go
// Fichier : adapters/console/notifier.go
// Adaptateur de SORTIE : le port Notifier, vers un io.Writer.
package console

import (
	"context"
	"fmt"
	"io"
)

type Notifier struct{ Out io.Writer }

func (n Notifier) Notify(_ context.Context, member, msg string) error {
	_, err := fmt.Fprintf(n.Out, "[notification] %s : %s\n", member, msg)
	return err
}
```

```go
// Fichier : adapters/httpin/handler.go
// Adaptateur d'ENTRÉE : HTTP → port Lending.
package httpin

import (
	"encoding/json"
	"errors"
	"net/http"

	"exemple/core"
)

func Handler(l core.Lending) http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("POST /members/{id}/loans/{book}", func(w http.ResponseWriter, r *http.Request) {
		due, err := l.Borrow(r.Context(), r.PathValue("id"), r.PathValue("book"))
		switch {
		case errors.Is(err, core.ErrTooManyLoans):
			http.Error(w, err.Error(), http.StatusConflict)
		case err != nil:
			http.Error(w, "erreur interne", http.StatusInternalServerError)
		default:
			w.WriteHeader(http.StatusCreated)
			json.NewEncoder(w).Encode(map[string]any{"due": due})
		}
	})
	return mux
}
```

```go
// Fichier : adapters/httpin/handler_test.go
// L'adaptateur d'entrée se teste avec un FAUX du port d'entrée : sans le cœur.
package httpin

import (
	"context"
	"net/http/httptest"
	"testing"
	"time"

	"exemple/core"
)

type lendingFunc func(context.Context, string, string) (time.Time, error)

func (f lendingFunc) Borrow(ctx context.Context, m, b string) (time.Time, error) { return f(ctx, m, b) }

func TestStatuses(t *testing.T) {
	for _, c := range []struct {
		err  error
		want int
	}{{nil, 201}, {core.ErrTooManyLoans, 409}, {context.DeadlineExceeded, 500}} {
		h := Handler(lendingFunc(func(context.Context, string, string) (time.Time, error) { return time.Time{}, c.err }))
		rec := httptest.NewRecorder()
		h.ServeHTTP(rec, httptest.NewRequest("POST", "/members/ada/loans/x", nil))
		if rec.Code != c.want {
			t.Errorf("%v → %d, attendu %d", c.err, rec.Code, c.want)
		}
	}
}
```

```go
// Fichier : adapters/cliin/cli.go
// Adaptateur d'ENTRÉE : ligne de commande → port Lending.
package cliin

import (
	"context"
	"fmt"
	"io"

	"exemple/core"
)

func Run(ctx context.Context, l core.Lending, args []string, out io.Writer) int {
	if len(args) != 2 {
		fmt.Fprintln(out, "usage : emprunt <adhérent> <livre>")
		return 2
	}
	due, err := l.Borrow(ctx, args[0], args[1])
	if err != nil {
		fmt.Fprintln(out, "refusé :", err)
		return 1
	}
	fmt.Fprintln(out, "à rendre le", due.Format("2006-01-02"))
	return 0
}
```

```go
// Fichier : cmd/app/main_test.go
// L'assemblage : un cœur, deux adaptateurs de sortie, deux adaptateurs d'entrée.
package main

import (
	"context"
	"fmt"
	"net/http/httptest"
	"os"
	"time"

	"exemple/adapters/cliin"
	"exemple/adapters/console"
	"exemple/adapters/httpin"
	"exemple/adapters/memory"
	"exemple/core"
)

func Example() {
	now := func() time.Time { return time.Date(2026, 3, 1, 0, 0, 0, 0, time.UTC) }
	lending := core.New(memory.NewMembers(), console.Notifier{Out: os.Stdout}, now)

	// Par la ligne de commande…
	for range 2 {
		cliin.Run(context.Background(), lending, []string{"ada", "La Peste"}, os.Stdout)
	}
	// … et par HTTP : le MÊME cœur, le même état.
	h := httpin.Handler(lending)
	for range 2 {
		rec := httptest.NewRecorder()
		h.ServeHTTP(rec, httptest.NewRequest("POST", "/members/ada/loans/Vol%20de%20nuit", nil))
		fmt.Println("HTTP", rec.Code)
	}
	// Output:
	// [notification] ada : « La Peste » à rendre le 2026-03-22
	// à rendre le 2026-03-22
	// [notification] ada : « La Peste » à rendre le 2026-03-22
	// à rendre le 2026-03-22
	// [notification] ada : « Vol de nuit » à rendre le 2026-03-22
	// HTTP 201
	// HTTP 409
}
```
`go test -race ./...` passe. Les dépendances, mesurées avec `go list` :

```
exemple/adapters/cliin   → context exemple/core fmt io
exemple/adapters/console → context fmt io
exemple/adapters/httpin  → encoding/json errors exemple/core net/http
exemple/adapters/memory  → context sync time
exemple/core             → context errors fmt time
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `core.Lending` (port d'entrée) | Ce que le cœur offre ; `New` le rend, la struct `service` reste non exportée. |
| `core.Members`, `core.Notifier` (ports de sortie) | Déclarés par le cœur, selon ses besoins. |
| Échec de notification ignoré | Une règle métier explicite : l'emprunt est fait même si le courriel échoue (la leçon 8 fera mieux). |
| `memory`, `console` sans import de `core` | Satisfaction implicite des ports : propre à Go. |
| `httpin`, `cliin` importent `core` | Les adaptateurs d'entrée connaissent le port qu'ils appellent et ses erreurs. |
| `TestStatuses` avec `lendingFunc` | L'adaptateur HTTP testé sans le cœur, grâce au port d'entrée. |
| `TestBorrow` avec `fakeMembers` et `notifyFunc` | Le cœur testé sans infrastructure. |
| `Example` dans `cmd/app` | Le même cœur et le même état, pilotés par la CLI puis par HTTP (le 3ᵉ emprunt passe, le 4ᵉ est refusé). |

## Erreurs fréquentes

1. **Appliquer l'architecture complète à un CRUD.**
2. **Un port pour chaque dépendance**, y compris celles qui n'auront jamais qu'une implémentation.
3. **Une arborescence de dix dossiers** avant la première règle métier.
4. **Des conversions de types à chaque couche** quand les types sont identiques et le resteront.
5. **Un cœur qui importe un adaptateur** (pour une constante, une erreur) : la règle de dépendance est
   rompue.
6. **Débattre de vocabulaire** (hexagonale contre oignon contre clean) au lieu de vérifier le sens des
   imports.

## Bonnes pratiques Go

- Garder l'essentiel : domaine sans technologie, dépendances vers lui, ports de sortie déclarés par lui.
- Ajouter un port d'entrée quand un adaptateur doit être testé sans le cœur, ou quand il y a plusieurs
  implémentations du cas d'usage.
- Organiser par fonctionnalité ; un paquet d'adaptateur par technologie, quand il a du contenu.
- Vérifier la règle de dépendance avec `go list` (et par un test, leçon 7).
- Mesurer le coût (fichiers, lignes, conversions) et le comparer aux bénéfices attendus.

## Ce que je dois retenir

- Hexagonale, ports et adaptateurs, clean : le domaine au centre, la technologie autour, les
  dépendances vers l'intérieur.
- Port d'entrée = ce que le cœur offre ; port de sortie = ce dont il a besoin ; adaptateur = une
  technologie branchée sur un port.
- En Go, les adaptateurs de sortie satisfont les ports implicitement.
- Le coût (indirection, conversions, paquets) est réel ; le bénéfice vient avec les règles et les
  portes d'entrée multiples.
- L'usage raisonnable en Go est partiel.

---

➡️ [Exercices](exercices.md)
