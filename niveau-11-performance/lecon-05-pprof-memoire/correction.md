# Leçon 5 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Les corrections des exercices faciles, ajoutées au paquet `prof` du cours :

```go
// Fichier : fix.go
package prof

import (
	"strconv"
	"strings"
	"time"
)

// E1 : Report sans concaténation quadratique ni fmt.
func ReportFast(rows int) string {
	var b strings.Builder
	line := strings.Repeat("x", 20)
	b.Grow(rows * (len("ligne  : \n") + 6 + len(line)))
	for i := range rows {
		b.WriteString("ligne ")
		b.WriteString(strconv.Itoa(i))
		b.WriteString(" : ")
		b.WriteString(line)
		b.WriteByte('\n')
	}
	return b.String()
}

// E2 : le calcul HORS du verrou.
func (c *Cache) GetFast(k int) int {
	c.mu.Lock()
	v, ok := c.m[k]
	c.mu.Unlock()
	if ok {
		return v
	}
	v = slowSquare(k) // peut être calculé deux fois pour la même clé : sans gravité ici
	c.mu.Lock()
	c.m[k] = v
	c.mu.Unlock()
	return v
}

// E3 : canal bufferisé.
func PipelineBuffered(n int) int {
	ch := make(chan int, 100)
	done := make(chan int)
	go func() {
		sum := 0
		for v := range ch {
			time.Sleep(20 * time.Microsecond)
			sum += v
		}
		done <- sum
	}()
	for i := range n {
		ch <- i
	}
	close(ch)
	return <-done
}

// E4 : fuite corrigée, et la variante capturée non corrigée.
func FirstOf3Fixed() int {
	ch := make(chan int, 3)
	for i := range 3 {
		go func(v int) { ch <- v }(i)
	}
	return <-ch
}

func FirstOf3Captured() int {
	ch := make(chan int)
	for i := range 3 {
		go func() { ch <- i }()
	}
	return <-ch
}
```

```go
// Fichier : fix_test.go
package prof

import (
	"bytes"
	"runtime"
	"runtime/pprof"
	"strings"
	"testing"
	"time"
)

func BenchmarkReportFast(b *testing.B) {
	for b.Loop() {
		sink = ReportFast(200)
	}
}

func BenchmarkCacheFast(b *testing.B) {
	c := &Cache{m: map[int]int{}}
	b.RunParallel(func(pb *testing.PB) {
		i := 0
		for pb.Next() {
			c.GetFast(i % 5000)
			i++
		}
	})
}

func BenchmarkPipelineBuffered(b *testing.B) {
	for b.Loop() {
		PipelineBuffered(100)
	}
}

func leakTotal() string {
	var buf bytes.Buffer
	pprof.Lookup("goroutineleak").WriteTo(&buf, 1)
	return strings.SplitN(buf.String(), "\n", 2)[0]
}

func TestE4(t *testing.T) {
	g0 := runtime.NumGoroutine()
	for range 10 {
		FirstOf3Fixed()
	}
	time.Sleep(50 * time.Millisecond)
	t.Logf("corrigé : %s, goroutines en plus=%d", leakTotal(), runtime.NumGoroutine()-g0)
	g0 = runtime.NumGoroutine()
	for range 10 {
		FirstOf3Captured()
	}
	time.Sleep(50 * time.Millisecond)
	t.Logf("capturé : %s, goroutines en plus=%d", leakTotal(), runtime.NumGoroutine()-g0)
}
```

## E1 — Les quatre vues de la mémoire

Mesuré (par opération, `-benchmem`) : `Report` 702 754 o et 800 allocations ; `ReportFast` 8 536 o et
102 allocations, et 6 µs au lieu de 245 µs. Dans le profil :

- `alloc_space` et `alloc_objects` montrent `Report` (puis `ReportFast`) en tête : ces vues **cumulent**
  depuis le début du programme, donc leurs totaux dépendent du nombre d'itérations du benchmark (40 fois plus
  pour `ReportFast`, plus rapide) — on compare les **par opération** du benchmark, pas les totaux du profil ;
- `inuse_space` et `inuse_objects` sont **vides** pour les deux : aucune de ces chaînes n'est encore vivante
  au moment du dernier cycle. Pour un problème de GC, ce sont les mauvaises vues.

Les 102 allocations restantes viennent surtout de `strconv.Itoa` (une petite chaîne par nombre au-delà de
99) ; `strconv.AppendInt` dans le tampon les supprimerait (leçon 2).

