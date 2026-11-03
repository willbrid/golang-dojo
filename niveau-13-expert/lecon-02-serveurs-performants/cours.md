# Module 2 — Serveurs haute performance

> Un serveur Go s'écrit avec une goroutine par connexion et des lectures bloquantes. C'est le code le plus
> simple qui soit, et il tient pourtant des dizaines de milliers de connexions avec une poignée de fils
> système. Ce module ouvre le capot : ce que fait le *runtime* sous un `conn.Read`, ce que coûte une
> connexion, comment éviter de copier les données dans l'espace utilisateur, et comment tenir un **budget
> d'allocation** sur le chemin chaud, en sachant ce que `net/http` dépense déjà de son côté.

## Objectifs

- Expliquer comment le *runtime* multiplexe des milliers de goroutines bloquées en E/S sur quelques fils
  (le *netpoller*).
- Mesurer le coût d'une connexion, en mémoire et en fils.
- Exploiter les copies sans passage par l'espace utilisateur (`sendfile`) que `io.Copy` sait déclencher, et
  savoir ce qui les empêche.
- Fixer un budget d'allocation par requête et le faire respecter par un test.
- Situer le gain d'une optimisation du handler dans le coût total du serveur.

## Explication

### Une goroutine par connexion, et le *netpoller*

```go
for {
	c, _ := ln.Accept()
	go handle(c) // lit avec c.Read, bloque, répond, recommence
}
```

Sous Linux, un descripteur réseau ouvert par Go est en mode **non bloquant**, et enregistré auprès d'`epoll`
(`kqueue` sous BSD et macOS, des ports de complétion sous Windows). Quand une goroutine appelle `c.Read` et
qu'il n'y a rien à lire :

1. l'appel système répond « rien pour l'instant » (`EAGAIN`) ;
2. le *runtime* **gare** la goroutine, sans bloquer le fil système qui l'exécutait ;
3. ce fil exécute d'autres goroutines ;
4. quand `epoll` signale que le descripteur est prêt, la goroutine redevient exécutable et reprend son `Read`.

C'est le **netpoller**. Le code garde l'apparence d'une lecture bloquante, et le *runtime* fait la boucle
d'événements.

Mesuré dans l'exemple :

| | Avant | Après 10 000 connexions ouvertes et inactives |
|---|---|---|
| goroutines | 2 | 10 002 |
| fils système | 8 | **11** |
| mémoire du *runtime* | 8 Mio | 113 Mio |

Le nombre de fils ne suit pas le nombre de connexions : il reste lié à `GOMAXPROCS` (8 ici), plus quelques
fils de service. Une connexion coûte environ **11 Kio**, client et serveur réunis puisque les deux sont dans le
même processus :
- environ 4 Kio de pile pour la goroutine serveur ;
- un tampon `bufio` de 4 Kio ;
- les structures de connexion des deux côtés.

Un aller-retour sur chacune des 10 000 connexions prend 276 ms, sans qu'un fil de plus soit créé.

