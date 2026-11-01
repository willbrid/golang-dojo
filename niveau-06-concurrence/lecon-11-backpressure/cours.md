# Leçon 11 — Backpressure

> Un service reçoit plus de travail qu'il ne peut en traiter. Que doit-il faire ? La réponse
> instinctive — « mettre en file d'attente, il finira bien par rattraper » — est la cause d'un
> nombre impressionnant de pannes : la file grossit, la latence explose, la mémoire déborde, et le
> service s'effondre au moment précis où on en a le plus besoin. La **backpressure** (contre-pression)
> est l'ensemble des techniques qui font **remonter** la surcharge vers l'amont au lieu de
> l'absorber jusqu'à la rupture.

## Objectifs

1. Comprendre pourquoi une file non bornée est une bombe à retardement.
2. Borner les files, et choisir entre **bloquer**, **attendre un temps borné** et **rejeter**.
3. Mesurer l'effet de la taille d'une file sur la latence.
4. Connaître les stratégies de délestage (*load shedding*).

## Explication

### La règle fondamentale

Soit un service qui traite μ tâches par seconde (sa **capacité**) et en reçoit λ (le **débit
d'arrivée**).

- Si λ < μ : la file reste courte en moyenne ; un buffer absorbe les rafales (leçon 3).
- Si λ > μ **durablement** : la file grandit de (λ − μ) tâches par seconde, **quelle que soit sa
  taille**. Aucun buffer ne résout un débit d'entrée supérieur au débit de sortie.

Dans le second cas, il n'y a que trois issues possibles :

1. **ralentir l'amont** (le faire attendre : c'est la contre-pression au sens strict) ;
2. **refuser** une partie du travail (le délestage) ;
3. **augmenter μ** (plus de workers, plus de machines) — utile, mais jamais instantané, et toujours borné.

Une file non bornée choisit une quatrième voie, implicite : **tout accepter et laisser exploser**
la latence puis la mémoire.

### Pourquoi la latence explose

Une tâche qui arrive derrière N autres attend N × (temps de traitement) / (nombre de workers). Avec
une file de 1 000 et des tâches de 3 ms sur 2 workers, la dernière attend **1,5 seconde** — alors que
le client a peut-être abandonné au bout de 500 ms. Le service travaille alors pour des clients **qui
ne sont plus là** : il est occupé à 100 %, et utile à 0 %. C'est l'effondrement sous charge, et il
persiste après la fin du pic, le temps de vider la file.

Une file **courte** garde une latence bornée pour le travail accepté, et rend la surcharge
**visible** immédiatement (par des refus) au lieu de la cacher dans une latence qui monte
lentement.

### Les trois politiques d'admission

| Politique | Code | Quand |
|---|---|---|
| **Bloquer** | `q <- t` | producteur interne qui peut attendre (un étage de pipeline) : la contre-pression se propage naturellement vers l'amont |
| **Attendre un temps borné** | `select { case q <- t: case <-ctx.Done(): }` | requête d'un client qui a un délai : on attend une place, mais pas au-delà de son délai |
| **Rejeter** | `select { case q <- t: default: return ErrOverloaded }` | point d'entrée d'un service (requêtes HTTP) : refuser vite coûte presque rien et permet au client de réessayer ailleurs ou plus tard |

Ces politiques s'appuient toutes sur ce qui a été vu : channel **bufferisé** borné (leçon 3),
`select` avec `default` ou `ctx.Done()` (leçons 4 et 8).

Le rejet doit être **explicite et typé** (`ErrOverloaded`) : l'appelant le reconnaît avec `errors.Is`,
et en HTTP il devient un statut `503 Service Unavailable` ou `429 Too Many Requests` avec un en-tête
`Retry-After` (niveau 7).

### Dimensionner la file

La taille de la file n'est pas un réglage de débit, c'est un réglage de **latence maximale** :

```
latence max ≈ (taille de la file / nombre de workers) × temps de traitement + temps de traitement
```

On part donc de la latence acceptable (le délai des clients), on en déduit la taille. Le niveau 11
(leçon 7) formalisera la relation avec la **loi de Little** : nombre moyen d'éléments dans le système
= débit × temps moyen passé dans le système.

