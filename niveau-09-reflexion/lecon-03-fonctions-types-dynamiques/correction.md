# Leçon 3 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Les mesures des exercices faciles E1 à E3 sont produites par `TestExplore`, dans le code de l'exercice
intermédiaire ci-dessous.

## E1 — Valeur ou pointeur

`MethodByName("Incr")` sur une **copie** de valeur rend une `Value` **invalide** (mesuré : `false`) :
toutes les méthodes modifiantes ont un récepteur pointeur, et l'ensemble de méthodes de `T` ne les
contient pas. `Call` renverrait donc « méthode inconnue ». Si `Add` avait un récepteur valeur, l'appel
réussirait… sur une **copie** : la map est partagée (une map est une référence), donc l'ajout serait
visible, mais toute modification d'un champ ordinaire serait perdue. Deux bonnes raisons de toujours
passer un pointeur à un répartiteur.

## E2 — Le mauvais type d'argument

```
reflect: Call using string as type int
reflect: Call using int as type time.Duration
```

`Call` exige le type **exact** (ou assignable) : `int` n'est pas assignable à `time.Duration`, bien
qu'ils aient la même sorte. Un appel direct `a.Sleep(3)` compile, lui, parce que `3` est une **constante**
non typée ; la réflexion manipule des valeurs déjà typées. D'où `reflect.New(param).Elem()` puis `Set…`.

## E3 — Signature par le type

```
par le type : func(*dispatch.Admin, int) int (NumIn 2) ; liée : func(int) int (NumIn 1)
Method.Func.Call(récepteur, 4) = 4
```

Une méthode obtenue par le **type** est une fonction ordinaire dont le premier paramètre est le
récepteur : on le passe en premier argument de `Call`. C'est l'équivalent réflexif des **expressions de
méthode** (`(*Admin).Incr(a, 4)`).

## E4 — `DeepEqual` dans tes tests

`json.Unmarshal` de `"tags": []` produit `[]string{}` (non nil) ; l'attendu `Tags: nil` est différent pour
`DeepEqual`. Trois corrections :

- écrire l'attendu `Tags: []string{}` : exact, mais fragile (le jour où le décodage rend nil pour un
  champ absent, le test recasse) ;
- `slices.Equal(got.Tags, want.Tags)` : traite nil et vide comme égaux (deux slices de longueur 0) —
  c'est en général la sémantique voulue ;
- une fonction de comparaison du type, qui utilise `slices.Equal` et `Equal` pour les dates.

Préférer la deuxième ou la troisième : elles expriment ce qu'on veut vérifier (« pas d'étiquettes »)
et non un détail de représentation. Pour des structures plus grosses, `go-cmp` avec
`cmpopts.EquateEmpty()` fait la même chose et affiche la différence.

## E5 — Mesurer `Traced`

Mesuré (sans le `Printf`) :

```
BenchmarkDirect     1,47 ns/op    0 B/op   0 allocs/op
BenchmarkMakeFunc   ≈ 500 ns/op  96 B/op   5 allocs/op
```

Environ **340 fois** plus lent. Ce qui alloue : la slice `[]reflect.Value` des arguments construite par
`MakeFunc`, le stockage des arguments dans des `Value` (chacun sort de la pile vers le tas), la slice des
résultats de `fv.Call`, et la conversion des résultats. Pour une trace autour d'un appel réseau de
plusieurs millisecondes, c'est négligeable ; autour d'une addition, c'est absurde.

## Exercice intermédiaire — Un répartiteur sûr

