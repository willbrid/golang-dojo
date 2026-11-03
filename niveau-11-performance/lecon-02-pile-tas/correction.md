# Leçon 2 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Lire les raisons

Extraits de `go build -gcflags='-m -m'` :

```
./esc.go:12:43: &Point{...} escapes to heap in NewPointer:
./esc.go:12:43:   flow: ~r0 ← &{storage for &Point{...}}:
./esc.go:12:43:     from return &Point{...} (return) at ./esc.go:12:36

./esc.go:23:51: p escapes to heap in Describe:
./esc.go:23:51:   flow: {storage for ... argument} ← &{storage for p}:
./esc.go:23:51:     from ... argument (slice-literal-element) at ./esc.go:23:50
./esc.go:23:51:   flow: {heap} ← {storage for ... argument}:
./esc.go:23:51:     from fmt.Sprint(... argument...) (call parameter) at ./esc.go:23:50

./esc.go:52:2: Counter capturing by ref: n (addr=false assign=true width=8)
./esc.go:52:2: n escapes to heap in Counter:
./esc.go:52:2:   flow: {storage for func literal} ← &n:
./esc.go:52:2:     from n (captured by a closure) at ./esc.go:53:22
```

Le cas 5 est le plus difficile à lire : `p` est placée dans la slice d'arguments variadiques de
`fmt.Sprint` (`... argument`), sous la forme d'une interface, et `fmt.Sprint` est un paramètre dont le
compilateur ne peut pas prouver qu'il ne garde rien (il passe par la réflexion) : la valeur part au tas.
Le cas 8 l'est moins : `n` est capturée **par référence** parce que la closure la modifie (`assign=true`).

## E2 — L'*inlining* change tout

Mesuré :

```
NewPointer-inline-local       1.5 ns    0 B   0 allocs
NewPointer-noinline-local    17.4 ns   16 B   1 allocs
DynamicBuffer(33) sans noinline   13 ns    0 B   0 allocs
```

*Inlinée*, `NewPointer` devient du code de l'appelant : le compilateur voit que le pointeur ne quitte pas la
boucle, et `Point` reste sur la pile. Sans *inlining*, il ne voit que « cette fonction rend un pointeur
vers une variable locale » : tas. De même, sans `//go:noinline`, `DynamicBuffer(33)` est *inlinée* avec la
**constante** 33 : `make([]byte, 33)` devient une taille connue, sur la pile. La même ligne de code alloue
ou non selon le contexte d'appel : c'est pourquoi on mesure dans des conditions réalistes.

## E3 — La frontière des 32 octets

```
buf-1        3 ns       0 B   0 allocs
buf-16      10 ns       0 B   0 allocs
buf-31      17 ns       0 B   0 allocs
buf-32      17 ns       0 B   0 allocs
buf-33      40 ns      48 B   1 allocs
buf-64      67 ns      64 B   1 allocs
buf-1024   713 ns    1024 B   1 allocs
buf-65536 40296 ns  65536 B   1 allocs
```

Jusqu'à 32 octets, le tampon réservé sur la pile suffit ; à partir de 33, allocation sur le tas. `B/op`
vaut 48 pour 33 octets demandés parce que l'allocateur de Go range les petits objets dans des **classes de
taille** fixes (8, 16, 24, 32, 48, 64… octets) : 33 octets prennent une case de 48. La différence de temps
entre 32 et 33 octets (17 → 40 ns) est le coût d'une allocation.

## E4 — Le `fmt` caché

```
key-fmt      159 ns   40 B   3 allocs   fmt.Sprintf("%s:%d", kind, id)
key-concat    65 ns   24 B   2 allocs   kind + ":" + strconv.Itoa(id)
key-append    39 ns   16 B   1 allocs   strconv.AppendInt dans un [32]byte
```

