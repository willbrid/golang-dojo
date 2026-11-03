# Leçon 5 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Le code ci-dessous complète le module du cours (les fichiers du cours ne sont pas répétés).

## E1 — Rompre la règle

Le cœur qui importe `console` **compile** : rien ne l'interdit, tant qu'il n'y a pas de cycle. `go list`
montre la violation :

```
exemple/core → context errors exemple/adapters/console fmt time
```

En ajoutant en plus l'import de `core` par `console`, le compilateur refuse :

```
package exemple/adapters/cliin
	imports exemple/core from cli.go
	imports exemple/adapters/console from format.go
	imports exemple/core from notifier.go: import cycle not allowed
```

L'import `console → core` respecte la règle (un adaptateur dépend du cœur) ; c'est `core → console` qui la
viole. Mais le compilateur n'interdit que les **cycles**, pas le **sens** : la violation seule passe
inaperçue tant que personne ne fait l'import inverse. Seuls `go list` (lu par un humain) ou un **test
d'architecture** (leçon 7) la détectent de façon fiable.

## E2 — Un adaptateur de plus

```go
// Fichier : adapters/file/notifier.go
// E2 : un adaptateur de sortie de plus. Aucune ligne du cœur ne change.
package file

import (
	"context"
	"fmt"
	"os"
	"sync"
)

type Notifier struct {
	mu   sync.Mutex
	path string
}

func NewNotifier(path string) *Notifier { return &Notifier{path: path} }

func (n *Notifier) Notify(_ context.Context, member, msg string) error {
	n.mu.Lock()
	defer n.mu.Unlock()
	f, err := os.OpenFile(n.path, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0o600)
	if err != nil {
		return err
	}
	_, werr := fmt.Fprintf(f, "%s : %s\n", member, msg)
	if cerr := f.Close(); werr == nil {
		werr = cerr // une erreur à la fermeture d'un fichier écrit est une vraie erreur
	}
	return werr
}
```

```go
// Fichier : adapters/file/notifier_test.go
package file

import (
	"os"
	"path/filepath"
	"testing"
	"time"

	"exemple/adapters/memory"
	"exemple/core"
)

func TestFileNotifierWithCore(t *testing.T) {
	path := filepath.Join(t.TempDir(), "notifications.log")
	l := core.New(memory.NewMembers(), NewNotifier(path), func() time.Time {
		return time.Date(2026, 3, 1, 0, 0, 0, 0, time.UTC)
	})
	l.Borrow(t.Context(), "ada", "a")
	l.Borrow(t.Context(), "ada", "b")
	b, err := os.ReadFile(path)
	if err != nil || string(b) != "ada : « a » à rendre le 2026-03-22\nada : « b » à rendre le 2026-03-22\n" {
		t.Fatalf("%q %v", b, err)
	}
}
```

Aucune ligne du cœur ne change ; dans `cmd/app`, une seule (le notificateur passé à `core.New`). Deux
détails de qualité : l'erreur de `Close` est rendue (sur un fichier **écrit**, elle peut signaler une
écriture perdue, N4 L7), et un mutex sérialise les écritures concurrentes.

## E3 — Sans port d'entrée

On perd la possibilité de tester l'adaptateur HTTP **sans le cœur** : `TestStatuses` passait un faux de
`Lending` qui rendait n'importe quelle erreur à la demande. Sans l'interface, le test HTTP doit construire
le vrai service avec des faux de ses ports de sortie, et provoquer chaque erreur à travers eux (un faux
`Members` qui rend une erreur pour obtenir le 500, trois emprunts préalables pour obtenir le 409). Ce
n'est pas grave : le test devient un test d'intégration légère du handler **et** du service, souvent plus
utile. Le port d'entrée vaut surtout quand le cœur est coûteux à construire, ou quand plusieurs
implémentations du cas d'usage existent réellement.

## E4 — Compter

```go
// Fichier : flat/lending.go
// E4 : le même comportement en UN paquet. Seules restent les interfaces de sortie.
package flat

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"time"
)

var ErrTooManyLoans = errors.New("nombre maximal d'emprunts atteint")

type Members interface {
	Loans(ctx context.Context, member string) (int, error)
	AddLoan(ctx context.Context, member, book string, due time.Time) error
}

type Notifier interface {
	Notify(ctx context.Context, member, msg string) error
}

type Service struct {
	Members Members
	Notify  Notifier
	Now     func() time.Time
}

func (s *Service) Borrow(ctx context.Context, member, book string) (time.Time, error) {
	n, err := s.Members.Loans(ctx, member)
	if err != nil {
		return time.Time{}, err
	}
	if n >= 3 {
		return time.Time{}, ErrTooManyLoans
	}
	due := s.Now().Add(21 * 24 * time.Hour)
	if err := s.Members.AddLoan(ctx, member, book, due); err != nil {
		return time.Time{}, err
	}
	_ = s.Notify.Notify(ctx, member, fmt.Sprintf("« %s » à rendre le %s", book, due.Format(time.DateOnly)))
	return due, nil
}

func (s *Service) Handler() http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("POST /members/{id}/loans/{book}", func(w http.ResponseWriter, r *http.Request) {
		due, err := s.Borrow(r.Context(), r.PathValue("id"), r.PathValue("book"))
		switch {
		case errors.Is(err, ErrTooManyLoans):
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

func (s *Service) RunCLI(ctx context.Context, args []string, out io.Writer) int {
	if len(args) != 2 {
		fmt.Fprintln(out, "usage : emprunt <adhérent> <livre>")
		return 2
	}
	due, err := s.Borrow(ctx, args[0], args[1])
	if err != nil {
		fmt.Fprintln(out, "refusé :", err)
		return 1
	}
	fmt.Fprintln(out, "à rendre le", due.Format("2006-01-02"))
	return 0
}
```

