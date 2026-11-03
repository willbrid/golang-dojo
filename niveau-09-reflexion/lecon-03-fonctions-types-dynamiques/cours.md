# Leçon 3 — Fonctions, méthodes et types dynamiques

> La réflexion ne sert pas qu'à lire des données : elle peut **appeler** une fonction ou une méthode
> dont on ne connaît la signature qu'à l'exécution, **fabriquer** une fonction d'un type donné, et
> **créer** des valeurs d'un type choisi par son nom. C'est ainsi que fonctionnent les serveurs RPC
> anciens (`net/rpc`), les répartiteurs de commandes, certains conteneurs d'injection de dépendances.
> Cette leçon montre ces trois mécanismes, puis termine par l'outil réflexif que tout le monde utilise
> sans le savoir — `reflect.DeepEqual` — et ses surprises.

## Objectifs

1. Trouver une méthode par son nom, examiner sa signature, l'appeler avec `Call`.
2. Comprendre les **ensembles de méthodes** vus par la réflexion (récepteur valeur ou pointeur).
3. Fabriquer une fonction d'un type quelconque avec `reflect.MakeFunc`.
4. Créer des valeurs à l'exécution (`New`, `MakeSlice`, `MakeMap`) et savoir quand préférer un
   constructeur.
5. Connaître les résultats surprenants de `reflect.DeepEqual`, et savoir quoi utiliser à la place.

## Explication

### Appeler une méthode par son nom

```go
m := reflect.ValueOf(target).MethodByName("Add")  // une Value de sorte Func, invalide si absente
mt := m.Type()                                      // func(string, int) error
mt.NumIn(), mt.In(i)                                // paramètres
mt.NumOut(), mt.Out(i)                              // résultats
out := m.Call([]reflect.Value{reflect.ValueOf("La Peste"), reflect.ValueOf(3)})
```

- `MethodByName` sur une **valeur** rend la méthode **liée** à cette valeur : le récepteur n'est pas un
  paramètre (`NumIn` vaut 2 pour `Add(title string, copies int)`). Sur un **type**
  (`t.MethodByName`), le récepteur est le premier paramètre.
- Seules les méthodes **exportées** sont visibles. `reset` n'existe pas pour la réflexion.
- `Call` **panique** si le nombre ou le type des arguments ne correspond pas. On vérifie donc la signature
  avant, et on construit chaque argument avec le **bon type** (`reflect.New(mt.In(i)).Elem()`, puis
  `Set…`).
- Les résultats reviennent en `[]reflect.Value`. La convention Go « erreur en dernier » se détecte en
  comparant `mt.Out(n-1)` au type `error`, obtenu par `reflect.TypeFor[error]()`.

### Les ensembles de méthodes

La règle de N2 L3 se lit directement dans la réflexion : le type `T` a pour méthodes celles à récepteur
**valeur** ; `*T` a celles à récepteur valeur **et** pointeur. Pour un type dont toutes les méthodes
modifiantes ont un récepteur pointeur, `reflect.ValueOf(lib)` (une copie de valeur) ne voit que `Size` ;
il faut passer `&lib`. C'est l'erreur classique d'un répartiteur qui « ne trouve pas » une méthode
pourtant bien écrite.

### Fabriquer une fonction : `MakeFunc`

`reflect.MakeFunc(typ, impl)` crée une fonction du type `typ` dont le corps est `impl`, qui reçoit et rend
des `[]reflect.Value`. Avec les génériques, on obtient un enrobage **universel** et typé :

```go
func Traced[F any](name string, fn F) F
add := Traced("add", func(a, b int) int { return a + b }) // add est un func(int, int) int
```

L'appelant garde un appel typé et vérifié par le compilateur (`add(2, 3)`) ; seule l'implémentation de
`Traced` est réflexive. Usages : traces, mesure de durée, *mocks* génériques dans les tests. Coût : chaque
appel passe par la conversion des arguments en `[]reflect.Value` et retour (leçon 4).

### Créer des valeurs à l'exécution

| Fonction | Rend |
|---|---|
| `reflect.New(t)` | un **pointeur** vers une valeur zéro de type `t` (comme `new(T)`) |
| `reflect.Zero(t)` | une valeur zéro **non modifiable** |
| `reflect.MakeSlice(t, len, cap)` | une slice |
| `reflect.MakeMap(t)`, `MakeMapWithSize` | une map |
| `reflect.MakeChan(t, size)` | un canal |
| `reflect.StructOf(fields)`, `FuncOf`, `SliceOf`, `MapOf` | un **type** construit à l'exécution |

