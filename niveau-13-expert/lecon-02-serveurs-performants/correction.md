# Module 2 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Moins de mémoire par connexion

```go
// Fichier : cmd/conns2/main.go
// Commande conns : 10 000 connexions TCP ouvertes et inactives, une goroutine chacune côté serveur.
// Que coûte une connexion ? Combien de fils système le runtime utilise-t-il ?
package main

import (
	"fmt"
	"log"
	"net"
	"os"
	"runtime"
	"runtime/metrics"
	"strings"
	"sync"
	"time"
)

func threads() string {
	b, _ := os.ReadFile("/proc/self/status")
	for l := range strings.Lines(string(b)) {
		if strings.HasPrefix(l, "Threads:") {
			return strings.TrimSpace(strings.TrimPrefix(l, "Threads:"))
		}
	}
	return "?"
}

func mem() (total, stacks uint64) {
	s := []metrics.Sample{{Name: "/memory/classes/total:bytes"}, {Name: "/memory/classes/heap/stacks:bytes"}}
	metrics.Read(s)
	return s[0].Value.Uint64(), s[1].Value.Uint64()
}

var bufs = sync.Pool{New: func() any { b := make([]byte, 4096); return &b }}

// serve attend le premier octet dans un tableau d'un octet (sur la pile), puis emprunte un tampon le
// temps de lire et de répondre, et le rend aussitôt.
func serve(c net.Conn) {
	defer c.Close()
	var first [1]byte
	for {
		if _, err := c.Read(first[:]); err != nil { // la goroutine attend ici, sans tampon
			return
		}
		bp := bufs.Get().(*[]byte)
		buf := *bp
		buf[0] = first[0]
		n := 1
		for buf[n-1] != '\n' && n < len(buf) { // lire la suite de la ligne
			m, err := c.Read(buf[n:])
			if err != nil {
				bufs.Put(bp)
				return
			}
			n += m
		}
		c.Write(buf[:n])
		bufs.Put(bp)
	}
}

func main() {
	ln, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		log.Fatal(err)
	}
	var accepted sync.WaitGroup
	const n = 10_000
	accepted.Add(n)
	go func() {
		for {
			c, err := ln.Accept()
			if err != nil {
				return
			}
			accepted.Done()
			go serve(c) // E1 : aucun tampon tant que la connexion est inactive
		}
	}()
	runtime.GC()
	t0, s0 := mem()
	fmt.Printf("avant : goroutines %d, fils %s, mémoire du runtime %d Mio\n", runtime.NumGoroutine(), threads(), t0>>20)
	conns := make([]net.Conn, 0, n)
	start := time.Now()
	for range n {
		c, err := net.Dial("tcp", ln.Addr().String())
		if err != nil {
			log.Fatal(err)
		}
		conns = append(conns, c)
	}
	accepted.Wait()
	time.Sleep(200 * time.Millisecond) // chaque goroutine serveur est entrée dans son Read
	runtime.GC()
	t1, s1 := mem()
	fmt.Printf("après %d connexions (%v) : goroutines %d, fils %s, mémoire %d Mio\n",
		n, time.Since(start).Round(time.Millisecond), runtime.NumGoroutine(), threads(), t1>>20)
	fmt.Printf("par connexion (client + serveur dans ce processus) : %.1f Kio, dont piles %.1f Kio\n",
		float64(t1-t0)/n/1024, float64(s1-s0)/n/1024)
	// Une requête sur chacune : elles sont servies par le netpoller, sans un fil par connexion.
	start = time.Now()
	buf := make([]byte, 64)
	for _, c := range conns {
		c.Write([]byte("ping\n"))
		c.Read(buf)
	}
	fmt.Printf("un aller-retour sur chacune : %v ; fils ensuite : %s\n", time.Since(start).Round(time.Millisecond), threads())
}
```

| Version du serveur | Par connexion (client + serveur) | Dont piles | Mémoire pour 10 000 |
|---|---|---|---|
| `bufio.Reader` de 4 Kio permanent (cours) | 10,8 Kio | 4,0 Kio | 113 Mio |
| tampon emprunté après le premier octet | **4,5 Kio** | 2,0 Kio | 56 Mio |

La goroutine attend dans `c.Read(first[:])`, un tableau d'**un** octet sur la pile. Tant que la connexion est
inactive, elle ne tient aucun tampon. Dès qu'un octet arrive, elle emprunte un tampon de 4 Kio, lit la fin de
la ligne, répond, et le rend.

Deux effets s'additionnent :
- les 4 Kio du `bufio` disparaissent ;
- la pile de la goroutine est plus petite (2 Kio contre 4), car la fonction n'a plus de `bufio.Reader` dans
  son cadre.

Il reste la pile minimale de chaque goroutine, les structures de connexion (`netFD`, côté client **et**
serveur, puisque le test les réunit) et l'enregistrement auprès du netpoller. Descendre plus bas
demanderait de ne plus avoir **une goroutine par connexion**, et donc de quitter le modèle idiomatique.
C'est rarement justifié : 4,5 Kio × 100 000 connexions font 450 Mio (voir le défi).

Le prix : un appel système de plus pour le premier octet de chaque ligne. On ne le paie que si les
connexions sont nombreuses et inactives, typiquement des connexions persistantes de clients mobiles ou de
WebSocket (N7 L11).

## E2 — Échapper sans allouer

