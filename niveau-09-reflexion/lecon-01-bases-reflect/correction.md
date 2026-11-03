# Leçon 1 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Type ou sorte ?

Mesuré :

```
int8                                     int8
time.Duration                            int64
[3]int                                   array
[]uint8                                  slice
*errors.errorString                      ptr
struct {}                                struct
*struct {}                               ptr
chan int                                 chan
func(...interface {}) (int, error)       func
```

`[]byte` s'affiche `[]uint8` (`byte` est un alias). `errors.New("x")` rend une **interface** `error`,
mais `reflect.TypeOf` reçoit un `any` : la valeur d'interface `error` est convertie en `any`, et seul son
**type dynamique** survit, `*errors.errorString`, un pointeur. Une `Value` de sorte `Interface` n'apparaît
qu'en **descendant** dans une structure : le champ `Any any` d'une struct, ou l'élément d'un `[]any`.

## E2 — Les paniques

```
mauvaise sorte   reflect: call of reflect.Value.Int on string Value
invalide         reflect: call of reflect.Value.Type on zero Value
non modifiable   reflect: reflect.Value.SetInt using unaddressable value
non exporté      reflect: reflect.Value.SetInt using value obtained using unexported field
Elem sur int     reflect: call of reflect.Value.Elem on int Value
```

Toutes auraient pu être évitées par une vérification préalable : `v.Kind() == reflect.Int`,
`v.IsValid()`, `v.CanSet()` (qui couvre les deux cas « non modifiable » et « non exporté »), et
`v.Kind() == reflect.Pointer || v.Kind() == reflect.Interface`. Une bibliothèque réflexive rend des
**erreurs** dans ces cas ; elle ne laisse paniquer que ce qui est une faute de programmation de
l'appelant, et le documente.

## E3 — Compléter `Describe`

Les ajouts suivent le même modèle : `reflect.Bool` → `v.Bool()` ; `reflect.Array` comme `Slice` ;
`reflect.Interface` → `if v.IsNil() {…}` puis `describe(v.Elem())` ; les non signés → `v.Uint()`.

