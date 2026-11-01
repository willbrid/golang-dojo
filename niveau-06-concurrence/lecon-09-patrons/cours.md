# Leçon 9 — Patrons de concurrence

> Les huit leçons précédentes ont fourni les briques : goroutines, channels, `select`,
> `WaitGroup`, verrous, `context`. Cette leçon les assemble en **patrons** — des formes
> éprouvées qu'on retrouve dans tout service Go : le **pipeline**, le **worker pool**, le
> **fan-out/fan-in**, et leur version industrielle, **`errgroup`**. Connaître ces formes, c'est
> lire en trois secondes un code concurrent qu'on n'a pas écrit.

## Objectifs

1. Construire un pipeline dont chaque étage ferme sa sortie et respecte l'annulation.
2. Écrire un worker pool à N goroutines (fan-out) qui fusionne ses résultats (fan-in).
3. Utiliser `errgroup` : première erreur, annulation des autres, limite de concurrence.
4. Choisir le bon patron, et savoir quand n'en utiliser aucun.

## Explication

### Le pipeline

Une suite d'étages reliés par des channels ; chaque étage est une fonction qui **reçoit** un
`<-chan` et **rend** un `<-chan` :

```go
func stage(ctx context.Context, in <-chan A) <-chan B {
	out := make(chan B)
	go func() {
		defer close(out)                 // 1. l'étage ferme sa sortie (leçon 3)
		for a := range in {              // 2. il s'arrête quand l'amont a fermé
			select {
			case out <- transform(a):
			case <-ctx.Done():           // 3. il s'arrête si l'aval a abandonné
				return
			}
		}
	}()
	return out
}
```

Les trois règles du pipeline :

1. **Chaque étage ferme sa sortie** : la fin se propage vers l'aval.
2. **Chaque étage écoute l'annulation** : si le consommateur final s'arrête (erreur, délai),
   l'annulation remonte vers l'amont et **toutes** les goroutines se terminent. Sans le `case
   <-ctx.Done()`, un étage resterait bloqué pour toujours sur un envoi que plus personne ne lit —
   la fuite classique des pipelines.
3. **Chaque étage rend `<-chan`** : l'appelant ne peut ni écrire, ni fermer.

Rappel de la leçon 2 : un échange sur un channel coûte ~100 ns. Un pipeline n'est utile que si
chaque étage a un travail **conséquent** par élément — sinon, envoyer des **lots**.

### Le worker pool (fan-out / fan-in)

Pour paralléliser un étage coûteux : N goroutines lisent **le même** channel d'entrée (*fan-out* :
chaque élément est pris par **un seul** worker), et écrivent sur **un** channel de sortie commun
(*fan-in*), fermé par un coordinateur quand tous les workers ont fini (leçon 3).

```go
for range workers {
	wg.Go(func() {
		for j := range jobs {        // les workers se partagent les tâches
			out <- process(j)
		}
	})
}
go func() { wg.Wait(); close(out) }()
```

Différence avec « une goroutine par tâche » (leçon 1) : le nombre de goroutines est **borné**, quel
que soit le nombre de tâches. C'est indispensable dès que les tâches consomment une ressource limitée
(descripteurs de fichiers, connexions à une base, mémoire).

**L'ordre est perdu** : les résultats arrivent dans l'ordre de **fin**, pas d'entrée. Si l'ordre
compte, transporter un indice et trier ou ranger à la fin (exemple).

Combien de workers ? Pour du **calcul**, de l'ordre de `GOMAXPROCS` (leçon 1). Pour des **E/S**
(réseau, disque), davantage, limité par la ressource en aval (une base qui accepte 20 connexions →
au plus 20 workers). On **mesure**.

### Le sémaphore

Variante plus simple quand on lance déjà une goroutine par tâche mais qu'on veut en limiter le
nombre **actif** : un channel bufferisé de capacité N (leçon 5, exercice intermédiaire). Ou
`errgroup.SetLimit`, ci-dessous.

### `errgroup` : le patron industriel

