# Leçon 3 — Méthodes génériques

> Pendant quatre ans, de Go 1.18 à Go 1.26, une méthode ne pouvait pas déclarer ses propres
> paramètres de type : seul le **type** pouvait en avoir. Go 1.27 (août 2026) lève cette
> restriction. C'est la nouveauté de langage la plus visible depuis les génériques eux-mêmes,
> et elle s'accompagne de limites qu'il faut connaître **avant** de s'en servir — parce
> qu'elles sont la raison pour laquelle on a attendu si longtemps.

> ℹ️ Tout le code de cette leçon exige `go 1.27` (ou plus) dans `go.mod`. Avec une directive
> plus ancienne, le compilateur refuse la syntaxe, même si l'outil installé est en 1.27
> (niveau 2, leçon 9 : la ligne `go` est un contrat de langage).

## Objectifs

1. Déclarer et appeler une méthode qui a ses propres paramètres de type.
2. Comprendre pourquoi une méthode générique **ne participe pas** aux interfaces.
3. Choisir entre méthode générique, fonction générique et type générique.

## Explication

### Le problème avant 1.27

Soit une liste générique. On veut une opération `Map` qui transforme chaque élément en un
autre type :

```go
type List[T any] []T

func (l List[T]) Map(f func(T) ???) List[???]   // quel type de sortie ?
```

Le type de sortie n'est **pas** `T` : il dépend de `f`. Il faudrait un second paramètre de
type, propre à l'appel. Jusqu'en 1.26, c'était impossible, et on écrivait une fonction :

```go
func Map[T, U any](l List[T], f func(T) U) List[U]
```

Ce qui fonctionne, mais casse l'enchaînement : `Map(Map(l, f), g)` se lit de l'intérieur vers
l'extérieur, au lieu de `l.Map(f).Map(g)`.

### La syntaxe Go 1.27

```go
func (l List[T]) Map[U any](f func(T) U) List[U] {
	out := make(List[U], 0, len(l))
	for _, v := range l {
		out = append(out, f(v))
	}
	return out
}
```

Les crochets se placent **après le nom de la méthode**, exactement comme pour une fonction.
Le récepteur garde ses propres paramètres (`T`), la méthode ajoute les siens (`U`).

L'appel profite de l'inférence, comme pour une fonction :

```go
l := List[int]{1, 2, 3}
s := l.Map(strconv.Itoa)            // U inféré : string
s2 := l.Map[float64](toFloat)       // instanciation explicite, si nécessaire
```

Une méthode générique peut aussi exister sur un type **non** générique :

```go
type Rows struct{ data [][]string }

// Col décode la colonne i avec la fonction parse fournie.
func (r *Rows) Col[T any](i int, parse func(string) (T, error)) ([]T, error)

ints, err := rows.Col(1, strconv.Atoi)   // T = int
```

### La limite fondamentale : pas d'interface

C'est la règle à retenir de la leçon :

> **Une méthode générique ne fait pas partie de l'ensemble de méthodes de son type.
> Elle ne peut donc satisfaire aucune interface.**

Deux conséquences, vérifiables :

```go
// 1. Une interface ne peut pas DÉCLARER de méthode générique.
type Mapper interface {
	Map[U any](func(int) U)       // ✗ interface method must have no type parameters
}

// 2. Une méthode générique ne satisfait pas une interface ordinaire,
//    même si une instanciation aurait la bonne signature.
type Box struct{ v int }
func (b Box) Conv[T ~int | ~int64]() T { return T(b.v) }

type Converter interface{ Conv() int }
var _ Converter = Box{}
// ✗ Box does not implement Converter (wrong type for method Conv)
//       have Conv[T ~int | ~int64]() T
//       want Conv() int
```

**Pourquoi ?** Rappel du niveau 2, leçon 4 : un appel à travers une interface est résolu
**à l'exécution** — on cherche la méthode dans une table construite à la compilation pour le
type dynamique. Pour une méthode générique, cette table devrait contenir **une entrée par
instanciation possible** : `Conv[int]`, `Conv[int64]`, `Conv[MyInt]`, `Conv[Duration]`… dont
la liste n'est connue qu'en voyant **tout** le programme, y compris des paquets compilés
séparément. Ou alors il faudrait générer du code pendant l'exécution, ce que Go refuse.
C'est cette impasse qui a retardé la fonctionnalité de quatre ans : la solution retenue est
de ne **pas** essayer.

Conséquences pratiques à connaître :

