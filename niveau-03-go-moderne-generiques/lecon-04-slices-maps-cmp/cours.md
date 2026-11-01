# Leçon 4 — `slices`, `maps`, `cmp`

> Les trois paquets de cette leçon sont le premier **fruit** des génériques : ajoutés entre Go
> 1.21 et 1.23, ils remplacent des centaines de boucles écrites à la main dans chaque base de
> code. Les lire, c'est aussi lire du code générique écrit par l'équipe Go — un modèle de
> conception à imiter.

## Objectifs

1. Connaître les fonctions de `slices`, `maps` et `cmp` qu'on utilise tous les jours.
2. Comprendre **lesquelles modifient** le slice reçu, et ce que ça implique pour l'aliasing.
3. Trier sur plusieurs critères avec `slices.SortFunc` et `cmp.Or`.
4. Savoir quand une boucle reste préférable.

## Explication

### `slices` : les opérations courantes

Le paquet travaille sur des slices **ordinaires**, via une signature récurrente :

```go
func Index[S ~[]E, E comparable](s S, v E) int
```

Deux paramètres de type : `S` est le type slice, `E` le type d'élément. Pourquoi pas
simplement `s []E` ? Pour **préserver le type défini** de l'appelant : si `type IDs []int`,
`slices.Clone(ids)` rend un `IDs`, pas un `[]int`. Retenir ce patron `S ~[]E` : c'est la
bonne façon d'écrire une fonction générique qui rend un slice.

Les fonctions se rangent en familles :

| Famille | Fonctions | Modifie `s` ? |
|---|---|---|
| Recherche | `Contains`, `Index`, `IndexFunc`, `ContainsFunc` | non |
| Recherche triée | `BinarySearch`, `BinarySearchFunc` — **suppose** `s` trié | non |
| Comparaison | `Equal`, `EqualFunc`, `Compare` | non |
| Extremums | `Min`, `Max`, `MinFunc`, `MaxFunc` — **paniquent** sur un slice vide | non |
| Tri | `Sort`, `SortFunc`, `SortStableFunc`, `IsSorted` | **oui**, en place |
| Réarrangement | `Reverse` | **oui**, en place |
| Copie | `Clone`, `Concat`, `Repeat` | non — nouveau tableau |
| Édition | `Insert`, `Delete`, `DeleteFunc`, `Replace`, `Compact`, `CompactFunc` | **oui** — et rendent le nouveau slice |
| Capacité | `Grow`, `Clip` | rendent un nouveau slice |

### Les fonctions d'édition : la règle d'`append`

`Insert`, `Delete`, `Compact` suivent **exactement** la règle d'`append` (niveau 1, leçon 6) :
elles travaillent dans le tableau existant si la capacité le permet, et **rendent** le slice
à utiliser. On écrit donc toujours :

```go
s = slices.Delete(s, i, j)   // jamais slices.Delete(s, i, j) seul
```

Et l'ancien slice ? Mesuré :

```go
s := []int{1, 2, 3, 4, 5}
t := slices.Delete(s, 1, 3)
fmt.Println(t, s)   // [1 4 5] [1 4 5 0 0]
```

`t` et `s` partagent le tableau. Depuis Go 1.22, `Delete` **met à zéro** les cases libérées
(`0 0`) : sans cela, une `[]*Image` garderait des références vers des images supprimées, et
le ramasse-miettes ne pourrait pas les libérer. Mais toute autre variable qui pointait sur le
même tableau voit désormais des zéros. Règle : **après une édition, n'utiliser que le slice
retourné.**

### Trier : `Sort`, `SortFunc`, et la fonction de comparaison

`slices.Sort` trie tout type `cmp.Ordered`. Pour tout le reste, ou pour un autre ordre,
`slices.SortFunc` prend une fonction de comparaison qui rend un **entier** :

- négatif si `a` vient avant `b` ;
- zéro si équivalents ;
- positif si `a` vient après `b`.

