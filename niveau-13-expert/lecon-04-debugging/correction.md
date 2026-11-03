# Module 4 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Qui modifie cette variable ?

```go
// Fichier : cmd/watch/main.go
// Commande watch : quelqu'un modifie limit par erreur (E1).
package main

import "fmt"

var limit = 100

type config struct{ max *int }

func load() config { return config{max: &limit} } // un pointeur vers la globale… qui fuit

func normalize(c config, v int) int {
	if v > *c.max {
		*c.max = v // l'erreur : on voulait plafonner v, on modifie la limite
	}
	return v
}

func main() {
	c := load()
	for _, v := range []int{50, 80, 250, 90} {
		normalize(c, v)
	}
	fmt.Println("limit =", limit)
}
```

```
$ cat dlv-watch.txt
break main.main
continue
watch -w main.limit
continue
stack
print v
exit
$ dlv debug ./cmd/watch --init dlv-watch.txt
Breakpoint 1 set at 0x4b6413 for main.main() ./cmd/watch/main.go:19
Watchpoint main.limit set at 0x572480
> watchpoint on [main.limit] main.normalize() ./cmd/watch/main.go:14 (hits goroutine(1):1 total:1) (PC: 0x4b63ec)
=>  14:			*c.max = v // l'erreur : on voulait plafonner v, on modifie la limite
0  0x00000000004b63ec in main.normalize
   at ./cmd/watch/main.go:14
1  0x00000000004b64ce in main.main
   at ./cmd/watch/main.go:22
…
250
```

Le point d'observation s'arrête sur **l'instruction qui écrit**, `*c.max = v`, ligne 14, alors que le code
ne mentionne jamais `limit` à cet endroit. L'écriture passe par un pointeur obtenu dans `load`. La pile dit
qui a appelé `normalize`, et `v` vaut 250, la valeur qui a remplacé 100. Lire le code pour chercher « où
`limit` change » n'aurait trouvé que la déclaration.

Les limites :
- **un point d'observation utilise un registre de débogage du processeur** : quatre au plus sur x86-64, et
  chacun surveille au plus 8 octets alignés ;
- **une variable locale** se surveille seulement tant que sa portée existe ;
- **on pose le point quand la variable existe**, d'où le `break main.main` d'abord. Pour une globale, ce
  n'est pas indispensable, mais c'est sûr.

## E2 — Un interblocage dans un vidage

```go
// Fichier : cmd/deadlock/main.go
// Commande deadlock : deux verrous pris dans l'ordre inverse ; le programme se fige (E2).
package main

import (
	"fmt"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

var accounts, audit sync.Mutex

func transfer() { // accounts puis audit
	for {
		accounts.Lock()
		time.Sleep(time.Millisecond)
		audit.Lock()
		audit.Unlock()
		accounts.Unlock()
	}
}

func report() { // audit puis accounts : l'ordre inverse
	for {
		audit.Lock()
		time.Sleep(time.Millisecond)
		accounts.Lock()
		accounts.Unlock()
		audit.Unlock()
	}
}

func main() {
	go transfer()
	go report()
	fmt.Println("pid", os.Getpid())
	sig := make(chan os.Signal, 1)
	signal.Notify(sig, syscall.SIGTERM) // main attend : le runtime ne voit pas d'interblocage global
	<-sig
}
```

`gdb` (et donc `gcore`) n'est pas installé ici : on provoque le vidage par `kill -ABRT` sur le processus
figé, avec `GOTRACEBACK=crash`.

