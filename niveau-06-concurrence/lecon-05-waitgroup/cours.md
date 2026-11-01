# Leçon 5 — `sync.WaitGroup`

> Les channels synchronisent en transmettant des valeurs. Parfois, on veut seulement savoir
> qu'un **groupe** de goroutines a fini. C'est le rôle de `sync.WaitGroup` — l'outil le plus
> simple du paquet `sync`, utilisé depuis la leçon 1 sous sa forme moderne `wg.Go`. Cette leçon
> le détaille, avec la forme classique `Add`/`Done` qu'on lit dans tout le code existant, et les
> quatre erreurs qui le font paniquer ou mentir.

## Objectifs

1. Attendre un groupe de goroutines avec `wg.Go` (Go 1.25) et avec `Add`/`Done`/`Wait`.
2. Collecter des résultats et des erreurs sans *data race*.
3. Éviter les pièges : `Add` au mauvais endroit, compteur négatif, copie, réutilisation.

## Explication

### Le principe : un compteur

Un `WaitGroup` est un compteur protégé :

| Méthode | Effet |
|---|---|
| `wg.Add(n)` | ajoute `n` au compteur |
| `wg.Done()` | retire 1 (équivaut à `Add(-1)`) |
| `wg.Wait()` | bloque jusqu'à ce que le compteur vaille 0 |
| `wg.Go(f)` (Go 1.25) | `Add(1)`, lance `f` dans une goroutine, et appelle `Done` quand `f` se termine |

Sa zéro-valeur est prête à l'emploi : `var wg sync.WaitGroup`, pas de constructeur.

Comme un channel, `Wait` apporte une garantie du modèle mémoire : tout ce que les goroutines ont
écrit **avant** leur `Done` est visible **après** le retour de `Wait`. C'est ce qui permet de lire
les résultats sans verrou, une fois `Wait` passé.

### `wg.Go` : la forme moderne

```go
var wg sync.WaitGroup
for _, item := range items {
	wg.Go(func() { process(item) })
}
wg.Wait()
```

`wg.Go` rend impossibles deux des erreurs classiques ci-dessous (`Add` mal placé, `Done` oublié).
C'est la forme à écrire dans tout code neuf. `go fix` (N5 L8, modernisateur `waitgroupgo`)
convertit automatiquement l'ancienne forme.

### `Add`/`Done` : la forme classique

```go
var wg sync.WaitGroup
wg.Add(len(items))              // 1. AVANT de lancer
for _, item := range items {
	go func() {
		defer wg.Done()          // 2. defer : même en cas de retour anticipé
		process(item)
	}()
}
wg.Wait()
```

Deux règles :

1. **`Add` avant le `go`, dans la goroutine qui attendra.** Jamais dans la goroutine lancée.
2. **`defer wg.Done()` en première ligne** de la goroutine.

### Piège 1 : `Add` dans la goroutine

```go
for range 100 {
	go func() {
		wg.Add(1)          // ✗ trop tard
		defer wg.Done()
		…
	}()
}
wg.Wait()
```

`wg.Wait()` peut s'exécuter **avant** que les goroutines aient atteint leur `Add` : le compteur
vaut encore 0, `Wait` rend immédiatement, et `main` continue pendant que le travail n'a pas commencé.
Mesuré sur 100 goroutines, le programme affichait `fini` alors que seules 93, puis 58 avaient
compté. `go vet` signale ce cas (Go 1.25+) :

```
WaitGroup.Add called from inside new goroutine
```

### Piège 2 : compteur négatif

Un `Done` de trop (un `Done` dans deux chemins, ou `Add(1)` pour deux goroutines) fait paniquer :

```
panic: sync: negative WaitGroup counter
```

### Piège 3 : la copie

```go
func worker(wg sync.WaitGroup) { defer wg.Done() }   // ✗ reçoit une COPIE
```

Le `Done` décrémente la copie ; l'original n'atteint jamais 0, `Wait` bloque pour toujours. Un
`WaitGroup` (comme un `Mutex`, leçon 6) ne se copie **jamais** : on le passe par **pointeur**, ou
mieux, on ne le passe pas (la closure le capture). `go vet` le signale :

```
work passes lock by value: sync.WaitGroup contains sync.noCopy
```

### Piège 4 : réutiliser avant la fin

