# Leçon 1 — Paramètres de type

> Les génériques sont arrivés en Go 1.18 (mars 2022), treize ans après le langage. Ce retard
> n'était pas un oubli : l'équipe Go a refusé plusieurs conceptions jugées trop coûteuses en
> complexité. Ce qu'on a obtenu est volontairement **modeste**. Cette leçon apprend à s'en
> servir, et tout autant à **ne pas** s'en servir.

## Objectifs

1. Écrire une fonction générique et un type générique.
2. Comprendre ce que fait l'**inférence de type**, et quand elle échoue.
3. Distinguer ce qu'apporte un générique de ce qu'apporte une interface.
4. Savoir refuser un générique qui n'apporte rien.

## Explication

### Le problème que les génériques résolvent

Au niveau 1, on a écrit des fonctions comme celle-ci :

```go
func ContainsInt(xs []int, v int) bool {
	for _, x := range xs {
		if x == v {
			return true
		}
	}
	return false
}
```

Puis la même pour `string`. Puis pour `float64`. Le corps est **identique**, seul le type
change. Trois solutions existaient avant 1.18, toutes mauvaises :

| Solution | Défaut |
|---|---|
| Copier-coller par type | duplication, corrections à faire N fois |
| `func Contains(xs []any, v any) bool` | un `[]int` n'est **pas** un `[]any` (il faut le recopier), et on perd tout contrôle de type |
| Générer le code (`go generate`) | un outil de plus, du code illisible à relire |

### Un paramètre de type

```go
func Contains[T comparable](xs []T, v T) bool {
	for _, x := range xs {
		if x == v {
			return true
		}
	}
	return false
}
```

Entre crochets, **avant** les paramètres ordinaires : la **liste des paramètres de type**.

- `T` est un **paramètre de type** : un nom qui désigne un type, à fixer plus tard.
- `comparable` est sa **contrainte** : l'ensemble des types autorisés pour `T`. Ici, ceux qui
  acceptent `==`. Sans elle, `x == v` ne compilerait pas : le compilateur doit pouvoir
  vérifier le corps de la fonction **une seule fois**, pour tous les `T` possibles.

C'est la différence de fond avec les *templates* C++ : en Go, le corps est vérifié contre la
contrainte, pas contre chaque type réel. Une fonction générique qui compile fonctionne pour
**tous** les types qui satisfont la contrainte.

### Instancier

```go
Contains[int]([]int{1, 2, 3}, 2)        // instanciation explicite
Contains([]int{1, 2, 3}, 2)             // T inféré : int
Contains([]string{"a", "b"}, "c")       // T inféré : string
```

Fournir un argument de type à `Contains` s'appelle l'**instanciation**. Le résultat,
`Contains[int]`, est une fonction ordinaire, non générique — on peut même la ranger dans une
variable :

```go
f := Contains[string]            // f est de type func([]string, string) bool
g := Contains                    // ✗ ne compile pas : une fonction générique n'est pas une valeur
```

### L'inférence de type

Dans 95 % des cas on n'écrit pas les crochets à l'appel : le compilateur **infère** `T` à
partir des arguments. Le mécanisme est une unification : il aligne les types des paramètres
(`[]T`, `T`) sur ceux des arguments (`[]int`, `int`) et en déduit `T = int`.

L'inférence échoue dans deux situations à connaître :

```go
// 1. T n'apparaît que dans le résultat : aucun argument pour le déduire.
func Zero[T any]() T { var z T; return z }
Zero()          // ✗ cannot infer T
Zero[int]()     // ✓

// 2. Des arguments contradictoires.
Contains([]int{1, 2}, int64(2))   // ✗ T vaudrait int ET int64
```

Et une situation qui **réussit** de façon surprenante, avec des constantes non typées
(niveau 1, leçon 2) :

```go
func Max[T int | float64](a, b T) T { … }
Max(1, 2.5)     // T = float64 : les deux constantes sont non typées,
                // le compilateur choisit le type par défaut le plus « large »
```

### Un type générique

Les types aussi peuvent prendre des paramètres :

```go
type Stack[T any] struct {
	items []T
}

func (s *Stack[T]) Push(v T) {
	s.items = append(s.items, v)
}

func (s *Stack[T]) Pop() (T, bool) {
	var zero T
	if len(s.items) == 0 {
		return zero, false
	}
	v := s.items[len(s.items)-1]
	s.items = s.items[:len(s.items)-1]
	return v, true
}
```

Trois points de syntaxe :

1. Le récepteur s'écrit `*Stack[T]` : le type n'existe pas sans son paramètre. On peut
   renommer le paramètre dans le récepteur (`*Stack[E]`), mais on ne le fait pas.
2. `var zero T` est **la** façon d'obtenir la zéro-valeur d'un type inconnu. Il n'y a pas
   d'autre écriture : `nil` n'est pas une valeur valide pour un `T` quelconque, `0` non plus.
