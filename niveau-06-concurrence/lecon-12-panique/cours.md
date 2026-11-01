# Leçon 12 — Panique et goroutines

> Au niveau 2 (leçon 7), `recover` arrêtait une panique dans un `defer`. Une règle avait été
> annoncée sans être démontrée : **`recover` ne protège que sa propre goroutine**. Dans un
> programme concurrent, c'est décisif : une seule goroutine qui panique, n'importe où, **arrête
> tout le processus** — tous les clients d'un serveur, toutes les tâches en cours. Cette leçon
> montre pourquoi, et comment construire des frontières qui contiennent une panique sans en cacher
> la cause.

## Objectifs

1. Comprendre pourquoi une panique non récupérée dans une goroutine tue le programme.
2. Placer `recover` au bon endroit : à l'entrée de chaque goroutine qu'on veut protéger.
3. Transformer une panique en erreur **avec sa pile**.
4. Superviser un worker : redémarrer avec un plafond et une attente croissante.

## Explication

### Une panique tue le processus

```go
func main() {
	defer func() { recover() }()     // ne sert à rien pour l'autre goroutine
	go func() { panic("boum") }()
	time.Sleep(time.Second)
}
```

```
panic: boum

goroutine 7 [running]:
main.main.func2()
…
exit status 2
```

Une panique remonte la pile **de la goroutine** qui panique, en exécutant ses `defer`. Si elle
atteint le sommet de cette pile sans être récupérée, le runtime **arrête le programme entier**. Il n'y a
pas de relation parent-enfant entre goroutines : celle qui a lancé `go func()` n'est pas « au-dessus »
dans la pile de l'autre. Son `defer`/`recover` ne voit rien.

Conséquence : **toute goroutine qui exécute du code susceptible de paniquer — y compris du code
qu'on n'a pas écrit — est un point de défaillance du processus entier**.

C'est vrai aussi des goroutines lancées par des bibliothèques : `errgroup` (v0.22, mesuré) ne
récupère pas les paniques — une panique dans `g.Go` tue le programme, et le `recover` de `main` n'y
peut rien.

### Le `recover` à l'entrée de la goroutine

Pour protéger une goroutine, le `defer`/`recover` doit être **dans** sa fonction, en premier :

```go
go func() {
	defer func() {
		if r := recover(); r != nil {
			// la goroutine s'arrête, le programme continue
		}
	}()
	travail()
}()
```

On l'encapsule dans une fonction utilitaire (`safe`, `Go` dans l'exemple) pour ne pas le répéter.
C'est exactement ce que fait `net/http` : chaque requête est servie dans sa goroutine, protégée par un
`recover` qui journalise la panique et coupe **cette** connexion seulement (niveau 7).

### Ne pas cacher la panique

Récupérer une panique n'est **pas** la corriger. Une panique signale un **bug** (N2 L7) : accès hors
bornes, map nil, déréférencement nil, invariant violé. La récupérer sans rien dire, c'est transformer
un plantage visible en corruption silencieuse.

Une frontière de récupération correcte :

1. **capture la pile** au moment de la panique (`runtime/debug.Stack()`, dans le `defer`) — après,
   elle est perdue ;
2. **transforme** la panique en **erreur** typée (`*PanicError`), que l'appelant peut tester avec
   `errors.As` ;
3. **journalise** avec la pile (niveau 12) et alimente une métrique : une panique en production doit
   déclencher une alerte et un correctif.

### Où placer les frontières

| Frontière | Pourquoi |
|---|---|
| entrée de chaque **requête** d'un serveur | un bug sur une requête ne doit pas faire tomber les autres (`net/http` le fait déjà) |
| entrée de chaque **tâche** d'un worker pool | une tâche corrompue ne doit pas tuer le pool |
| entrée des goroutines de **longue durée** (boucles de service) | pour les **superviser** |
| **pas** dans chaque fonction | ce serait cacher des bugs ; les frontières sont rares et explicites |

Et **jamais** pour du flux de contrôle normal : une erreur attendue est une `error`, pas une panique.

### Ce qu'on ne peut pas récupérer

Certaines erreurs fatales du runtime **ne sont pas des paniques** et ne passent pas par `recover` :
`fatal error: concurrent map writes`, `fatal error: all goroutines are asleep - deadlock!`,
`fatal error: sync: unlock of unlocked mutex`, l'épuisement de la mémoire, le débordement de pile.
Aucune frontière ne les arrête. Ils se préviennent (`-race`, conception), ils ne se rattrapent pas.

### Superviser

Un **superviseur** relance un worker qui s'est arrêté sur une erreur ou une panique — l'idée vient du
monde Erlang (« let it crash ») :

- **un plafond** de redémarrages : un worker qui échoue toujours (disque plein, configuration fausse)
  ne doit pas être relancé à l'infini ;
- **une attente croissante** entre deux essais (*backoff*), pour ne pas boucler à 100 % de CPU ni
  marteler une dépendance en panne ;
- **un arrêt** sur annulation du contexte ;
- **une remontée** finale de l'erreur quand on abandonne.

Le redémarrage a du sens si l'échec est **transitoire** (connexion perdue, ressource temporairement
indisponible) ou si l'état du worker est reconstruit à chaque démarrage. Relancer un worker dont l'état
partagé a été laissé incohérent par la panique ne fait que propager la corruption.

## Exemple

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"runtime/debug"
	"strings"
	"sync"
	"time"
)

// PanicError transporte une panique récupérée, avec la pile de la goroutine qui a paniqué.
type PanicError struct {
	Value any
	Stack []byte
}

func (e *PanicError) Error() string { return fmt.Sprintf("panique : %v", e.Value) }