| | Exemple hexagonal | Un seul paquet |
|---|---|---|
| Paquets (hors `cmd`) | 6 | 1 |
| Fichiers (hors tests) | 6 | 1 |
| Lignes (hors tests) | 156 | 76 |
| Interfaces | 3 (`Lending`, `Members`, `Notifier`) | 2 (`Members`, `Notifier`) |

Ce qui devient plus difficile : tester le handler HTTP sans le service (E3), et remplacer un adaptateur
sans toucher au fichier qui contient les autres. Ce qui reste facile : tester le service avec des faux
de `Members` et `Notifier`, ajouter une porte d'entrée (une méthode de plus). Pour ce volume de règles, la
version plate est plus simple à lire ; elle deviendra trop grosse bien avant d'être difficile à tester.

## E5 — Quel découpage ?

- **(a) Authentification en HTTP, gRPC et bibliothèque** : complète, ou presque. Trois portes d'entrée
  réelles sur le même cœur, des règles de sécurité à tester seules : le port d'entrée a ici une vraie
  raison d'être.
- **(b) Tableau de bord interne** : aucune. Des requêtes et des graphiques ; pas de règles, une seule
  entrée.
- **(c) Moteur de calcul de primes** : partielle. Domaine riche et testé seul ; deux entrées (API et
  traitements par lots) qui appellent le service ; des ports de sortie pour les tarifs et l'historique.
- **(d) Redimensionnement d'images** : aucune, ou minimale. Une fonction de transformation, testée
  directement ; l'entrée HTTP et le stockage d'objets autour, sans cérémonie.

## Exercice intermédiaire — Une troisième porte d'entrée : les messages

```go
// Fichier : adapters/queuein/consume.go
// Adaptateur d'ENTRÉE : file de messages → port Lending.
package queuein

import (
	"context"
	"encoding/json"
	"fmt"
	"io"

	"exemple/core"
)

type message struct {
	Member string `json:"member"`
	Book   string `json:"book"`
}

// Consume traite les messages jusqu'à la fermeture du canal ou l'annulation de ctx.
// Un message illisible est journalisé et ignoré : il ne bloque pas les suivants.
func Consume(ctx context.Context, l core.Lending, msgs <-chan []byte, out io.Writer) {
	for {
		select {
		case <-ctx.Done():
			return
		case raw, ok := <-msgs:
			if !ok {
				return
			}
			var m message
			if err := json.Unmarshal(raw, &m); err != nil || m.Member == "" || m.Book == "" {
				fmt.Fprintf(out, "message ignoré (%q) : illisible\n", raw)
				continue
			}
			due, err := l.Borrow(ctx, m.Member, m.Book)
			if err != nil {
				fmt.Fprintf(out, "%s/%s : refusé : %v\n", m.Member, m.Book, err)
				continue
			}
			fmt.Fprintf(out, "%s/%s : à rendre le %s\n", m.Member, m.Book, due.Format("2006-01-02"))
		}
	}
}
```

```go
// Fichier : adapters/queuein/consume_test.go
package queuein

import (
	"bytes"
	"context"
	"testing"
	"time"

	"exemple/core"
)

type lendingFunc func(context.Context, string, string) (time.Time, error)

func (f lendingFunc) Borrow(ctx context.Context, m, b string) (time.Time, error) { return f(ctx, m, b) }

func TestConsumeWithFake(t *testing.T) {
	due := time.Date(2026, 3, 22, 0, 0, 0, 0, time.UTC)
	fake := lendingFunc(func(_ context.Context, m, _ string) (time.Time, error) {
		if m == "bob" {
			return time.Time{}, core.ErrTooManyLoans
		}
		return due, nil
	})
	msgs := make(chan []byte, 4)
	msgs <- []byte(`{"member":"ada","book":"a"}`)
	msgs <- []byte(`pas du json`)
	msgs <- []byte(`{"member":"bob","book":"b"}`)
	msgs <- []byte(`{"member":"ada","book":"c"}`)
	close(msgs)
	var out bytes.Buffer
	Consume(t.Context(), fake, msgs, &out)
	want := "ada/a : à rendre le 2026-03-22\n" +
		"message ignoré (\"pas du json\") : illisible\n" +
		"bob/b : refusé : nombre maximal d'emprunts atteint\n" +
		"ada/c : à rendre le 2026-03-22\n"
	if out.String() != want {
		t.Fatalf("\n%s", out.String())
	}
}

func TestConsumeStopsOnCancel(t *testing.T) {
	ctx, cancel := context.WithCancel(t.Context())
	done := make(chan struct{})
	go func() {
		Consume(ctx, nil, make(chan []byte), new(bytes.Buffer)) // canal jamais alimenté
		close(done)
	}()
	cancel()
	select {
	case <-done:
	case <-time.After(time.Second):
		t.Fatal("Consume ne s'arrête pas à l'annulation")
	}
}
```