Conséquence : **une bibliothèque de boucle d'événements n'est presque jamais nécessaire en Go**. Le modèle
goroutine-par-connexion passe à des dizaines de milliers de connexions. Au-delà de quelques centaines de
milliers, le coût mémoire par connexion devient le sujet. On réduit alors les tampons par connexion (un
tampon prêté seulement pendant la lecture, au lieu d'un `bufio` permanent), et on vérifie les limites du
système (`ulimit -n`, ports éphémères).

Les appels système qui **ne** passent **pas** par le netpoller (lecture de fichiers ordinaires, certains
appels bloquants, cgo) bloquent bien leur fil. Le *runtime* en crée alors un autre pour continuer à exécuter
les goroutines. Un service qui lit beaucoup de fichiers sur un disque lent peut ainsi voir son nombre de fils
grimper.

### Ne pas copier : `io.Copy` et `sendfile`

`io.Copy(dst, src)` cherche d'abord si `src` implémente `io.WriterTo`, ou `dst` `io.ReaderFrom`. Un
`*net.TCPConn` implémente `ReadFrom`. Quand `src` est un `*os.File`, il utilise sous Linux l'appel système
**`sendfile`** : le noyau envoie les pages du fichier vers la socket **sans les recopier** dans la mémoire du
processus. Mesuré dans l'exemple (512 Mio vers une connexion locale, trois exécutions) :

| | Débit | Processeur (envoi et réception, même processus) | Appels système d'envoi |
|---|---|---|---|
| `io.Copy(conn, fichier)` | 2 144 à 2 274 Mio/s | 411 à 455 ms | 1 147 `sendfile` |
| copie par tampon de 32 Kio | 1 269 à 1 352 Mio/s | 574 à 609 ms | environ 16 700 `write` (plus autant de `read` du fichier) |

`sendfile` est **1,7 fois plus rapide**, et moins coûteux en processeur. Ce qui l'empêche :

- **envelopper** la connexion ou le fichier dans un type qui cache leur type concret : un `io.Writer` qui
  compte les octets, un middleware qui enveloppe le `ResponseWriter` sans exposer `ReadFrom`. L'exemple le
  fait exprès avec `struct{ io.Writer }{c}` ;
- **TLS** : les données doivent être chiffrées dans le processus, il n'y a plus de copie directe possible
  (sauf avec le chiffrement par le noyau, hors du périmètre de ce cours) ;
- **la compression** à la volée.

`http.ServeContent` et `http.ServeFile` utilisent `io.Copy`. Un fichier statique servi en HTTP clair
bénéficie donc de `sendfile`, à condition qu'aucun middleware ne cache le `ResponseWriter`. Un middleware qui
enveloppe le `ResponseWriter` (N7 L3) peut rester transparent en implémentant `Unwrap()`, que
`http.ResponseController` sait suivre. Il doit se demander ce qu'il fait perdre.

### Le budget d'allocation

Un **budget d'allocation** est un nombre maximal d'allocations par requête sur le chemin chaud, vérifié par un
test. Le test échoue si une modification fait dépasser le budget, même si elle ne change rien aux réponses.
`testing.AllocsPerRun` mesure les allocations d'une fonction, en moyenne sur N exécutions.

L'exemple compare trois versions du même handler, qui renvoient la **même** réponse octet pour octet (un test
le vérifie) :

| Handler | Allocations du handler | Temps du handler |
|---|---|---|
| `Naive` : `json.NewEncoder(w).Encode(b)` | 3 | 601 ns |
| `Pooled` : encodage à la main dans un tampon réutilisé | 3 | 400 ns |
| `Fast` : JSON encodé **une fois**, à l'écriture du livre | **0** | 49 ns |

Le test de budget a fait son travail **dès la première écriture** : `Pooled` avait été écrit avec un budget de
1 allocation, et le test a échoué (« 3 allocations, budget 1 »). Le `json.Marshal` qui échappe le titre alloue,
lui. Le budget est maintenant fixé à la valeur mesurée, 3, et l'exercice E2 demande de le ramener à 0.

Encoder à l'**écriture** plutôt qu'à la lecture est la meilleure optimisation des trois. Elle vaut pour une
donnée beaucoup plus lue qu'écrite, et elle ne demande aucune astuce, seulement de déplacer le travail.

### Le handler n'est pas tout le serveur

Mesuré côté **serveur**, avec un client dans un autre processus, sur 20 000 requêtes en connexion
persistante :

| Handler | Allocations du processus serveur par requête |
|---|---|
| `Naive` | 22,4 |
| `Pooled` | 22,5 |
| `Fast` | 19,3 |

`net/http` lui-même alloue **environ 19 fois par requête** : la requête, ses en-têtes, le contexte, le
`ResponseWriter`… Passer le handler de 3 à 0 allocations retire 14 % des allocations du serveur, pas 100 %.
C'est la loi d'Amdahl (N11 L8). Avant de supprimer les dernières allocations d'un handler, on mesure ce que
le reste coûte.

Si `net/http` lui-même devient le poste principal du profil, les leviers sont ailleurs :
- des connexions persistantes (éviter la poignée de main TCP et TLS à chaque requête) ;
- des réponses en lot plutôt qu'une requête par élément ;
- HTTP/2, qui multiplexe les requêtes ;
- dans des cas extrêmes, un protocole binaire sur TCP brut (N7 L10).

### Le reste de la liste

- **Pas de goroutine ni de minuterie par requête** sans nécessité. `time.AfterFunc` par requête crée une
  minuterie ; un `context.WithTimeout` aussi. On les annule (`defer cancel()`) pour les libérer tout de suite.