```go
// Fichier : dispatch/dispatch.go
package dispatch

import (
	"errors"
	"fmt"
	"reflect"
	"slices"
	"strconv"
	"strings"
	"time"
)

var (
	errorType    = reflect.TypeFor[error]()
	durationType = reflect.TypeFor[time.Duration]()
)

type method struct {
	fn     reflect.Value
	params []reflect.Type
	hasErr bool
	hasVal bool
}

type Dispatcher struct {
	methods map[string]method
}

// NewDispatcher vérifie TOUT à la construction : un appel ne peut plus échouer pour une
// raison de signature.
func NewDispatcher(target any, allowed ...string) (*Dispatcher, error) {
	v := reflect.ValueOf(target)
	d := &Dispatcher{methods: map[string]method{}}
	var errs []error
	for _, name := range allowed {
		m := v.MethodByName(name)
		if !m.IsValid() {
			errs = append(errs, fmt.Errorf("%s : méthode introuvable sur %T", name, target))
			continue
		}
		t := m.Type()
		var md method
		md.fn = m
		for i := range t.NumIn() {
			p := t.In(i)
			if !supported(p) {
				errs = append(errs, fmt.Errorf("%s : paramètre %d de type %s non géré", name, i+1, p))
			}
			md.params = append(md.params, p)
		}
		outs := t.NumOut()
		if outs > 0 && t.Out(outs-1) == errorType {
			md.hasErr = true
			outs--
		}
		if outs > 1 {
			errs = append(errs, fmt.Errorf("%s : au plus une valeur et une error", name))
		}
		md.hasVal = outs == 1
		d.methods[name] = md
	}
	if err := errors.Join(errs...); err != nil {
		return nil, err
	}
	return d, nil
}

func supported(t reflect.Type) bool {
	if t == durationType {
		return true
	}
	switch t.Kind() {
	case reflect.String, reflect.Bool, reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		return true
	}
	return false
}

func convert(t reflect.Type, raw string) (reflect.Value, error) {
	v := reflect.New(t).Elem()
	if t == durationType {
		d, err := time.ParseDuration(raw)
		v.SetInt(int64(d))
		return v, err
	}
	switch t.Kind() {
	case reflect.String:
		v.SetString(raw)
	case reflect.Bool:
		b, err := strconv.ParseBool(raw)
		v.SetBool(b)
		return v, err
	default: // entiers (garantis par supported)
		n, err := strconv.ParseInt(raw, 10, t.Bits())
		v.SetInt(n)
		return v, err
	}
	return v, nil
}

// Dispatch exécute « Méthode arg1 arg2… ». Une panique de la méthode devient une erreur.
func (d *Dispatcher) Dispatch(line string) (result string, err error) {
	fields := strings.Fields(line)
	if len(fields) == 0 {
		return "", errors.New("commande vide")
	}
	name, args := fields[0], fields[1:]
	m, ok := d.methods[name]
	if !ok {
		names := make([]string, 0, len(d.methods))
		for n := range d.methods {
			names = append(names, n)
		}
		slices.Sort(names)
		return "", fmt.Errorf("%q non autorisée (autorisées : %s)", name, strings.Join(names, ", "))
	}
	if len(args) != len(m.params) {
		return "", fmt.Errorf("%s attend %d argument(s), %d fourni(s)", name, len(m.params), len(args))
	}
	in := make([]reflect.Value, len(args))
	for i, raw := range args {
		if in[i], err = convert(m.params[i], raw); err != nil {
			return "", fmt.Errorf("%s, argument %d : %w", name, i+1, err)
		}
	}
	defer func() {
		if p := recover(); p != nil {
			result, err = "", fmt.Errorf("%s a paniqué : %v", name, p)
		}
	}()
	out := m.fn.Call(in)
	if m.hasErr {
		if e, _ := out[len(out)-1].Interface().(error); e != nil {
			return "", e
		}
	}
	if m.hasVal {
		return fmt.Sprint(out[0].Interface()), nil
	}
	return "ok", nil
}
```

