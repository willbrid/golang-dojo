# Leçon 6 — Tableaux et slices

> **La leçon la plus importante du niveau 1.** Les slices sont partout en Go, et leur
> mécanique — partage d'un tableau sous-jacent — est la première source de bugs subtils
> chez les débutants. À lire deux fois.

## Objectifs

1. Distinguer un **tableau** (`[5]int`) d'un **slice** (`[]int`) et savoir lequel utiliser.
2. Expliquer les trois champs d'un slice : pointeur, longueur, capacité.
3. Prédire le comportement d'`append` — quand il réalloue et quand il écrase.
4. Reconnaître les trois pièges classiques : aliasing, sous-slice, capture de mémoire.

## Explication

### Le tableau : taille fixe, valeur copiable

```go
var a [5]int              // [0 0 0 0 0]
b := [3]string{"x", "y", "z"}
c := [...]int{1, 2, 3}    // taille déduite : [3]int
```

**La taille fait partie du type.** `[3]int` et `[4]int` sont deux types différents, aussi
incompatibles que `int` et `string`. Un tableau est une **valeur** : l'affecter ou le
passer à une fonction **copie tous ses éléments**.

```go
x := [3]int{1, 2, 3}
y := x        // COPIE complète
y[0] = 99
fmt.Println(x[0], y[0])   // 1 99
```

En pratique, on utilise rarement des tableaux directement : ils servent de support aux
slices, et pour des tailles fixes connues (`[32]byte` pour un hash, `[16]byte` pour un UUID).

### Le slice : une vue sur un tableau

Un slice n'est **pas** un tableau dynamique : c'est une **fenêtre** sur un tableau
sous-jacent. En interne, c'est une structure de trois champs :

```
type slice struct {
	ptr *T    // pointeur vers le premier élément visible
	len int   // nombre d'éléments visibles
	cap int   // nombre d'éléments disponibles à partir de ptr
}
```

```go
s := []int{1, 2, 3}                  // len=3 cap=3
s2 := make([]int, 3)                 // [0 0 0]      len=3 cap=3
s3 := make([]int, 0, 10)             // []           len=0 cap=10
var s4 []int                         // nil          len=0 cap=0
```

`len` = ce qu'on voit. `cap` = ce qu'on peut atteindre sans réallouer.

Le slice `nil` mérite une note : il est **parfaitement utilisable**. `len(nil slice)` vaut
0, `range` dessus ne fait rien, `append` fonctionne. `var s []int` est donc l'idiome
préféré à `s := []int{}` — même comportement, aucune allocation. Seule différence
observable : `s == nil` est vrai pour l'un, faux pour l'autre (et l'encodage JSON produit
`null` au lieu de `[]`).

### Découper : `s[low:high]`

```go
s := []int{0, 1, 2, 3, 4, 5}
fmt.Println(s[1:4])    // [1 2 3]  — high est EXCLU
fmt.Println(s[:3])     // [0 1 2]
fmt.Println(s[3:])     // [3 4 5]
fmt.Println(s[:])      // tout
```

**Un sous-slice ne copie rien.** Il crée un nouveau descripteur qui pointe dans le **même**
tableau. Ce qui donne le premier piège :

```go
s := []int{0, 1, 2, 3, 4, 5}
part := s[1:3]         // [1 2]
part[0] = 99
fmt.Println(s)         // [0 99 2 3 4 5]  ← s est modifié !
```

C'est l'**aliasing**. Ce n'est pas un bug de Go : c'est le principe même du slice, et c'est
ce qui rend `s[i+1:]` gratuit dans une boucle. Mais il faut le savoir.

Détail à connaître : `s[1:3]` a `len=2` mais `cap=5` — la capacité va du début de la
fenêtre jusqu'à la fin du tableau sous-jacent. C'est ce qui rend le piège d'`append`
possible (plus bas).

### `append` : le cœur du sujet

```go
s := make([]int, 0, 2)     // len=0 cap=2
s = append(s, 1)           // len=1 cap=2 — écrit dans le tableau existant
s = append(s, 2)           // len=2 cap=2
s = append(s, 3)           // len=3 cap=4 — PLUS DE PLACE : nouveau tableau, copie
```

Règle : si `len < cap`, `append` écrit sur place. Sinon il **alloue un nouveau tableau,
copie les éléments**, et retourne un slice pointant vers ce nouveau tableau.

