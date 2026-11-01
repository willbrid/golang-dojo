# Leçon 2 — Contraintes de type

> Au niveau 2, une interface était un **ensemble de méthodes**. Pour écrire `a + b` ou
> `a < b` sur un `T`, ce n'est plus suffisant : aucune méthode ne dit « ce type supporte `<` ».
> Go 1.18 a donc élargi la définition. Une interface est désormais un **ensemble de types**.
> Toute la leçon découle de ce changement de point de vue.

## Objectifs

1. Comprendre une interface comme un **ensemble de types** (*type set*).
2. Écrire des contraintes avec des **unions** et l'opérateur `~`.
3. Connaître `any`, `comparable`, `cmp.Ordered` et leurs pièges.
4. Combiner méthodes et types dans une même contrainte.

## Explication

### Une interface définit un ensemble de types

Relisons l'interface du niveau 2 autrement :

```go
type Stringer interface{ String() string }
```

Ancienne lecture : « une liste de méthodes ». Nouvelle lecture : « **l'ensemble de tous les
types** qui ont une méthode `String() string` ». C'est le même objet ; seule la formulation
change. Mais la nouvelle permet d'écrire des interfaces qui énumèrent des types :

```go
type Integer interface {
	int | int8 | int16 | int32 | int64
}
```

`Integer` est l'ensemble de ces cinq types, ni plus ni moins. Le `|` est une **union**.

Une fonction contrainte par `Integer` peut utiliser **toute opération que les cinq types
supportent** : `+`, `-`, `*`, `/`, `%`, `<`, `==`, la conversion vers `float64`… C'est la
règle unique de la leçon : **une opération est permise sur `T` si elle l'est pour chaque type
de l'ensemble.**

```go
func Sum[T Integer](xs []T) T {
	var s T
	for _, x := range xs {
		s += x // ✓ les cinq types supportent +
	}
	return s
}
```

### L'opérateur `~` : le type sous-jacent

```go
type Celsius float64

type Float interface{ float32 | float64 }
func Avg[T Float](xs []T) T { … }

Avg([]Celsius{20, 22})   // ✗ Celsius does not satisfy Float (possibly missing ~ for float64 in Float)
```

`Celsius` est un **type défini** (niveau 1, leçon 2) : distinct de `float64`, même s'il en a
la représentation. L'union `float32 | float64` ne le contient pas. La correction :

```go
type Float interface{ ~float32 | ~float64 }
```

`~float64` signifie « **tout type dont le type sous-jacent est `float64`** » : `float64`
lui-même, `Celsius`, `time.Duration` pour `~int64`, etc. Dans une bibliothèque, on écrit
**presque toujours** `~` : sans lui, on refuse les types définis de ses utilisateurs, qui
sont précisément ce que Go encourage.

Restriction : `~T` exige que `T` soit son propre type sous-jacent. `~Celsius` ne compile pas.

### Les interfaces de contrainte ne sont pas des types ordinaires

```go
var n Integer   // ✗ cannot use type Integer outside a type constraint: interface contains type constraints
```

Une interface qui contient des unions ou des `~` ne peut servir **que** de contrainte. On ne
peut pas en faire une variable, un paramètre ordinaire, un élément de slice. Raison : une
variable `Integer` serait une boîte (type, valeur), et `n + m` sur deux boîtes contenant un
`int8` et un `int64` n'aurait pas de sens.

On distingue donc deux familles :

| Interface | Exemple | Utilisable comme type ? | Comme contrainte ? |
|---|---|---|---|
| **de base** : seulement des méthodes | `io.Reader`, `error`, `any` | oui | oui |
| **générale** : contient des types | `Integer`, `cmp.Ordered` | **non** | oui |

### Combiner types et méthodes

Une contrainte peut exiger les deux à la fois. Le `;` (ou un retour à la ligne) signifie
**intersection** :

```go
type Level int
func (l Level) String() string { … }

type IntStringer interface {
	~int
	String() string
}
```

Est accepté tout type dont le sous-jacent est `int` **et** qui a une méthode `String()`.
`Level` oui, `int` non (pas de méthode). Dans le corps, on peut à la fois écrire `x + 1` et
`x.String()`.

### Contraintes en ligne

Quand une contrainte ne sert qu'une fois, on l'écrit directement dans les crochets — le
mot-clé `interface{}` est alors facultatif :

```go
func Abs[T ~int | ~float64](x T) T {
	if x < 0 {
		return -x
	}
	return x
}
```

### `comparable` : les types qui ont `==`