```go
// Fichier : dispatch/dispatch_test.go
package dispatch

import (
	"errors"
	"reflect"
	"strings"
	"testing"
	"time"
)

type Admin struct {
	n int
}

func (a *Admin) Incr(by int) int             { a.n += by; return a.n }
func (a *Admin) Sleep(d time.Duration) error { if d > time.Second { return errors.New("trop long") }; return nil }
func (a *Admin) Boom()                       { panic("plus de mémoire") }
func (a *Admin) Shutdown()                   {}
func (a *Admin) Bad(m map[string]int)        {}
func (a Admin) Get() int                     { return a.n }

func TestConstruction(t *testing.T) {
	_, err := NewDispatcher(&Admin{}, "Incr", "Absente", "Bad")
	if err == nil || !strings.Contains(err.Error(), "Absente") || !strings.Contains(err.Error(), "map[string]int") {
		t.Fatalf("err = %v", err)
	}
	t.Log(err)
}

func TestDispatch(t *testing.T) {
	d, err := NewDispatcher(&Admin{}, "Incr", "Sleep", "Boom", "Get")
	if err != nil {
		t.Fatal(err)
	}
	tests := []struct{ line, want, wantErr string }{
		{"Incr 5", "5", ""},
		{"Incr 2", "7", ""},
		{"Get", "7", ""},
		{"Sleep 10ms", "ok", ""},
		{"Sleep 2s", "", "trop long"},
		{"Incr", "", "attend 1 argument"},
		{"Incr beaucoup", "", "argument 1"},
		{"Boom", "", "Boom a paniqué : plus de mémoire"},
		{"Shutdown", "", "non autorisée"},
		{"", "", "commande vide"},
	}
	for _, tt := range tests {
		got, err := d.Dispatch(tt.line)
		if got != tt.want || (tt.wantErr == "") != (err == nil) || (err != nil && !strings.Contains(err.Error(), tt.wantErr)) {
			t.Errorf("Dispatch(%q) = %q, %v ; attendu %q, %q", tt.line, got, err, tt.want, tt.wantErr)
		}
	}
}

// Les mesures des exercices faciles E1 à E3.
func TestExplore(t *testing.T) {
	a := &Admin{}
	t.Log("E1 copie : MethodByName(Incr) valide ?", reflect.ValueOf(*a).MethodByName("Incr").IsValid())
	try := func(name string, f func()) {
		defer func() { t.Logf("E2 %s : %v", name, recover()) }()
		f()
	}
	try("string pour int", func() { reflect.ValueOf(a).MethodByName("Incr").Call([]reflect.Value{reflect.ValueOf("5")}) })
	try("int pour Duration", func() { reflect.ValueOf(a).MethodByName("Sleep").Call([]reflect.Value{reflect.ValueOf(3)}) })
	mt, _ := reflect.TypeFor[*Admin]().MethodByName("Incr")
	bound := reflect.ValueOf(a).MethodByName("Incr").Type()
	t.Logf("E3 par le type : %s (NumIn %d) ; liée : %s (NumIn %d)", mt.Type, mt.Type.NumIn(), bound, bound.NumIn())
	out := mt.Func.Call([]reflect.Value{reflect.ValueOf(a), reflect.ValueOf(4)})
	t.Log("E3 Method.Func.Call(récepteur, 4) =", out[0].Int())
}
```

`go test -race ./...` passe. Points de conception :

- **Tout est vérifié à la construction** : méthode introuvable, paramètre de type non géré, trop de
  résultats. Les erreurs sont jointes (`Absente : méthode introuvable…` et `Bad : paramètre 1 de type
  map[string]int non géré`). Un répartiteur mal configuré empêche le programme de démarrer, au lieu
  d'échouer à la première commande.
- **La panique** est contenue par un `recover` dans `Dispatch`, avec le nom de la méthode
  (`Boom a paniqué : plus de mémoire`) : un bug dans une commande ne fait pas tomber le service (N6 L12).
- **La liste blanche** est indispensable dès que les lignes viennent du réseau : sans elle, **toute**
  méthode exportée devient appelable à distance — `Shutdown` en est l'exemple. Une méthode exportée
  pour les besoins du code (appelée par un autre paquet) n'est pas pour autant destinée à des
  utilisateurs. Exporter n'est pas autoriser.