- **Des tampons réutilisés** pour les gros objets temporaires (`sync.Pool`, N11 L8), des tailles bornées.
- **`GOMEMLIMIT`** près de la mémoire allouée au conteneur, et `GOGC` ajusté si le GC domine le profil
  (N11 L3).
- **Limiter la concurrence** (N6 L11) plutôt que d'accepter un nombre illimité de requêtes en cours : la
  latence d'un serveur saturé explose (N11 L7).
- **Les délais du serveur** (N7 L9) : une connexion lente ne doit pas garder une goroutine et un tampon
  indéfiniment.

## Exemple

Trois programmes et un paquet. `conns` ouvre 10 000 connexions, `sendfile` compare les deux façons d'envoyer
un fichier, et le paquet `netx` contient les trois handlers et leur budget. `server` et `client` mesurent
les allocations du serveur réel.

```go.mod
// Fichier : go.mod
module example.com/hp

go 1.27
```

```go
// Fichier : cmd/conns/main.go
// Commande conns : 10 000 connexions TCP ouvertes et inactives, une goroutine chacune côté serveur.
// Que coûte une connexion ? Combien de fils système le runtime utilise-t-il ?
package main

import (
	"bufio"
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
			go func() { // une goroutine par connexion, en lecture bloquante : le style idiomatique
				r := bufio.NewReaderSize(c, 4096)
				for {
					line, err := r.ReadString('\n')
					if err != nil {
						c.Close()
						return
					}
					c.Write([]byte(line))
				}
			}()
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

```go
// Fichier : cmd/sendfile/main.go
// Commande sendfile : envoyer un fichier de 512 Mio sur une connexion TCP locale, avec io.Copy
// (qui délègue au noyau : sendfile) ou par une copie dans un tampon de l'espace utilisateur.
package main

import (
	"fmt"
	"io"
	"log"
	"net"
	"os"
	"path/filepath"
	"syscall"
	"time"
)

const size = 512 << 20

func cpu() time.Duration {
	var ru syscall.Rusage
	syscall.Getrusage(syscall.RUSAGE_SELF, &ru)
	return time.Duration(ru.Utime.Nano() + ru.Stime.Nano())
}

func main() {
	dir, _ := os.MkdirTemp("", "sendfile")
	defer os.RemoveAll(dir)
	path := filepath.Join(dir, "gros.bin")
	f, _ := os.Create(path)
	f.Truncate(size) // un fichier creux : les pages viennent du cache, la mesure porte sur la copie
	f.Close()

	mode := os.Args[1]
	ln, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		log.Fatal(err)
	}
	done := make(chan int64)
	go func() { // le client : lit et jette
		c, err := net.Dial("tcp", ln.Addr().String())
		if err != nil {
			log.Fatal(err)
		}
		n, _ := io.Copy(io.Discard, c)
		done <- n
	}()
	c, _ := ln.Accept()
	src, _ := os.Open(path)
	c0, t0 := cpu(), time.Now()
	var n int64
	switch mode {
	case "io.Copy":
		n, err = io.Copy(c, src) // *net.TCPConn implémente ReaderFrom : sendfile sous Linux
	case "tampon":
		// Cacher les types concrets empêche io.Copy de trouver ReaderFrom/WriterTo : copie par tampon.
		n, err = io.Copy(struct{ io.Writer }{c}, struct{ io.Reader }{src})
	}
	if err != nil {
		log.Fatal(err)
	}
	c.Close()
	<-done
	el, used := time.Since(t0), cpu()-c0
	fmt.Printf("%-8s %d Mio en %v (%.0f Mio/s), processeur %v\n", mode, n>>20, el.Round(time.Millisecond),
		float64(n>>20)/el.Seconds(), used.Round(time.Millisecond))
}
```

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

// appendJSONString encode une chaîne comme encoding/json (échappement de <, >, & compris).
func appendJSONString(dst []byte, s string) []byte {
	j, _ := json.Marshal(s) // simple et sûr ; voir l'exercice E2 pour l'écrire sans allocation
	return append(dst, j...)
}
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
	}{{"Naive", s.Naive, 3}, {"Fast", s.Fast, 0}, {"Pooled", s.Pooled, 3}} {
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
// Fichier : cmd/server/main.go
// Commande server : sert /books/{id} avec un handler donné, et rapporte les allocations du
// PROCESSUS SERVEUR par requête (net/http compris), mesurées par runtime/metrics.
package main

import (
	"fmt"
	"log"
	"net"
	"net/http"
	"os"
	"runtime/metrics"
	"sync/atomic"

	"example.com/hp/netx"
)

func mallocs() uint64 {
	s := []metrics.Sample{{Name: "/gc/heap/allocs:objects"}}
	metrics.Read(s)
	return s[0].Value.Uint64()
}

func main() {
	s := netx.NewStore([]netx.Book{{ID: 1, Title: "Le Petit Prince", Price: 899}, {ID: 2, Title: "Ça & <là>", Price: 1250}})
	h := map[string]http.HandlerFunc{"naive": s.Naive, "fast": s.Fast, "pooled": s.Pooled}[os.Args[1]]
	var n atomic.Int64
	var start uint64
	mux := http.NewServeMux()
	mux.HandleFunc("GET /books/{id}", func(w http.ResponseWriter, r *http.Request) {
		if n.Add(1) == 1000 { // après l'échauffement
			start = mallocs()
		}
		h(w, r)
	})
	mux.HandleFunc("GET /report", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "%.1f", float64(mallocs()-start)/float64(n.Load()-1000))
	})
	ln, err := net.Listen("tcp", "127.0.0.1:18321")
	if err != nil {
		log.Fatal(err)
	}
	http.Serve(ln, mux)
}
```