D'où l'obligation absolue de réaffecter :

```go
append(s, 1)         // ← INUTILE, le résultat est jeté ; go vet le signale
s = append(s, 1)     // ← correct
```

La croissance est amortie : historiquement le double sous 256 éléments, puis ~1,25× ; les
détails exacts sont un choix d'implémentation, **jamais garanti par la spécification**. Ne
jamais écrire de code qui en dépend.

**Le piège d'`append` sur un sous-slice** — celui qui coûte des heures de débogage :

```go
s := []int{0, 1, 2, 3, 4, 5}
part := s[1:3]              // len=2 cap=5
part = append(part, 99)     // len < cap → écrit DANS s !
fmt.Println(s)              // [0 1 2 99 4 5]  ← s[3] écrasé
```

La parade est le **slice à trois indices**, `s[low:high:max]`, qui borne la capacité :

```go
part := s[1:3:3]            // len=2 cap=2
part = append(part, 99)     // cap atteinte → réalloue, s est intact
```

À utiliser dès qu'on retourne un sous-slice depuis une fonction : c'est la différence entre
« ça fonctionne » et « c'est correct ».

### `copy` : quand on veut vraiment dupliquer

```go
dst := make([]int, len(src))
n := copy(dst, src)          // retourne le nombre d'éléments copiés = min(len(dst), len(src))
```

`copy` ne redimensionne jamais la destination. Oublier de l'allouer à la bonne taille
(`var dst []int` puis `copy(dst, src)`) copie **zéro** élément, sans erreur : bug silencieux
classique.

Depuis Go 1.21, `slices.Clone(src)` fait la même chose plus lisiblement.

### La fuite de mémoire par sous-slice

```go
func firstBytes(data []byte) []byte {
	return data[:10]     // garde en vie TOUT le tableau de 100 Mo
}
```

Le sous-slice pointe dans le grand tableau : tant qu'il est vivant, le ramasse-miettes ne
peut rien libérer. Pour un gros tableau, il faut copier : `return slices.Clone(data[:10])`.
Ce cas est réel et se rencontre en production (parsing de gros fichiers, buffers réseau).

### La stdlib moderne : le paquet `slices` (Go 1.21+)

```go
slices.Contains(s, 3)          slices.Sort(s)
slices.Index(s, 3)             slices.SortFunc(s, cmp)
slices.Reverse(s)              slices.Max(s) / slices.Min(s)
slices.Equal(a, b)             slices.Clone(s)
slices.Insert(s, i, vals...)   slices.Delete(s, i, j)
slices.BinarySearch(s, v)      slices.Compact(s)   // retire les doublons ADJACENTS
```

Avant Go 1.21 il fallait tout écrire à la main ou passer par `sort.Slice`. Beaucoup de code
et de tutoriels en ligne datent d'avant : **connaître `slices` est un marqueur de code
moderne**. On y reviendra au niveau 2 (ces fonctions sont génériques).

## Exemple

Aucune fonction passée en paramètre ici : les types fonction arrivent à la leçon 11.

