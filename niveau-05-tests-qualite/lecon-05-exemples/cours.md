# Leçon 5 — Exemples exécutables

> Sur pkg.go.dev, sous la documentation de `strings.Cut`, un bloc « Example » montre un
> programme et sa sortie. Ce programme n'a pas été écrit dans la documentation : c'est une
> fonction de **test**, exécutée par `go test`, dont la sortie est **vérifiée**. Une
> documentation qui ne peut pas mentir — c'est l'un des meilleurs outils de l'écosystème, et
> l'un des moins utilisés.

## Objectifs

1. Écrire des fonctions `Example` et les faire vérifier par `go test`.
2. Respecter les conventions de nommage qui les rattachent à un identifiant.
3. Lire les exemples d'un paquet avec `go doc -ex` (Go 1.27).
4. Savoir quand un exemple remplace avantageusement un test, et quand non.

## Explication

### Une fonction `Example`

Dans un fichier `_test.go` :

```go
func ExampleSlug() {
	fmt.Println(slug.Slug("Ça, c'est l'été !"))
	// Output: ca-c-est-l-ete
}
```

- Le nom commence par `Example`, la fonction n'a **ni paramètre ni résultat**.
- Le commentaire final `// Output:` contient la sortie standard attendue.
- `go test` exécute la fonction, capture ce qu'elle écrit sur `os.Stdout`, et compare au
  commentaire (espaces de début et de fin de chaque ligne ignorés). Une différence = un échec :

```
--- FAIL: ExampleSlug (0.00s)
got:
ca-cest-lete
want:
ca-c-est-l-ete
```

Sans commentaire `// Output:`, l'exemple est **compilé** mais pas exécuté : utile pour un
exemple qui dépend du réseau ou de l'heure, et qui doit au moins rester valide.

### Le nom décide où l'exemple apparaît

| Nom de la fonction | Rattaché à |
|---|---|
| `Example()` | le paquet lui-même |
| `ExampleSlug()` | la fonction `Slug` |
| `ExampleQueue()` | le type `Queue` |
| `ExampleQueue_Push()` | la méthode `Queue.Push` |
| `ExampleSlug_accents()` | `Slug`, avec le **suffixe** « accents » (suffixe en **minuscule**) |

Plusieurs exemples pour un même identifiant se distinguent par un suffixe en minuscules.
`go vet` signale un exemple qui se réfère à un identifiant inexistant
(`ExampleSlugg refers to unknown identifier: Slugg`) — une faute de frappe ne passe pas.

### Sortie non ordonnée

Quand l'ordre de la sortie n'est pas déterministe (itération sur une map), on écrit :

```go
// Unordered output:
// b 2
// a 1
```

`go test` compare alors les lignes **indépendamment de leur ordre**. Mais un exemple de
documentation devrait plutôt montrer la bonne pratique : trier avant d'afficher.

### Un exemple est écrit pour un **lecteur**

Un exemple est lu sur pkg.go.dev, dans l'éditeur, par `go doc` — par quelqu'un qui découvre le
paquet. Il doit donc :

- s'écrire dans un paquet **externe** (`package slug_test`) : le lecteur voit `slug.Slug(…)`,
  exactement ce qu'il écrira ;