| Mécanisme | Voit-il les méthodes génériques ? |
|---|---|
| appel direct `x.Map(f)` sur un type concret | **oui** |
| interface, assertion de type `x.(interface{ … })` | **non** |
| `reflect` (`NumMethod`, `MethodByName`, niveau 9) | **non** |
| `fmt` (qui cherche `String()` par assertion) | **non** — un `String[T any]()` ne sera jamais appelé |

### Les valeurs de méthode doivent être instanciées

```go
f := l.Map                 // ✗ cannot use generic function l.Map without instantiation
f := l.Map[string]         // ✓ func(func(int) string) List[string]
g := (*Rows).Col[int]      // ✓ expression de méthode : func(*Rows, int, func(string) (int, error)) ([]int, error)
```

Même règle que pour une fonction générique (leçon 1) : une valeur doit avoir un type
concret.

### Méthode générique ou fonction générique ?

La nouveauté ne rend pas les fonctions obsolètes. Choisir avec cette grille :

| Question | Si oui |
|---|---|
| L'opération est-elle **symétrique** entre deux valeurs (`Union(a, b)`, `Equal(a, b)`) ? | **fonction** |
| Doit-elle accepter **plusieurs types** de « récepteur » (tous les slices, pas seulement `List`) ? | **fonction** — `slices.Index` marche sur tout `[]E`, une méthode seulement sur `List` |
| Doit-elle satisfaire une interface ? | ni l'une ni l'autre : une méthode **non** générique |
| L'opération est-elle naturellement **attachée** au type, et enchaînée ? | **méthode générique** |
| Accède-t-elle à des champs non exportés depuis un autre paquet ? | **méthode** (une fonction du même paquet le peut aussi) |

La bibliothèque standard, écrite avant 1.27, est presque entièrement en fonctions — et le
restera, par compatibilité (niveau 13, leçon 6). Le paquet `slices` (leçon 4) opère sur des
slices **ordinaires**, pas sur un type `List` : une fonction y est le bon choix, et le
resterait même avec les méthodes génériques.

### Le piège de la chaîne trop longue

L'enchaînement est le principal attrait, et le principal risque :

```go
total := orders.Filter(isPaid).Map(toLines).Flatten().Filter(inStock).Fold(0, addPrice)
```

Lisible ? Peut-être. Débogable ? Mal : pas de variable intermédiaire à inspecter, une
allocation par étape, et aucun moyen de s'arrêter au premier élément. La culture Go préfère
une boucle explicite dès que la chaîne dépasse deux maillons. Les **itérateurs** (leçon 5)
résolvent la question des allocations ; aucune syntaxe ne résout celle de la lisibilité.

## Exemple

```go
package main

import (
	"errors"
	"fmt"
	"strconv"
	"strings"
)

// Table est un tableau de chaînes, typiquement lu depuis un CSV.
// Le type n'est PAS générique ; seules ses méthodes de décodage le sont.
type Table struct {
	header []string
	rows   [][]string
}

func NewTable(lines []string) (*Table, error) {
	if len(lines) == 0 {
		return nil, errors.New("table vide")
	}
	t := &Table{header: strings.Split(lines[0], ",")}
	for i, l := range lines[1:] {
		row := strings.Split(l, ",")
		if len(row) != len(t.header) {
			return nil, fmt.Errorf("ligne %d : %d colonnes, %d attendues", i+2, len(row), len(t.header))
		}
		t.rows = append(t.rows, row)
	}
	return t, nil
}

// Column décode la colonne nommée avec parse. T est propre à chaque appel :
// la même table fournit des []int, des []float64, des []bool…
func (t *Table) Column[T any](name string, parse func(string) (T, error)) ([]T, error) {
	idx := -1
	for i, h := range t.header {
		if h == name {
			idx = i
			break
		}
	}
	if idx < 0 {
		return nil, fmt.Errorf("colonne %q introuvable", name)
	}
	out := make([]T, 0, len(t.rows))
	for i, row := range t.rows {
		v, err := parse(row[idx])
		if err != nil {
			return nil, fmt.Errorf("colonne %q, ligne %d : %w", name, i+2, err)
		}
		out = append(out, v)
	}
	return out, nil
}

// List montre l'enchaînement : Map change le type des éléments.
type List[T any] []T

func (l List[T]) Map[U any](f func(T) U) List[U] {
	out := make(List[U], 0, len(l))
	for _, v := range l {
		out = append(out, f(v))
	}
	return out
}

func (l List[T]) Filter(keep func(T) bool) List[T] { // pas besoin de paramètre propre
	var out List[T]
	for _, v := range l {
		if keep(v) {
			out = append(out, v)
		}
	}
	return out
}

func parseFloat(s string) (float64, error) { return strconv.ParseFloat(s, 64) }

func main() {
	t, err := NewTable([]string{
		"produit,stock,prix,actif",
		"clavier,12,49.90,true",
		"souris,0,19.50,true",
		"écran,3,189.00,false",
	})
	if err != nil {
		fmt.Println(err)
		return
	}

	stock, _ := t.Column("stock", strconv.Atoi)      // T = int
	prix, _ := t.Column("prix", parseFloat)          // T = float64
	actif, _ := t.Column("actif", strconv.ParseBool) // T = bool
	fmt.Println(stock, prix, actif)

	if _, err := t.Column("stock", strconv.ParseBool); err != nil {
		fmt.Println("erreur :", err)
	}
	if _, err := t.Column("poids", strconv.Atoi); err != nil {
		fmt.Println("erreur :", err)
	}

	labels := List[int](stock).
		Filter(func(n int) bool { return n > 0 }).
		Map(func(n int) string { return strconv.Itoa(n) + " u." })
	fmt.Printf("%q\n", labels)

	toStr := List[float64](prix).Map[string] // valeur de méthode : instanciation obligatoire
	fmt.Printf("%T\n", toStr)
}
```