## Défi — Un *mock* universel avec `MakeFunc`

```go
// Fichier : spy/spy.go
package spy

import (
	"fmt"
	"reflect"
	"sync"
)

type Calls struct {
	mu      sync.Mutex
	args    [][]any
	results [][]any
}

func (c *Calls) record(in, out []reflect.Value) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.args = append(c.args, values(in))
	c.results = append(c.results, values(out))
}

func values(vs []reflect.Value) []any {
	out := make([]any, len(vs))
	for i, v := range vs {
		out[i] = v.Interface()
	}
	return out
}

func (c *Calls) Len() int              { c.mu.Lock(); defer c.mu.Unlock(); return len(c.args) }
func (c *Calls) Args(i int) []any      { c.mu.Lock(); defer c.mu.Unlock(); return c.args[i] }
func (c *Calls) Results(i int) []any   { c.mu.Lock(); defer c.mu.Unlock(); return c.results[i] }

// Spy rend une fonction du type de impl, qui lui délègue et enregistre chaque appel.
func Spy[F any](impl F) (F, *Calls) {
	c := &Calls{}
	fv := reflect.ValueOf(impl)
	if fv.Kind() != reflect.Func {
		panic(fmt.Sprintf("Spy : fonction attendue, reçu %T", impl)) // faute de programmation
	}
	f := reflect.MakeFunc(fv.Type(), func(in []reflect.Value) []reflect.Value {
		out := fv.Call(in)
		c.record(in, out)
		return out
	})
	return f.Interface().(F), c
}

// Stub rend une fonction de type F qui rend toujours results. Les types sont vérifiés ICI.
func Stub[F any](results ...any) (F, *Calls, error) {
	var zero F
	ft := reflect.TypeFor[F]()
	if ft.Kind() != reflect.Func {
		return zero, nil, fmt.Errorf("Stub : %s n'est pas un type de fonction", ft)
	}
	if ft.NumOut() != len(results) {
		return zero, nil, fmt.Errorf("Stub : %s rend %d valeur(s), %d fournie(s)", ft, ft.NumOut(), len(results))
	}
	out := make([]reflect.Value, len(results))
	for i, r := range results {
		want := ft.Out(i)
		if r == nil { // nil : valeur zéro du type attendu (une error nil, par exemple)
			switch want.Kind() {
			case reflect.Interface, reflect.Pointer, reflect.Slice, reflect.Map, reflect.Func, reflect.Chan:
				out[i] = reflect.Zero(want)
				continue
			}
			return zero, nil, fmt.Errorf("Stub : résultat %d nil pour le type %s", i+1, want)
		}
		v := reflect.ValueOf(r)
		if !v.Type().AssignableTo(want) {
			return zero, nil, fmt.Errorf("Stub : résultat %d de type %s, %s attendu", i+1, v.Type(), want)
		}
		rv := reflect.New(want).Elem()
		rv.Set(v)
		out[i] = rv
	}
	c := &Calls{}
	f := reflect.MakeFunc(ft, func(in []reflect.Value) []reflect.Value {
		c.record(in, out)
		return out
	})
	return f.Interface().(F), c, nil
}
```