```go
// Fichier : cmd/client/main.go
// Commande client : 21 000 requêtes en connexion persistante, puis lit le rapport du serveur.
package main

import (
	"fmt"
	"io"
	"log"
	"net/http"
)

func main() {
	for range 21_000 {
		resp, err := http.Get("http://127.0.0.1:18321/books/2")
		if err != nil {
			log.Fatal(err)
		}
		io.Copy(io.Discard, resp.Body)
		resp.Body.Close()
	}
	resp, err := http.Get("http://127.0.0.1:18321/report")
	if err != nil {
		log.Fatal(err)
	}
	b, _ := io.ReadAll(resp.Body)
	fmt.Println(string(b))
}
```

```
$ go run ./cmd/conns
avant : goroutines 2, fils 8, mémoire du runtime 8 Mio
après 10000 connexions (2.403s) : goroutines 10002, fils 11, mémoire 113 Mio
par connexion (client + serveur dans ce processus) : 10.9 Kio, dont piles 4.0 Kio
un aller-retour sur chacune : 276ms ; fils ensuite : 11
$ for i in 1 2 3; do ./sendfile io.Copy; ./sendfile tampon; done
io.Copy  512 Mio en 225ms (2274 Mio/s), processeur 411ms
tampon   512 Mio en 379ms (1352 Mio/s), processeur 574ms
io.Copy  512 Mio en 239ms (2144 Mio/s), processeur 455ms
tampon   512 Mio en 397ms (1290 Mio/s), processeur 600ms
io.Copy  512 Mio en 237ms (2164 Mio/s), processeur 430ms
tampon   512 Mio en 404ms (1269 Mio/s), processeur 609ms
$ strace -f -c -e trace=sendfile,read,write,splice ./sendfile io.Copy      # extrait
 68,14    0,672572          10     65548         1 read
 31,86    0,314468         274      1147       511 sendfile
$ strace -f -c -e trace=sendfile,read,write,splice ./sendfile tampon       # extrait
 80,42    1,212559          14     81934         2 read
 19,58    0,295247          17     16685       146 write
$ go test -v ./netx/
    books_test.go:37: "{\"id\":2,\"title\":\"Ça \\u0026 \\u003clà\\u003e\",\"price\":1250}\napplication/json"
--- PASS: TestSameBytes (0.00s)
    books_test.go:51: Naive   3 allocation(s) par requête (budget 3)
    books_test.go:51: Fast    0 allocation(s) par requête (budget 0)
    books_test.go:51: Pooled  3 allocation(s) par requête (budget 3)
--- PASS: TestAllocationBudget (0.00s)
ok  	example.com/hp/netx	0.007s
$ go test -run XXX -bench Handlers -benchmem -count 5 ./netx/      # médianes
BenchmarkHandlers/Naive-8     601.0 ns/op   80 B/op   3 allocs/op
BenchmarkHandlers/Pooled-8    400.4 ns/op   64 B/op   3 allocs/op
BenchmarkHandlers/Fast-8       49.5 ns/op    0 B/op   0 allocs/op
$ ./server naive & ./client     # puis fast, puis pooled
naive   allocations du serveur par requête : 22.4
fast    allocations du serveur par requête : 19.3
pooled  allocations du serveur par requête : 22.5
```