```go
// Fichier : netx/jsonstr.go
package netx

import "unicode/utf8"

const hex = "0123456789abcdef"

// AppendJSONString écrit s comme le ferait json.Marshal(s), sans allocation (E2) : guillemets,
// \ et ", caractères de contrôle, <, > et & (sûrs pour HTML), U+2028 et U+2029, UTF-8 invalide.
func AppendJSONString(dst []byte, s string) []byte {
	dst = append(dst, '"')
	start := 0
	for i := 0; i < len(s); {
		if b := s[i]; b < utf8.RuneSelf {
			if b >= 0x20 && b != '"' && b != '\\' && b != '<' && b != '>' && b != '&' {
				i++
				continue
			}
			dst = append(dst, s[start:i]...)
			switch b {
			case '\\', '"':
				dst = append(dst, '\\', b)
			case '\b':
				dst = append(dst, '\\', 'b')
			case '\f':
				dst = append(dst, '\\', 'f')
			case '\n':
				dst = append(dst, '\\', 'n')
			case '\r':
				dst = append(dst, '\\', 'r')
			case '\t':
				dst = append(dst, '\\', 't')
			default: // autres contrôles, et < > &
				dst = append(dst, '\\', 'u', '0', '0', hex[b>>4], hex[b&0xF])
			}
			i++
			start = i
			continue
		}
		r, size := utf8.DecodeRuneInString(s[i:])
		if r == utf8.RuneError && size == 1 { // UTF-8 invalide
			dst = append(dst, s[start:i]...)
			dst = append(dst, `�`...)
			i += size
			start = i
			continue
		}
		if r == ' ' || r == ' ' {
			dst = append(dst, s[start:i]...)
			dst = append(dst, '\\', 'u', '2', '0', '2', hex[r&0xF])
			i += size
			start = i
			continue
		}
		i += size
	}
	dst = append(dst, s[start:]...)
	return append(dst, '"')
}
```

La fonction `appendJSONString` de `books.go` délègue désormais à cette version, et le budget de `Pooled`
passe à 0 dans `books_test.go` :

```go
// Fichier : netx/books.go
// Package netx : un handler au chemin chaud, et son budget d'allocation.
package netx

import (
	"encoding/json"
	"net/http"
	"strconv"
	"sync"
)

type Book struct {
	ID    int64  `json:"id"`
	Title string `json:"title"`
	Price int64  `json:"price"` // centimes
}

// Store garde les livres, et, pour le chemin chaud, leur représentation JSON déjà encodée.
type Store struct {
	books   map[int64]Book
	encoded map[int64][]byte
}

func NewStore(books []Book) *Store {
	s := &Store{books: map[int64]Book{}, encoded: map[int64][]byte{}}
	for _, b := range books {
		s.books[b.ID] = b
		j, _ := json.Marshal(b) // encodé une fois, à l'écriture, avec le vrai encodeur JSON
		s.encoded[b.ID] = append(j, '\n')
	}
	return s
}

// Naive : le handler qu'on écrit d'abord.
func (s *Store) Naive(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.ParseInt(r.PathValue("id"), 10, 64)
	if err != nil {
		http.Error(w, "identifiant invalide", http.StatusBadRequest)
		return
	}
	b, ok := s.books[id]
	if !ok {
		http.NotFound(w, r)
		return
	}
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(b)
}

var contentType = []string{"application/json"} // une slice partagée, jamais modifiée

// Fast : même réponse, octet pour octet, sans encodage à chaque requête.
func (s *Store) Fast(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.ParseInt(r.PathValue("id"), 10, 64)
	if err != nil {
		http.Error(w, "identifiant invalide", http.StatusBadRequest)
		return
	}
	j, ok := s.encoded[id]
	if !ok {
		http.NotFound(w, r)
		return
	}
	w.Header()["Content-Type"] = contentType // évite la normalisation et l'allocation de Set
	w.Write(j)
}

// Pooled : encoder à chaque requête (si la réponse change), mais dans un tampon réutilisé.
var bufPool = sync.Pool{New: func() any { b := make([]byte, 0, 256); return &b }}

func (s *Store) Pooled(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.ParseInt(r.PathValue("id"), 10, 64)
	if err != nil {
		http.Error(w, "identifiant invalide", http.StatusBadRequest)
		return
	}
	b, ok := s.books[id]
	if !ok {
		http.NotFound(w, r)
		return
	}
	bp := bufPool.Get().(*[]byte)
	out := append((*bp)[:0], `{"id":`...)
	out = strconv.AppendInt(out, b.ID, 10)
	out = append(out, `,"title":`...)
	out = appendJSONString(out, b.Title)
	out = append(out, `,"price":`...)
	out = strconv.AppendInt(out, b.Price, 10)
	out = append(out, "}\n"...)
	w.Header()["Content-Type"] = contentType
	w.Write(out)
	*bp = out
	bufPool.Put(bp)
}

// appendJSONString : la version sans allocation de E2 (jsonstr.go).
func appendJSONString(dst []byte, s string) []byte { return AppendJSONString(dst, s) }
```

```go
// Fichier : netx/books_test.go
package netx

import (
	"net/http"
	"net/http/httptest"
	"testing"
)

// discard : un ResponseWriter minimal, pour mesurer le HANDLER seul (httptest.NewRecorder alloue lui-même).
type discard struct{ h http.Header }

func (d *discard) Header() http.Header         { return d.h }
func (d *discard) Write(p []byte) (int, error) { return len(p), nil }
func (d *discard) WriteHeader(int)             {}

func store() *Store {
	return NewStore([]Book{{1, "Le Petit Prince", 899}, {2, "Ça & <là>", 1250}})
}

func request() *http.Request {
	r := httptest.NewRequest("GET", "/books/2", nil)
	r.SetPathValue("id", "2")
	return r
}

func TestSameBytes(t *testing.T) {
	s := store()
	var outs []string
	for _, h := range []http.HandlerFunc{s.Naive, s.Fast, s.Pooled} {
		rec := httptest.NewRecorder()
		h(rec, request())
		outs = append(outs, rec.Body.String()+rec.Header().Get("Content-Type"))
	}
	if outs[0] != outs[1] || outs[0] != outs[2] {
		t.Fatalf("réponses différentes :\n%q", outs)
	}
	t.Logf("%q", outs[0])
}

// Le budget : un test qui échoue si le chemin chaud se met à allouer.
func TestAllocationBudget(t *testing.T) {
	s := store()
	r := request()
	w := &discard{h: http.Header{}}
	for _, tc := range []struct {
		name   string
		h      http.HandlerFunc
		budget float64
	}{{"Naive", s.Naive, 3}, {"Fast", s.Fast, 0}, {"Pooled", s.Pooled, 0}} {
		got := testing.AllocsPerRun(1000, func() { tc.h(w, r) })
		t.Logf("%-7s %v allocation(s) par requête (budget %v)", tc.name, got, tc.budget)
		if got > tc.budget {
			t.Errorf("%s : %v allocations, budget %v", tc.name, got, tc.budget)
		}
	}
}

func BenchmarkHandlers(b *testing.B) {
	s := store()
	r := request()
	for _, tc := range []struct {
		name string
		h    http.HandlerFunc
	}{{"Naive", s.Naive}, {"Fast", s.Fast}, {"Pooled", s.Pooled}} {
		b.Run(tc.name, func(b *testing.B) {
			w := &discard{h: http.Header{}}
			for b.Loop() {
				tc.h(w, r)
			}
		})
	}
}
```

```go
// Fichier : netx/jsonstr_test.go
package netx

import (
	"encoding/json"
	"testing"
)

func FuzzAppendJSONString(f *testing.F) {
	for _, s := range []string{"", "abc", `"\`, "a\nb\tc\r\b\f\x00\x1f", "<script>&", "  ", "\xff\xfe", "Ça é 日本"} {
		f.Add(s)
	}
	f.Fuzz(func(t *testing.T, s string) {
		want, _ := json.Marshal(s)
		if got := AppendJSONString(nil, s); string(got) != string(want) {
			t.Fatalf("%q : obtenu %s, attendu %s", s, got, want)
		}
	})
}