### Le délestage, plus finement

Rejeter « ce qui arrive quand c'est plein » est la politique la plus simple. D'autres existent :

- rejeter en priorité les requêtes **les moins importantes** (une file par priorité) ;
- rejeter les requêtes **déjà trop vieilles** au moment où un worker les prend (leur client est
  sans doute parti) — vérifier `ctx.Err()` avant de commencer (leçon 9) ;
- limiter la **concurrence** plutôt que la file (sémaphore, `errgroup.SetLimit`) ;
- limiter le **débit** par client (le limiteur à jetons de la leçon 4) pour qu'un client bruyant
  n'affame pas les autres.

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : queue/queue.go
// Package queue est une file de travail BORNÉE : elle refuse plutôt que d'accumuler.
package queue

import (
	"context"
	"errors"
	"sync"
	"time"
)

// ErrOverloaded est rendue quand la file est pleine : l'appelant doit réessayer plus tard
// (en HTTP : 503 ou 429, niveau 7).
var ErrOverloaded = errors.New("service surchargé")

type task struct {
	work     func()
	enqueued time.Time
	done     chan time.Duration // latence totale (attente + traitement), capacité 1
}

type Queue struct {
	tasks chan task
	wg    sync.WaitGroup
}

// New démarre workers goroutines ; la file contient au plus capacity tâches en attente.
func New(workers, capacity int) *Queue {
	q := &Queue{tasks: make(chan task, capacity)}
	for range workers {
		q.wg.Go(func() {
			for t := range q.tasks {
				t.work()
				t.done <- time.Since(t.enqueued)
			}
		})
	}
	return q
}

// Submit dépose une tâche SANS JAMAIS BLOQUER : file pleine → ErrOverloaded immédiatement.
func (q *Queue) Submit(work func()) (<-chan time.Duration, error) {
	t := task{work: work, enqueued: time.Now(), done: make(chan time.Duration, 1)}
	select {
	case q.tasks <- t:
		return t.done, nil
	default:
		return nil, ErrOverloaded
	}
}

// SubmitWait attend une place au plus jusqu'à l'annulation de ctx : une attente BORNÉE.
func (q *Queue) SubmitWait(ctx context.Context, work func()) (<-chan time.Duration, error) {
	t := task{work: work, enqueued: time.Now(), done: make(chan time.Duration, 1)}
	select {
	case q.tasks <- t:
		return t.done, nil
	case <-ctx.Done():
		return nil, ErrOverloaded
	}
}

// Close cesse d'accepter des tâches et attend que celles en file soient traitées.
func (q *Queue) Close() {
	close(q.tasks)
	q.wg.Wait()
}
```

```go
// Fichier : queue/queue_test.go
package queue

import (
	"errors"
	"slices"
	"testing"
	"testing/synctest"
	"time"
)

// simulate envoie 200 tâches, une toutes les `every`, chacune durant `service`.
// Elle rend le nombre de refus et la latence médiane et maximale des tâches acceptées.
func simulate(t *testing.T, workers, capacity int, every, service time.Duration) (rejected int, p50, max time.Duration) {
	var lat []time.Duration
	synctest.Test(t, func(t *testing.T) {
		q := New(workers, capacity)
		var dones []<-chan time.Duration
		for range 200 {
			done, err := q.Submit(func() { time.Sleep(service) })
			switch {
			case errors.Is(err, ErrOverloaded):
				rejected++
			case err != nil:
				t.Fatal(err)
			default:
				dones = append(dones, done)
			}
			time.Sleep(every)
		}
		q.Close()
		for _, d := range dones {
			lat = append(lat, <-d)
		}
	})
	slices.Sort(lat)
	return rejected, lat[len(lat)/2], lat[len(lat)-1]
}