C'est exactement ce que rend `cmp.Compare`. On écrit donc presque toujours :

```go
slices.SortFunc(users, func(a, b User) int {
	return cmp.Compare(a.Age, b.Age)
})
```

**Ne pas écrire `return a.Age - b.Age`** : sur des entiers proches des bornes, la
soustraction déborde (niveau 1, leçon 3) et inverse le signe. `cmp.Compare` ne déborde
jamais.

Pour un ordre **décroissant**, inverser les arguments : `cmp.Compare(b.Age, a.Age)`.

### Plusieurs critères : `cmp.Or`

`cmp.Or` (Go 1.22) rend son premier argument **non nul**. Combiné à `cmp.Compare`, c'est
l'idiome du tri multicritère :

```go
slices.SortFunc(users, func(a, b User) int {
	return cmp.Or(
		cmp.Compare(a.City, b.City),   // d'abord par ville
		cmp.Compare(b.Age, a.Age),     // puis par âge décroissant
		cmp.Compare(a.Name, b.Name),   // enfin par nom
	)
})
```

Le premier critère qui départage gagne. Lisible, sans `if` imbriqués. `cmp.Or` sert aussi
pour les valeurs par défaut : `port := cmp.Or(os.Getenv("PORT"), "8080")`.

### Tri stable

`slices.SortFunc` n'est **pas stable** : deux éléments équivalents peuvent changer d'ordre
relatif. `slices.SortStableFunc` l'est, un peu plus lentement. On a besoin de stabilité quand
on trie successivement sur plusieurs critères, ou quand l'ordre d'origine porte une
information (ordre d'arrivée). Avec `cmp.Or` et un dernier critère discriminant, on s'en passe
généralement.

### Les chaînes ne se trient pas « alphabétiquement »

```go
words := []string{"Éclair", "abricot", "Banane", "cerise"}
slices.Sort(words)   // [Banane abricot cerise Éclair]
```

`<` sur des chaînes compare les **octets** : toutes les majuscules ASCII (65–90) précèdent
toutes les minuscules (97–122), et `É` (encodé `0xC3 0x89`) vient après tout l'ASCII. Pour un
tri insensible à la casse, comparer `strings.ToLower` des deux. Pour un vrai tri
**linguistique** (où `é` se range près de `e`), il faut le paquet `golang.org/x/text/collate`
— la bibliothèque standard ne le fait pas.

### `maps` : moins de fonctions, mêmes principes

| Fonction | Rôle |
|---|---|
| `maps.Clone(m)` | copie **superficielle** ; `Clone(nil)` rend `nil` |
| `maps.Copy(dst, src)` | copie les entrées de `src` dans `dst` (écrase) |
| `maps.Equal(a, b)`, `EqualFunc` | mêmes clés, mêmes valeurs |
| `maps.DeleteFunc(m, f)` | supprime les entrées pour lesquelles `f` est vrai |
| `maps.Keys`, `maps.Values`, `maps.All` | rendent des **itérateurs** — leçon 5 |

« Superficielle » : cloner une `map[string][]int` duplique la table, mais chaque valeur est
un slice qui **partage** son tableau avec l'original. Le tableau *[Copié ou
partagé ?](../../ressources/copie-ou-partage.md)* s'applique entrée par entrée.

Supprimer pendant une boucle `range` sur une map est autorisé en Go, mais `maps.DeleteFunc`
dit l'intention en une ligne.

### `cmp` : trois choses

| | |
|---|---|
| `cmp.Ordered` | la contrainte (leçon 2) |
| `cmp.Compare(a, b) int`, `cmp.Less(a, b) bool` | comparaison à ordre total, NaN compris |
| `cmp.Or(vals...)` | le premier non nul |

### Ce que ces paquets remplacent

