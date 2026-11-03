# Leçon 1 — Bases de `reflect`

> `json.Marshal` reçoit un `any` et produit le bon JSON pour une struct qu'il n'a jamais vue.
> `fmt.Println` affiche n'importe quoi. GORM associe des colonnes à des champs dont il ne connaît pas
> les noms. Tous utilisent le paquet `reflect`, qui permet à un programme d'**examiner ses propres
> valeurs à l'exécution** : leur type, leurs champs, leurs éléments. C'est puissant, lent et fragile.
> Cette leçon pose le modèle, assez précisément pour lire le code de ces bibliothèques et éviter les
> paniques classiques.

## Objectifs

1. Obtenir un `reflect.Type` et un `reflect.Value` à partir d'une valeur `any`.
2. Distinguer **type** (`main.Celsius`) et **sorte** (`Kind` : `float64`).
3. Parcourir une valeur de structure inconnue (slices, maps, pointeurs, structs).
4. Appliquer les trois lois de la réflexion, en particulier : modifier exige une valeur **modifiable**.
5. Reconnaître une `Value` invalide (interface `nil`).

## Explication

### D'où part la réflexion

Une valeur de type interface (`any`) contient deux choses : un **type dynamique** et une **valeur
dynamique** (N2 L4). Le paquet `reflect` donne accès aux deux :

```go
t := reflect.TypeOf(x)   // reflect.Type  : la description du type
v := reflect.ValueOf(x)  // reflect.Value : la valeur, avec son type (v.Type() == t)
```

Les deux fonctions prennent un `any` : `x` est d'abord placé dans une interface, puis examiné. C'est
pourquoi la réflexion commence toujours par une interface.

### Type et `Kind`

Le **type** est le type exact, nommé ou non : `int`, `main.Celsius`, `[]string`, `*main.Book`. La
**sorte** (`Kind`) est la catégorie de base parmi une liste fermée : `Int`, `Float64`, `String`, `Slice`,
`Map`, `Pointer`, `Struct`, `Func`, `Interface`, `Chan`, `Array`, `Bool`…

```go
type Celsius float64
reflect.TypeOf(Celsius(21.5))         // main.Celsius
reflect.TypeOf(Celsius(21.5)).Kind()  // float64
```

Le code réflexif se branche presque toujours sur le `Kind` : il traite **tous** les types de sorte
`float64` de la même façon (`v.Float()`), qu'ils s'appellent `float64`, `Celsius` ou `Prix`. C'est ainsi
que `encoding/json` encode un `Celsius` comme un nombre sans rien savoir de ce type.

Remarque : la sorte `reflect.Pointer` (son ancien nom, `reflect.Ptr`, existe encore) s'affiche `ptr`.

### Lire une `Value`

Chaque sorte a ses méthodes. Les appeler sur la mauvaise sorte **panique** :

| Sorte | Lire | Parcourir |
|---|---|---|
| entiers | `v.Int()` (rend un `int64`) ; `v.Uint()` pour les non signés | — |
| flottants | `v.Float()` | — |
| `String` | `v.String()` | — |
| `Bool` | `v.Bool()` | — |
| `Slice`, `Array` | `v.Len()` | `v.Index(i)` |
| `Map` | `v.Len()` | `v.MapRange()` (itérateur), `v.MapIndex(k)` |
| `Pointer`, `Interface` | `v.IsNil()` | `v.Elem()` (la valeur pointée ou contenue) |
| `Struct` | `v.NumField()` | `v.Field(i)`, et `v.Type().Field(i)` pour le nom et la balise |
| `Func` | `v.IsNil()` | `v.Call(args)` (leçon 3) |

`v.String()` est une exception : sur une valeur qui n'est pas une chaîne, il ne panique pas mais rend
`"<int Value>"`, ce qui surprend.

### La `Value` invalide

`reflect.ValueOf(nil)` rend une `Value` **zéro** : pas de type, pas de valeur. `v.IsValid()` vaut `false`,
et presque toute méthode (dont `v.Type()`) panique. Un code réflexif robuste teste `IsValid` avant tout.
Attention : un **pointeur nil typé** (`(*Book)(nil)`) n'est pas dans ce cas ; il a un type (`*main.Book`)
et `v.IsNil()` vaut `true` (c'est la distinction interface nil / interface contenant un pointeur nil de
N2 L4).

### Les trois lois

L'article officiel *The Laws of Reflection* du blog de Go les énonce ainsi :

1. **De l'interface vers l'objet de réflexion** : `reflect.ValueOf(x)` et `reflect.TypeOf(x)`.
2. **De l'objet de réflexion vers l'interface** : `v.Interface()` rend un `any`, qu'on récupère par
   assertion (`v.Interface().(int)`). C'est l'inverse exact de la première loi.