func TestAppendJSONStringAllocs(t *testing.T) {
	buf := make([]byte, 0, 256)
	if n := testing.AllocsPerRun(1000, func() { buf = AppendJSONString(buf[:0], "Ça & <là> ") }); n != 0 {
		t.Fatalf("%v allocations", n)
	}
}
```

```
$ go test -run XXX -fuzz FuzzAppendJSONString -fuzztime 30s ./netx/
fuzz: elapsed: 31s, execs: 662760 (0/sec), new interesting: 55 (total: 63)
PASS
$ go test -v -run Budget ./netx/
    books_test.go:51: Naive   3 allocation(s) par requête (budget 3)
    books_test.go:51: Fast    0 allocation(s) par requête (budget 0)
    books_test.go:51: Pooled  0 allocation(s) par requête (budget 0)
```

- **Le fuzzing** a exécuté 662 760 entrées sans trouver d'écart avec `json.Marshal`. Il est parti des cas
  écrits à la main : guillemets, contrôles, `<>&`, U+2028, UTF-8 invalide, accents. La référence est
  l'encodeur de la bibliothèque standard, pas une spécification relue de mémoire.
- **`\b` et `\f`** s'écrivent `\b` et `\f`, et non `\u0008` : c'est ce que fait `encoding/json` dans cette
  version de Go. Le test de fuzzing l'aurait signalé si l'on s'était trompé.
- **`Pooled` passe à 0 allocation et 134 ns**, contre 400 ns et 3 allocations. Le budget du test est abaissé
  à 0 : il ne pourra plus remonter sans que le test échoue.
- **`Fast` reste plus rapide (49 ns)** : il n'encode rien du tout. Encoder à l'écriture reste la meilleure
  option quand c'est possible.

## E3 — Un middleware qui ne casse pas `sendfile`

```go
// Fichier : cmd/servefile/main.go
// Commande servefile : un fichier de 512 Mio servi par http.ServeContent, à travers un middleware
// qui compte les octets : opaque, transparent, ou absent (E3).
package main

import (
	"fmt"
	"io"
	"log"
	"net"
	"net/http"
	"os"
	"path/filepath"
	"time"
)

// counting compte les octets écrits. Seul, il cache le ResponseWriter d'origine.
type counting struct {
	http.ResponseWriter
	n int64
}

func (c *counting) Write(p []byte) (int, error) {
	n, err := c.ResponseWriter.Write(p)
	c.n += int64(n)
	return n, err
}

// transparent ajoute ReadFrom, qui délègue au ResponseWriter d'origine (lequel sait faire sendfile).
type transparent struct{ *counting }

func (t transparent) ReadFrom(r io.Reader) (int64, error) {
	n, err := io.Copy(t.counting.ResponseWriter, r) // io.Copy trouve le ReadFrom de net/http
	t.n += n
	return n, err
}