```go
// Fichier : cmd/app/three_test.go
// Les trois entrées sur le même cœur.
package main

import (
	"bytes"
	"context"
	"net/http/httptest"
	"strings"
	"testing"
	"time"

	"exemple/adapters/cliin"
	"exemple/adapters/console"
	"exemple/adapters/httpin"
	"exemple/adapters/memory"
	"exemple/adapters/queuein"
	"exemple/core"
)

func TestThreeEntries(t *testing.T) {
	var out bytes.Buffer
	l := core.New(memory.NewMembers(), console.Notifier{Out: new(bytes.Buffer)}, time.Now)
	cliin.Run(context.Background(), l, []string{"ada", "a"}, &out)
	httpin.Handler(l).ServeHTTP(httptest.NewRecorder(), httptest.NewRequest("POST", "/members/ada/loans/b", nil))
	msgs := make(chan []byte, 2)
	msgs <- []byte(`{"member":"ada","book":"c"}`)
	msgs <- []byte(`{"member":"ada","book":"d"}`)
	close(msgs)
	queuein.Consume(t.Context(), l, msgs, &out)
	if !strings.Contains(out.String(), "ada/d : refusé : nombre maximal") {
		t.Fatalf("le 4e emprunt, par une 3e porte, doit être refusé :\n%s", out.String())
	}
}
```

`go test -race ./...` passe. **Aucune** ligne du cœur n'a changé. Le bénéfice du port d'entrée se voit
ici deux fois : l'adaptateur de messages se teste avec un faux de `Lending` (y compris le message
empoisonné et le refus métier), et le test d'intégration montre que la règle « 3 emprunts au plus »
s'applique quelle que soit la porte (CLI, HTTP, messages) : le quatrième emprunt, arrivé par la file, est
refusé.

## Défi — Mesurer la cérémonie

Pas de corrigé unique : il dépend du service étudié. Une règle d'équipe raisonnable, en dix lignes :

1. Le domaine n'importe aucune technologie ; c'est vérifié par un test (leçon 7).
2. Une interface naît d'un besoin : un test qui veut un faux, ou une seconde implémentation réelle.
3. Elle est déclarée par son consommateur et contient les méthodes qu'il appelle.
4. Un port d'entrée n'existe que s'il y a plusieurs portes d'entrée **et** un intérêt à tester un
   adaptateur sans le cœur.
5. Un type de frontière (DTO, ligne) n'existe que s'il diffère du type du domaine, ou risque de diverger.
6. Un paquet est créé quand il a un nom métier ou technique précis et du contenu ; jamais « au cas où ».
7. On organise par fonctionnalité ; les adaptateurs d'une fonctionnalité vivent avec elle.
8. Toute abstraction ajoutée est justifiée dans la revue de code par le test ou le changement qu'elle
   facilite.
9. Une abstraction sans usage depuis six mois est supprimée.
10. Le but est un code facile à changer, pas la conformité à une école.

## Quiz

1. Port d'entrée : ce que le cœur offre ; port de sortie : ce dont il a besoin.
2. Le branchement d'une technologie sur un port : HTTP ou CLI (entrée), PostgreSQL ou courriel (sortie).
3. Les dépendances pointent toujours vers le cœur.
4. Le domaine au centre, sans technologie, et la règle de dépendance.
5. Parce que la satisfaction d'une interface est implicite : il suffit d'avoir les méthodes.
6. À tester un adaptateur sans le cœur et à brancher plusieurs implémentations ; on s'en passe quand le
   cœur est simple à construire et unique.
7. Indirection, conversions de types, multiplication des paquets (aussi : abstractions prématurées).
8. Avec des règles non triviales, plusieurs portes d'entrée, ou des technologies susceptibles de changer.
9. Domaine sans technologie et ports de sortie déclarés par lui, adaptateurs d'entrée qui dépendent du
   service concret.
10. `go list` sur les imports, et un test d'architecture qui échoue en cas de violation.