```go
// Fichier : spy/spy_test.go
package spy

import (
	"context"
	"errors"
	"strings"
	"sync"
	"testing"
)

type SendMail func(ctx context.Context, to, body string) error

// Le service testé : il reçoit sa dépendance sous forme de fonction.
func Welcome(ctx context.Context, send SendMail, emails []string) error {
	var errs []error
	for _, e := range emails {
		if err := send(ctx, e, "Bienvenue !"); err != nil {
			errs = append(errs, err)
		}
	}
	return errors.Join(errs...)
}

func TestSpy(t *testing.T) {
	send, calls := Spy(SendMail(func(_ context.Context, to, _ string) error {
		if strings.HasSuffix(to, "@invalide") {
			return errors.New("refusé")
		}
		return nil
	}))
	err := Welcome(t.Context(), send, []string{"a@x.fr", "b@invalide"})
	if err == nil || calls.Len() != 2 || calls.Args(1)[1] != "b@invalide" || calls.Results(1)[0].(error).Error() != "refusé" {
		t.Fatalf("err=%v len=%d", err, calls.Len())
	}
}

func TestStub(t *testing.T) {
	send, calls, err := Stub[SendMail](nil) // error nil
	if err != nil {
		t.Fatal(err)
	}
	var wg sync.WaitGroup
	for range 50 {
		wg.Go(func() { _ = send(t.Context(), "x", "y") })
	}
	wg.Wait()
	if calls.Len() != 50 {
		t.Fatalf("%d appels", calls.Len())
	}
	if _, _, err := Stub[SendMail]("pas une error"); err == nil {
		t.Fatal("type de résultat incorrect accepté")
	} else {
		t.Log(err)
	}
	if _, _, err := Stub[SendMail](); err == nil {
		t.Fatal("nombre de résultats incorrect accepté")
	}
}
```

```go
// Fichier : spy/bench_test.go
package spy

import (
	"reflect"
	"testing"
)

func add(a, b int) int { return a + b }

func traced(fn func(int, int) int) func(int, int) int {
	fv := reflect.ValueOf(fn)
	return reflect.MakeFunc(fv.Type(), func(args []reflect.Value) []reflect.Value {
		return fv.Call(args)
	}).Interface().(func(int, int) int)
}

var sink int

func BenchmarkDirect(b *testing.B) {
	f := add
	for b.Loop() {
		sink = f(2, 3)
	}
}

func BenchmarkMakeFunc(b *testing.B) {
	f := traced(add)
	for b.Loop() {
		sink = f(2, 3)
	}
}
```

**Comparaison avec un faux écrit à la main.**

| | *Spy* réflexif | Faux écrit à la main (N5 L3) |
|---|---|---|
| Écriture | une ligne par dépendance | une struct ou une fonction par dépendance |
| Sécurité de type | l'appel est typé (`F`) ; mais `calls.Args(1)[1]` est un `any` à convertir | entièrement typé |
| Erreurs | `Stub` vérifie les types à la création, avec un message ; une erreur d'indice dans `Args` panique | erreurs de compilation |
| Lisibilité | magique pour qui ne connaît pas la réflexion | évidente |

Le *spy* vaut la peine quand il y a **beaucoup** de dépendances-fonctions à observer, de formes variées,
dans une grosse suite de tests. Pour une ou deux dépendances, un faux de cinq lignes est plus lisible et
plus sûr. Les bibliothèques de *mocks* de l'écosystème font ce choix pour vous, avec les mêmes
compromis ; la culture Go penche nettement vers les faux écrits à la main.

## Quiz

1. La méthode liée à cette valeur ; `NumIn` ne compte pas le récepteur.
2. Parce que l'ensemble de méthodes de `T` ne contient que les méthodes à récepteur valeur.
3. Une panique : `reflect: Call using string as type int`.
4. `t.NumOut() > 0 && t.Out(t.NumOut()-1) == reflect.TypeFor[error]()`.
5. Elle crée une fonction d'un type donné dont le corps reçoit des `[]reflect.Value` ; avec les
   génériques, l'appelant manipule une fonction typée.
6. Un pointeur vers une valeur zéro du type.
7. Parce qu'une valeur zéro est souvent inutilisable et que les champs non exportés ne peuvent pas être
   initialisés par réflexion ; un constructeur initialise correctement, sans réflexion.
8. `false` ; `true` (structs pointées égales), alors que `==` et `errors.Is` disent `false`.
9. Fuseau et lecture monotone font différer des instants identiques ; utiliser `Equal`.
10. `slices.Equal`, `maps.Equal`, `Equal` pour les dates, `errors.Is` pour les erreurs, ou `go-cmp`.