3. Pour **utiliser** le type, il faut l'instancier : `Stack[int]{}`, `var s Stack[string]`.
   L'inférence ne s'applique pas aux littéraux de type.

`Stack[int]` et `Stack[string]` sont **deux types distincts**, aussi distincts que `int` et
`string`. On ne peut pas affecter l'un à l'autre, et on ne peut pas mettre les deux dans un
même slice — sauf via une interface.

### Ce que coûte un générique à l'exécution

Rien de spectaculaire, dans un sens comme dans l'autre. Le compilateur Go n'écrit **ni** une
copie par type (comme C++), **ni** une seule version qui manipule tout par pointeur (comme
Java). Il regroupe les types par **forme mémoire** (*GC shape*) : tous les pointeurs partagent
une même version compilée, `int` et `int64` en ont chacun une. Les appels de méthode sur `T`
passent par un dictionnaire, comme une interface.

Conséquence pratique : **un générique n'est pas une optimisation.** On l'écrit pour la
sûreté de type et l'absence de duplication, jamais en espérant aller plus vite qu'une
interface. Si la vitesse compte, on mesure (niveau 11).

### Générique ou interface ?

C'est la question qui compte vraiment, et la réponse officielle tient en une règle :

> **Si on n'a besoin que d'appeler des méthodes sur une valeur, une interface suffit.**

```go
// Inutilement générique :
func PrintAll[T fmt.Stringer](xs []T) { for _, x := range xs { fmt.Println(x.String()) } }

// Équivalent, plus simple :
func PrintOne(s fmt.Stringer) { fmt.Println(s.String()) }
```

Le générique se justifie quand **le type lui-même** doit être préservé dans la signature :

| Situation | Générique justifié ? |
|---|---|
| Une fonction sur des slices, maps ou channels de n'importe quel type d'élément | **oui** — `[]T` en entrée, `[]T` en sortie |
| Une structure de données (pile, file, arbre, cache) | **oui** — `Pop()` doit rendre un `T`, pas un `any` |
| Le résultat doit avoir le **même type** que l'argument | **oui** — `Max(a, b T) T` |
| On appelle seulement des méthodes | **non** — une interface |
| Le code ne marche que pour deux types | **non** — deux fonctions sont plus lisibles |
| « Au cas où on en aurait besoin plus tard » | **non** — jamais |

Le signal d'alarme : un paramètre de type qui n'apparaît **qu'une fois** dans la signature.
`func F[T any](x T)` n'apporte rien par rapport à `func F(x any)` — sauf de la complexité.

### `any` comme contrainte n'est pas `any` comme type

```go
func A(x any)          // x est une INTERFACE : couple (type, valeur), boxing possible
func B[T any](x T)     // x est un T CONCRET : à l'appel B(3), x est un vrai int
```

Dans `A`, on a perdu le type : il faut une assertion pour retrouver l'`int`. Dans `B`, le
type est connu du compilateur à chaque site d'appel. Même mot, deux rôles. Le mot `any` est
un alias de `interface{}` (niveau 2, leçon 4) : en position de contrainte, il veut dire
« aucune exigence ».

## Exemple

```go
package main

import (
	"errors"
	"fmt"
	"strconv"
	"strings"
)

// Map applique f à chaque élément. Deux paramètres de type : l'entrée et la sortie
// peuvent différer, et la signature le dit.
func Map[T, U any](xs []T, f func(T) U) []U {
	out := make([]U, 0, len(xs))
	for _, x := range xs {
		out = append(out, f(x))
	}
	return out
}

// Filter garde les éléments pour lesquels keep est vrai. Le type de sortie
// est EXACTEMENT celui d'entrée : c'est ce qu'aucune interface ne sait exprimer.
func Filter[T any](xs []T, keep func(T) bool) []T {
	var out []T
	for _, x := range xs {
		if keep(x) {
			out = append(out, x)
		}
	}
	return out
}

// Stack est une pile typée : Pop rend un T, pas un any.
type Stack[T any] struct {
	items []T
}

func (s *Stack[T]) Push(v T) { s.items = append(s.items, v) }

func (s *Stack[T]) Pop() (T, error) {
	var zero T
	if len(s.items) == 0 {
		return zero, errors.New("pile vide")
	}
	v := s.items[len(s.items)-1]
	s.items = s.items[:len(s.items)-1]
	return v, nil
}

func (s *Stack[T]) Len() int { return len(s.items) }

// evalRPN évalue une expression en notation polonaise inverse : "3 4 + 2 *" → 14.
func evalRPN(expr string) (int, error) {
	var st Stack[int]
	for _, tok := range strings.Fields(expr) {
		switch tok {
		case "+", "-", "*":
			b, errB := st.Pop()
			a, errA := st.Pop()
			if err := errors.Join(errA, errB); err != nil {
				return 0, fmt.Errorf("opérateur %q : %w", tok, err)
			}
			switch tok {
			case "+":
				st.Push(a + b)
			case "-":
				st.Push(a - b)
			case "*":
				st.Push(a * b)
			}
		default:
			n, err := strconv.Atoi(tok)
			if err != nil {
				return 0, fmt.Errorf("jeton %q : %w", tok, err)
			}
			st.Push(n)
		}
	}
	if st.Len() != 1 {
		return 0, fmt.Errorf("expression mal formée : %d valeurs restantes", st.Len())
	}
	v, _ := st.Pop()
	return v, nil
}

func main() {
	words := []string{"go", "rust", "c", "zig", "python"}

	lengths := Map(words, func(s string) int { return len(s) }) // T=string, U=int inférés
	fmt.Println(lengths)

	short := Filter(words, func(s string) bool { return len(s) <= 3 })
	fmt.Println(short)

	upper := Map(short, strings.ToUpper) // une fonction ordinaire convient aussi
	fmt.Printf("%q\n", upper)

	for _, e := range []string{"3 4 + 2 *", "1 +", "2 x *"} {
		v, err := evalRPN(e)
		if err != nil {
			fmt.Printf("%-10s → erreur : %v\n", e, err)
			continue
		}
		fmt.Printf("%-10s → %d\n", e, v)
	}

	var names Stack[string]
	names.Push("alice")
	fmt.Printf("%T %d\n", names, names.Len())
}
```