func main() {
	mode := os.Args[1]
	dir, _ := os.MkdirTemp("", "servefile")
	defer os.RemoveAll(dir)
	path := filepath.Join(dir, "gros.bin")
	f, _ := os.Create(path)
	f.Truncate(512 << 20)
	f.Close()
	var counted int64
	h := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		src, _ := os.Open(path)
		defer src.Close()
		st, _ := src.Stat()
		var out http.ResponseWriter = w
		cw := &counting{ResponseWriter: w}
		switch mode {
		case "opaque":
			out = cw
		case "transparent":
			out = transparent{cw}
		}
		http.ServeContent(out, r, "gros.bin", st.ModTime(), src)
		counted = cw.n
	})
	ln, _ := net.Listen("tcp", "127.0.0.1:0")
	go http.Serve(ln, h)
	start := time.Now()
	resp, err := http.Get("http://" + ln.Addr().String())
	if err != nil {
		log.Fatal(err)
	}
	n, _ := io.Copy(io.Discard, resp.Body)
	resp.Body.Close()
	el := time.Since(start)
	fmt.Printf("%-11s %d Mio en %v (%.0f Mio/s), compté par le middleware : %d Mio\n", mode, n>>20,
		el.Round(time.Millisecond), float64(n>>20)/el.Seconds(), counted>>20)
}
```

```
$ for m in aucun opaque transparent; do ./servefile $m; done
aucun       512 Mio en 234ms (2186 Mio/s), compté par le middleware : 0 Mio
opaque      512 Mio en 387ms (1323 Mio/s), compté par le middleware : 512 Mio
transparent 512 Mio en 243ms (2104 Mio/s), compté par le middleware : 512 Mio
$ strace -f -c -e trace=sendfile,write ./servefile …        # appels comptés
aucun        1028 sendfile   4 write
opaque                       16630 write
transparent   780 sendfile   4 write
```

Le middleware **opaque** n'a que `Write`. `io.Copy`, appelé par `http.ServeContent`, ne trouve pas de
`ReadFrom` et copie par tampons de 32 Kio : 16 630 `write`, et 40 % de débit en moins.

La version **transparente** ajoute `ReadFrom`. Il délègue à `io.Copy` vers le `ResponseWriter` d'origine, que
`net/http` sait transformer en `sendfile`, et compte les octets au passage : 780 `sendfile`, le débit d'origine,
et le bon total.

Le même problème touche tout ce qui enveloppe un `ResponseWriter` : métriques, journaux, compression. Une
enveloppe doit exposer ce qu'elle cache. Les méthodes à considérer : `ReadFrom`, `Flush`, `Hijack`, et
`Unwrap` pour `http.ResponseController`.

## E4 — Quand le *runtime* crée des fils

```go
// Fichier : cmd/threads/main.go
// Commande threads : 200 goroutines qui dorment, garées ou bloquées dans un appel système (E4).
package main

import (
	"fmt"
	"os"
	"strings"
	"sync"
	"syscall"
	"time"
)

func threads() string {
	b, _ := os.ReadFile("/proc/self/status")
	for l := range strings.Lines(string(b)) {
		if v, ok := strings.CutPrefix(l, "Threads:"); ok {
			return strings.TrimSpace(v)
		}
	}
	return "?"
}

func run(name string, sleep func()) {
	var wg sync.WaitGroup
	for range 200 {
		wg.Go(sleep)
	}
	time.Sleep(500 * time.Millisecond) // au milieu du sommeil
	fmt.Printf("%-28s fils pendant : %s\n", name, threads())
	wg.Wait()
}

func main() {
	fmt.Println("au départ : fils", threads())
	run("time.Sleep (garée)", func() { time.Sleep(time.Second) })
	run("syscall.Nanosleep (bloquée)", func() {
		ts := syscall.NsecToTimespec(int64(time.Second))
		syscall.Nanosleep(&ts, nil)
	})
	time.Sleep(100 * time.Millisecond)
	fmt.Println("après : fils", threads())
}
```

```
$ go run ./cmd/threads
au départ : fils 5
time.Sleep (garée)           fils pendant : 7
syscall.Nanosleep (bloquée)  fils pendant : 203
après : fils 204
```

- **`time.Sleep`** gare la goroutine dans l'ordonnanceur : aucun fil n'est occupé. Deux fils de plus
  apparaissent pour l'activité générale.
- **`syscall.Nanosleep`** est un appel système **bloquant** que le *runtime* ne sait pas transformer en
  attente garée. Le fil reste bloqué dans le noyau. Pour que les autres goroutines continuent, le *runtime*
  détache le processeur logique (P) de ce fil et en démarre un autre : **un fil par appel bloqué**, 203 au
  total.
- **Les fils ne sont pas rendus** : il y en a encore 204 après. Le *runtime* les garde pour plus tard.

Avec 10 000 lectures de fichiers sur un disque réseau lent, chacune bloquée dans un `read`, on obtiendrait de
l'ordre de 10 000 fils. Ils coûtent de la mémoire (une pile système chacun), et on atteint la limite par
défaut de `runtime/debug.SetMaxThreads`, 10 000, au-delà de laquelle le programme **s'arrête**. D'où la règle :
borner la concurrence des E/S qui ne passent pas par le netpoller, par un sémaphore (N6 L9).

## E5 — Le prix d'une connexion neuve

```go
// Fichier : cmd/client/main.go
// Commande client : 21 000 requêtes, en connexion persistante ou non (-new), puis lit le rapport
// du serveur.
package main

import (
	"flag"
	"fmt"
	"io"
	"log"
	"net/http"
	"time"
)