```
$ ulimit -c unlimited; GOTRACEBACK=crash ./deadlock & sleep 2; kill -ABRT $!
pid 853431
… Abandon (core dumped)
$ dlv core ./deadlock dl.core --init dlv-dl.txt      # goroutines -s
  Goroutine 1 - Start: …/runtime/proc.go:153 runtime.main [chan receive]
  …
  Goroutine 7 - Start: ./cmd/deadlock/main.go:15 main.transfer [sync.Mutex.Lock]
  Goroutine 8 - Start: ./cmd/deadlock/main.go:25 main.report [sync.Mutex.Lock]
$ dlv core ./deadlock dl.core --init dlv-dl2.txt
(*sync.Mutex)(0x5a16f8)                 # &main.accounts
(*sync.Mutex)(0x5a1700)                 # &main.audit
Switched from 0 to 7
Frame 6: /usr/local/go/src/sync/mutex.go:46
(*sync.Mutex)(0x5a1700)                 # la goroutine 7 attend audit…
Frame 7: cmd/deadlock/main.go:19
=>  19:			audit.Lock()
Switched from 7 to 8
Frame 6: /usr/local/go/src/sync/mutex.go:46
(*sync.Mutex)(0x5a16f8)                 # … la goroutine 8 attend accounts
Frame 7: cmd/deadlock/main.go:29
=>  29:			accounts.Lock()
```

(Le second script : `print &main.accounts`, `print &main.audit`, puis pour chaque goroutine `frame 6`,
`print m` et `frame 7`.)

On identifie le cycle sans reproduire, en comparant **adresses** :
- `transfer` (goroutine 7) tient `accounts`, car il l'a pris ligne 17, et attend `audit` ligne 19 ;
- `report` (goroutine 8) tient `audit`, et attend `accounts` ligne 29.

Le *runtime* ne signale **pas** d'interblocage (`all goroutines are asleep`) : `main` attend un signal, il
reste donc une goroutine qui pourrait progresser. Ce sont les interblocages **partiels**, les plus fréquents
dans un service, et c'est là que le vidage, ou `/debug/pprof/goroutine?debug=2` (N12 L9), est
indispensable.

## E3 — Corriger le pic

```go
// rebuild corrigé (dans cmd/spike/main.go)
var held atomic.Int64 // durée maximale de détention du verrou par rebuild, en ns

func rebuild() {
	next := map[int]int{}
	for i := range 2_000_000 {
		next[i%1000] += i
	}
	time.Sleep(100 * time.Millisecond) // un appel lent « pour charger les données »
	t0 := time.Now()
	mu.Lock()
	cache = next
	mu.Unlock()
	…
}
```

```
$ ./spike           # deux exécutions
verrou tenu par rebuild au plus 629ns
enregistreur=true : 67492 requêtes, p50 1.17ms, p99 1.46ms, p99,9 2.16ms, max 4ms
verrou tenu par rebuild au plus 290ns
enregistreur=true : 67468 requêtes, p50 1.17ms, p99 1.46ms, p99,9 2.28ms, max 4ms
$ ls pic.trace
aucun instantané : pas de pic
```

Le maximum passe de **153 ms à 4 ms**. Le verrou n'est plus tenu que pour échanger une référence : 0,3 à
0,6 µs, au lieu de 150 ms. Le débit gagne 7 %, et l'enregistreur de vol ne se déclenche plus. p50, p99 et
p99,9 ne bougent presque pas, ce qui confirme qu'ils ne voyaient pas le défaut.

## E4 — La latence de l'ordonnanceur

```go
// Fichier : sched/doc.go
// Package sched : mesurer la latence de l'ordonnanceur (E4).
package sched
```

```go
// Fichier : sched/sched_test.go
package sched

import (
	"fmt"
	"runtime"
	"slices"
	"testing"
	"time"
)

var sink uint64

// spin calcule sans aucun appel de fonction ni allocation : sans préemption asynchrone, il garderait
// son fil jusqu'à la fin.
func spin(d time.Duration) {
	end := time.Now().Add(d)
	x := uint64(1)
	for time.Now().Before(end) {
		for range 1_000_000 {
			x = x*6364136223846793005 + 1
		}
	}
	sink = x
}

// E4 : une goroutine veut s'exécuter toutes les 1 ms pendant qu'une autre calcule sans relâche.
func TestSchedLatency(t *testing.T) {
	for _, procs := range []int{1, 8} {
		prev := runtime.GOMAXPROCS(procs)
		go spin(2 * time.Second)
		var late []time.Duration
		for range 1000 {
			t0 := time.Now()
			time.Sleep(time.Millisecond)
			late = append(late, time.Since(t0)-time.Millisecond) // le retard au réveil
		}
		slices.Sort(late)
		fmt.Printf("GOMAXPROCS=%d : retard au réveil p50 %v, p99 %v, max %v\n", procs,
			late[500].Round(time.Microsecond), late[990].Round(time.Microsecond), late[999].Round(time.Microsecond))
		runtime.GOMAXPROCS(prev)
		time.Sleep(2 * time.Second) // laisser finir spin
	}
}
```