func TestBackpressure(t *testing.T) {
	// Capacité de traitement : 2 workers / 3 ms = 0,67 tâche par ms. Arrivées : 1 par ms.
	for _, c := range []int{4, 50, 1000} {
		rej, p50, max := simulate(t, 2, c, time.Millisecond, 3*time.Millisecond)
		t.Logf("file %4d : refusées %3d/200  latence médiane %-7v max %v", c, rej, p50, max)
	}
	// Sous la capacité : arrivées toutes les 2 ms.
	rej, p50, max := simulate(t, 2, 4, 2*time.Millisecond, 3*time.Millisecond)
	t.Logf("charge 75%% : refusées %3d/200  latence médiane %-7v max %v", rej, p50, max)
}
```

Sortie de `go test -v ./queue`, préfixes `queue_test.go:NN:` retirés (temps **virtuel** grâce à `synctest` : la simulation de 200 ms dure
quelques millisecondes réelles) :

```
file    4 : refusées  63/200  latence médiane 8ms     max 9ms
file   50 : refusées  17/200  latence médiane 48ms    max 78ms
file 1000 : refusées   0/200  latence médiane 53ms    max 102ms
charge 75% : refusées   0/200  latence médiane 3ms     max 3ms
```

*(La ligne « file 50 » donne 16 ou 17 refus selon l'exécution : quand un worker se libère au même
instant virtuel qu'une soumission arrive, l'ordre des deux événements n'est pas fixé.)*

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `make(chan task, capacity)` | La file **est** le buffer du channel : bornée par construction. |
| `Submit` avec `default` | Ne bloque jamais : refuse en temps constant quand la file est pleine. |
| `SubmitWait` avec `ctx.Done()` | Attend une place, mais seulement aussi longtemps que l'appelant l'accepte. |
| `done` de capacité 1 | Le worker dépose la latence et continue, même si personne ne l'attend (leçon 3). |
| `Close` : `close(q.tasks)` puis `wg.Wait()` | Arrêt propre : plus d'admissions, les tâches en file sont terminées (leçon 3 : l'émetteur ferme — ici, la file elle-même). |
| Arrivées 1/ms, capacité 0,67/ms | λ > μ : la surcharge est **réelle**, aucun réglage ne la fait disparaître. |
| File de 4 | 63 refus, mais les tâches acceptées sont servies en **9 ms au plus**. |
| File de 1 000 | **Aucun** refus… et une latence maximale de 102 ms, qui croîtrait sans fin si les arrivées continuaient. Au bout d'une minute de surcharge, la dernière tâche attendrait 20 secondes. |
| Charge à 75 % | Sous la capacité, la file reste vide : latence = temps de traitement, aucun refus. Le buffer ne sert qu'aux rafales. |

## Erreurs fréquentes

1. **Des files non bornées** (slice qui grandit, channel énorme, `go handle(req)` sans limite).
2. **Augmenter la taille de la file** pour « faire disparaître » les erreurs de surcharge.
3. **Bloquer indéfiniment** le traitement d'une requête client en attente d'une place.
4. **Traiter des requêtes dont le client est parti.**
5. **Rejeter sans le dire** (perdre silencieusement des tâches) au lieu d'une erreur explicite.
6. **Ne mesurer que la moyenne de latence** : la surcharge se voit d'abord dans les percentiles
   élevés (p99) et la longueur de file.
7. **Relancer immédiatement après un refus**, côté client : on aggrave la surcharge (*retry storm*,
   niveau 10).

## Bonnes pratiques Go

- Toute file est **bornée** ; toute admission a une politique explicite (bloquer, attendre borné, rejeter).
- Une erreur sentinelle de surcharge, testable avec `errors.Is`.
- Dimensionner les files à partir de la latence acceptable, pas du débit.
- Vérifier `ctx.Err()` avant de commencer une tâche qui a attendu.
- Exposer la longueur de la file et le nombre de refus comme métriques (niveau 12).
- Tester le comportement sous surcharge, avec `synctest` pour simuler des heures en millisecondes.

## Ce que je dois retenir

- Si le débit d'entrée dépasse durablement la capacité, **aucun buffer** ne suffit.
- Issues possibles : ralentir l'amont, refuser, augmenter la capacité — jamais « tout garder ».
- Une file longue = une latence longue ; on la dimensionne par la latence acceptable.
- `select` + `default` pour rejeter, + `ctx.Done()` pour attendre de façon bornée.
- Rejeter tôt et explicitement vaut mieux que servir tard des clients partis.

---

➡️ [Exercices](exercices.md)