Sortie :

```
[12 0 3] [49.9 19.5 189] [true true false]
erreur : colonne "stock", ligne 2 : strconv.ParseBool: parsing "12": invalid syntax
erreur : colonne "poids" introuvable
["12 u." "3 u."]
func(func(float64) string) main.List[string]
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `Table` non générique | Le type n'a aucune raison d'être paramétré : ses données sont des chaînes. C'est **chaque décodage** qui a un type. |
| `Column[T any]` | Avant 1.27, il aurait fallu une fonction `Column[T any](t *Table, …)` : même logique, appel moins naturel. |
| `t.Column("stock", strconv.Atoi)` | `T` inféré depuis le type de `strconv.Atoi` : `func(string) (int, error)`. |
| `parseFloat` | `strconv.ParseFloat` prend deux arguments ; il faut l'adapter à `func(string) (T, error)`. |
| `Filter` sans paramètre propre | Le type de sortie est celui d'entrée : le `T` du récepteur suffit. Ne pas ajouter de paramètre inutile. |
| `List[int](stock)` | Conversion d'un `[]int` en `List[int]` — même sous-jacent, conversion gratuite. |
| `.Map[string]` | Valeur de méthode : il faut instancier, on obtient une fonction ordinaire. |

## Erreurs fréquentes

1. **Déclarer une méthode générique dans une interface** : refusé par le compilateur.
2. **Croire qu'une méthode générique satisfait une interface**, par exemple écrire
   `String[T any]() string` et s'étonner que `fmt` ne l'appelle pas.
3. **Oublier `go 1.27` dans `go.mod`** et chercher l'erreur dans le code.
4. **Transformer toutes les fonctions génériques en méthodes** « parce qu'on peut » : une
   méthode sur `List[T]` ne marche plus sur un `[]T` ordinaire.
5. **Des chaînes de cinq maillons** qui allouent cinq slices et ne se déboguent pas.
6. **Prendre une valeur de méthode sans instancier.**

## Bonnes pratiques Go

- Une méthode générique pour une opération **attachée** au type et dont le type de sortie
  dépend de l'appel (décodage, conversion, projection).
- Une **fonction** pour les opérations symétriques, ou qui doivent s'appliquer à des types
  ordinaires (`[]T`, `map[K]V`).
- Jamais de méthode générique là où une interface devra être satisfaite.
- Garder les chaînes courtes ; au-delà de deux maillons, nommer les résultats intermédiaires
  ou écrire une boucle.
- Dans une bibliothèque publique, se souvenir qu'ajouter une méthode générique **exige** que
  tous les utilisateurs passent à Go 1.27 : c'est un changement de version minimale.

## Ce que je dois retenir

- Depuis Go 1.27 : `func (r R) Nom[U any](…)`, les crochets après le nom.
- L'inférence fonctionne comme pour les fonctions.
- Une méthode générique **n'est pas dans l'ensemble de méthodes** : ni interface, ni `reflect`, ni `fmt`.
- Une valeur de méthode générique doit être instanciée.
- Fonction pour ce qui est symétrique ou s'applique aux types ordinaires ; méthode pour ce qui
  est attaché au type.
- La limite vient du dispatch dynamique des interfaces : on ne peut pas tabuler une infinité
  d'instanciations.

---

➡️ [Exercices](exercices.md)