func main() {
	fresh := flag.Bool("new", false, "une connexion TCP neuve par requête")
	flag.Parse()
	c := &http.Client{Transport: &http.Transport{DisableKeepAlives: *fresh}}
	start := time.Now()
	for range 21_000 {
		resp, err := c.Get("http://127.0.0.1:18321/books/2")
		if err != nil {
			log.Fatal(err)
		}
		io.Copy(io.Discard, resp.Body)
		resp.Body.Close()
	}
	el := time.Since(start)
	resp, err := http.Get("http://127.0.0.1:18321/report")
	if err != nil {
		log.Fatal(err)
	}
	b, _ := io.ReadAll(resp.Body)
	fmt.Printf("%5.0f requêtes/s, %5.1f µs par requête ; allocations du serveur par requête : %s\n",
		21_000/el.Seconds(), float64(el.Microseconds())/21_000, b)
}
```

```
$ ./server fast & ./client         # deux exécutions de chaque
persist  9263 requêtes/s, 108.0 µs par requête ; allocations du serveur par requête : 19.3
persist 10674 requêtes/s,  93.7 µs par requête ; allocations du serveur par requête : 19.3
-new     3346 requêtes/s, 298.8 µs par requête ; allocations du serveur par requête : 38.0
-new     2932 requêtes/s, 341.1 µs par requête ; allocations du serveur par requête : 38.0
```

Une connexion TCP neuve par requête, **en local et sans TLS**, triple la durée d'une requête (environ 100 µs →
320 µs) et **double** les allocations du serveur (19 → 38). Il faut accepter la connexion, créer sa goroutine
et ses tampons, et la fermer.

Sur un réseau réel, il faut ajouter un aller-retour réseau pour la poignée de main TCP. TLS en ajoute un ou
deux de plus, et des calculs cryptographiques des deux côtés. La différence se compte alors en
**millisecondes**. Les connexions persistantes sont la première optimisation d'un client HTTP : un seul
`http.Client` (et son `Transport`) partagé par tout le programme, jamais un client par requête.

---

## Exercice intermédiaire — Un serveur clé-valeur sur TCP

```go
// Fichier : kv/kv.go
// Package kv : un serveur clé-valeur sur TCP, à protocole texte, avec pipelining.
//
//	SET clé valeur\n  →  OK\n
//	GET clé\n         →  VALUE valeur\n | NOTFOUND\n
package kv

import (
	"bufio"
	"bytes"
	"io"
	"net"
	"sync"
)

type Store struct {
	mu sync.RWMutex
	m  map[string][]byte
}

func NewStore() *Store { return &Store{m: map[string][]byte{}} }

var (
	okResp       = []byte("OK\n")
	notFoundResp = []byte("NOTFOUND\n")
	valuePrefix  = []byte("VALUE ")
	errResp      = []byte("ERR commande inconnue\n")
)

// Handle exécute une commande et écrit la réponse dans w. C'est le chemin chaud : 0 allocation pour GET.
func (s *Store) Handle(line []byte, w *bufio.Writer) {
	cmd, rest, _ := bytes.Cut(line, []byte{' '})
	switch string(cmd) { // switch sur string(b) : pas d'allocation
	case "GET":
		s.mu.RLock()
		v, ok := s.m[string(rest)] // recherche avec string(b) : pas d'allocation
		s.mu.RUnlock()
		if !ok {
			w.Write(notFoundResp)
			return
		}
		w.Write(valuePrefix)
		w.Write(v)
		w.WriteByte('\n')
	case "SET":
		k, v, ok := bytes.Cut(rest, []byte{' '})
		if !ok {
			w.Write(errResp)
			return
		}
		val := bytes.Clone(v) // la ligne appartient au bufio.Reader : il faut copier ce qu'on garde
		s.mu.Lock()
		s.m[string(k)] = val // ici l'allocation de la clé est voulue : elle est stockée
		s.mu.Unlock()
		w.Write(okResp)
	default:
		w.Write(errResp)
	}
}

// Serve sert une connexion : lit les commandes, répond dans un tampon, le vide quand il n'y a plus
// rien à lire (pipelining : 100 commandes envoyées d'un coup donnent un seul write de réponses).
func (s *Store) Serve(c net.Conn) {
	defer c.Close()
	r := bufio.NewReaderSize(c, 16<<10)
	w := bufio.NewWriterSize(c, 16<<10)
	for {
		line, err := r.ReadSlice('\n') // une vue dans le tampon du Reader, sans copie
		if err != nil {
			if err != io.EOF {
				w.Flush()
			}
			return
		}
		s.Handle(line[:len(line)-1], w)
		if r.Buffered() == 0 { // plus rien en attente : on envoie les réponses accumulées
			if err := w.Flush(); err != nil {
				return
			}
		}
	}
}

func (s *Store) ListenAndServe(ln net.Listener) error {
	for {
		c, err := ln.Accept()
		if err != nil {
			return err
		}
		go s.Serve(c)
	}
}
```

```go
// Fichier : kv/kv_test.go
package kv

import (
	"bufio"
	"fmt"
	"io"
	"net"
	"net/http"
	"net/http/httptest"
	"strconv"
	"sync"
	"testing"
	"time"
)

func TestBudget(t *testing.T) {
	s := NewStore()
	w := bufio.NewWriter(io.Discard)
	s.Handle([]byte("SET k12 une-valeur"), w)
	line := []byte("GET k12")
	if n := testing.AllocsPerRun(1000, func() { s.Handle(line, w) }); n != 0 {
		t.Fatalf("GET : %v allocations, budget 0", n)
	}
}

func TestProtocol(t *testing.T) {
	s := NewStore()
	ln, _ := net.Listen("tcp", "127.0.0.1:0")
	go s.ListenAndServe(ln)
	defer ln.Close()
	c, _ := net.Dial("tcp", ln.Addr().String())
	defer c.Close()
	io.WriteString(c, "SET a 1\nGET a\nGET b\nNOP\n") // pipelining : tout d'un coup
	r := bufio.NewReader(c)
	for _, want := range []string{"OK\n", "VALUE 1\n", "NOTFOUND\n", "ERR commande inconnue\n"} {
		if got, _ := r.ReadString('\n'); got != want {
			t.Fatalf("%q, attendu %q", got, want)
		}
	}
}