- montrer **un** usage, typique, complet (avec la gestion d'erreur réelle, pas `_`) ;
- tenir en un écran ;
- avoir une sortie **parlante** (`fmt.Println(got)`, pas `fmt.Println(got == want)`).

Un **exemple complet** (*whole file example*) : un fichier `example_xxx_test.go` qui contient un
seul `Example`, plus des types ou fonctions auxiliaires. pkg.go.dev l'affiche alors comme un
programme entier, avec ses déclarations.

### Lire les exemples : `go doc -ex` (Go 1.27)

```bash
$ go doc -ex strings.Cut
func Cut(s, sep string) (before, after string, found bool)
    Cut slices s around the first instance of sep, …

    func ExampleCut()

$ go doc -ex strings.ExampleCut
package main
…                         ← l'exemple, présenté comme un programme complet
```

Le drapeau `-ex` liste les exemples d'un identifiant, et affiche le programme d'un exemple
nommé. La documentation devient disponible hors ligne, avec du code qui compile.

### Exemple ou test ?

| Un exemple… | Un test… |
|---|---|
| documente un usage typique | vérifie aussi les cas limites et les erreurs |
| est lu par un humain | est lu par la machine |
| compare une sortie texte | compare n'importe quoi, avec un message ciblé |
| un ou deux par identifiant important | autant que nécessaire |

Un exemple **complète** les tests ; il ne les remplace pas. Mais un paquet public sans aucun
exemple se prive de la meilleure documentation possible : celle qui est garantie à jour.

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : queue/queue.go
// Package queue fournit une file FIFO de capacité fixe.
package queue

import "errors"

// ErrFull est retournée par Push quand la file est pleine.
var ErrFull = errors.New("file pleine")

// Queue est une file FIFO de capacité fixe. Sa zéro-valeur n'est pas utilisable :
// créer une Queue avec New.
type Queue[T any] struct {
	buf     []T
	head, n int
}

// New crée une file pouvant contenir capacity éléments.
func New[T any](capacity int) *Queue[T] {
	return &Queue[T]{buf: make([]T, capacity)}
}

// Push ajoute v en fin de file, ou rend ErrFull.
func (q *Queue[T]) Push(v T) error {
	if q.n == len(q.buf) {
		return ErrFull
	}
	q.buf[(q.head+q.n)%len(q.buf)] = v
	q.n++
	return nil
}

// Pop retire et rend l'élément de tête ; ok vaut false si la file est vide.
func (q *Queue[T]) Pop() (v T, ok bool) {
	if q.n == 0 {
		return v, false
	}
	v = q.buf[q.head]
	var zero T
	q.buf[q.head] = zero
	q.head = (q.head + 1) % len(q.buf)
	q.n--
	return v, true
}

// Len rend le nombre d'éléments présents.
func (q *Queue[T]) Len() int { return q.n }
```

```go
// Fichier : queue/example_test.go
package queue_test

import (
	"errors"
	"fmt"

	"exemple/queue"
)

// Une file de capacité 2 : le troisième Push échoue.
func Example() {
	q := queue.New[string](2)
	for _, job := range []string{"compiler", "tester", "publier"} {
		if err := q.Push(job); err != nil {
			fmt.Printf("%s refusé : %v\n", job, err)
		}
	}
	for q.Len() > 0 {
		job, _ := q.Pop()
		fmt.Println("traité :", job)
	}
	// Output:
	// publier refusé : file pleine
	// traité : compiler
	// traité : tester
}

func ExampleQueue_Push() {
	q := queue.New[int](1)
	fmt.Println(q.Push(1))
	err := q.Push(2)
	fmt.Println(errors.Is(err, queue.ErrFull))
	// Output:
	// <nil>
	// true
}

func ExampleQueue_Pop() {
	q := queue.New[int](3)
	q.Push(42)
	v, ok := q.Pop()
	fmt.Println(v, ok)
	v, ok = q.Pop() // file vide : zéro-valeur et false
	fmt.Println(v, ok)
	// Output:
	// 42 true
	// 0 false
}
```

Sortie de `go test -v -run Example ./queue` :

```
=== RUN   Example
--- PASS: Example (0.00s)
=== RUN   ExampleQueue_Push
--- PASS: ExampleQueue_Push (0.00s)
=== RUN   ExampleQueue_Pop
--- PASS: ExampleQueue_Pop (0.00s)
PASS
ok  	exemple/queue	0.003s
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `example_test.go` | Nom conventionnel du fichier d'exemples d'un paquet. |
| `package queue_test` | Le lecteur voit `queue.New`, `queue.ErrFull` : le code qu'il copiera. |
| `Example()` | Rattaché au paquet : c'est la vitrine en tête de la documentation. |
| `ExampleQueue_Push` | Rattaché à la méthode `Push` du type `Queue` (le nom ignore le paramètre de type). |
| Le commentaire au-dessus de `Example` | Affiché comme introduction de l'exemple sur pkg.go.dev. |
| `// Output:` sur plusieurs lignes | Une ligne de commentaire par ligne de sortie. |
| `fmt.Println(q.Push(1))` → `<nil>` | Un exemple peut montrer ce qu'on voit réellement, y compris `<nil>`. |
| Le cas de la file vide | La documentation de `Pop` dit ce qui se passe ; l'exemple le **montre**, et le garantit. |

## Erreurs fréquentes

1. **Oublier `// Output:`** : l'exemple compile mais n'est jamais vérifié.
2. **`// output:` en minuscule** ou un espace manquant : le commentaire n'est pas reconnu.
3. **Une sortie non déterministe** (map, heure) avec `// Output:` : exemple instable.
4. **Un suffixe en majuscule** (`ExampleSlug_Accents`) : interprété comme une méthode `Accents`
   de `Slug` — `go vet` le signale.
5. **Des exemples dans le paquet interne** : le lecteur voit des appels sans préfixe.
6. **Des exemples qui ignorent les erreurs** avec `_` : le lecteur copiera ce style.
7. **Des exemples qui servent de tests de cas limites** : illisibles en documentation.

## Bonnes pratiques Go

- Un `Example()` pour chaque paquet public, et un exemple pour chaque identifiant central.
- `package xxx_test`, fichier `example_test.go`.
- Toujours `// Output:` quand la sortie est déterministe.
- Un usage typique et complet par exemple, lisible en un écran.
- Consulter les exemples de la stdlib avec `go doc -ex` avant de chercher ailleurs.

## Ce que je dois retenir

- `func ExampleXxx()` + `// Output:` = documentation **vérifiée** par `go test`.
- Sans `// Output:`, l'exemple est seulement compilé.
- Le nom rattache l'exemple : `Example`, `ExampleF`, `ExampleT`, `ExampleT_M`, suffixe `_minuscule`.
- `// Unordered output:` pour une sortie sans ordre (mais mieux vaut trier).
- `go doc -ex` liste et affiche les exemples (Go 1.27).
- Un exemple complète les tests, il ne les remplace pas.

---

➡️ [Exercices](exercices.md)