3. **Pour modifier un objet de réflexion, la valeur doit être modifiable** (*settable*).

La troisième loi est celle qui surprend. `reflect.ValueOf(x)` reçoit une **copie** de `x` (placée dans
l'interface) ; modifier cette copie n'aurait aucun effet visible, donc `reflect` l'interdit :

```
reflect: reflect.Value.SetInt using unaddressable value
```

Pour modifier `x`, on passe **son adresse**, puis on prend l'élément pointé :

```go
p := reflect.ValueOf(&x).Elem()   // l'élément pointé : x lui-même
p.CanSet()                        // true
p.SetInt(2)                       // x vaut maintenant 2
```

C'est la raison pour laquelle `json.Unmarshal(data, &v)` exige un **pointeur** : sans lui, il ne pourrait
remplir que sa propre copie. Il le vérifie d'ailleurs et rend une erreur claire
(`json: Unmarshal(non-pointer main.T)`).

Les champs **non exportés** d'une struct se lisent par réflexion (en partie) mais ne se modifient
**jamais** : `CanSet()` vaut `false`, même à travers un pointeur. L'encapsulation des paquets (N2 L8)
tient aussi contre la réflexion.

### Quand utiliser `reflect`

Presque jamais dans le code d'application. Ses usages légitimes sont des **bibliothèques génériques** au
sens large : encodeurs (JSON, YAML, CSV), *mapping* de bases de données, chargeurs de configuration,
outils de test (comparaison profonde), injection de dépendances. Avant les génériques (Go 1.18), c'était
aussi le seul moyen d'écrire une fonction pour « n'importe quel type » ; ce n'est plus le cas
(leçon 4).

Coûts à garder en tête dès maintenant : le code réflexif n'est **vérifié qu'à l'exécution** (une
mauvaise sorte = une panique), il est **plus lent** (leçon 4) et plus difficile à lire.

## Exemple

Une fonction qui décrit la structure de n'importe quelle valeur, puis les trois lois en action.

```go
package main

import (
	"fmt"
	"reflect"
	"strings"
)

type Celsius float64

type Book struct {
	Title string
	Tags  []string
}

// Describe parcourt une valeur quelconque et décrit sa structure, sans rien savoir d'elle.
func Describe(v any) string {
	var b strings.Builder
	describe(&b, reflect.ValueOf(v), 0)
	return b.String()
}

func describe(b *strings.Builder, v reflect.Value, depth int) {
	pad := strings.Repeat("  ", depth)
	if !v.IsValid() { // reflect.ValueOf(nil) : aucune valeur, aucun type
		fmt.Fprintf(b, "%s<invalide : interface nil>\n", pad)
		return
	}
	t := v.Type()
	fmt.Fprintf(b, "%stype=%-14s kind=%-7s", pad, t, t.Kind())
	switch v.Kind() {
	case reflect.Int, reflect.Int64:
		fmt.Fprintf(b, " valeur=%d\n", v.Int())
	case reflect.Float64:
		fmt.Fprintf(b, " valeur=%g\n", v.Float())
	case reflect.String:
		fmt.Fprintf(b, " valeur=%q\n", v.String())
	case reflect.Slice:
		fmt.Fprintf(b, " len=%d\n", v.Len())
		for i := range v.Len() {
			describe(b, v.Index(i), depth+1)
		}
	case reflect.Map:
		fmt.Fprintf(b, " len=%d\n", v.Len())
		iter := v.MapRange()
		for iter.Next() {
			fmt.Fprintf(b, "%s  clé %v :\n", pad, iter.Key())
			describe(b, iter.Value(), depth+2)
		}
	case reflect.Pointer:
		if v.IsNil() {
			fmt.Fprintf(b, " nil\n")
			return
		}
		fmt.Fprintf(b, " → élément :\n")
		describe(b, v.Elem(), depth+1)
	case reflect.Struct:
		fmt.Fprintf(b, " %d champ(s)\n", v.NumField())
		for i := range v.NumField() {
			fmt.Fprintf(b, "%s  .%s :\n", pad, t.Field(i).Name)
			describe(b, v.Field(i), depth+2)
		}
	default:
		fmt.Fprintf(b, " (non détaillé)\n")
	}
}

func main() {
	var nilBook *Book
	for _, v := range []any{
		42,
		Celsius(21.5),
		[]string{"a", "b"},
		map[string]int{"go": 1},
		&Book{Title: "La Peste", Tags: []string{"roman"}},
		nilBook,
		nil,
		func() {},
	} {
		fmt.Print(Describe(v))
	}

	// Troisième loi : pour modifier, il faut une valeur MODIFIABLE (obtenue à travers un pointeur).
	x := 1
	v := reflect.ValueOf(x) // une COPIE de x dans l'interface : la modifier n'aurait aucun sens
	fmt.Println("ValueOf(x).CanSet() :", v.CanSet())
	func() {
		defer func() { fmt.Println("panique :", recover()) }()
		v.SetInt(2)
	}()
	p := reflect.ValueOf(&x).Elem() // l'élément pointé : c'est x lui-même
	fmt.Println("ValueOf(&x).Elem().CanSet() :", p.CanSet())
	p.SetInt(2)
	fmt.Println("x =", x)

	// Deuxième loi : du reflet vers l'interface.
	back := p.Interface().(int)
	fmt.Println("Interface().(int) :", back)
}
```