| Avant (Go ≤ 1.20) | Maintenant |
|---|---|
| `sort.Slice(s, func(i, j int) bool { … })` | `slices.SortFunc(s, func(a, b T) int { … })` |
| `sort.Ints(s)`, `sort.Strings(s)` | `slices.Sort(s)` |
| `sort.SearchInts` | `slices.BinarySearch` |
| boucle `for` + `==` | `slices.Contains`, `slices.Index` |
| `append([]T(nil), s...)` | `slices.Clone(s)` |
| `append(s[:i], s[i+1:]...)` | `slices.Delete(s, i, i+1)` |
| `reflect.DeepEqual` sur deux slices | `slices.Equal` |

`sort.Slice` passait des **indices** et utilisait la réflexion ; `slices.SortFunc` passe des
**valeurs**, est typé, et plus rapide. Le paquet `sort` n'est pas déprécié, mais tout code
neuf utilise `slices`.

### Quand garder la boucle

`slices` ne rend pas les boucles obsolètes. Une boucle reste préférable quand :

- on fait **plusieurs choses** en un passage (compter, filtrer et sommer à la fois) ;
- on a besoin de **s'arrêter** sur une condition complexe ;
- la fonction anonyme passée à `IndexFunc` serait plus longue que la boucle elle-même.

Le critère n'est pas « moins de lignes » mais « intention plus visible ».

## Exemple

```go
package main

import (
	"cmp"
	"fmt"
	"maps"
	"slices"
	"strings"
)

type Employee struct {
	Name   string
	Team   string
	Salary int
	Skills []string
}

func main() {
	staff := []Employee{
		{"Inès", "back", 52000, []string{"go", "sql"}},
		{"Marc", "front", 48000, []string{"ts"}},
		{"Awa", "back", 61000, []string{"go", "k8s"}},
		{"Zoé", "front", 48000, []string{"ts", "go"}},
		{"Bilal", "back", 52000, []string{"go"}},
	}

	// Tri multicritère : équipe, puis salaire décroissant, puis nom.
	slices.SortFunc(staff, func(a, b Employee) int {
		return cmp.Or(
			cmp.Compare(a.Team, b.Team),
			cmp.Compare(b.Salary, a.Salary),
			cmp.Compare(a.Name, b.Name),
		)
	})
	for _, e := range staff {
		fmt.Printf("%-6s %-5s %d\n", e.Name, e.Team, e.Salary)
	}

	// Recherche par prédicat.
	i := slices.IndexFunc(staff, func(e Employee) bool { return slices.Contains(e.Skills, "k8s") })
	fmt.Println("k8s :", staff[i].Name)

	// Extremum par fonction : MaxFunc rend le PREMIER maximum en cas d'égalité.
	richest := slices.MaxFunc(staff, func(a, b Employee) int { return cmp.Compare(a.Salary, b.Salary) })
	fmt.Println("max :", richest.Name)

	// Supprimer en place les profils sans Go — et ne garder que le slice rendu.
	gophers := slices.DeleteFunc(slices.Clone(staff), func(e Employee) bool {
		return !slices.Contains(e.Skills, "go")
	})
	fmt.Println("gophers :", len(gophers), "sur", len(staff))

	// Compter les compétences, puis supprimer les rares.
	count := map[string]int{}
	for _, e := range staff {
		for _, s := range e.Skills {
			count[s]++
		}
	}
	frequent := maps.Clone(count)
	maps.DeleteFunc(frequent, func(_ string, n int) bool { return n < 2 })
	fmt.Println(count, frequent) // fmt trie les clés de map à l'affichage

	// Dédoublonner : Compact n'enlève que les doublons ADJACENTS, d'où le tri préalable.
	var all []string
	for _, e := range staff {
		all = append(all, e.Skills...)
	}
	slices.Sort(all)
	all = slices.Compact(all)
	fmt.Println(strings.Join(all, ","))

	// Recherche dichotomique dans un slice trié.
	pos, found := slices.BinarySearch(all, "rust")
	fmt.Println(pos, found)
}
```

Sortie :