## E2 — Sortir le calcul du verrou

À travail **égal** (`-benchtime 20000x`, deux séries) : `Cache` 22,2 et 22,7 µs par appel, `GetFast` 8,0 et
6,6 µs. Contention mesurée : **4,31 s** contre **6 ms**.

Piège de mesure rencontré : sans nombre d'itérations fixe, les résultats de `BenchmarkCache` variaient de
527 ns à 26 µs d'une exécution à l'autre. Le coût par appel dépend de la proportion d'absences dans le cache
(les 5 000 premières clés), donc du nombre total d'itérations choisi par le cadre de test. Un benchmark dont
l'état évolue doit fixer son travail (`-benchtime Nx`) ou se remettre à zéro.

Le nouveau risque : deux goroutines qui manquent la même clé en même temps calculent toutes deux
`slowSquare` et écrivent la même valeur. Ici c'est sans gravité (calcul pur, résultat identique). Si le
calcul était coûteux ou avait des effets, il faudrait un mécanisme « une seule exécution par clé »
(`golang.org/x/sync/singleflight`, ou une `map[int]*sync.Once`-like).

## E3 — Le canal bufferisé

```
Pipeline           27,6 ms
PipelineBuffered   28,0 ms
```

Aucun gain. Le profil de blocage montre que l'attente a **changé de place** : le producteur ne bloque plus
sur l'envoi (il remplit le tampon), mais sur `return <-done`, en attendant que le consommateur ait fini. Le
débit est fixé par le consommateur (100 éléments × ≈ 280 µs : `time.Sleep(20 µs)` dure en réalité bien plus
longtemps, la granularité des minuteries du système étant de l'ordre de la centaine de microsecondes). Un
tampon absorbe les **irrégularités** d'un débit ; il ne rend pas un étage lent plus rapide (N6 L3). Pour
cela, il faut plus de consommateurs, ou un consommateur plus rapide.

## E4 — Corriger la fuite

```
corrigé : goroutineleak profile: total 0, goroutines en plus=0
capturé : goroutineleak profile: total 0, goroutines en plus=20
```

Corrigée (canal de capacité 3), aucune goroutine ne reste. Avec la capture de la variable de boucle et le
canal non corrigé, 20 goroutines fuient (`runtime.NumGoroutine`) mais le profil affiche **0** : le faux
négatif du cours. Le nombre de goroutines, suivi dans le temps, reste l'indicateur qui ne ment pas.

## E5 — La fuite de mémoire

`inuse_space` désigne la ligne qui **alloue** (`make([]byte, …)` dans le handler), pas la map globale qui
**retient**. Le profil mémoire de Go enregistre où chaque objet a été créé, pas qui le référence. Pour
trouver le responsable :

- lire le code à partir du point d'allocation : où va la valeur ? (ici, dans la map) ;
- regarder **ce qui grandit** : un compteur de la taille de la map en métrique, ou des profils `inuse`
  successifs comparés (`go tool pprof -base ancien.out nouveau.out` montre ce qui a augmenté entre deux
  instants) ;
- en dernier recours, un vidage du tas (`runtime/debug.WriteHeapDump`) analysé par un outil dédié.

La technique `-base` est la plus utile en production : deux profils pris à une heure d'intervalle, et la
différence pointe ce qui accumule.

## Exercice intermédiaire — Enquête en production

Un service avec les trois défauts, et une option `-fixed` pour la version corrigée :

```go
// Fichier : cmd/svc/main.go
// Un service avec trois défauts : à retrouver par les profils.
package main

import (
	"encoding/json"
	"flag"
	"log"
	"net/http"
	_ "net/http/pprof"
	"runtime"
	"sync"
)

var fixed = flag.Bool("fixed", false, "version corrigée")

type item struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

var (
	mu    sync.Mutex
	items = func() []item {
		s := make([]item, 200)
		for i := range s {
			s[i] = item{i, "élément"}
		}
		return s
	}()
	pool = sync.Pool{New: func() any { return new([1 << 20]byte) }}
)

// Défaut 1 : une goroutine par requête qui attend un canal que personne n'alimentera.
func notify(w http.ResponseWriter, r *http.Request) {
	ch := make(chan string)
	if !*fixed {
		go func(c chan string) { <-c }(ch) // fuite (paramètre : détectable)
	}
	w.Write([]byte("ok"))
}

// Défaut 2 : le verrou global est tenu pendant l'encodage JSON.
func list(w http.ResponseWriter, r *http.Request) {
	if *fixed {
		mu.Lock()
		snapshot := append([]item(nil), items...) // copier sous verrou, encoder hors verrou
		mu.Unlock()
		json.NewEncoder(w).Encode(snapshot)
		return
	}
	mu.Lock()
	defer mu.Unlock()
	json.NewEncoder(w).Encode(items)
}

// Défaut 3 : un tampon de 1 Mio alloué à chaque requête.
func checksum(w http.ResponseWriter, r *http.Request) {
	var buf *[1 << 20]byte
	if *fixed {
		buf = pool.Get().(*[1 << 20]byte)
		defer pool.Put(buf)
	} else {
		buf = new([1 << 20]byte)
	}
	n := 0
	for i := 0; i < len(buf); i += 4096 {
		buf[i] = byte(i)
		n += int(buf[i])
	}
	w.Write([]byte{byte(n)})
}

func main() {
	flag.Parse()
	runtime.SetBlockProfileRate(1_000_000) // ≈ un événement par milliseconde d'attente
	runtime.SetMutexProfileFraction(100)   // une contention sur 100
	go func() { log.Fatal(http.ListenAndServe("127.0.0.1:6062", nil)) }()
	mux := http.NewServeMux()
	mux.HandleFunc("GET /notify", notify)
	mux.HandleFunc("GET /list", list)
	mux.HandleFunc("GET /checksum", checksum)
	log.Fatal(http.ListenAndServe("127.0.0.1:8082", mux))
}
```

Sous charge (12 clients, 10 s), profils lus par HTTP sur l'adresse d'administration :

| Défaut | Profil qui le révèle | Commande | Mesure (version défectueuse) |
|---|---|---|---|
| goroutine qui attend pour toujours | `goroutineleak` (et `goroutine`) | `curl …/debug/pprof/goroutineleak?debug=1` | **27 105** goroutines fuies (27 802 au total) |
| verrou tenu pendant le JSON | `mutex` **et** `block` | `go tool pprof -top -cum …/mutex` | `main.list` : **83 %** de la contention ; 51 % du temps de blocage dans `Mutex.Lock` |
| 1 Mio par requête | `allocs` (`alloc_space`) | `go tool pprof -sample_index=alloc_space -top …/allocs` | `main.checksum` : **9,2 Go** alloués, 98 % du total |

Résultat après correction : **333 851** requêtes contre **89 049** (×3,7), 0 goroutine fuie, et le premier
poste d'allocations devient l'encodage JSON de `list` (202 Mo), suivi du `sync.Pool` lui-même (53 Mo : les
tampons qu'il crée au démarrage et quand le GC le vide).

Deux remarques de méthode :

- dans le profil de contention, `top` (trié par temps propre) met en tête `sync.(*Mutex).Unlock`, et
  `main.list` n'apparaît pas dans les trois premières lignes ; il faut `top -cum` (ou `-focus=main.list`) pour
  le voir à 83 %. Même règle qu'à la leçon 4 ;
- le coût des profils échantillonnés (`SetBlockProfileRate(1_000_000)`, `SetMutexProfileFraction(100)`) n'a
  pas été distinguable du bruit sur cette charge (point 5 à mesurer sur ta machine, avec et sans, en
  alternant).

## Défi — Comprendre le faux négatif de `goroutineleak`

Variantes mesurées (un programme par cas, fuites réelles connues) :

```go
// Fichier : leakvariants/main.go
package main

import (
	"fmt"
	"runtime/pprof"
	"strings"
	"time"
)

func capturedNoLoop() {
	ch := make(chan int)
	x := 7
	go func() { ch <- x }()
}

func capturedModified() {
	ch := make(chan int)
	x := 7
	go func() { ch <- x }()
	x++ // capture par RÉFÉRENCE (variable modifiée après la capture)
}

func loopRange() {
	ch := make(chan int)
	for i := range 2 {
		go func() { ch <- i }()
	}
}

func loopReadBefore() {
	ch := make(chan int)
	for i := range 2 {
		v := i
		go func() { ch <- v }()
	}
}

func loopClassic() {
	ch := make(chan int)
	for i := 0; i < 2; i++ {
		go func() { ch <- i }()
	}
}

func loopSlice() {
	ch := make(chan string)
	for _, s := range []string{"a", "b"} {
		go func() { ch <- s }()
	}
}

func param() {
	ch := make(chan int)
	for i := range 2 {
		go func(v int) { ch <- v }(i)
	}
}

func total() int {
	var b strings.Builder
	pprof.Lookup("goroutineleak").WriteTo(&b, 1)
	var n int
	fmt.Sscanf(strings.SplitN(b.String(), "\n", 2)[0], "goroutineleak profile: total %d", &n)
	return n
}

func main() {
	prev := 0
	for _, c := range []struct {
		name string
		fn   func()
		n    int
	}{
		{"capturée, hors boucle", capturedNoLoop, 1},
		{"capturée et modifiée", capturedModified, 1},
		{"boucle range, capturée", loopRange, 2},
		{"boucle, copie v := i", loopReadBefore, 2},
		{"boucle classique, capturée", loopClassic, 2},
		{"range sur slice, capturée", loopSlice, 2},
		{"paramètre", param, 2},
	} {
		c.fn()
		time.Sleep(50 * time.Millisecond)
		t := total()
		fmt.Printf("%-28s fuites réelles=%d  détectées=%d\n", c.name, c.n, t-prev)
		prev = t
	}
}
```

```
capturée, hors boucle        fuites réelles=1  détectées=0
capturée et modifiée         fuites réelles=1  détectées=1
boucle range, capturée       fuites réelles=2  détectées=0
boucle, copie v := i         fuites réelles=2  détectées=0
boucle classique, capturée   fuites réelles=2  détectées=0
range sur slice, capturée    fuites réelles=2  détectées=0
paramètre                    fuites réelles=2  détectées=2
```

1. **Le motif** n'est pas lié aux boucles : toute fermeture qui capture une variable **par valeur** en plus
   du canal (la variable n'est pas modifiée après la capture, le compilateur en recopie la valeur dans la
   fermeture) masque la fuite. La capturer **par référence** (`capturée et modifiée` : `moved to heap: x`
   dans `-gcflags=-m`), la passer en **paramètre**, ou ne capturer que le canal (cours) : la fuite est
   détectée.
2. **`-gcflags=-m`** : `func literal escapes to heap` pour toutes les variantes (la fermeture d'une goroutine
   part toujours sur le tas) ; seule la variante « modifiée » ajoute `moved to heap: x`. La corrélation tient
   donc à la **forme de l'objet fermeture** (le canal et une valeur copiée, ou le canal et un pointeur vers une
   variable du tas), pas au fait de s'échapper.
3. **Hypothèse** (non vérifiée dans le code du *runtime*) : la détection considère une goroutine comme
   « pouvant encore être débloquée » si l'objet sur lequel elle attend est atteignable depuis des objets
   marqués ; selon la façon dont l'objet fermeture est référencé et parcouru, le canal serait considéré comme
   atteignable dans certains cas. Une hypothèse à confirmer ou réfuter en lisant `findMaybeRunnableGoroutines`
   dans `runtime/mgc.go`.
4. **Signaler** : sur le gestionnaire de tickets officiel du projet Go (`github.com/golang/go/issues`), avec
   la version exacte (`go version`), le programme minimal ci-dessus, la sortie obtenue et la sortie attendue.
   C'est ainsi que les défauts des outils se corrigent.

## Quiz

1. Le profil de blocage (attentes), et celui de contention si des verrous sont en jeu.
2. `alloc_space` : tout ce qui a été alloué depuis le début (charge du GC) ; `inuse_space` : ce qui est
   encore vivant (occupation de la mémoire).
3. `block` et `mutex` : `runtime.SetBlockProfileRate`, `runtime.SetMutexProfileFraction`, ou
   `go test -blockprofile` / `-mutexprofile`.
4. À la pile de la goroutine qui tenait le verrou, au moment de son `Unlock`.
5. Parce que le `defer c.mu.Unlock()` s'exécute au retour de la fonction.
6. Parce que le cadre de test, dans `main.main`, attend la fin du benchmark.
7. Il garantit que les goroutines listées sont réellement perdues ; il ne garantit pas de toutes les trouver.
8. Par `net/http/pprof`, sur une adresse d'administration.
9. Pour limiter leur coût tout en gardant une image statistiquement fidèle.
10. En comparant deux profils `inuse` dans le temps (`-base`), en suivant ce qui grandit, et en lisant le
    code à partir du point d'allocation.