Un « registre » de types par nom (`map[string]reflect.Type` + `reflect.New`) donne des valeurs **zéro** :
les champs non exportés ne peuvent pas être initialisés par réflexion, et une valeur zéro n'est souvent
pas utilisable (ici, une map nil : `Add` paniquerait). L'alternative idiomatique, sans réflexion, est un
registre de **constructeurs** : `map[string]func() any`. C'est le choix d'`database/sql` pour ses pilotes
(leçon N8 L1 : `sql.Register` reçoit une **valeur** de pilote, pas un type).

### `reflect.DeepEqual` et ses surprises

`DeepEqual` compare deux valeurs **récursivement** : deux slices sont égales si elles ont la même
longueur et des éléments égaux un à un, deux pointeurs si ce qu'ils pointent est égal, etc. C'est la
comparaison qu'utilisent beaucoup de tests. Ses règles exactes surprennent ; mesuré :

| Comparaison | Résultat | Pourquoi |
|---|---|---|
| `[]int(nil)` et `[]int{}` | **false** | nil et vide sont distingués (de même pour les maps) |
| `NaN` et `NaN` | **false** | `NaN != NaN` en virgule flottante |
| `[]float64{NaN}` et une **autre** `[]float64{NaN}` | **false** | même règle, élément par élément |
| une slice contenant `NaN` et **elle-même** | **true** | raccourci : même tableau sous-jacent, pas de comparaison des éléments |
| la même fonction non nil, deux fois | **false** | deux fonctions ne sont égales que si elles sont **toutes deux nil** |
| même instant `time.Time`, fuseaux différents | **false** | les champs internes (fuseau) diffèrent ; il faut `Equal` |
| `time.Now()` et `time.Now().Round(0)` | **false** | lecture monotone présente d'un côté seulement (N4 L4) |
| `errors.New("x")` et `errors.New("x")` | **true** | deux pointeurs différents, mais vers des structs égales — alors que `==` donne `false` et `errors.Is` aussi |
| `1` et `int64(1)` | **false** | types différents |
| deux structures **cycliques** de même forme | true | les cycles sont détectés |
| `error(nil)` et `(*Node)(nil)` | **false** | interface nil contre pointeur nil typé |

Conséquences pratiques :

- dans les tests, une slice nil attendue contre une slice vide obtenue échoue « sans raison » ;
- `DeepEqual` sur des erreurs dit vrai pour deux erreurs **distinctes** : `errors.Is` reste la seule
  bonne comparaison d'erreurs (N2 L6) ;
- `DeepEqual` sur des `time.Time` est presque toujours faux pour de mauvaises raisons.

Alternatives : `slices.Equal`, `maps.Equal` (N3 L4) pour les cas simples, typés et sans surprise sur
nil/vide ; une comparaison écrite pour le type (avec `Equal` pour les dates) ; dans les tests, le module
`github.com/google/go-cmp` (`cmp.Diff`), qui affiche **où** les valeurs diffèrent et accepte des options
(`cmpopts.EquateEmpty` pour nil/vide, `cmpopts.EquateNaNs`).

## Exemple

Un répartiteur de commandes textuelles qui appelle des méthodes par leur nom, un enrobage universel
avec `MakeFunc`, et deux façons de créer une valeur à partir d'un nom.