Pour une clé de cache appelée un million de fois par seconde : la version `AppendInt` (une seule allocation,
celle de la chaîne finale, inévitable si l'on veut une `string`). `fmt.Sprintf` coûte quatre fois plus et
trois allocations (la conversion des arguments en `any`, l'état interne, le résultat). Si la clé peut
rester un `[]byte` (par exemple comme clé d'une map via une conversion que le compilateur optimise, ou
écrite directement dans un tampon), on descend à zéro.

## E5 — L'échappement par un canal

```
chan   143 ns   136 B   3 allocs
```

Le compilateur ne suit pas les valeurs à travers un canal : tout ce qu'on y envoie est considéré comme
s'échappant (une autre goroutine pourrait le recevoir), et le canal lui-même, créé par `make(chan …)`, est
alloué sur le tas. Même si l'envoi et la réception sont dans la même fonction.

## Exercice intermédiaire — Un encodeur sans allocation

```go
// Fichier : event/event.go
// Exercice intermédiaire : encoder un événement de journal sans allocation.
package event

import (
	"fmt"
	"maps"
	"slices"
	"strconv"
	"strings"
	"time"
)

type Attr struct{ Key, Value string }

type Event struct {
	Time  time.Time
	Level int
	Msg   string
	Attrs []Attr // fournis TRIÉS par clé : c'est le contrat (voir le corrigé)
}

// Naive : la version « évidente », pour comparaison.
func Naive(e Event) string {
	m := map[string]string{}
	for _, a := range e.Attrs {
		m[a.Key] = a.Value
	}
	parts := []string{e.Time.UTC().Format(time.RFC3339), fmt.Sprintf("niveau=%d", e.Level), fmt.Sprintf("msg=%q", e.Msg)}
	for _, k := range slices.Sorted(maps.Keys(m)) {
		parts = append(parts, fmt.Sprintf("%s=%s", k, m[k]))
	}
	return strings.Join(parts, " ")
}

// AppendEvent ajoute l'événement à dst. Aucune allocation si dst a assez de capacité.
func AppendEvent(dst []byte, e Event) []byte {
	dst = e.Time.UTC().AppendFormat(dst, time.RFC3339)
	dst = append(dst, " niveau="...)
	dst = strconv.AppendInt(dst, int64(e.Level), 10)
	dst = append(dst, " msg="...)
	dst = strconv.AppendQuote(dst, e.Msg)
	for _, a := range e.Attrs {
		dst = append(dst, ' ')
		dst = append(dst, a.Key...)
		dst = append(dst, '=')
		dst = append(dst, a.Value...)
	}
	return dst
}
```

```go
// Fichier : event/event_test.go
package event

import (
	"slices"
	"strings"
	"testing"
	"time"
)

var e = Event{
	Time:  time.Date(2026, 3, 1, 10, 0, 0, 0, time.UTC),
	Level: 2,
	Msg:   `emprunt "refusé"`,
	Attrs: []Attr{{"livre", "EX-1"}, {"membre", "ada"}, {"raison", "trop_d_emprunts"}},
}

func TestSameOutput(t *testing.T) {
	if got, want := string(AppendEvent(nil, e)), Naive(e); got != want {
		t.Fatalf("\n%s\n%s", got, want)
	}
}

func FuzzSameOutput(f *testing.F) {
	f.Add(int64(0), 1, "msg", "k", "v")
	f.Fuzz(func(t *testing.T, sec int64, lvl int, msg, k, v string) {
		if strings.ContainsAny(k+v, " =") || k == "" || sec < 0 || sec > 1<<40 {
			t.Skip() // hors du format de journal (clés et valeurs simples)
		}
		ev := Event{Time: time.Unix(sec, 0), Level: lvl, Msg: msg, Attrs: []Attr{{k, v}}}
		if got, want := string(AppendEvent(nil, ev)), Naive(ev); got != want {
			t.Fatalf("\n%q\n%q", got, want)
		}
	})
}

func TestZeroAlloc(t *testing.T) {
	buf := make([]byte, 0, 256)
	n := testing.AllocsPerRun(1000, func() { buf = AppendEvent(buf[:0], e) })
	if n != 0 {
		t.Fatalf("%v allocations par appel", n)
	}
}

func TestAttrsMustBeSorted(t *testing.T) {
	if !slices.IsSortedFunc(e.Attrs, func(a, b Attr) int { return strings.Compare(a.Key, b.Key) }) {
		t.Fatal("contrat : attributs triés par clé")
	}
}

var sink []byte
var sinkS string

func BenchmarkNaive(b *testing.B) {
	for b.Loop() {
		sinkS = Naive(e)
	}
}

func BenchmarkAppend(b *testing.B) {
	buf := make([]byte, 0, 256)
	for b.Loop() {
		buf = AppendEvent(buf[:0], e)
	}
	sink = buf
}
```

Mesuré :

```
BenchmarkNaive-8     1686 ns/op   616 B/op   21 allocs/op
BenchmarkAppend-8     191 ns/op     0 B/op    0 allocs/op
```

Neuf fois plus rapide, et **aucune** allocation (vérifié par `testing.AllocsPerRun`). Même sortie que la
version naïve (table de tests et fuzzing, 15 s sans écart).

Le point 3 (attributs triés) est un choix de **contrat** : trier une map à chaque appel demande de collecter
les clés dans une slice, donc une allocation (ou une slice de travail fournie par l'appelant). Ici, les
attributs sont une **slice** de paires, fournie triée par l'appelant, et un test documente ce contrat.
`log/slog` (niveau 12) fait un choix proche : ses attributs sont une liste ordonnée, pas une map.

## Défi — La pile qui grandit

```go
// Fichier : stack/stack_test.go
// Défi : la pile d'une goroutine grandit en se recopiant.
package stack

import (
	"runtime"
	"sync"
	"testing"
	"time"
)

//go:noinline
func deep(n int) byte {
	var local [1024]byte // 1 Kio par appel
	local[n%1024] = byte(n)
	if n == 0 {
		return local[0]
	}
	return deep(n-1) + local[n%1024]
}

func stackInUse() uint64 {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	return m.StackInuse
}

func TestOneDeepStack(t *testing.T) {
	before := stackInUse()
	start := time.Now()
	done := make(chan uint64)
	go func() {
		deep(100_000)        // ≈ 100 Mio de pile
		done <- stackInUse() // mesuré pendant que la pile est encore grande
	}()
	during := <-done
	t.Logf("1 goroutine, profondeur 100 000 × 1 Kio : %v, StackInuse %d Mio → %d Mio",
		time.Since(start).Round(time.Millisecond), before>>20, during>>20)
}

func TestManyShallowStacks(t *testing.T) {
	before := stackInUse()
	var wg sync.WaitGroup
	release := make(chan struct{})
	var mu sync.Mutex
	var during uint64
	for i := range 100_000 {
		wg.Go(func() {
			deep(10)
			if i == 0 {
				mu.Lock()
				during = stackInUse()
				mu.Unlock()
			}
			<-release
		})
	}
	time.Sleep(200 * time.Millisecond)
	mu.Lock()
	d := stackInUse()
	mu.Unlock()
	close(release)
	wg.Wait()
	_ = during
	t.Logf("100 000 goroutines, profondeur 10 × 1 Kio : StackInuse %d Mio → %d Mio", before>>20, d>>20)
}
```

1. **Profondeur 1 000 000 × 1 Kio** (l'énoncé) : ≈ 1 Go de pile, et le programme s'arrête net :

   ```
   runtime: goroutine stack exceeds 1000000000-byte limit
   fatal error: stack overflow
   ```

   C'est une **erreur fatale**, pas une panique : `recover` ne peut rien (point 3). La limite par défaut est
   de 1 Go sur une machine 64 bits ; `runtime/debug.SetMaxStack` la modifie. Avec une profondeur de 100 000 :

   ```
   1 goroutine, profondeur 100 000 × 1 Kio : 151ms, StackInuse 0 Mio → 128 Mio
   ```

   ≈ 100 Mio demandés, 128 Mio occupés : la pile **double** à chaque agrandissement (puissance de deux
   suivante), et chaque doublement **recopie** la pile entière.
2. **100 000 goroutines de profondeur 10** : `StackInuse` → 2 284 Mio. Chacune a besoin d'environ 11 Kio, plus
   que sa pile initiale ; elle double jusqu'à 16 Kio (et davantage avec les marges du *runtime*). Beaucoup de
   petites piles coûtent ainsi plus cher qu'une seule grande. Les goroutines sont légères **tant que leur
   pile reste petite**.
3. Voir le point 1.
4. Puisque la pile se **déplace** en mémoire quand elle grandit, un pointeur vers elle tenu par une autre
   goroutine deviendrait invalide. Le *runtime* ne met à jour que les pointeurs internes à la pile qu'il
   recopie. D'où la règle : toute valeur dont l'adresse peut être vue en dehors de la goroutine (envoyée
   dans un canal, rangée dans une globale, capturée par une goroutine) doit vivre sur le **tas** — c'est
   exactement ce que garantit l'analyse d'échappement.

## Quiz

1. Réserver la place revient à déplacer un pointeur, et tout est libéré au retour de la fonction.
2. L'allocation elle-même, puis surtout le travail du ramasse-miettes.
3. Le compilateur ; une valeur reste sur la pile s'il peut prouver qu'aucune référence ne lui survit.
4. Valeur allouée sur le tas ; variable locale déplacée sur le tas ; paramètre pointeur conservé au-delà de
   l'appel.
5. L'interface contient un pointeur vers la valeur, souvent conservé par la fonction appelée.
6. La variable doit vivre aussi longtemps que la closure qui la référence.
7. Non : un `make` de taille variable passe par le tas au-delà de 32 octets (cas 7).
8. Une fonction *inlinée* est analysée dans le contexte de son appelant, ce qui évite souvent l'échappement.
9. La forme `Append…(dst []byte, …) []byte`, avec un tampon réutilisé par l'appelant.
10. Quand un profil montre une fonction qui alloue beaucoup sur un chemin chaud.