// throughput : clients connexions, chacune envoie des lots de depth GET puis lit les depth réponses.
func throughput(addr string, clients, depth int, d time.Duration) float64 {
	var total int64
	var mu sync.Mutex
	var wg sync.WaitGroup
	deadline := time.Now().Add(d)
	batch := []byte{}
	for i := range depth {
		batch = append(batch, "GET k"+strconv.Itoa(i%100)+"\n"...)
	}
	for range clients {
		wg.Go(func() {
			c, err := net.Dial("tcp", addr)
			if err != nil {
				return
			}
			defer c.Close()
			r := bufio.NewReaderSize(c, 64<<10)
			n := int64(0)
			for time.Now().Before(deadline) {
				c.Write(batch)
				for range depth {
					if _, err := r.ReadSlice('\n'); err != nil {
						return
					}
				}
				n += int64(depth)
			}
			mu.Lock()
			total += n
			mu.Unlock()
		})
	}
	wg.Wait()
	return float64(total) / d.Seconds()
}

func TestThroughput(t *testing.T) {
	if testing.Short() {
		t.Skip()
	}
	s := NewStore()
	w := bufio.NewWriter(io.Discard)
	for i := range 100 {
		s.Handle([]byte("SET k"+strconv.Itoa(i)+" valeur-"+strconv.Itoa(i)), w)
	}
	ln, _ := net.Listen("tcp", "127.0.0.1:0")
	go s.ListenAndServe(ln)
	defer ln.Close()
	for _, clients := range []int{1, 8, 64} {
		for _, depth := range []int{1, 100} {
			fmt.Printf("TCP   clients %2d  profondeur %3d : %9.0f GET/s\n", clients, depth,
				throughput(ln.Addr().String(), clients, depth, time.Second))
		}
	}
	// La même chose en HTTP, sans pipelining (un GET par requête).
	mux := http.NewServeMux()
	mux.HandleFunc("GET /kv/{k}", func(w http.ResponseWriter, r *http.Request) {
		s.mu.RLock()
		v, ok := s.m[r.PathValue("k")]
		s.mu.RUnlock()
		if !ok {
			http.NotFound(w, r)
			return
		}
		w.Write(v)
	})
	srv := httptest.NewServer(mux)
	defer srv.Close()
	for _, clients := range []int{1, 8, 64} {
		var n int64
		var mu sync.Mutex
		var wg sync.WaitGroup
		tr := &http.Transport{MaxIdleConnsPerHost: 64}
		deadline := time.Now().Add(time.Second)
		for range clients {
			wg.Go(func() {
				c := &http.Client{Transport: tr}
				k := int64(0)
				for time.Now().Before(deadline) {
					resp, err := c.Get(srv.URL + "/kv/k7")
					if err != nil {
						return
					}
					io.Copy(io.Discard, resp.Body)
					resp.Body.Close()
					k++
				}
				mu.Lock()
				n += k
				mu.Unlock()
			})
		}
		wg.Wait()
		fmt.Printf("HTTP  clients %2d                 : %9.0f GET/s\n", clients, float64(n))
	}
}
```

```
$ go test -v ./kv/
--- PASS: TestBudget (0.00s)
--- PASS: TestProtocol (0.00s)
TCP   clients  1  profondeur   1 :     76084 GET/s
TCP   clients  1  profondeur 100 :   4433100 GET/s
TCP   clients  8  profondeur   1 :    217405 GET/s
TCP   clients  8  profondeur 100 :  12805700 GET/s
TCP   clients 64  profondeur   1 :    299555 GET/s
TCP   clients 64  profondeur 100 :  13856800 GET/s
HTTP  clients  1                 :     14190 GET/s
HTTP  clients  8                 :     72250 GET/s
HTTP  clients 64                 :     49164 GET/s
```

- **Le budget de 0 allocation par `GET`** est vérifié par `TestBudget`. `ReadSlice` rend une **vue** dans le
  tampon du `bufio.Reader`, sans copie. Les `switch string(cmd)` et `m[string(rest)]` n'allouent pas : le
  compilateur reconnaît ces conversions. Seul `SET` alloue, et volontairement : la clé et la valeur sont
  **gardées**, il faut donc les copier hors du tampon, qui sera réutilisé (`bytes.Clone`).
- **Le pipelining multiplie le débit par 46 à 58** à nombre de clients égal. Sans lui, chaque `GET` coûte un
  aller-retour complet : un `write` et un `read` de chaque côté, et deux passages par le netpoller. Avec
  100 commandes par lot, ces coûts sont partagés. Le serveur ne vide son tampon de réponses que quand il n'a
  plus rien à lire (`r.Buffered() == 0`) : 100 réponses partent en un seul `write`.
- **HTTP, sans pipelining**, plafonne à 72 000 requêtes/s, et **baisse** à 64 clients (49 000). Le client et le
  serveur partagent les 8 fils matériels, et l'analyse des en-têtes et l'allocation par requête dominent. Le
  protocole texte de 20 octets n'a aucun de ces coûts.

**4. Le profil à 64 clients :**

```
profondeur 1
    16.41s 82.50%  internal/runtime/syscall/linux.Syscall6
     0.17s  0.85%  bufio.(*Reader).fill          (cum 23.63 %)
profondeur 100
     8.90s 46.87%  internal/runtime/syscall/linux.Syscall6
     3.54s 18.64%  sync/atomic.(*Int32).Add (inline)
     1.04s  5.48%  example.com/hp/kv.(*Store).Handle   (cum 36.97 %)
     0.66s  3.48%  bufio.(*Reader).ReadSlice
     0.51s  2.69%  runtime.mapaccess2_faststr
