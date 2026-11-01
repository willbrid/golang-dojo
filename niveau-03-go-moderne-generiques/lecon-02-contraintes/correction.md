# Leçon 2 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Le `~` manquant

```
Celsius does not satisfy Float (possibly missing ~ for float64 in Float)
```

Le compilateur nomme `Celsius`, désigne l'ensemble `Float`, et **propose la correction** :
il a vu que le sous-jacent de `Celsius` est `float64`, qui figure dans l'union. Correction :
`type Float interface{ ~float32 | ~float64 }`.

Quand le sous-jacent n'est pas du tout dans l'union (un `Celsius` passé à une contrainte
`int | int64`), le message change : `Celsius missing in int | int64`. Deux messages, deux
diagnostics : lire lequel on a reçu.

## E2 — Prédire, puis mesurer

| Fonction | Compile ? | Raison |
|---|---|---|
| `A` (`~int \| ~float64`, `%`) | **non** | `operator % not defined on x` : `%` n'existe pas sur les flottants |
| `B` (`~int \| ~uint`, `%`) | oui | les deux sont entiers |
| `C` (`~string \| ~[]byte`, `len`) | oui | les deux ont une longueur |
| `D` (`~int \| ~string`, `len`) | **non** | `invalid argument: x … for built-in len` : `int` n'a pas de longueur |
| `E` (`~int \| ~string`, `+`) | **oui** | piège : `+` existe pour les deux — addition pour l'un, **concaténation** pour l'autre |
| `F` (`comparable`, `<`) | **non** | `type parameter T cannot use operator <` : `comparable` donne `==`, pas `<` |

`E` montre la limite de la règle « permis pour tous » : elle vérifie que l'**opérateur**
existe, pas qu'il a le **même sens**. `E(3)` vaut `6`, `E("ab")` vaut `"abab"`. Une
contrainte qui mélange nombres et chaînes est presque toujours une erreur de conception.

## E3 — Contrainte, pas type

```
cannot use type cmp.Ordered outside a type constraint: interface contains type constraints
```

Une variable d'interface est une boîte (type dynamique, valeur). Un `[]cmp.Ordered` pourrait
contenir un `int` en `xs[0]` et une `string` en `xs[1]` ; `xs[0] < xs[1]` n'aurait alors aucun
sens, et le compilateur ne pourrait rien vérifier. Le langage coupe court : une interface qui
énumère des types ne sert qu'à **contraindre**, jamais à **contenir**.

## E4 — `comparable` qui panique

```go
func Count[T comparable](xs []T, v T) int {
	n := 0
	for _, x := range xs {
		if x == v {
			n++
		}
	}
	return n
}
```

Ça **compile** : `any` satisfait `comparable` depuis Go 1.20. À l'exécution, tout dépend des
valeurs comparées :

```go
Count([]any{1, []int{1}}, any(1))          // 1 — aucune panique !
Count([]any{[]int{1}}, any([]int{1}))      // panic: runtime error: comparing uncomparable type []int
```

Le premier appel ne panique pas : comparer deux interfaces commence par comparer leurs
**types dynamiques**. `int` contre `[]int` : types différents, résultat `false` sans jamais
regarder les valeurs. La panique n'arrive que quand les deux interfaces contiennent **le même
type** et que ce type n'a pas de `==`.

La trace pointe la ligne `if x == v` dans `main.Count[...]` — les `[...]` signalent une
fonction générique instanciée.

C'est le genre de bug qui passe tous les tests (aucun n'a deux slices identiques) et éclate
en production. Si une fonction générique doit recevoir des interfaces, le documenter, ou
exiger une contrainte plus stricte.

## E5 — NaN

| Entrée | `>` | `cmp.Compare` | `slices.Max` |
|---|---|---|---|
| `{1, NaN, 3}` | `3` | `3` | `NaN` |
| `{NaN, 1, 3}` | **`NaN`** | `3` | `NaN` |