```
Awa    back  61000
Bilal  back  52000
Inès   back  52000
Marc   front 48000
Zoé    front 48000
k8s : Awa
max : Awa
gophers : 4 sur 5
map[go:4 k8s:1 sql:1 ts:2] map[go:4 ts:2]
go,k8s,sql,ts
2 false
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `cmp.Or(cmp.Compare…, …)` | Trois critères, aucun `if`. Le dernier (le nom) rend l'ordre **total** : la stabilité ne compte plus. |
| `cmp.Compare(b.Salary, a.Salary)` | Arguments inversés = ordre décroissant. |
| `slices.IndexFunc` + `slices.Contains` | Une recherche dans une recherche, sans boucle explicite. On ne vérifie pas `i == -1` ici parce qu'on **sait** que les données contiennent un `k8s` — en vrai code, il le faudrait. |
| `slices.DeleteFunc(slices.Clone(staff), …)` | `DeleteFunc` modifie en place : sans `Clone`, `staff` serait abîmé (et ses dernières cases mises à zéro). |
| `maps.Clone` puis `DeleteFunc` | On garde `count` intact. |
| `slices.Sort` puis `Compact` | `Compact` fonctionne comme la commande `uniq` : seulement les voisins. |
| `BinarySearch` → `2 false` | Pas trouvé : `2` est la position où il **faudrait** insérer `"rust"` pour garder l'ordre. |

## Erreurs fréquentes

1. **Ignorer le résultat** de `Delete`, `Insert`, `Compact` : `slices.Delete(s, 0, 1)` seul ne
   fait pas ce qu'on croit.
2. **Continuer à utiliser l'ancien slice** après une édition : il contient des zéros en fin.
3. **`DeleteFunc` sur les données de l'appelant** sans `Clone` dans une fonction qui n'est
   pas censée les modifier.
4. **`return a - b`** dans une fonction de comparaison : débordement possible.
5. **`Compact` sur un slice non trié** en espérant un dédoublonnage complet.
6. **`BinarySearch` sur un slice non trié** : aucun plantage, juste des réponses fausses.
7. **`slices.Max` sur un slice vide** : panique.
8. **Croire que `maps.Clone` est une copie profonde.**
9. **Trier des chaînes françaises avec `slices.Sort`** et s'étonner de `Éclair` en dernier.

## Bonnes pratiques Go

- `slices` et `maps` dans tout code neuf ; `sort` seulement pour maintenir l'existant.
- `cmp.Compare` dans toutes les fonctions de comparaison ; `cmp.Or` pour enchaîner les critères.
- Toujours **réaffecter** le résultat d'une fonction d'édition.
- Signature `func F[S ~[]E, E any](s S) S` quand on écrit soi-même une fonction qui rend un slice.
- Une boucle quand elle est plus claire que la composition de trois fonctions.
- Lire le code source de `slices` (`go doc -src slices.Compact`) : c'est court et instructif.

## Ce que je dois retenir

- `slices` : recherche, tri, édition, copie ; `maps` : clone, copie, égalité, suppression ;
  `cmp` : `Compare`, `Less`, `Or`, `Ordered`.
- Les fonctions d'édition suivent la règle d'`append` : **réaffecter**, et ne plus toucher l'ancien slice.
- `slices.Delete` remet à zéro les cases libérées (Go 1.22+).
- Comparateur : négatif / zéro / positif, écrit avec `cmp.Compare`, enchaîné avec `cmp.Or`.
- `S ~[]E` préserve le type défini de l'appelant.
- `Compact` = voisins seulement ; `BinarySearch` = slice trié seulement.
- Le tri de chaînes est un tri d'**octets**.

---

🧵 **Fil rouge :** `slices.Clone`, `maps.Clone` et l'ancien slice après `Delete` sont trois nouvelles lignes pour *[Copié ou partagé ?](../../ressources/copie-ou-partage.md)*.

➡️ [Exercices](exercices.md)