```go
package main

import (
	"errors"
	"fmt"
	"reflect"
	"strconv"
	"strings"
)

type Library struct{ books map[string]int }

func (l *Library) Add(title string, copies int) error {
	if copies <= 0 {
		return errors.New("nombre d'exemplaires invalide")
	}
	l.books[title] += copies
	return nil
}

func (l *Library) Count(title string) int { return l.books[title] }
func (l Library) Size() int               { return len(l.books) } // récepteur valeur
func (l *Library) reset()                 { clear(l.books) }      // non exportée : invisible

var errorType = reflect.TypeFor[error]()

// Call appelle la méthode name de target, en convertissant les arguments textuels.
func Call(target any, name string, args ...string) ([]any, error) {
	m := reflect.ValueOf(target).MethodByName(name)
	if !m.IsValid() {
		return nil, fmt.Errorf("méthode %q inconnue sur %T", name, target)
	}
	mt := m.Type() // le type de la méthode LIÉE : le récepteur n'est pas un paramètre
	if mt.NumIn() != len(args) {
		return nil, fmt.Errorf("%s attend %d argument(s), %d fourni(s)", name, mt.NumIn(), len(args))
	}
	in := make([]reflect.Value, len(args))
	for i, raw := range args {
		v, err := parse(mt.In(i), raw)
		if err != nil {
			return nil, fmt.Errorf("%s, argument %d : %w", name, i+1, err)
		}
		in[i] = v
	}
	out := m.Call(in) // panique si les types ne correspondent pas : on les a vérifiés avant
	// Convention Go : une erreur en dernier résultat.
	if n := mt.NumOut(); n > 0 && mt.Out(n-1) == errorType {
		if err, _ := out[n-1].Interface().(error); err != nil {
			return nil, err
		}
		out = out[:n-1]
	}
	res := make([]any, len(out))
	for i, o := range out {
		res[i] = o.Interface()
	}
	return res, nil
}

func parse(t reflect.Type, raw string) (reflect.Value, error) {
	v := reflect.New(t).Elem() // une valeur zéro du type demandé, modifiable
	switch t.Kind() {
	case reflect.String:
		v.SetString(raw)
	case reflect.Int:
		n, err := strconv.Atoi(raw)
		if err != nil {
			return v, err
		}
		v.SetInt(int64(n))
	default:
		return v, fmt.Errorf("type %s non géré", t)
	}
	return v, nil
}

// Traced enrobe N'IMPORTE QUELLE fonction : MakeFunc fabrique une fonction du même type.
func Traced[F any](name string, fn F) F {
	fv := reflect.ValueOf(fn)
	wrapper := reflect.MakeFunc(fv.Type(), func(args []reflect.Value) []reflect.Value {
		out := fv.Call(args)
		fmt.Printf("  trace : %s(%s) = %s\n", name, show(args), show(out))
		return out
	})
	return wrapper.Interface().(F)
}

func show(vs []reflect.Value) string {
	parts := make([]string, len(vs))
	for i, v := range vs {
		parts[i] = fmt.Sprint(v.Interface())
	}
	return strings.Join(parts, ", ")
}

func methods(t reflect.Type) []string {
	var names []string
	for i := range t.NumMethod() {
		names = append(names, t.Method(i).Name)
	}
	return names
}

func main() {
	lib := &Library{books: map[string]int{}}
	for _, cmd := range [][]string{
		{"Add", "La Peste", "3"},
		{"Add", "La Peste", "2"},
		{"Count", "La Peste"},
		{"Size"},
		{"Add", "Vide", "0"},
		{"Add", "La Peste", "beaucoup"},
		{"Count"},
		{"reset"},
	} {
		res, err := Call(lib, cmd[0], cmd[1:]...)
		fmt.Printf("%-24s → %v %v\n", strings.Join(cmd, " "), res, err)
	}

	// Ensembles de méthodes : T n'a que les méthodes à récepteur valeur, *T a les deux.
	fmt.Println("Library  :", methods(reflect.TypeFor[Library]()))
	fmt.Println("*Library :", methods(reflect.TypeFor[*Library]()))

	add := Traced("add", func(a, b int) int { return a + b })
	split := Traced("cut", strings.Cut)
	fmt.Println("résultat :", add(2, 3))
	before, after, _ := split("clé=valeur", "=")
	fmt.Println("résultat :", before, after)

	// Créer une valeur dont on ne connaît le type qu'à l'exécution.
	types := map[string]reflect.Type{"library": reflect.TypeFor[Library]()}
	zero := reflect.New(types["library"]).Interface().(*Library) // *Library, valeur ZÉRO
	fmt.Printf("reflect.New : %T, map interne nil = %t (Add paniquerait)\n", zero, zero.books == nil)

	// L'alternative sans réflexion : un registre de constructeurs.
	constructors := map[string]func() any{
		"library": func() any { return &Library{books: map[string]int{}} },
	}
	fresh := constructors["library"]().(*Library)
	fresh.Add("Vol de nuit", 1)
	fmt.Printf("constructeur : %T avec %d livre(s)\n", fresh, fresh.Size())
}
```