Le module `golang.org/x/sync` (maintenu par l'équipe Go, hors bibliothèque standard) fournit
`errgroup` :

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(8)                            // au plus 8 goroutines actives (g.Go bloque au-delà)
for _, item := range items {
	g.Go(func() error {
		return process(ctx, item)        // une erreur annule ctx pour toutes les autres
	})
}
if err := g.Wait(); err != nil {         // attend TOUTES les goroutines, rend la PREMIÈRE erreur
	return err
}
```

Il réunit en quelques lignes ce que la leçon 8 construisait à la main (`WaitGroup` +
`WithCancelCause`) : attente de toutes les goroutines, **première erreur** rendue, **annulation** du
contexte partagé dès cette erreur, et limite de concurrence. `g.TryGo` lance seulement si la limite
n'est pas atteinte.

Deux précisions :

- `Wait` attend **toutes** les goroutines, même après la première erreur : les autres doivent donc
  **respecter `ctx`**, sinon `Wait` attend leur fin normale.
- Les erreurs suivantes sont **perdues**. Si toutes comptent, collecter dans un slice (une case par
  tâche) et `errors.Join`.

Ajouter la dépendance : `go get golang.org/x/sync`.

### Choisir

| Besoin | Patron |
|---|---|
| quelques tâches indépendantes, toutes nécessaires, arrêt à la première erreur | **`errgroup`** |
| beaucoup de tâches, ressource limitée | **`errgroup` + `SetLimit`** ou **worker pool** |
| flux continu d'éléments, plusieurs transformations successives | **pipeline** |
| un étage coûteux dans un pipeline | **worker pool** pour cet étage |
| une seule tâche en arrière-plan | une goroutine + `context` |
| du travail rapide et en petite quantité | **aucun** — une boucle |

Le dernier cas est le plus fréquent. La concurrence a un coût de lecture, de test et de débogage :
elle se justifie par un gain mesuré ou par une structure naturellement concurrente (serveur).

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27

require golang.org/x/sync v0.22.0
```

```go
// Fichier : main.go
package main

import (
	"cmp"
	"context"
	"crypto/sha256"
	"errors"
	"fmt"
	"slices"
	"strings"
	"sync"

	"golang.org/x/sync/errgroup"
)

type job struct {
	id   int
	text string
}

type result struct {
	id    int
	words int
	sum   string
}

// ---- 1. Pipeline : générer → transformer → collecter, chaque étage ferme sa sortie. ----

func generate(ctx context.Context, texts []string) <-chan job {
	out := make(chan job)
	go func() {
		defer close(out)
		for i, t := range texts {
			select {
			case out <- job{id: i, text: t}:
			case <-ctx.Done(): // le consommateur a abandonné : on arrête de produire
				return
			}
		}
	}()
	return out
}

// ---- 2. Worker pool : N goroutines lisent le même channel (fan-out),
//         écrivent sur un channel commun (fan-in), fermé par un coordinateur. ----

func workerPool(ctx context.Context, jobs <-chan job, workers int) <-chan result {
	out := make(chan result)
	var wg sync.WaitGroup
	for range workers {
		wg.Go(func() {
			for j := range jobs {
				r := result{id: j.id, words: len(strings.Fields(j.text)),
					sum: fmt.Sprintf("%x", sha256.Sum256([]byte(j.text)))[:8]}
				select {
				case out <- r:
				case <-ctx.Done():
					return
				}
			}
		})
	}
	go func() {
		wg.Wait()
		close(out)
	}()
	return out
}

// ---- 3. errgroup : la même chose, avec arrêt à la première erreur et limite de concurrence. ----

var ErrEmpty = errors.New("texte vide")

func analyze(ctx context.Context, texts []string, limit int) ([]result, error) {
	g, ctx := errgroup.WithContext(ctx) // ctx est annulé à la première erreur
	g.SetLimit(limit)                   // au plus `limit` goroutines actives
	results := make([]result, len(texts))
	for i, t := range texts {
		g.Go(func() error {
			if err := ctx.Err(); err != nil { // déjà annulé : inutile de commencer
				return err
			}
			if strings.TrimSpace(t) == "" {
				return fmt.Errorf("texte %d : %w", i, ErrEmpty)
			}
			results[i] = result{id: i, words: len(strings.Fields(t))}
			return nil
		})
	}
	if err := g.Wait(); err != nil { // attend TOUTES les goroutines, rend la première erreur
		return nil, err
	}
	return results, nil
}

func main() {
	texts := []string{"le chat dort", "go est simple", "un deux trois quatre", "fin"}

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	var got []result
	for r := range workerPool(ctx, generate(ctx, texts), 3) {
		got = append(got, r)
	}
	slices.SortFunc(got, func(a, b result) int { return cmp.Compare(a.id, b.id) }) // l'ordre d'arrivée est quelconque
	for _, r := range got {
		fmt.Printf("#%d %d mots %s\n", r.id, r.words, r.sum)
	}

	res, err := analyze(context.Background(), texts, 2)
	fmt.Println(len(res), "analysés, erreur :", err)

	_, err = analyze(context.Background(), []string{"a", " ", "b"}, 2)
	fmt.Println("erreur :", err, "| vide ?", errors.Is(err, ErrEmpty))
}
```

Sortie (`go get golang.org/x/sync` au préalable) :

```
#0 3 mots f15433f0
#1 3 mots 0f18b063
#2 4 mots 117d6bf5
#3 1 mots a383bcb7
4 analysés, erreur : <nil>
erreur : texte 1 : texte vide | vide ? true
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `generate` | Premier étage : ferme sa sortie, et abandonne si `ctx` est annulé — un générateur qui ignorerait `ctx` resterait bloqué pour toujours si le consommateur s'arrêtait tôt. |
| `workerPool(…, 3)` | Trois goroutines seulement, quel que soit le nombre de textes. Les quatre textes sont répartis entre elles. |
| `select` sur `out <- r` et `ctx.Done()` | Un worker ne reste jamais bloqué sur un résultat que plus personne n'attend. |
| `wg.Wait()` puis `close(out)` | Fan-in : plusieurs émetteurs, un coordinateur qui ferme (leçon 3). |
| `slices.SortFunc` par `id` | Les résultats arrivent dans l'ordre de fin ; on rétablit l'ordre d'entrée. |
| `errgroup.WithContext` + `SetLimit(2)` | Au plus deux analyses simultanées ; la première erreur annule `ctx`. |
| `if err := ctx.Err(); err != nil` en tête | Une tâche lancée après l'annulation ne commence même pas. |
| `results[i] = …` | Une case par tâche (leçon 5) : pas de verrou. |
| `errors.Is(err, ErrEmpty)` | L'erreur rendue par `Wait` est celle de la goroutine, avec son contexte. |

## Erreurs fréquentes

1. **Un étage de pipeline qui n'écoute pas `ctx`** : fuite dès que l'aval s'arrête tôt.
2. **Un étage qui oublie de fermer sa sortie** : l'aval attend pour toujours.
3. **Un worker qui ferme le channel de sortie** commun : les autres paniquent.
4. **Une goroutine par tâche sans limite** sur des millions de tâches ou une ressource limitée.
5. **Supposer l'ordre des résultats** d'un worker pool.
6. **`errgroup` avec des tâches qui ignorent `ctx`** : `Wait` attend la fin de tout, l'annulation ne
   sert à rien.
7. **Croire que `errgroup` rend toutes les erreurs.**
8. **Un pipeline pour du travail minuscule** : plus lent que la boucle.

## Bonnes pratiques Go

- Étage de pipeline : `defer close(out)`, `range in`, `select` avec `ctx.Done()`, rend `<-chan`.
- Worker pool : nombre de workers **mesuré**, résultats indexés si l'ordre compte.
- `errgroup.WithContext` + `SetLimit` comme choix par défaut pour « N tâches, arrêt à la première erreur ».
- Toute fonction lancée dans un groupe reçoit et respecte le `ctx` du groupe.
- Commencer séquentiel ; paralléliser sur mesure.

## Ce que je dois retenir

- **Pipeline** : étages reliés par channels ; chacun ferme sa sortie et écoute l'annulation.
- **Worker pool** : N goroutines sur un même channel d'entrée (fan-out), une sortie commune fermée
  par un coordinateur (fan-in).
- **`errgroup`** : `WithContext`, `Go`, `SetLimit`, `Wait` → première erreur + annulation des autres.
- L'ordre des résultats d'un pool n'est pas garanti.
- Le nombre de goroutines doit être **borné** quand les tâches sont nombreuses ou les ressources limitées.

---

➡️ [Exercices](exercices.md)