```

- **À profondeur 1**, 82 % du temps est **dans le noyau** (appels système). Le code Go est négligeable : il
  n'y a rien à optimiser dans le serveur, seulement le nombre d'allers-retours (pipelining, lots).
- **À profondeur 100**, les appels système tombent à 47 %, et un nouveau poste apparaît :
  `sync/atomic.(*Int32).Add`, 18,6 %. C'est le compteur de lecteurs du **`RWMutex`** partagé par les 64
  connexions. À 14 millions de `RLock` par seconde, sa ligne de cache fait la navette entre les cœurs, comme
  dans le faux partage du N11 L6. Pour aller plus loin, on découperait la map en segments, ou, pour des
  données presque jamais écrites, on publierait une map **immuable** par un `atomic.Pointer` et on la
  remplacerait à chaque écriture. On n'a pas fait ce changement : on ne le ferait que si le profil
  l'imposait.

---

## Défi — 100 000 connexions

```go
// Fichier : cmd/conns100k/main.go
// Commande conns : 10 000 connexions TCP ouvertes et inactives, une goroutine chacune côté serveur.
// Que coûte une connexion ? Combien de fils système le runtime utilise-t-il ?
package main

import (
	"fmt"
	"log"
	"net"
	"os"
	"runtime"
	"runtime/metrics"
	"strconv"
	"strings"
	"sync"
	"syscall"
	"time"
)

func threads() string {
	b, _ := os.ReadFile("/proc/self/status")
	for l := range strings.Lines(string(b)) {
		if strings.HasPrefix(l, "Threads:") {
			return strings.TrimSpace(strings.TrimPrefix(l, "Threads:"))
		}
	}
	return "?"
}

func mem() (total, stacks uint64) {
	s := []metrics.Sample{{Name: "/memory/classes/total:bytes"}, {Name: "/memory/classes/heap/stacks:bytes"}}
	metrics.Read(s)
	return s[0].Value.Uint64(), s[1].Value.Uint64()
}

var bufs = sync.Pool{New: func() any { b := make([]byte, 4096); return &b }}

// serve attend le premier octet dans un tableau d'un octet (sur la pile), puis emprunte un tampon le
// temps de lire et de répondre, et le rend aussitôt.
func serve(c net.Conn) {
	defer c.Close()
	var first [1]byte
	for {
		if _, err := c.Read(first[:]); err != nil { // la goroutine attend ici, sans tampon
			return
		}
		bp := bufs.Get().(*[]byte)
		buf := *bp
		buf[0] = first[0]
		n := 1
		for buf[n-1] != '\n' && n < len(buf) { // lire la suite de la ligne
			m, err := c.Read(buf[n:])
			if err != nil {
				bufs.Put(bp)
				return
			}
			n += m
		}
		c.Write(buf[:n])
		bufs.Put(bp)
	}
}

func nFromArgs() int {
	n, _ := strconv.Atoi(os.Args[1])
	return n
}

func sourcesFromArgs() []net.IP {
	k, _ := strconv.Atoi(os.Args[2])
	var ips []net.IP
	for i := range k {
		ips = append(ips, net.IPv4(127, 0, 0, byte(1+i)))
	}
	return ips
}