- Avec `>`, le résultat **dépend de la position** de NaN : s'il est en tête, il devient le
  maximum initial et aucune comparaison `x > NaN` n'est jamais vraie. Non déterministe vis-à-vis
  de l'ordre — le pire des trois.
- Avec `cmp.Compare`, NaN est **plus petit que tout** : il est ignoré pour un maximum. Déterministe.
- `slices.Max` **propage** NaN, comme la fonction prédéclarée `max` : « une donnée invalide
  rend le résultat invalide ». Déterministe aussi, et souvent le comportement le plus honnête.

Deux comportements défendables (ignorer ou propager), un indéfendable (dépendre de l'ordre).

## Exercice intermédiaire — `stats`

```go
// Package stats fournit des statistiques simples sur des slices numériques.
package stats

import (
	"cmp"
	"errors"
	"slices"
)

// ErrEmpty est retournée quand une statistique est demandée sur une liste vide.
var ErrEmpty = errors.New("liste vide")

// Number regroupe les types numériques, types définis compris.
type Number interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
		~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
		~float32 | ~float64
}

// Sum rend la somme des éléments. Le calcul se fait dans le type T :
// sur des entiers, un dépassement de capacité est silencieux
// (Sum([]int8{100, 100}) vaut -56).
func Sum[T Number](xs []T) T {
	var s T
	for _, x := range xs {
		s += x
	}
	return s
}

// Mean rend la moyenne arithmétique, calculée en float64.
func Mean[T Number](xs []T) (float64, error) {
	if len(xs) == 0 {
		return 0, ErrEmpty
	}
	var s float64
	for _, x := range xs {
		s += float64(x)
	}
	return s / float64(len(xs)), nil
}

// Median rend la médiane. Le slice de l'appelant n'est pas modifié.
func Median[T Number](xs []T) (float64, error) {
	if len(xs) == 0 {
		return 0, ErrEmpty
	}
	s := slices.Clone(xs) // ← le tri ne doit pas toucher les données de l'appelant
	slices.Sort(s)
	mid := len(s) / 2
	if len(s)%2 == 1 {
		return float64(s[mid]), nil
	}
	return (float64(s[mid-1]) + float64(s[mid])) / 2, nil
}

// MinMax rend le plus petit et le plus grand élément.
func MinMax[T cmp.Ordered](xs []T) (lo, hi T, err error) {
	if len(xs) == 0 {
		return lo, hi, ErrEmpty
	}
	lo, hi = xs[0], xs[0]
	for _, x := range xs[1:] {
		if cmp.Less(x, lo) {
			lo = x
		}
		if cmp.Less(hi, x) {
			hi = x
		}
	}
	return lo, hi, nil
}

// Histogram compte les occurrences de chaque valeur.
func Histogram[T cmp.Ordered](xs []T) map[T]int {
	h := make(map[T]int)
	for _, x := range xs {
		h[x]++
	}
	return h
}
```

*`slices.Clone` et `slices.Sort` sont au programme de la leçon 4 ; une copie avec `make` +
`copy` et un tri à la main étaient la réponse attendue à ce stade.*

Points de correction :

1. **Exporter `Number` ?** Oui : un utilisateur qui écrit `Variance[T stats.Number]` doit
   pouvoir réutiliser exactement le même ensemble, sinon ses fonctions et celles du paquet
   divergeront au premier type ajouté.
2. `ErrEmpty` en sentinelle : `errors.Is(err, stats.ErrEmpty)`.
3. **`Median` ne modifie pas l'appelant** : sans `Clone`, trier `xs` trierait le tableau de
   l'appelant, qui verrait ses données réordonnées après un simple calcul de médiane — effet
   de bord invisible dans la signature.
4. `Sum([]int8{100, 100})` vaut **-56** : 200 ne tient pas dans un `int8`, on garde les 8
   bits de poids faible, interprétés en complément à deux (niveau 1, leçon 2). Le documenter
   est le minimum ; une alternative est de faire rendre `Sum` en `float64` comme `Mean`, au
   prix de la précision sur les grands `int64`.
5. `Mean([]time.Duration{…})` fonctionne grâce à `~int64`.
6. `Sum`, `Mean`, `Median` refusent les `string` : `string does not satisfy Number`.
   `MinMax` et `Histogram` les acceptent.

`Histogram` aurait pu se contenter de `comparable` : une map n'a besoin que de `==`. Exiger
`cmp.Ordered` est **trop strict** — à moins de vouloir offrir plus tard un parcours trié.
Choisir la contrainte la **plus faible** qui permet le corps.

## Défi — Une contrainte qui exige une méthode

```go
type Comparer[T any] interface {
	Compare(other T) int
}

func SortSlice[T Comparer[T]](xs []T) {
	for i := 1; i < len(xs); i++ {
		for j := i; j > 0 && xs[j].Compare(xs[j-1]) < 0; j-- {
			xs[j], xs[j-1] = xs[j-1], xs[j]
		}
	}
}

type Version struct{ Major, Minor, Patch int }

func (v Version) Compare(o Version) int {
	if c := cmp.Compare(v.Major, o.Major); c != 0 {
		return c
	}
	if c := cmp.Compare(v.Minor, o.Minor); c != 0 {
		return c
	}
	return cmp.Compare(v.Patch, o.Patch)
}

func (v Version) String() string { return fmt.Sprintf("%d.%d.%d", v.Major, v.Minor, v.Patch) }
```

1. **`Comparer[T]`** : la méthode `Compare` doit prendre un argument **du même type** que le
   récepteur. `Comparer` seul ne dirait pas quel type. En écrivant `T Comparer[T]`, on
   exprime « `T` a une méthode `Compare(T) int` » — la contrainte **se réfère à `T`**. C'est
   l'idiome générique pour toute opération binaire.
2. Tri par insertion : voir ci-dessus. Sortie : `[0.9.9 1.2.0 1.9.3 1.10.0]`.
3. Un tri par `String()` compare des **textes** : `"1.10.0" < "1.2.0"` parce que `'1' < '2'`
   au troisième caractère. On obtient `0.9.9 1.10.0 1.2.0 1.9.3` — l'erreur classique des
   numéros de version triés comme des chaînes.
4. `int` **ne satisfait pas** `Comparer[int]` : il n'a pas de méthode. Les deux mondes ne se
   rejoignent pas — `SortSlice` ne trie pas des `int`, `slices.Sort` (contrainte
   `cmp.Ordered`) ne trie pas des `Version`. C'est pourquoi la bibliothèque standard propose
   **deux** fonctions : `slices.Sort` pour les types ordonnés et `slices.SortFunc` avec une
   fonction de comparaison pour tout le reste (leçon 4). Passer une **fonction** est la
   solution qui couvre les deux.

## Quiz

1. Un **ensemble de types** : ceux qui satisfont toutes ses exigences (méthodes, unions, `~`).
2. Tout type dont le **type sous-jacent** est `float64`. `~Celsius` est refusé parce que le
   sous-jacent de `Celsius` n'est pas `Celsius` : `~` n'accepte qu'un type qui est son propre
   sous-jacent.
3. Une opération est permise si elle l'est pour **chaque** type de l'ensemble.
4. Parce que `cmp.Ordered` contient des éléments de type ; une telle interface ne peut servir
   que de contrainte.
5. Que `==` et `!=` **compilent**. Pas qu'ils réussissent : sur des interfaces contenant le
   même type non comparable, ils paniquent.
6. Tous les entiers signés et non signés, `uintptr`, les deux flottants et les chaînes, chacun
   avec `~`. Oui, `~string` en fait partie.
7. Il définit un ordre **total**, NaN compris ; `<` donne un résultat qui dépend de l'ordre
   des arguments dès qu'un NaN apparaît.
8. `interface{ ~int; String() string }`.
9. Non. Il faut passer par une méthode.
10. Une **intersection** : le type doit satisfaire chaque ligne. L'union s'écrit `|` sur une
    même ligne.