(Les `read` de `strace` comptent aussi ceux du client, qui est dans le même processus : 65 548 dans les deux
cas. La différence est côté émetteur. Les `sendfile` en erreur, 511, sont des `EAGAIN` : la socket était
pleine, et le netpoller a garé la goroutine jusqu'à ce qu'elle se vide.)

## Explication du code

| Élément | Rôle |
|---|---|
| `/proc/self/status`, ligne `Threads:` | Le nombre de fils système du processus, lu directement au noyau. |
| `/memory/classes/heap/stacks:bytes` | La mémoire des piles de goroutines (`runtime/metrics`, N11 L3). |
| `f.Truncate(size)` | Un fichier creux de 512 Mio : pas de disque à lire, les pages viennent du cache. La mesure porte sur la copie seule. |
| `struct{ io.Writer }{c}` | Une enveloppe qui cache `*net.TCPConn` : `io.Copy` ne voit plus `ReadFrom` et copie par tampon. C'est exactement ce que fait, sans le vouloir, un middleware maladroit. |
| `syscall.Getrusage` | Le temps processeur (utilisateur et système) du processus. |
| `NewStore` encode chaque livre une fois | Le travail est fait à l'écriture ; `Fast` n'a plus qu'à écrire des octets. |
| `w.Header()["Content-Type"] = contentType` | Évite `Set`, qui normalise la clé et alloue une slice à chaque appel. Possible parce que la clé est déjà au format canonique et que la slice partagée n'est jamais modifiée. |
| `discard` | Un `ResponseWriter` qui n'alloue rien, pour mesurer le handler **seul**. `httptest.NewRecorder` alloue ses propres tampons. |
| `testing.AllocsPerRun(1000, …)` | La moyenne des allocations sur 1 000 appels : c'est le budget vérifié. |
| `/gc/heap/allocs:objects` dans `server` | Le nombre total d'objets alloués par le processus, lu avant et après la charge. |

## Erreurs fréquentes

1. **Chercher une bibliothèque de boucle d'événements** pour tenir 10 000 connexions : le *runtime* le fait
   déjà.
2. **Un tampon de 64 Kio par connexion** « pour la performance », multiplié par 100 000 connexions.
3. **Envelopper un `io.Writer`** sur le chemin d'un gros transfert, et perdre `sendfile` sans le savoir.
4. **Optimiser le handler à zéro allocation** quand le serveur en fait 19 par requête de toute façon.
5. **Pas de test de budget** : les allocations reviennent au premier changement anodin.
6. **Mesurer avec `httptest.NewRecorder`** et lui attribuer ses propres allocations.
7. **Des appels bloquants hors du netpoller** (fichiers sur un disque lent, cgo) dans chaque requête, et un
   nombre de fils qui grimpe.

## Bonnes pratiques Go

- Goroutine par connexion, lectures bloquantes, délais du serveur : le modèle par défaut.
- `io.Copy` entre types concrets pour les gros transferts ; des enveloppes qui exposent `Unwrap` ou
  `ReadFrom` si elles sont nécessaires.
- Un test `AllocsPerRun` par chemin chaud, avec un budget égal à la mesure ; on le baisse quand on optimise.
- Déplacer le travail vers l'écriture (pré-encodage, pré-calcul) quand les données sont beaucoup plus lues
  qu'écrites.
- Mesurer les allocations **du serveur entier** avant d'optimiser le handler.

## Ce que je dois retenir

- Le netpoller gare les goroutines bloquées en E/S : 10 000 connexions, 11 fils, environ 11 Kio par
  connexion.
- `io.Copy(conn, fichier)` utilise `sendfile` : 1,7 fois plus rapide ; une enveloppe ou TLS l'empêche.
- Un budget d'allocation est un test, et il a attrapé une erreur dès sa première exécution.
- Encoder à l'écriture : 0 allocation et 49 ns, contre 3 allocations et 601 ns.
- `net/http` fait environ 19 allocations par requête : le handler n'est qu'une partie du coût.

---

➡️ [Exercices](exercices.md)