`comparable` est une contrainte prédéclarée : tous les types sur lesquels `==` et `!=` sont
définis. Booléens, nombres, chaînes, pointeurs, channels, tableaux et structs **de
comparables**, interfaces. **Pas** les slices, maps et fonctions (niveau 1, leçon 7 : ce sont
eux qui ne peuvent pas servir de clé de map).

Elle sert surtout pour les **clés de map** : `func Keys[K comparable, V any](m map[K]V)`.

Le piège : les **interfaces** satisfont `comparable` depuis Go 1.20. Or comparer deux
interfaces qui contiennent des slices compile… et panique :

```go
func Eq[T comparable](a, b T) bool { return a == b }

var a, b any = []int{1}, []int{1}
Eq(a, b)   // panic: runtime error: comparing uncomparable type []int
```

`comparable` garantit que `==` **compile**, pas qu'il réussit. C'est exactement la garantie
qu'offrait déjà `==` sur deux `any`, au niveau 2.

### `cmp.Ordered` : les types qui ont `<`

Le paquet `cmp` (Go 1.21) définit :

```go
type Ordered interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
		~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
		~float32 | ~float64 |
		~string
}
```

Tous les types qui supportent `<`, `<=`, `>`, `>=`. Remarquer `~string` : les chaînes sont
ordonnées (ordre des octets, pas l'ordre alphabétique d'un dictionnaire français).

**Le piège des flottants** : `NaN` (*not a number*, le résultat de `0.0/0.0` ou de
`math.Sqrt(-1)`) n'est ni plus petit, ni plus grand, ni égal à quoi que ce soit — y compris
lui-même. Une fonction écrite avec `<` ou `>` donne alors un résultat qui **dépend de l'ordre
des arguments** :

```go
func Max[T cmp.Ordered](a, b T) T {
	if a > b {
		return a
	}
	return b
}

Max(math.NaN(), 1)   // 1     (NaN > 1 est faux → on rend b)
Max(1, math.NaN())   // NaN   (1 > NaN est faux → on rend b)
```

Le paquet `cmp` fournit `cmp.Compare` et `cmp.Less`, qui définissent un ordre **total** :
NaN y est plus petit que tout autre nombre. Et les fonctions prédéclarées `min` et `max` (Go
1.21) rendent NaN dès qu'un argument l'est. Morale : pour comparer dans du code générique,
**préférer `cmp.Compare`** à `<` dès que `T` peut être un flottant.

### Ce qu'on ne peut pas faire dans une contrainte

- La règle « permise pour tous » joue dans les deux sens : `len(x)` compile pour
  `~string | ~[]byte` (les deux ont une longueur), pas pour `~int | ~string`. Et `%` est
  refusé dès qu'un flottant est dans l'ensemble. En cas de doute, le compilateur tranche —
  et son message nomme la contrainte fautive.
- Pas de champ de struct : `interface{ ~struct{ Name string } }` est accepté, mais `x.Name`
  ne compile pas. Les génériques Go ne savent pas exprimer « a un champ `Name` » — c'est une
  **méthode** qu'il faut.
- Pas d'union qui contient une interface à méthodes : `int | fmt.Stringer` est refusé.

Ces limites sont voulues : les concepteurs ont préféré un système simple, vérifiable une
seule fois, à un système expressif.

## Exemple

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"time"
)

// Number : tous les numériques, y compris les types définis par l'utilisateur.
type Number interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
		~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
		~float32 | ~float64
}

// Mean : la moyenne, calculée en float64 pour ne pas tronquer les entiers.
func Mean[T Number](xs []T) (float64, bool) {
	if len(xs) == 0 {
		return 0, false
	}
	var sum float64
	for _, x := range xs {
		sum += float64(x) // conversion permise : chaque type de Number se convertit en float64
	}
	return sum / float64(len(xs)), true
}

// Clamp borne v à [lo, hi]. cmp.Compare plutôt que < : ordre total, NaN compris.
func Clamp[T cmp.Ordered](v, lo, hi T) T {
	if cmp.Compare(v, lo) < 0 {
		return lo
	}
	if cmp.Compare(v, hi) > 0 {
		return hi
	}
	return v
}

// Labeled : un entier qui sait se nommer. Intersection d'un ensemble de types et d'une méthode.
type Labeled interface {
	~int
	String() string
}

type Priority int

const (
	Low Priority = iota
	Medium
	High
)