func main() {
	ln, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		log.Fatal(err)
	}
	var accepted sync.WaitGroup
	n := nFromArgs()
	accepted.Add(n)
	go func() {
		for {
			c, err := ln.Accept()
			if err != nil {
				return
			}
			accepted.Done()
			go serve(c) // E1 : aucun tampon tant que la connexion est inactive
		}
	}()
	runtime.GC()
	t0, s0 := mem()
	fmt.Printf("avant : goroutines %d, fils %s, mémoire du runtime %d Mio\n", runtime.NumGoroutine(), threads(), t0>>20)
	conns := make([]net.Conn, 0, n)
	start := time.Now()
	sources := sourcesFromArgs()
	noPort := len(os.Args) > 3 && os.Args[3] == "noport"
	for i := range n {
		// Chaque adresse source locale (127.0.0.1, .2, …) a sa propre plage de ports éphémères.
		d := net.Dialer{LocalAddr: &net.TCPAddr{IP: sources[i%len(sources)]}}
		if noPort {
			// IP_BIND_ADDRESS_NO_PORT (Linux) : le port est choisi au connect, pas au bind. Le noyau
			// peut alors réutiliser un même port source vers des destinations différentes, et ne
			// cherche plus un port libre à chaque bind.
			d.Control = func(_, _ string, c syscall.RawConn) error {
				return c.Control(func(fd uintptr) {
					syscall.SetsockoptInt(int(fd), syscall.IPPROTO_IP, 24, 1) // 24 = IP_BIND_ADDRESS_NO_PORT
				})
			}
		}
		if i%20000 == 0 && i > 0 {
			fmt.Printf("  … %d connexions en %v\n", i, time.Since(start).Round(time.Millisecond))
		}
		c, err := d.Dial("tcp", ln.Addr().String())
		if err != nil {
			log.Fatalf("connexion %d : %v", i, err)
		}
		conns = append(conns, c)
	}
	accepted.Wait()
	time.Sleep(200 * time.Millisecond) // chaque goroutine serveur est entrée dans son Read
	runtime.GC()
	t1, s1 := mem()
	fmt.Printf("après %d connexions (%v) : goroutines %d, fils %s, mémoire %d Mio\n",
		n, time.Since(start).Round(time.Millisecond), runtime.NumGoroutine(), threads(), t1>>20)
	fmt.Printf("par connexion (client + serveur dans ce processus) : %.1f Kio, dont piles %.1f Kio\n",
		float64(t1-t0)/float64(n)/1024, float64(s1-s0)/float64(n)/1024)
	// Une requête sur chacune : elles sont servies par le netpoller, sans un fil par connexion.
	start = time.Now()
	buf := make([]byte, 64)
	for _, c := range conns {
		c.Write([]byte("ping\n"))
		c.Read(buf)
	}
	fmt.Printf("un aller-retour sur chacune : %v ; fils ensuite : %s\n", time.Since(start).Round(time.Millisecond), threads())
}
```

**1. La première limite** n'est ni la mémoire ni les descripteurs (`ulimit -n` vaut 1 048 576 ici). C'est la
plage de **ports éphémères** :

```
$ cat /proc/sys/net/ipv4/ip_local_port_range
32768	60999
$ ./c100k 100000 1
connexion 28227 : dial tcp 127.0.0.1:0->127.0.0.1:34567: bind: address already in use
```

Une connexion TCP est identifiée par son quadruplet (adresse et port source, adresse et port destination).
Avec une seule adresse source vers un seul serveur, il n'y a que 28 232 ports sources possibles.

**2. Les adresses sources.** Plusieurs adresses `127.0.0.x`, et une source choisie par connexion
(`net.Dialer{LocalAddr: …}`) :

| Essai | Progression |
|---|---|
| 4 adresses, `bind` classique | 20 000 en 1,9 s, 40 000 en 3,7 s, 60 000 en **30,5 s**, puis arrêté au bout de 120 s |
| 4 adresses, `IP_BIND_ADDRESS_NO_PORT` | 60 000 en 24,6 s, 80 000 en **2 min 6 s**, puis arrêté au bout de 200 s |
| **16 adresses**, `IP_BIND_ADDRESS_NO_PORT` | **100 000 en 5,7 s** |

Avec 4 adresses, il y a en principe 113 000 ports sources possibles, et pourtant tout ralentit au-delà de la
moitié. L'option `IP_BIND_ADDRESS_NO_PORT` (choix du port au `connect` plutôt qu'au `bind`) n'y change
presque rien. Avec 16 adresses, chacune n'utilise qu'environ 6 000 ports, et l'ouverture reste linéaire.

L'hypothèse la plus cohérente avec ces mesures : le noyau **cherche** un port libre pour chaque connexion
dans la plage d'une adresse, et cette recherche s'allonge à mesure que la plage se remplit. On ne l'a pas
vérifiée dans le code du noyau.

En pratique, on évite de remplir une plage. On ajoute des adresses sources, des ports de destination (le
serveur écoute sur plusieurs ports), ou on élargit la plage par `sysctl`.

Un piège de mesure, rencontré trois fois pendant ce défi : chaque essai interrompu laisse jusqu'à 65 536
sockets en `TIME_WAIT` (`ss -tan state time-wait`), qui occupent les ports pendant 60 s. Il faut attendre
qu'ils expirent avant de remesurer, sinon les chiffres mesurent l'essai précédent.

**3. À 100 000 connexions** (version de E1) :

```
après 100000 connexions (5.74s) : goroutines 100002, fils 12, mémoire 452 Mio
par connexion (client + serveur dans ce processus) : 4.5 Kio, dont piles 2.0 Kio
un aller-retour sur chacune : 1.989s ; fils ensuite : 12
```

100 002 goroutines, **12 fils**, 4,5 Kio par connexion : la même valeur qu'à 10 000. Le coût par connexion ne
dépend pas de leur nombre, et le netpoller ne crée pas de fils. Un aller-retour séquentiel sur chacune prend
2 s, soit 20 µs par connexion.

**4. Pour un million de connexions sur une machine**, par exemple :

> - Mémoire : environ 4,5 Gio en tout (2,3 Gio côté serveur seul) avec la version de E1, contre 11 Gio avec un
>   `bufio` permanent par connexion.
> - Noyau : `ulimit -n` au-dessus d'un million ; `net.core.somaxconn` et la file d'attente de `Listen` pour les
>   rafales de connexion ; des tampons de socket réduits (`net.ipv4.tcp_rmem`, `tcp_wmem`), chaque socket
>   en réservant par défaut.
> - Côté clients : des milliers de machines, ou de nombreuses adresses sources, pour les ports.
> - Code : aucun tampon par connexion inactive (E1), des délais d'inactivité (une connexion morte coûte comme
>   une vivante), des sondes TCP de maintien de connexion (*keepalive*).
> - Et d'abord une question : faut-il **une** machine ? Dix machines à 100 000 connexions sont plus simples à
>   exploiter, et en perdre une est moins grave.

---

## Quiz

1. L'appel système répond « rien pour l'instant ». Le *runtime* gare la goroutine et enregistre le
   descripteur auprès d'`epoll`, et le fil exécute d'autres goroutines. La goroutine reprend quand `epoll`
   signale des données.
2. Les goroutines bloquées sur le réseau sont garées, sans fil. Les fils correspondent aux processeurs
   logiques (`GOMAXPROCS`) et à quelques fils de service : 11 pour 10 000 connexions, 12 pour 100 000.
3. La pile de la goroutine (2 à 4 Kio mesurés), les tampons par connexion (`bufio` : 4 Kio), les structures
   de connexion, et les tampons du noyau pour la socket.
4. Quand une goroutine fait un appel système **bloquant** hors du netpoller (fichiers ordinaires, certains
   appels, cgo) : le *runtime* démarre un autre fil pour continuer à exécuter les autres goroutines.
5. `io.Copy` cherche `WriterTo` sur la source ou `ReaderFrom` sur la destination. `*net.TCPConn` implémente
   `ReadFrom`, qui utilise `sendfile` quand la source est un `*os.File`.
6. Une enveloppe qui cache le type concret (`struct{ io.Writer }`, un middleware sans `ReadFrom`), TLS, et la
   compression.
7. Un nombre maximal d'allocations par requête sur le chemin chaud, vérifié par un test `AllocsPerRun` qui
   échoue au dépassement.
8. `httptest.NewRecorder` alloue ses propres tampons et en-têtes : on mesurerait le banc de test en plus du
   handler.
9. Environ 19 côté serveur. Un handler à 0 allocation ne retire que la part du handler (14 % ici) : on mesure
   le tout avant d'optimiser une partie.
10. Encoder à l'écriture (`Fast`) : 49 ns et 0 allocation, contre 601 ns. On ne fait plus le travail à chaque
    lecture, au lieu de le faire plus vite.