// safe exécute f et transforme une panique en erreur. Le recover est DANS la goroutine
// qui exécute f : c'est la seule place où il peut fonctionner.
func safe(f func() error) (err error) {
	defer func() {
		if r := recover(); r != nil {
			err = &PanicError{Value: r, Stack: debug.Stack()}
		}
	}()
	return f()
}

// Go lance f dans une goroutine protégée et rend son erreur (panique comprise) sur un channel.
func Go(wg *sync.WaitGroup, f func() error) <-chan error {
	errc := make(chan error, 1)
	wg.Go(func() { errc <- safe(f) })
	return errc
}

// supervise relance worker après une erreur ou une panique, au plus maxRestarts fois,
// avec une attente croissante ; s'arrête quand ctx est annulé.
func supervise(ctx context.Context, name string, maxRestarts int, worker func(context.Context) error) error {
	backoff := time.Millisecond
	for attempt := 0; ; attempt++ {
		err := safe(func() error { return worker(ctx) })
		if err == nil || ctx.Err() != nil {
			return err
		}
		if attempt == maxRestarts {
			return fmt.Errorf("%s : abandon après %d redémarrages : %w", name, maxRestarts, err)
		}
		fmt.Printf("%s : redémarrage %d après « %v »\n", name, attempt+1, err)
		select {
		case <-time.After(backoff):
			backoff *= 2
		case <-ctx.Done():
			return ctx.Err()
		}
	}
}

func main() {
	// 1. Une panique dans une goroutine protégée devient une erreur ; le programme continue.
	var wg sync.WaitGroup
	e1 := Go(&wg, func() error { return nil })
	e2 := Go(&wg, func() error {
		var m map[string]int
		m["x"] = 1 // panique : assignment to entry in nil map
		return nil
	})
	wg.Wait()
	fmt.Println("tâche 1 :", <-e1)
	err := <-e2
	fmt.Println("tâche 2 :", err)
	var pe *PanicError
	if errors.As(err, &pe) {
		fmt.Println("pile capturée :", strings.Contains(string(pe.Stack), "main.main.func"))
	}

	// 2. Un superviseur : un worker qui panique deux fois puis réussit.
	calls := 0
	err = supervise(context.Background(), "indexeur", 3, func(ctx context.Context) error {
		calls++
		if calls <= 2 {
			panic(fmt.Sprintf("index corrompu (essai %d)", calls))
		}
		return nil
	})
	fmt.Println("indexeur :", err, "après", calls, "essais")

	// 3. Un worker qui échoue toujours : le superviseur abandonne.
	err = supervise(context.Background(), "export", 2, func(ctx context.Context) error {
		return errors.New("disque plein")
	})
	fmt.Println(err)
}
```

Sortie :

```
tâche 1 : <nil>
tâche 2 : panique : assignment to entry in nil map
pile capturée : true
indexeur : redémarrage 1 après « panique : index corrompu (essai 1) »
indexeur : redémarrage 2 après « panique : index corrompu (essai 2) »
indexeur : <nil> après 3 essais
export : redémarrage 1 après « disque plein »
export : redémarrage 2 après « disque plein »
export : abandon après 2 redémarrages : disque plein
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `safe` avec résultat nommé `err` | Le `defer` peut remplacer la valeur de retour par l'erreur de panique (N2 L7). |
| `debug.Stack()` **dans** le `defer` | La pile de la panique n'existe qu'à cet instant ; c'est elle qui permettra de corriger le bug. |
| `*PanicError` | Une erreur typée : `errors.As` la distingue d'une erreur ordinaire. |
| `Go` : `wg.Go(func() { errc <- safe(f) })` | Le `recover` s'exécute **dans** la goroutine de `f`. Le channel de capacité 1 rend l'erreur sans risque de fuite. |
| La map nil | Un vrai bug, récupéré : le programme continue, la cause est conservée. |
| `supervise` | Plafond, attente doublée à chaque échec, sortie sur `ctx.Done()`, erreur finale enveloppée. |
| `calls` sans verrou | `supervise` appelle le worker **séquentiellement**, dans sa propre goroutine : pas d'accès concurrent. |

## Erreurs fréquentes

1. **Un `recover` dans `main`** en croyant protéger les goroutines.
2. **Lancer du code tiers dans une goroutine non protégée** dans un serveur long.
3. **Récupérer sans journaliser** ni capturer la pile : le bug devient invisible.
4. **Des `recover` partout** : on cache les bugs au lieu de les corriger.
5. **Utiliser panique/`recover` comme mécanisme d'erreur.**
6. **Croire qu'on peut récupérer `concurrent map writes`** ou un interblocage.
7. **Un superviseur sans plafond ni attente** : boucle infinie de redémarrages.
8. **Relancer un worker après une panique qui a laissé un état partagé incohérent.**

## Bonnes pratiques Go

- Une frontière de récupération à l'entrée de chaque requête, tâche ou goroutine de longue durée — et
  nulle part ailleurs.
- Panique → `*PanicError` avec la pile → journal + métrique + alerte.
- Superviser avec plafond, *backoff* et contexte.
- Préférer **prévenir** : `-race`, tests, fuzzing — une panique en production est un bug à corriger.

## Ce que je dois retenir

- Une panique non récupérée dans **n'importe quelle** goroutine arrête **tout** le programme.
- `recover` ne fonctionne que dans un `defer` de la goroutine qui panique.
- Une frontière : `defer` + `recover` à l'entrée de la goroutine, pile capturée, erreur typée.
- Les erreurs fatales du runtime (map concurrente, interblocage) ne se récupèrent pas.
- Superviser = redémarrer avec plafond, attente croissante et arrêt sur annulation.

---

➡️ [Exercices](exercices.md)