func (p Priority) String() string {
	switch p {
	case Low:
		return "basse"
	case Medium:
		return "moyenne"
	case High:
		return "haute"
	}
	return fmt.Sprintf("Priority(%d)", int(p))
}

// Next rend la valeur suivante, bornée : arithmétique ET méthode sur le même T.
func Next[T Labeled](x, last T) (T, string) {
	if x < last {
		x++
	}
	return x, x.String()
}

type Celsius float64

func main() {
	m1, _ := Mean([]int{1, 2, 4})
	m2, _ := Mean([]Celsius{19.5, 21, 22.5})
	m3, _ := Mean([]time.Duration{time.Second, 3 * time.Second}) // ~int64
	_, ok := Mean([]float64{})
	fmt.Printf("%.3f %.1f %v %v\n", m1, m2, time.Duration(m3), ok)

	fmt.Println(Clamp(15, 0, 10), Clamp("m", "a", "f"), Clamp(Celsius(-3), 0, 30))
	fmt.Println(Clamp(math.NaN(), 0, 1)) // NaN < 0 pour cmp.Compare : on rend lo

	p, label := Next(Medium, High)
	fmt.Println(int(p), label)
	p, label = Next(p, High)
	fmt.Println(int(p), label)
}
```

Sortie :

```
2.333 21.0 2s false
10 f 0
0
2 haute
2 haute
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `~int \| … \| ~float64` | Le `~` admet `Celsius` et `time.Duration` ; sans lui, les deux appels correspondants ne compileraient pas. |
| `float64(x)` | Permis parce que **chaque** type de l'ensemble se convertit en `float64`. |
| `(float64, bool)` | La moyenne d'une liste vide n'existe pas : on le dit dans la signature (niveau 1, leçon 10) plutôt que de diviser par zéro. |
| `cmp.Compare` | Ordre total : `Clamp(NaN, 0, 1)` rend `0` de façon déterministe. Avec `<`, il rendrait NaN. |
| `Clamp("m", "a", "f")` | `~string` fait partie de `cmp.Ordered` : les chaînes s'ordonnent octet par octet. |
| `Labeled` | Deux lignes = intersection : sous-jacent `int` **et** méthode `String`. |
| `x++` et `x.String()` | Le corps de `Next` mélange arithmétique (permise par `~int`) et appel de méthode. |

## Erreurs fréquentes

1. **Oublier le `~`** et refuser les types définis des utilisateurs (`Celsius`, `Duration`).
2. **Utiliser une contrainte comme un type** : `var n Number`, `[]Number`.
3. **Croire que `comparable` garantit un `==` sans panique** — faux pour les interfaces.
4. **Comparer des flottants génériques avec `<`** quand NaN est possible.
5. **Écrire sa propre contrainte `Ordered`** alors que `cmp.Ordered` existe. Idem pour
   `Integer`, `Float` : le module `golang.org/x/exp/constraints` les fournit, mais pour une
   bibliothèque, mieux vaut copier trois lignes que d'ajouter une dépendance expérimentale.
6. **Vouloir accéder à un champ** de struct via une contrainte : il faut une méthode.
7. **Des contraintes trop larges** : `Number` alors que le code n'a de sens que pour des
   entiers (un `%` sur un `float64` ne compile pas — le compilateur le dira, heureusement).

## Bonnes pratiques Go

- `any` quand aucune opération n'est requise, `comparable` pour `==` et les clés de map,
  `cmp.Ordered` pour `<`. Ces trois-là couvrent la grande majorité des cas.
- **Toujours `~`** dans les unions d'une bibliothèque.
- Une contrainte personnalisée ne s'exporte que si les utilisateurs en ont besoin pour écrire
  leurs propres fonctions ; sinon elle reste en ligne ou non exportée.
- Pour comparer génériquement : `cmp.Compare`, `cmp.Less`, `min`, `max`.
- Une contrainte avec méthodes et types mélangés est rare ; si on en écrit une, se demander
  si une interface ordinaire ne suffirait pas.

## Ce que je dois retenir

- Une interface est un **ensemble de types** ; une méthode, une union, un `~` restreignent cet ensemble.
- Une opération est permise sur `T` si elle l'est pour **tous** les types de l'ensemble.
- `~T` = tout type de **sous-jacent** `T`.
- Les interfaces contenant des types ne servent **que** de contraintes.
- `comparable` : `==` compile, mais peut paniquer sur des interfaces.
- `cmp.Ordered` : `<` existe, mais NaN casse l'ordre ; `cmp.Compare` le répare.

---

➡️ [Exercices](exercices.md)