```go
package main

import (
	"fmt"
	"slices"
)

// Evens retourne un NOUVEAU slice avec les nombres pairs.
// L'original n'est jamais modifié : contrat explicite et sans surprise.
func Evens(nums []int) []int {
	out := make([]int, 0, len(nums)) // préallocation : une seule allocation
	for _, n := range nums {
		if n%2 == 0 {
			out = append(out, n)
		}
	}
	return out
}

// EvensInPlace filtre SANS allouer, en réutilisant le tableau sous-jacent.
// Attention : l'appelant perd l'original. Le nom doit le dire.
func EvensInPlace(nums []int) []int {
	out := nums[:0] // len=0, même tableau, même capacité
	for _, n := range nums {
		if n%2 == 0 {
			out = append(out, n)
		}
	}
	return out
}

func describe(label string, s []int) {
	fmt.Printf("%-14s %v len=%d cap=%d\n", label, s, len(s), cap(s))
}

func main() {
	base := []int{1, 2, 3, 4, 5, 6}
	describe("base", base)

	even := Evens(base)
	describe("even", even)
	describe("base après", base) // inchangé

	// Aliasing : le sous-slice partage la mémoire
	window := base[1:3]
	describe("window", window) // cap = 5, pas 2 !
	window[0] = 99
	describe("base modifié", base)

	// Le piège d'append sur un sous-slice
	part := base[1:3]
	part = append(part, 42) // len(2) < cap(5) : écrit DANS base
	describe("part", part)
	describe("base écrasé", base)

	// La parade : borner la capacité, ou copier franchement
	safe := slices.Clone(base[1:3])
	safe[0] = -1
	describe("safe", safe)
	describe("base intact", base)

	bounded := base[1:3:3]              // len=2 cap=2
	bounded = append(bounded, 7)        // capacité atteinte → réalloue
	describe("bounded", bounded)
	describe("base intact", base)

	// EvensInPlace DÉTRUIT son entrée : à n'utiliser qu'en connaissance de cause
	scratch := []int{1, 2, 3, 4, 5, 6}
	describe("in place", EvensInPlace(scratch))
	describe("scratch", scratch) // les premiers éléments ont été réécrits
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `make([]int, 0, len(nums))` | `len=0`, `cap` suffisante : `append` ne réallouera jamais. Sur un million d'éléments, l'écart est mesurable. |
| `base[1:3:3]` | Le troisième indice **borne la capacité** : `append` est alors forcé de réallouer, et l'original est protégé. |
| `out := nums[:0]` | Astuce idiomatique : longueur nulle, même tableau. On réécrit par-dessus l'original en le parcourant — c'est sûr car on n'écrit jamais plus loin qu'on n'a lu. |
| `cap(window)` = 5 | La capacité s'étend jusqu'à la fin du tableau sous-jacent, pas jusqu'à la fin de la fenêtre. |
| `slices.Clone` | Copie explicite : l'appelant est protégé. |
| Nommage `EvensInPlace` | Une fonction qui modifie son argument **doit** le dire dans son nom. Contrat implicite = bug futur. |
| `scratch` après l'appel | L'entrée est réécrite sur place : c'est le prix de l'absence d'allocation. |

## Erreurs fréquentes

1. **Oublier `s = append(s, x)`.** Le résultat est perdu. `go vet` le détecte.
2. **Croire qu'un sous-slice copie.** Il partage. Toujours.
3. **`append` sur un sous-slice** qui écrase les éléments suivants de l'original. Parade : `s[a:b:b]`.
4. **`copy(dst, src)` avec `dst` non alloué** : copie 0 élément, silencieusement.
5. **Retourner `data[:n]` d'un gros buffer** : empêche le GC de libérer le reste.
6. **Comparer deux slices avec `==`** : ne compile pas (sauf contre `nil`). Utiliser `slices.Equal`.
7. **`s := []int{}` par réflexe** : préférer `var s []int`, qui n'alloue rien.
8. **Modifier un slice pendant un `range`** : la longueur est figée au début de la boucle.
9. **Utiliser un tableau `[N]int` en paramètre de fonction** : copie complète à chaque appel. Passer un slice.

## Bonnes pratiques Go

- **Slice par défaut**, tableau seulement pour une taille fixe imposée par le domaine.
- `var s []T` plutôt que `s := []T{}`.
- **Préallouer** avec `make([]T, 0, n)` dès que la taille finale est connue ou estimable.
- Documenter si une fonction modifie son slice d'entrée — ou, mieux, ne pas le faire.
- `s[a:b:b]` quand on retourne une vue sur un slice qu'on ne contrôle pas.
- Utiliser le paquet `slices` plutôt que de réécrire `Contains`, `Index` ou un tri.
- Ne jamais dépendre du facteur de croissance d'`append` : il n'est pas spécifié.

## Ce que je dois retenir

- Un tableau est une **valeur de taille fixe** qui se copie ; sa taille fait partie de son type.
- Un slice est une **vue** {pointeur, len, cap} sur un tableau sous-jacent.
- **Découper ne copie jamais.** Le partage de mémoire est le comportement normal.
- `append` écrit sur place si `len < cap`, sinon **réalloue et copie** → toujours réaffecter.
- `cap` d'un sous-slice s'étend jusqu'à la **fin du tableau**, d'où le piège d'`append`.
- `s[low:high:max]` borne la capacité ; `slices.Clone` copie franchement.
- Un slice `nil` est utilisable : `len`, `range`, `append` fonctionnent dessus.

➡️ [Exercices](exercices.md)