Sortie :

```
[2 4 1 3 6]
[go c zig]
["GO" "C" "ZIG"]
3 4 + 2 *  → 14
1 +        → erreur : opérateur "+" : pile vide
2 x *      → erreur : jeton "x" : strconv.Atoi: parsing "x": invalid syntax
main.Stack[string] 1
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `Map[T, U any]` | Deux paramètres de type qui partagent une contrainte : on peut les grouper comme des paramètres ordinaires. |
| `make([]U, 0, len(xs))` | On connaît la taille finale : préallocation (niveau 1, leçon 6). Un générique reste du Go ordinaire. |
| `Filter` retourne `[]T` | Le point clé : l'appelant récupère un `[]string`, pas un `[]any` à convertir. |
| `Map(short, strings.ToUpper)` | `strings.ToUpper` est de type `func(string) string` : `U` est inféré à `string`. |
| `var zero T` | La seule manière d'écrire « la zéro-valeur de T ». |
| `var st Stack[int]` | La zéro-valeur d'une `Stack` est utilisable : son slice nil accepte `append`. |
| `errors.Join(errA, errB)` | Rend `nil` si les deux sont nil — une seule vérification pour deux `Pop`. |
| `%T` sur `names` | Affiche `main.Stack[string]` : l'argument de type fait partie du nom du type. |

## Erreurs fréquentes

1. **Générique là où une interface suffit** : `func Log[T fmt.Stringer](v T)`. Le signal :
   `T` n'apparaît qu'une fois.
2. **Oublier la contrainte** et s'étonner que `a == b` ou `a < b` ne compile pas avec `any`.
3. **Retourner `nil` pour un `T`** : `return nil, err` ne compile pas si `T` peut être `int`.
   Il faut `var zero T`.
4. **Croire qu'on peut écrire `Stack{}`** : l'inférence ne marche pas sur les littéraux de type.
5. **Essayer de stocker une fonction générique non instanciée** dans une variable.
6. **Penser que `Stack[int]` et `Stack[int64]` sont compatibles.** Ce sont deux types sans
   aucun lien.
7. **Généraliser trop tôt** : écrire `Map`, `Filter`, `Reduce` génériques pour les utiliser
   une seule fois, là où une boucle de quatre lignes était plus claire.

## Bonnes pratiques Go

- **Écrire d'abord la version concrète.** Généraliser seulement quand une deuxième version
  identique apparaît.
- Nommer les paramètres de type d'une lettre majuscule : `T`, `K`/`V` pour clé et valeur,
  `E` pour un élément, `S` pour un type slice. Un nom long n'est justifié que s'il y en a
  beaucoup.
- Laisser l'inférence travailler : écrire `Map(xs, f)`, pas `Map[string, int](xs, f)`.
- Préférer une interface quand on n'a besoin que de comportements.
- Préférer une boucle quand le générique ne sert qu'une fois.
- Ne jamais rendre générique « pour la performance ».

## Ce que je dois retenir

- Les paramètres de type se déclarent **entre crochets** avant les paramètres ordinaires.
- Chaque paramètre de type a une **contrainte**, et le corps est vérifié contre elle, une seule fois.
- L'**inférence** déduit les arguments de type des arguments ordinaires ; elle échoue quand
  `T` n'apparaît que dans le résultat.
- `var zero T` donne la zéro-valeur.
- `Stack[int]` et `Stack[string]` sont des types distincts.
- Un générique se justifie quand **le type doit être préservé** ; sinon, une interface ou du code concret.
- Un générique n'est pas une optimisation.

---

➡️ [Exercices](exercices.md)