```
$ go test -v -run SchedLatency -trace sched.trace ./sched/
GOMAXPROCS=1 : retard au réveil p50 149µs, p99 19.277ms, max 39.483ms
GOMAXPROCS=8 : retard au réveil p50 96µs, p99 258µs, max 410µs
$ go tool trace -pprof=sched sched.trace > sched.pprof && go tool pprof -top sched.pprof
      flat  flat%   sum%        cum   cum%
    2.21ms  3.84%  3.84%     2.21ms  3.84%  runtime.asyncPreempt2
…
         0     0%  3.87%     2.21ms  3.84%  example.com/dbg/sched.spin
```

- **Avec un seul processeur logique**, la goroutine qui veut se réveiller toutes les millisecondes attend
  jusqu'à **39 ms**, et 19 ms au p99. Elle ne reprend la main que quand `spin` est **préempté**.
- **La préemption asynchrone existe** (depuis Go 1.14) : `runtime.asyncPreempt2` apparaît dans le profil
  d'ordonnancement, **dans** `spin`, une boucle sans appel ni allocation. Sans elle, l'attente durerait les
  2 s du calcul. Mais elle est grossière : le *runtime* préempte une goroutine qui s'exécute depuis environ
  10 ms, d'où des retards de 10 à 40 ms.
- **Avec 8 processeurs logiques**, la goroutine de réponse trouve un processeur libre : 410 µs au pire.