Sortie :

```
type=int            kind=int     valeur=42
type=main.Celsius   kind=float64 valeur=21.5
type=[]string       kind=slice   len=2
  type=string         kind=string  valeur="a"
  type=string         kind=string  valeur="b"
type=map[string]int kind=map     len=1
  clé go :
    type=int            kind=int     valeur=1
type=*main.Book     kind=ptr     → élément :
  type=main.Book      kind=struct  2 champ(s)
    .Title :
      type=string         kind=string  valeur="La Peste"
    .Tags :
      type=[]string       kind=slice   len=1
        type=string         kind=string  valeur="roman"
type=*main.Book     kind=ptr     nil
<invalide : interface nil>
type=func()         kind=func    (non détaillé)
ValueOf(x).CanSet() : false
panique : reflect: reflect.Value.SetInt using unaddressable value
ValueOf(&x).Elem().CanSet() : true
x = 2
Interface().(int) : 2
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `reflect.ValueOf(v)` dans `Describe` | Première loi : de l'interface vers la réflexion. |
| `v.IsValid()` en premier | `Describe(nil)` donnerait sinon une panique sur `v.Type()`. |
| `switch v.Kind()` | Le branchement se fait sur la **sorte** : `Celsius` est traité comme tout `float64`. |
| `type=main.Celsius kind=float64` | Le type nommé et sa sorte sous-jacente. |
| `v.Index(i)`, `v.MapRange()`, `v.Elem()`, `v.Field(i)` | Les parcours propres à chaque sorte. |
| `t.Field(i).Name` | Le **nom** d'un champ est une information de **type** ; sa valeur, une information de `Value`. |
| `*main.Book` nil | Un pointeur nil a un type ; `IsNil()` le détecte avant `Elem()`. |
| `func()` « non détaillé » | La réflexion voit une fonction, mais ne peut rien en dire d'utile sans l'appeler. |
| `ValueOf(x).SetInt` → panique | Troisième loi : une copie n'est pas modifiable. |
| `ValueOf(&x).Elem()` | Modifiable : c'est `x` lui-même. |
| `p.Interface().(int)` | Deuxième loi : retour vers une interface, puis assertion. |

## Erreurs fréquentes

1. **Appeler la méthode d'une autre sorte** (`v.Int()` sur une chaîne) : panique.
2. **Oublier `IsValid`** devant une valeur qui peut être `nil`.
3. **Confondre type et sorte** : comparer `t.Kind() == reflect.TypeOf(Celsius(0))`, ou `t == reflect.Float64`.
4. **Modifier à travers `ValueOf(x)`** au lieu de `ValueOf(&x).Elem()`.
5. **`Elem()` sur un pointeur nil** : `Value` invalide, puis panique plus loin.
6. **Utiliser `reflect` là où un `switch` de types suffit** (N2 L4) : `switch x := v.(type)` est vérifié
   par le compilateur et bien plus lisible pour un nombre fermé de types.
7. **Croire que `v.String()` convertit** n'importe quelle valeur en texte.

## Bonnes pratiques Go

- Préférer, dans l'ordre : types concrets, interfaces, `switch` de types, génériques ; `reflect` en dernier.
- Brancher sur `Kind`, traiter explicitement les sortes non gérées (erreur, pas panique).
- Tester `IsValid`, `IsNil` et `CanSet` avant d'agir.
- Exiger un pointeur dès qu'une fonction réflexive doit **remplir** une valeur, et le vérifier.
- Encapsuler le code réflexif dans une petite fonction bien testée, jamais le disséminer.

## Ce que je dois retenir

- `TypeOf` et `ValueOf` partent d'une interface ; `Interface()` y revient.
- **Type** = le type exact ; **Kind** = la catégorie ; on branche sur le `Kind`.
- Chaque sorte a ses méthodes ; la mauvaise sorte panique.
- Modifier exige une valeur modifiable : passer un pointeur, puis `Elem()`.
- `ValueOf(nil)` est invalide ; un pointeur nil typé ne l'est pas.

---

➡️ [Exercices](exercices.md)