Sortie :

```
Add La Peste 3           → [] <nil>
Add La Peste 2           → [] <nil>
Count La Peste           → [5] <nil>
Size                     → [1] <nil>
Add Vide 0               → [] nombre d'exemplaires invalide
Add La Peste beaucoup    → [] Add, argument 2 : strconv.Atoi: parsing "beaucoup": invalid syntax
Count                    → [] Count attend 1 argument(s), 0 fourni(s)
reset                    → [] méthode "reset" inconnue sur *main.Library
Library  : [Size]
*Library : [Add Count Size]
  trace : add(2, 3) = 5
résultat : 5
  trace : cut(clé=valeur, =) = clé, valeur, true
résultat : clé valeur
reflect.New : *main.Library, map interne nil = true (Add paniquerait)
constructeur : *main.Library avec 1 livre(s)
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `MethodByName` puis `IsValid` | Une méthode absente (ou non exportée, comme `reset`) donne une `Value` invalide, pas une panique. |
| `mt.NumIn() != len(args)` | Vérifié **avant** `Call`, qui paniquerait. |
| `reflect.New(t).Elem()` dans `parse` | Un argument du type exact du paramètre, modifiable. |
| `mt.Out(n-1) == errorType` | La convention Go « erreur en dernier », reconnue et convertie en vraie `error`. |
| `out[n-1].Interface().(error)` avec `_` | Une erreur nil donne une interface nil : l'assertion à deux résultats évite la panique. |
| `Library : [Size]` / `*Library : [Add Count Size]` | Les ensembles de méthodes de N2 L3, vus par la réflexion. |
| `Traced[F any]` + `MakeFunc(fv.Type(), …)` | Une fonction du **même type** que celle qu'on enrobe : l'appelant reste typé. |
| `Traced("cut", strings.Cut)` | Fonctionne avec trois résultats, sans code spécifique. |
| `reflect.New(types["library"])` | Une valeur zéro : sa map est nil, inutilisable en l'état. |
| `constructors` | Le registre de constructeurs : sans réflexion, et la valeur est correctement initialisée. |

## Erreurs fréquentes

1. **Chercher une méthode à récepteur pointeur sur une valeur** (`ValueOf(lib)` au lieu de `ValueOf(&lib)`).
2. **Appeler `Call` sans vérifier** nombre et types des arguments : panique.
3. **Construire un argument du mauvais type** (`ValueOf(3)` est un `int` ; un paramètre `int64` le refuse).
4. **Oublier qu'une méthode non exportée est invisible.**
5. **Un registre de types** là où un registre de constructeurs suffit.
6. **`reflect.DeepEqual` sur des erreurs, des dates ou des fonctions.**
7. **Confondre nil et vide** dans un test avec `DeepEqual`.
8. **Exposer un répartiteur par nom de méthode** à des entrées non fiables sans liste blanche : toute
   méthode exportée devient appelable de l'extérieur.

## Bonnes pratiques Go

- Préférer une interface ou une `map[string]func(…)` explicite à un appel par nom de méthode.
- Si la réflexion est nécessaire : vérifier la signature une fois, à l'enregistrement, pas à chaque appel.
- `MakeFunc` dans une fonction générique, pour que les appelants restent typés.
- Registre de constructeurs plutôt que registre de types.
- `slices.Equal`, `maps.Equal`, `Equal` des dates, `errors.Is` ; `go-cmp` dans les tests ; `DeepEqual`
  en connaissance de ses règles.

## Ce que je dois retenir

- `MethodByName` rend une méthode liée ; `Type()` donne sa signature ; `Call` exécute — et panique sur
  un mauvais appel.
- La réflexion voit les ensembles de méthodes de N2 L3 : `T` n'a pas les méthodes à récepteur pointeur.
- `MakeFunc` fabrique une fonction d'un type donné ; combinée aux génériques, l'appelant reste typé.
- `New`, `MakeSlice`, `MakeMap` créent des valeurs ; un constructeur vaut souvent mieux.
- `DeepEqual` : nil ≠ vide, `NaN` ≠ `NaN`, fonctions jamais égales, dates à éviter, erreurs « égales »
  à tort.

---

➡️ [Exercices](exercices.md)