La leçon pour un service : un calcul lourd (encodage d'un gros document, compression) dans une goroutine de
requête ne bloque pas les autres requêtes tant qu'il reste des processeurs logiques libres. Avec
`GOMAXPROCS` petit (un conteneur limité à 1 processeur), il ajoute des **dizaines de millisecondes** à leur
latence. On isole alors ces calculs (un pool borné de workers, N6 L9), ou on les découpe.

## E5 — Régler le détecteur de races

```go
// Fichier : races/doc.go
// Package races : deux courses pour essayer GORACE (E5).
package races
```

```go
// Fichier : races/races_test.go
package races

import (
	"sync"
	"testing"
)

var hits, misses int // deux compteurs partagés sans protection : deux courses distinctes

func TestTwoRaces(t *testing.T) {
	var wg sync.WaitGroup
	for range 4 {
		wg.Go(func() {
			for i := range 1000 {
				if i%2 == 0 {
					hits++
				} else {
					misses++
				}
			}
		})
	}
	wg.Wait()
}
```

| Réglage | Rapports affichés | Comportement |
|---|---|---|
| défaut | 4 | le test se termine, puis `race detected during execution of test` et `FAIL` |
| `GORACE=halt_on_error=1` | **1** | le processus s'arrête au premier rapport : pas de résumé du test, `FAIL` du paquet |
| `GORACE=log_path=/tmp/race` | 0 sur la sortie | les 4 rapports sont dans le fichier `/tmp/race.<pid>` |

Deux variables, chacune lue et écrite par plusieurs goroutines, donnent quatre rapports : chaque paire
d'accès conflictuels distincte en produit un.

- **`halt_on_error`** sert dans une instance de pré-production (cours) : la première course suffit, et un
  processus qui continue après une course peut corrompre d'autres données.
- **`log_path`** sépare les rapports des journaux du service, dans un fichier par processus.

**`failed to restore the stack`** apparaît quand le détecteur ne retrouve plus la pile de l'accès
**précédent**, trop ancien pour son historique (programmes longs, beaucoup d'accès entre les deux). Le
rapport ne montre alors qu'une des deux piles. On augmente `history_size` (de 0 à 7, la mémoire de
l'historique doublant à chaque cran), au prix de plus de mémoire. On ne l'a pas provoqué ici : il faut un
programme qui tourne longtemps entre les deux accès.

---

## Exercice intermédiaire — Une enquête complète

**Le journal d'enquête.**

| Symptôme | Ce qui l'a révélé | Cause |
|---|---|---|
| plantage sous charge | le test de charge sans `-race` : `fatal error: concurrent map read and map write` ; puis `-race` : 3 rapports, dont la lecture de `c.counts` dans la fonction de tri | `Top` trie en lisant `c.counts` **hors** du verrou, pendant que `Visit` écrit |
| page en double | un **invariant au repos** : chaque page de `counts` exactement une fois dans `order` | `Visit` relâche le verrou entre « la page est nouvelle » et l'ajout : deux visites simultanées l'ajoutent toutes les deux |
| page absente | le même invariant (pages de `counts` absentes d'`order`), et `-race` sur `c.order` | l'`append` hors du verrou : deux `append` concurrents peuvent écrire au même emplacement, et l'un des deux est perdu |
| mémoire qui grossit | un profil mémoire `inuse_space` (N11 L5), ou simplement la mesure du tas | `History` garde **chaque** classement, sans limite et sans verrou |

Pour la mémoire, la mesure avant et après correction :

```
 10000 classements : original +1795 Kio retenus, corrigé +20 Kio
100000 classements : original +18039 Kio retenus, corrigé +0 Kio
```

La version corrigée :

```go
// Fichier : top/top.go
// Package top : compter les visites par page et servir un classement (version corrigée).
//
// Invariants (sous mu) : chaque page de counts apparaît exactement une fois dans order, et
// len(order) == len(counts). Aucune donnée partagée n'est lue hors du verrou.
package top

import (
	"cmp"
	"slices"
	"sync"
)

type Counter struct {
	mu     sync.Mutex
	counts map[string]int
	order  []string

	histMu  sync.Mutex
	history [][]string // classements récents, bornés (anneau)
	next    int
}

const historySize = 100

func New() *Counter { return &Counter{counts: map[string]int{}} }

// Visit : la vérification « page nouvelle ? » et l'ajout se font sous le MÊME verrou. L'original
// relâchait le verrou entre les deux : deux visites simultanées d'une page nouvelle l'ajoutaient
// deux fois (doublon), et l'append concurrent pouvait en perdre une (page absente).
func (c *Counter) Visit(page string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	if _, ok := c.counts[page]; !ok {
		c.order = append(c.order, page)
	}
	c.counts[page]++
}

type entry struct {
	page string
	n    int
}

// Top copie, SOUS le verrou, les pages et leurs totaux, puis trie hors du verrou. L'original lisait
// c.counts pendant le tri, sans verrou : course de données, et « concurrent map read and map write ».
func (c *Counter) Top(n int) []string {
	c.mu.Lock()
	snap := make([]entry, len(c.order))
	for i, p := range c.order {
		snap[i] = entry{p, c.counts[p]}
	}
	c.mu.Unlock()
	slices.SortStableFunc(snap, func(a, b entry) int { return cmp.Compare(b.n, a.n) })
	out := make([]string, 0, min(n, len(snap)))
	for _, e := range snap[:min(n, len(snap))] {
		out = append(out, e.page)
	}
	return out
}

// TopLogged garde au plus historySize classements : l'original les accumulait sans fin (et sans verrou).
func (c *Counter) TopLogged(n int) []string {
	t := c.Top(n)
	c.histMu.Lock()
	if len(c.history) < historySize {
		c.history = append(c.history, t)
	} else {
		c.history[c.next] = t
		c.next = (c.next + 1) % historySize
	}
	c.histMu.Unlock()
	return t
}

// check vérifie les invariants (pour les tests).
func (c *Counter) check() (dups, missing int) {
	c.mu.Lock()
	defer c.mu.Unlock()
	seen := map[string]bool{}
	for _, p := range c.order {
		if seen[p] {
			dups++
		}
		seen[p] = true
	}
	for p := range c.counts {
		if !seen[p] {
			missing++
		}
	}
	return
}
```

```go
// Fichier : top/top_test.go
package top

import (
	"runtime"
	"strconv"
	"sync"
	"testing"
)

// load reproduit la charge des symptômes : 8 goroutines qui visitent et demandent le classement.
func load(c *Counter) {
	var wg sync.WaitGroup
	for g := range 8 {
		wg.Go(func() {
			for i := range 2000 {
				c.Visit("p" + strconv.Itoa((g*7+i)%300))
				if i%200 == 0 {
					c.TopLogged(10)
				}
			}
		})
	}
	wg.Wait()
}

// Symptômes 2 et 3 : doublons et pages absentes (invariants vérifiés au repos). Le symptôme 1
// (plantage) est couvert par go test -race sur ce même test.
func TestInvariants(t *testing.T) {
	for range 50 {
		c := New()
		load(c)
		if d, m := c.check(); d != 0 || m != 0 {
			t.Fatalf("doublons %d, absentes %d", d, m)
		}
	}
}

func TestTopOrder(t *testing.T) {
	c := New()
	for _, p := range []string{"a", "b", "b", "c", "c", "c"} {
		c.Visit(p)
	}
	if got := c.Top(2); len(got) != 2 || got[0] != "c" || got[1] != "b" {
		t.Fatalf("%v", got)
	}
}

// Symptôme 4 : la mémoire retenue par l'historique reste bornée.
func TestHistoryBounded(t *testing.T) {
	c := New()
	for range 10_000 {
		c.TopLogged(10)
	}
	if len(c.history) != historySize {
		t.Fatalf("historique : %d", len(c.history))
	}
	runtime.GC()
}
```

```
$ go test -race -v ./top/
--- PASS: TestInvariants (1.04s)
--- PASS: TestTopOrder (0.00s)
--- PASS: TestHistoryBounded (0.01s)
```

Quatre corrections :
- **Les invariants sous un seul verrou.** La vérification et l'ajout dans `Visit` ne font plus qu'une section
  critique.
- **Un instantané pris sous le verrou.** `Top` copie les couples (page, total) sous le verrou, puis trie la
  copie, qui n'appartient qu'à lui.
- **Un historique borné**, protégé par son propre verrou.
- **Un test par symptôme** : `-race` pour le plantage, l'invariant au repos pour les doublons et les
  absences, la borne pour la mémoire.

Un symptôme, deux causes : le plantage révèle **une** course, mais `-race` en signale **trois**. On les corrige
toutes, pas seulement celle qui plante.

---

## Défi — Une boîte noire pour la production

```go
// Fichier : blackbox/blackbox.go
// Package blackbox : une boîte noire pour la production. Un enregistreur de vol tourne en permanence ;
// une requête trop lente déclenche un instantané (trace + profil de contention), au plus un par
// période, gardé en mémoire et téléchargeable sur l'adresse d'administration.
package blackbox

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"runtime/pprof"
	"runtime/trace"
	"sync"
	"time"
)

// Recorder : ce dont la boîte noire a besoin de l'enregistreur de vol (remplaçable dans les tests).
type Recorder interface {
	WriteTo(io.Writer) (int64, error)
}

type Snapshot struct {
	ID       int           `json:"id"`
	At       time.Time     `json:"at"`
	Route    string        `json:"route"`
	Duration time.Duration `json:"duration"`
	TraceID  string        `json:"trace_id,omitempty"`
	Size     int           `json:"size"`
	trace    []byte
	mutex    []byte
}

type Box struct {
	rec       Recorder
	threshold time.Duration
	every     time.Duration // au plus un instantané par période
	keep      int           // nombre d'instantanés gardés
	now       func() time.Time
	traceID   func(*http.Request) string

	mu    sync.Mutex
	last  time.Time
	next  int
	snaps []*Snapshot
	busy  bool
}

type Options struct {
	Threshold, Every time.Duration
	Keep             int
	Now              func() time.Time
	TraceID          func(*http.Request) string // par exemple depuis le contexte OpenTelemetry (N12 L7)
}

// Start démarre un vrai enregistreur de vol et renvoie la boîte noire, et la fonction d'arrêt.
func Start(o Options) (*Box, func(), error) {
	fr := trace.NewFlightRecorder(trace.FlightRecorderConfig{MinAge: 5 * time.Second, MaxBytes: 16 << 20})
	if err := fr.Start(); err != nil {
		return nil, nil, err
	}
	return New(fr, o), fr.Stop, nil
}

func New(rec Recorder, o Options) *Box {
	if o.Now == nil {
		o.Now = time.Now
	}
	if o.TraceID == nil {
		o.TraceID = func(*http.Request) string { return "" }
	}
	return &Box{rec: rec, threshold: o.Threshold, every: o.Every, keep: o.Keep, now: o.Now, traceID: o.TraceID}
}

// Middleware mesure chaque requête et déclenche un instantané si elle dépasse le seuil.
func (b *Box) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := b.now()
		next.ServeHTTP(w, r)
		if d := b.now().Sub(start); d > b.threshold {
			b.trigger(r.Pattern, d, b.traceID(r))
		}
	})
}

func (b *Box) trigger(route string, d time.Duration, traceID string) {
	b.mu.Lock()
	now := b.now()
	if b.busy || (!b.last.IsZero() && now.Sub(b.last) < b.every) {
		b.mu.Unlock()
		return // limite de fréquence : ne pas aggraver un incident en écrivant des traces en rafale
	}
	b.busy, b.last = true, now
	b.next++
	id := b.next
	b.mu.Unlock()

	var tr, mx bytes.Buffer
	b.rec.WriteTo(&tr) // hors du verrou : quelques centaines de Kio à copier
	if p := pprof.Lookup("mutex"); p != nil {
		p.WriteTo(&mx, 0)
	}
	s := &Snapshot{ID: id, At: now, Route: route, Duration: d, TraceID: traceID, Size: tr.Len(), trace: tr.Bytes(), mutex: mx.Bytes()}

	b.mu.Lock()
	b.busy = false
	b.snaps = append(b.snaps, s)
	if len(b.snaps) > b.keep {
		b.snaps = b.snaps[len(b.snaps)-b.keep:]
	}
	b.mu.Unlock()
}

// Snapshots renvoie les métadonnées des instantanés gardés.
func (b *Box) Snapshots() []Snapshot {
	b.mu.Lock()
	defer b.mu.Unlock()
	out := make([]Snapshot, len(b.snaps))
	for i, s := range b.snaps {
		out[i] = *s
		out[i].trace, out[i].mutex = nil, nil
	}
	return out
}

// Admin : GET /debug/blackbox (liste), GET /debug/blackbox/{id} (trace), …/{id}/mutex (contention).
// À monter sur l'adresse d'ADMINISTRATION seulement : une trace contient des piles, des noms, des durées.
func (b *Box) Admin(mux *http.ServeMux) {
	mux.HandleFunc("GET /debug/blackbox", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(b.Snapshots())
	})
	get := func(w http.ResponseWriter, r *http.Request, pick func(*Snapshot) []byte, name string) {
		var id int
		fmt.Sscan(r.PathValue("id"), &id)
		b.mu.Lock()
		defer b.mu.Unlock()
		for _, s := range b.snaps {
			if s.ID == id {
				w.Header().Set("Content-Disposition", fmt.Sprintf("attachment; filename=%q", fmt.Sprintf("blackbox-%d.%s", id, name)))
				w.Write(pick(s))
				return
			}
		}
		http.NotFound(w, r)
	}
	mux.HandleFunc("GET /debug/blackbox/{id}", func(w http.ResponseWriter, r *http.Request) {
		get(w, r, func(s *Snapshot) []byte { return s.trace }, "trace")
	})
	mux.HandleFunc("GET /debug/blackbox/{id}/mutex", func(w http.ResponseWriter, r *http.Request) {
		get(w, r, func(s *Snapshot) []byte { return s.mutex }, "pprof")
	})
}
```

```go
// Fichier : blackbox/blackbox_test.go
package blackbox

import (
	"bytes"
	"encoding/json"
	"io"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"testing/synctest"
	"time"
)

type fakeRec struct{ calls int }

func (f *fakeRec) WriteTo(w io.Writer) (int64, error) {
	f.calls++
	n, err := io.WriteString(w, "go 1.26 trace (factice)")
	return int64(n), err
}

// Seuil et limite de fréquence, en temps virtuel.
func TestThresholdAndRate(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		rec := &fakeRec{}
		b := New(rec, Options{Threshold: 50 * time.Millisecond, Every: time.Minute, Keep: 3})
		mux := http.NewServeMux()
		mux.HandleFunc("GET /slow/{n}", func(w http.ResponseWriter, r *http.Request) {
			d, _ := time.ParseDuration(r.PathValue("n"))
			time.Sleep(d) // temps virtuel
		})
		h := b.Middleware(mux)
		do := func(d string) { h.ServeHTTP(httptest.NewRecorder(), httptest.NewRequest("GET", "/slow/"+d, nil)) }
		do("10ms") // sous le seuil
		do("80ms") // instantané 1
		do("90ms") // trop tôt : ignoré
		time.Sleep(time.Minute)
		do("70ms") // instantané 2
		if rec.calls != 2 {
			t.Fatalf("%d instantanés, attendu 2", rec.calls)
		}
		s := b.Snapshots()
		if len(s) != 2 || s[0].Route != "GET /slow/{n}" || s[0].Duration != 80*time.Millisecond {
			t.Fatalf("%+v", s)
		}
	})
}

// Avec le vrai enregistreur de vol : l'instantané téléchargé est bien une trace d'exécution.
func TestRealSnapshot(t *testing.T) {
	b, stop, err := Start(Options{Threshold: 10 * time.Millisecond, Every: time.Second, Keep: 2})
	if err != nil {
		t.Fatal(err)
	}
	defer stop()
	app := http.NewServeMux()
	app.HandleFunc("GET /lent", func(w http.ResponseWriter, r *http.Request) { time.Sleep(30 * time.Millisecond) })
	b.Middleware(app).ServeHTTP(httptest.NewRecorder(), httptest.NewRequest("GET", "/lent", nil))

	admin := http.NewServeMux()
	b.Admin(admin)
	rec := httptest.NewRecorder()
	admin.ServeHTTP(rec, httptest.NewRequest("GET", "/debug/blackbox", nil))
	var list []Snapshot
	json.NewDecoder(rec.Body).Decode(&list)
	if len(list) != 1 || list[0].Size == 0 {
		t.Fatalf("%+v", list)
	}
	rec = httptest.NewRecorder()
	admin.ServeHTTP(rec, httptest.NewRequest("GET", "/debug/blackbox/1", nil))
	if !bytes.HasPrefix(rec.Body.Bytes(), []byte("go 1.")) || !strings.Contains(rec.Body.String()[:16], " trace") {
		t.Fatalf("pas une trace : %q", rec.Body.Bytes()[:16])
	}
	t.Logf("instantané : %d octets, route %s, durée %v", list[0].Size, list[0].Route, list[0].Duration.Round(time.Millisecond))
}

var sink int

// Le surcoût par requête : sans, et avec la boîte noire (enregistreur actif, aucun déclenchement).
func BenchmarkOverhead(b *testing.B) {
	app := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) { sink++ })
	req := httptest.NewRequest("GET", "/", nil)
	w := httptest.NewRecorder()
	b.Run("sans", func(b *testing.B) {
		for b.Loop() {
			app.ServeHTTP(w, req)
		}
	})
	box, stop, _ := Start(Options{Threshold: time.Second, Every: time.Minute, Keep: 2})
	defer stop()
	h := box.Middleware(app)
	b.Run("avec", func(b *testing.B) {
		for b.Loop() {
			h.ServeHTTP(w, req)
		}
	})
}
```

```
$ go test -race -v ./blackbox/
--- PASS: TestThresholdAndRate (0.00s)
    blackbox_test.go:75: instantané : 4152 octets, route GET /lent, durée 30ms
--- PASS: TestRealSnapshot (0.04s)
$ go test -run XXX -bench Overhead -benchmem -count 5 ./blackbox/      # médianes
BenchmarkOverhead/sans-8      2.3 ns/op   0 B/op   0 allocs/op
BenchmarkOverhead/avec-8     84.9 ns/op   0 B/op   0 allocs/op
```

- **Le seuil et la limite de fréquence** sont testés en temps virtuel. Dans `TestThresholdAndRate`, les
  requêtes de 80 et 90 ms tombent dans la même minute, et seule la première déclenche ; la suivante, une minute
  plus tard, déclenche. Le `Recorder` est une **interface**, et le test utilise un faux enregistreur. Le vrai
  n'a rien à faire dans un test de logique.
- **Le vrai instantané** est vérifié par son en-tête (`go 1.26 trace`, le numéro du **format** de trace, et non
  la version de Go). Il ne fait que 4 Ko, car l'enregistreur venait de démarrer : sa fenêtre se remplit avec le
  temps.
- **Le surcoût** est de **82 ns par requête**, avec l'enregistreur actif et sans déclenchement. Ce sont les deux
  lectures d'horloge du middleware, plus le surcoût de l'enregistreur lui-même. C'est négligeable devant une
  requête réelle (1,8 % de débit dans le cours, pour des requêtes de 1 ms).
- **Le verrou n'est pas tenu pendant `WriteTo`.** Le drapeau `busy` empêche deux instantanés simultanés (un
  seul `WriteTo` à la fois est permis), sans bloquer les autres requêtes qui franchissent le seuil.

**6. Les données sensibles.** Une trace contient les **piles** de toutes les goroutines (donc la structure du
code), les noms de fonctions, les durées, et les étiquettes de profilage. Elle ne contient pas les valeurs des
variables. Le profil de contention contient des piles. Les métadonnées contiennent la route et l'identifiant de
trace.

C'est moins sensible qu'un vidage mémoire, mais cela renseigne un attaquant sur le code et son comportement. On
expose donc la boîte noire sur l'adresse d'**administration** seulement (N12 L8), et on réserve le
téléchargement à l'équipe d'exploitation, comme `/debug/pprof`.

---

## Quiz

1. Reproduire, observer, formuler une hypothèse qui explique tout, la vérifier par une expérience qui
   pourrait la réfuter, corriger, et écrire le test.
2. `delve` (programme vivant ou vidage) ; la trace de pile et le vidage mémoire ; la trace d'exécution ou la
   trace répartie ; les profils de blocage et de contention.
3. Sans optimisations ni *inlining*, les variables existent et les lignes se suivent : le débogueur montre ce
   que le code dit.
4. Le noyau peut interdire de tracer un processus qui n'est pas un enfant (`ptrace_scope`), et les conteneurs
   retirent la capacité `SYS_PTRACE`. En production, on capture un vidage ou une trace, et on analyse ailleurs.
5. `GOTRACEBACK=crash` (ou `debug.SetTraceback("crash")`), `ulimit -c unlimited`, et un `core_pattern` qui
   les accepte. Puis `dlv core binaire vidage`, avec le binaire non réduit de la même version.
6. L'ordre d'évaluation entre un appel et un accès à un champ n'est pas défini : `total(o)` a été évalué avant
   `o.id`. La pile désigne `total`, mais l'erreur est dans la donnée de l'appelant.
7. L'histoire **dans le temps** : qui s'exécutait, qui attendait quoi, et quand, pour une requête précise. Le
   profil ne donne que des totaux.
8. À garder les dernières secondes de trace en mémoire, pour les écrire au moment où l'on détecte un
   incident rare. Environ 1,8 % de débit dans l'exemple.
9. Les pics touchaient moins d'une requête sur mille : ils tombaient au-delà du p99,9. Seul le maximum les
   montrait.
10. Environ ×7 en temps et ×9 en mémoire ici, plus 1 s à la sortie par défaut. En CI, dans les tests de stress,
    et sur une instance de pré-production sous charge réaliste.