L'ordre de `MapRange` est **aléatoire** : lancer deux fois le programme sur une map de plusieurs clés
montre deux ordres différents (le *runtime* le fait exprès, pour qu'aucun code ne dépende de l'ordre
d'une map). Pour une sortie déterministe : `keys := v.MapKeys()`, trier avec une comparaison qui dépend
de la **sorte** de la clé (numérique pour les nombres, lexicale pour les chaînes, voir `compareKeys` dans
le défi), puis parcourir `v.MapIndex(k)`. C'est ce que fait `fmt` depuis Go 1.12.

## E4 — Mettre à zéro

```go
// Fichier : refl/zero.go
package refl

import (
	"fmt"
	"reflect"
)

// Zero remet à zéro la valeur pointée par ptr.
func Zero(ptr any) error {
	v := reflect.ValueOf(ptr)
	if v.Kind() != reflect.Pointer || v.IsNil() {
		return fmt.Errorf("Zero : pointeur non nil attendu, reçu %T", ptr)
	}
	e := v.Elem()
	e.Set(reflect.Zero(e.Type()))
	return nil
}
```

`reflect.Zero(t)` rend la valeur zéro du type, et `e.Set` la copie dans l'élément pointé, modifiable
puisqu'on est passé par un pointeur.

## E5 — Le cycle

`Describe(n)` descend `Next` → `Next` → … indéfiniment : la pile grossit jusqu'à
`goroutine stack exceeds 1000000000-byte limit` puis `fatal error: stack overflow` (une erreur fatale,
pas une panique : `recover` ne peut rien). La correction mémorise les pointeurs déjà visités :

```go
if v.Kind() == reflect.Pointer && !v.IsNil() {
	if seen[v.Pointer()] {
		fmt.Fprintf(b, "%s(cycle)\n", pad)
		return
	}
	seen[v.Pointer()] = true
}
```

Le même mécanisme, avec une map de l'adresse d'origine vers la **copie**, est utilisé par `DeepCopy`
ci-dessous et testé par `TestDeepCopyCycle`. `encoding/json` fait de même : il détecte les cycles et rend
`json: unsupported value: encountered a cycle via *main.Node`.

## Exercice intermédiaire — Un `Copy` profond

```go
// Fichier : refl/deepcopy.go
package refl

import "reflect"

// DeepCopy copie récursivement v. Politique pour les champs NON exportés : copie
// superficielle (ils sont recopiés tels quels avec la struct, mais leur contenu n'est
// pas dupliqué), car reflect interdit de les modifier. Les chan et func sont partagés.
func DeepCopy[T any](v T) T {
	src := reflect.ValueOf(&v).Elem()
	dst := reflect.New(src.Type()).Elem()
	copyValue(dst, src, map[uintptr]reflect.Value{})
	return dst.Interface().(T)
}

func copyValue(dst, src reflect.Value, seen map[uintptr]reflect.Value) {
	switch src.Kind() {
	case reflect.Pointer:
		if src.IsNil() {
			return
		}
		if done, ok := seen[src.Pointer()]; ok { // cycle : réutiliser la copie déjà faite
			dst.Set(done)
			return
		}
		p := reflect.New(src.Type().Elem())
		seen[src.Pointer()] = p
		copyValue(p.Elem(), src.Elem(), seen)
		dst.Set(p)
	case reflect.Slice:
		if src.IsNil() {
			return
		}
		s := reflect.MakeSlice(src.Type(), src.Len(), src.Len())
		for i := range src.Len() {
			copyValue(s.Index(i), src.Index(i), seen)
		}
		dst.Set(s)
	case reflect.Array:
		for i := range src.Len() {
			copyValue(dst.Index(i), src.Index(i), seen)
		}
	case reflect.Map:
		if src.IsNil() {
			return
		}
		m := reflect.MakeMapWithSize(src.Type(), src.Len())
		it := src.MapRange()
		for it.Next() {
			k := reflect.New(src.Type().Key()).Elem()
			copyValue(k, it.Key(), seen)
			e := reflect.New(src.Type().Elem()).Elem()
			copyValue(e, it.Value(), seen)
			m.SetMapIndex(k, e)
		}
		dst.Set(m)
	case reflect.Struct:
		dst.Set(src) // copie superficielle de TOUS les champs, non exportés compris
		for i := range src.NumField() {
			if dst.Field(i).CanSet() { // puis copie profonde des champs exportés
				copyValue(dst.Field(i), src.Field(i), seen)
			}
		}
	case reflect.Interface:
		if src.IsNil() {
			return
		}
		e := reflect.New(src.Elem().Type()).Elem()
		copyValue(e, src.Elem(), seen)
		dst.Set(e)
	default: // nombres, chaînes, booléens, chan, func : copie de la valeur
		dst.Set(src)
	}
}
```

Réponses :

- **Les champs non exportés.** `reflect` interdit de les **modifier** (`CanSet` faux), y compris dans une
  valeur neuve créée par `reflect.New`. Mais `dst.Set(src)` sur la **struct entière** est permis (la
  struct elle-même est modifiable) et recopie **tous** les champs, non exportés compris — par simple
  copie. Leur contenu (slice, map, pointeur) reste donc **partagé**. Politique choisie et documentée :
  copie superficielle des champs non exportés ; un test la fige (`cp.secret[0] = 100` modifie l'original).
  Une copie vraiment profonde des champs non exportés demanderait `unsafe` : c'est précisément ce que
  l'encapsulation est censée empêcher.
- **`chan` et `func`** : partagés. Copier un canal en créerait un **autre**, sans les goroutines qui
  l'écoutent ; une fonction ne se copie pas.
- **`sync.Mutex`** : un champ non exporté de la plupart des types, donc partagé… ou copié avec la struct
  (`dst.Set(src)`), y compris dans l'état **verrouillé** si la copie a lieu pendant qu'il est tenu. Le
  verrou copié ne protège plus rien. C'est pourquoi `go vet` (analyseur `copylocks`) signale toute copie
  d'une struct qui contient un `sync.Mutex`, et pourquoi copier « n'importe quoi » est une fausse bonne
  idée : les types qui contiennent un verrou doivent fournir leur propre méthode de copie.

## Défi — Un `fmt.Printf("%v")` minimal

```go
// Fichier : refl/format.go
package refl

import (
	"cmp"
	"fmt"
	"reflect"
	"slices"
	"strconv"
	"strings"
)

// Format imite fmt.Sprintf("%v", v) pour les sortes courantes.
func Format(v any) string {
	var b strings.Builder
	format(&b, reflect.ValueOf(v), true)
	return b.String()
}

var stringerType = reflect.TypeFor[fmt.Stringer]()

func format(b *strings.Builder, v reflect.Value, top bool) {
	if !v.IsValid() {
		b.WriteString("<nil>")
		return
	}
	// Stringer : vérifié par réflexion, car on travaille sur une Value et non une interface.
	if v.Type().Implements(stringerType) && v.CanInterface() &&
		!(v.Kind() == reflect.Pointer && v.IsNil()) {
		b.WriteString(v.Interface().(fmt.Stringer).String())
		return
	}
	switch v.Kind() {
	case reflect.Bool:
		b.WriteString(strconv.FormatBool(v.Bool()))
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		b.WriteString(strconv.FormatInt(v.Int(), 10))
	case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64, reflect.Uintptr:
		b.WriteString(strconv.FormatUint(v.Uint(), 10))
	case reflect.Float32:
		b.WriteString(strconv.FormatFloat(v.Float(), 'g', -1, 32))
	case reflect.Float64:
		b.WriteString(strconv.FormatFloat(v.Float(), 'g', -1, 64))
	case reflect.String:
		b.WriteString(v.String())
	case reflect.Slice, reflect.Array:
		b.WriteByte('[')
		for i := range v.Len() {
			if i > 0 {
				b.WriteByte(' ')
			}
			format(b, v.Index(i), false)
		}
		b.WriteByte(']')
	case reflect.Map:
		keys := v.MapKeys()
		slices.SortFunc(keys, compareKeys) // comme fmt : les nombres dans l'ordre NUMÉRIQUE
		b.WriteString("map[")
		for i, k := range keys {
			if i > 0 {
				b.WriteByte(' ')
			}
			format(b, k, false)
			b.WriteByte(':')
			format(b, v.MapIndex(k), false)
		}
		b.WriteByte(']')
	case reflect.Struct:
		b.WriteByte('{')
		for i := range v.NumField() {
			if i > 0 {
				b.WriteByte(' ')
			}
			format(b, v.Field(i), false)
		}
		b.WriteByte('}')
	case reflect.Pointer:
		if v.IsNil() {
			b.WriteString("<nil>")
			return
		}
		if top && (v.Elem().Kind() == reflect.Struct || v.Elem().Kind() == reflect.Slice || v.Elem().Kind() == reflect.Map) {
			b.WriteByte('&')
			format(b, v.Elem(), false)
			return
		}
		fmt.Fprintf(b, "%p", v.Interface())
	case reflect.Interface:
		format(b, v.Elem(), false)
	default:
		fmt.Fprintf(b, "%v", v) // chan, func : une adresse
	}
}

// compareKeys ordonne les clés de map selon leur sorte (un tri de leur texte mettrait 10 avant 2).
func compareKeys(a, c reflect.Value) int {
	switch a.Kind() {
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		return cmp.Compare(a.Int(), c.Int())
	case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64, reflect.Uintptr:
		return cmp.Compare(a.Uint(), c.Uint())
	case reflect.Float32, reflect.Float64:
		return cmp.Compare(a.Float(), c.Float())
	case reflect.String:
		return strings.Compare(a.String(), c.String())
	case reflect.Bool:
		return cmp.Compare(boolInt(a.Bool()), boolInt(c.Bool()))
	default:
		return strings.Compare(Format(a.Interface()), Format(c.Interface()))
	}
}

func boolInt(b bool) int {
	if b {
		return 1
	}
	return 0
}
```

```go
// Fichier : refl/refl_test.go
package refl

import (
	"fmt"
	"reflect"
	"strings"
	"testing"
	"time"
)

type Node struct {
	Name string
	Next *Node
}

type Inner struct{ Vals []int }

type Outer struct {
	Name   string
	In     *Inner
	Tags   map[string][]string
	Any    any
	Arr    [2]*Inner
	secret []int
}

func TestZero(t *testing.T) {
	o := Outer{Name: "x", In: &Inner{}}
	if err := Zero(&o); err != nil || !reflect.DeepEqual(o, Outer{}) {
		t.Fatalf("o=%+v err=%v", o, err)
	}
	for _, bad := range []any{o, (*Outer)(nil), nil} {
		if err := Zero(bad); err == nil {
			t.Errorf("Zero(%T) : erreur attendue", bad)
		}
	}
}

func TestDeepCopyIndependent(t *testing.T) {
	orig := Outer{
		Name: "a", In: &Inner{Vals: []int{1, 2}},
		Tags: map[string][]string{"k": {"v"}}, Any: []int{7},
		Arr: [2]*Inner{{Vals: []int{3}}}, secret: []int{9},
	}
	cp := DeepCopy(orig)
	if !reflect.DeepEqual(cp, orig) {
		t.Fatalf("copie différente : %+v", cp)
	}
	cp.In.Vals[0] = 100
	cp.Tags["k"][0] = "modifié"
	cp.Any.([]int)[0] = 100
	cp.Arr[0].Vals[0] = 100
	if orig.In.Vals[0] != 1 || orig.Tags["k"][0] != "v" || orig.Any.([]int)[0] != 7 || orig.Arr[0].Vals[0] != 3 {
		t.Fatalf("l'original a été modifié : %+v", orig)
	}
	// Politique documentée : les champs non exportés sont PARTAGÉS (copie superficielle).
	cp.secret[0] = 100
	if orig.secret[0] != 100 {
		t.Fatal("la politique « non exportés partagés » n'est plus celle documentée")
	}
}

func TestDeepCopyCycle(t *testing.T) {
	n := &Node{Name: "a"}
	n.Next = &Node{Name: "b", Next: n}
	cp := DeepCopy(n)
	if cp == n || cp.Next == n.Next || cp.Next.Next != cp {
		t.Fatal("le cycle doit être reproduit dans la copie, sans pointer vers l'original")
	}
}

type temp float64

func (t temp) String() string { return fmt.Sprintf("%.1f°C", float64(t)) }

func TestFormatLikeFmt(t *testing.T) {
	var nilMap map[string]int
	var nilPtr *Inner
	type pair struct {
		A int
		B string
	}
	values := []any{
		0, -5, uint8(200), 3.5, float32(0.1), 1e21, "", "go", true, false,
		[]int{1, 2}, []string{}, [2]bool{true}, map[string]int{"b": 2, "a": 1}, nilMap,
		pair{1, "x"}, &pair{2, "y"}, []pair{{3, "z"}}, nilPtr, nil,
		temp(21.5), []temp{1, 2}, time.Duration(1500) * time.Millisecond,
		map[int][]string{2: {"x"}, 1: nil}, map[int]bool{10: true, 2: false, -1: true}, struct{}{}, []any{1, "a", nil},
		[]*int{nil}, &[]int{1}, &map[string]int{"a": 1}, [][]int{{1}, {}},
	}
	for _, v := range values {
		if got, want := Format(v), fmt.Sprintf("%v", v); got != want {
			t.Errorf("Format(%#v) = %q, fmt = %q", v, got, want)
		}
	}
}

func FuzzFormat(f *testing.F) {
	f.Add(int64(1), "a", 1.5, true)
	f.Fuzz(func(t *testing.T, i int64, s string, x float64, b bool) {
		v := struct {
			I int64
			S []string
			M map[string]float64
			B *bool
		}{i, strings.Split(s, ","), map[string]float64{s: x}, &b}
		if got, want := Format(v), fmt.Sprintf("%v", v); !strings.HasPrefix(got, want[:strings.LastIndexByte(want, ' ')]) {
			t.Fatalf("Format = %q, fmt = %q", got, want)
		}
	})
}
```

`go test -race ./...` passe ; le fuzzing (20 s, 2 millions d'exécutions) aussi.

**Stringer : réflexion ou assertion ?** À l'entrée de `Format`, une assertion `v.(fmt.Stringer)` suffirait.
Mais `format` travaille sur des `reflect.Value` (éléments, champs) : il faut tester
`v.Type().Implements(stringerType)`, puis `v.Interface().(fmt.Stringer)`. Deux précautions : `CanInterface`
(un champ non exporté ne peut pas être converti en interface, E2 de la leçon), et le pointeur nil (appeler
`String()` sur un pointeur nil peut paniquer ; `fmt` affiche alors `<nil>`).

**Un bug trouvé par un test ajouté après coup.** La première version triait les clés de map par leur
**texte**. La table de 30 valeurs passait, parce que ses maps n'avaient que des clés à un chiffre. Une
ligne de plus (`map[int]bool{10: true, 2: false, -1: true}`) l'a montré :

```
Format(map[int]bool{-1:true, 2:false, 10:true}) = "map[-1:true 10:true 2:false]", fmt = "map[-1:true 2:false 10:true]"
```

`fmt` trie les clés selon leur **sorte** (numérique pour les nombres) ; `compareKeys` fait de même. Le
fuzzing ne l'avait pas trouvé non plus : sa map n'avait qu'une clé. Un fuzzer ne trouve que ce que sa
structure d'entrée permet de produire.

## Quiz

1. Un type dynamique et une valeur dynamique ; `TypeOf` donne le premier, `ValueOf` les deux.
2. Le type est exact (`main.Celsius`) ; le `Kind` est la catégorie de base (`float64`).
3. Pour traiter de la même façon tous les types d'une même catégorie, nommés ou non.
4. Une panique : `call of reflect.Value.Int on string Value`.
5. La `Value` zéro, sans type ni valeur ; `reflect.ValueOf(nil)`.
6. Interface → réflexion ; réflexion → interface ; modifier exige une valeur modifiable.
7. `ValueOf(x)` travaille sur une copie de `x`, non adressable.
8. Pour modifier la variable de l'appelant, il lui faut son adresse.
9. Non : `CanSet` est faux, même à travers un pointeur.
10. Encodeurs, *mapping* de bases de données, chargeurs de configuration (aussi : outils de test) ;
    dans le code d'application, préférer interfaces, `switch` de types et génériques.