Un `WaitGroup` peut être réutilisé **après** que `Wait` a rendu. Appeler `Add` pendant qu'un autre
`Wait` est en cours avec un compteur à zéro est une erreur de conception (le comportement est
indéfini, et la documentation l'interdit). Un groupe = un `WaitGroup` = une attente.

### Collecter des résultats et des erreurs

`WaitGroup` ne transporte rien : il faut un endroit où les goroutines **déposent** leurs résultats.
Trois options, de la plus simple à la plus générale :

| Option | Quand |
|---|---|
| **une case de slice par goroutine** (`results[i] = …`) | nombre de tâches connu ; ordre des résultats = ordre des entrées |
| un **channel** bufferisé de capacité N, lu après `Wait` | idem, ordre d'arrivée |
| un **channel** lu **pendant** le travail, fermé par un coordinateur après `Wait` | flux de résultats, nombre inconnu (leçon 3, `merge`) |

Pour les erreurs : collecter, puis `errors.Join` (N2 L6). Pour s'**arrêter à la première erreur**
et annuler les autres, il faudra `context` et `errgroup` (leçons 8 et 9).

## Exemple

```go
package main

import (
	"crypto/sha256"
	"errors"
	"fmt"
	"strings"
	"sync"
	"time"
)

type doc struct {
	name, body string
}

type result struct {
	name string
	sum  string
	err  error
}

// checksum simule un traitement qui peut échouer.
func checksum(d doc) (string, error) {
	time.Sleep(time.Duration(len(d.body)) * time.Millisecond) // durées différentes : ordre d'arrivée quelconque
	if strings.TrimSpace(d.body) == "" {
		return "", fmt.Errorf("%s : document vide", d.name)
	}
	return fmt.Sprintf("%x", sha256.Sum256([]byte(d.body)))[:12], nil
}

// processAll traite tous les documents en parallèle et rend les résultats
// DANS L'ORDRE DES ENTRÉES, plus toutes les erreurs réunies.
func processAll(docs []doc) ([]result, error) {
	results := make([]result, len(docs))
	var wg sync.WaitGroup
	for i, d := range docs {
		wg.Go(func() {
			sum, err := checksum(d)
			results[i] = result{name: d.name, sum: sum, err: err}
		})
	}
	wg.Wait() // après ce point, toutes les écritures dans results sont visibles

	var errs []error
	for _, r := range results {
		if r.err != nil {
			errs = append(errs, r.err)
		}
	}
	return results, errors.Join(errs...)
}

// processAllClassic : la même chose avec Add/Done, la forme d'avant Go 1.25.
func processAllClassic(docs []doc) []result {
	results := make([]result, len(docs))
	var wg sync.WaitGroup
	wg.Add(len(docs)) // AVANT de lancer les goroutines
	for i, d := range docs {
		go func() {
			defer wg.Done() // defer : Done est appelé même si checksum panique ou retourne tôt
			sum, err := checksum(d)
			results[i] = result{name: d.name, sum: sum, err: err}
		}()
	}
	wg.Wait()
	return results
}

func main() {
	docs := []doc{
		{"long.txt", strings.Repeat("go ", 30)},
		{"vide.txt", "   "},
		{"court.txt", "bonjour"},
		{"rien.txt", ""},
	}
	results, err := processAll(docs)
	for _, r := range results {
		if r.err == nil {
			fmt.Printf("%-10s %s\n", r.name, r.sum)
		}
	}
	fmt.Printf("erreurs :\n%v\n", err)

	same := processAllClassic(docs)
	fmt.Println("mêmes résultats :", same[0].sum == results[0].sum && same[2].sum == results[2].sum)
}
```

Sortie :

```
long.txt   410a2e575f1c
court.txt  2cb4b1431b84
erreurs :
vide.txt : document vide
rien.txt : document vide
mêmes résultats : true
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `results := make([]result, len(docs))` | Une case par tâche, allouée **avant** : aucune goroutine ne modifie le slice lui-même (pas d'`append` concurrent). |
| `results[i] = …` dans `wg.Go` | Chaque goroutine écrit **sa** case. `-race` silencieux. |
| Les `time.Sleep` de durées différentes | Les documents finissent dans un ordre quelconque ; les résultats sortent pourtant **dans l'ordre des entrées**, grâce à l'indice. |
| Lecture de `results` **après** `wg.Wait()` | Le modèle mémoire garantit que toutes les écritures sont visibles. |
| `errors.Join(errs...)` | Toutes les erreurs, pas seulement la première ; `nil` s'il n'y en a pas. |
| `processAllClassic` | `Add(len(docs))` **une fois**, avant la boucle ; `defer wg.Done()` en tête de goroutine. |
| `defer wg.Done()` | Si `checksum` paniquait, `Done` serait quand même appelé — mais la panique tuerait le programme de toute façon (leçon 12). |

## Erreurs fréquentes

1. **`wg.Add(1)` dans la goroutine lancée.**
2. **Oublier `Done`** dans un chemin de retour : `Wait` bloque pour toujours. `defer`.
3. **Un `Done` de trop** : `negative WaitGroup counter`.
4. **Passer un `WaitGroup` par valeur.**
5. **`append` sur un slice partagé** depuis plusieurs goroutines : race (et perte de valeurs).
6. **Lire les résultats avant `Wait`.**
7. **Croire que `WaitGroup` limite le nombre de goroutines** : il les compte, il ne les borne pas
   (leçon 9 : *worker pool*, sémaphore).

## Bonnes pratiques Go

- `wg.Go(f)` dans tout code neuf.
- Avec `Add`/`Done` : `Add` avant le `go`, `defer wg.Done()` en première ligne.
- Ne jamais copier un `WaitGroup` ; le capturer dans la closure plutôt que le passer.
- Pré-allouer un emplacement de résultat par tâche.
- `go vet` à chaque commit : il détecte trois des quatre pièges.

## Ce que je dois retenir

- `WaitGroup` = compteur : `Add`, `Done`, `Wait`, et `Go` (1.25) qui fait les trois.
- Les écritures faites avant `Done` sont visibles après `Wait`.
- `Add` **avant** le `go` ; `Done` en `defer`.
- Ne jamais copier ; zéro-valeur utilisable.
- Pour les résultats : une case par goroutine ; pour les erreurs : `errors.Join`.

---

➡️ [Exercices](exercices.md)
