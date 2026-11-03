# Module 7 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Le corrigé complète le module du cours (`example.com/resil`). `overload/server.go` et `overload/client.go`
reçoivent les options des exercices. Les valeurs par défaut gardent le comportement du cours : le tableau de
`TestMetastable` est inchangé.

```go
// Fichier : overload/server.go
// Paquet overload : un serveur à file d'attente, et ses protections contre la surcharge.
package overload

import (
	"context"
	"errors"
	"sync"
	"sync/atomic"
	"time"
)

// ErrOverloaded est renvoyée sans attendre quand la file est pleine.
var ErrOverloaded = errors.New("surchargé")

type Config struct {
	Workers     int           // nombre de requêtes traitées en même temps
	QueueCap    int           // capacité de la file d'attente
	DropExpired bool          // ne pas traiter une requête dont l'appelant a déjà renoncé
	Service     time.Duration // durée du travail pour une requête
	LIFO        bool          // E1 : servir la requête la plus récente d'abord
	ShedBelow   bool          // E2 : refuser les requêtes BestEffort quand la file est à moitié pleine
	RejectCost  time.Duration // défi : un refus occupe un travailleur pendant cette durée (TLS, authentification)
}

type Stats struct {
	Served   atomic.Int64 // requêtes traitées jusqu'au bout
	Wasted   atomic.Int64 // dont : traitées alors que l'appelant avait renoncé
	Dropped  atomic.Int64 // abandonnées sans travail (délai dépassé)
	Rejected atomic.Int64 // refusées à l'entrée (file pleine)
}

// Priority : E2.
type Priority int

const (
	Critical Priority = iota
	BestEffort
)

type job struct {
	ctx  context.Context
	done chan error
}

type Server struct {
	cfg   Config
	queue chan job      // FIFO
	cpu   chan struct{} // défi : les Workers « processeurs », partagés par le travail et les refus
	wg    sync.WaitGroup
	Stats Stats

	// LIFO (E1) : une pile protégée par un verrou ; ready compte les éléments, et réveille les travailleurs.
	mu    sync.Mutex
	stack []job
	ready chan struct{}
}

func New(cfg Config) *Server {
	s := &Server{cfg: cfg, cpu: make(chan struct{}, cfg.Workers)}
	if cfg.LIFO {
		s.ready = make(chan struct{}, cfg.QueueCap)
	} else {
		s.queue = make(chan job, cfg.QueueCap)
	}
	for range cfg.Workers {
		s.wg.Go(s.work)
	}
	return s
}

// Close arrête les travailleurs, et attend qu'ils aient vidé la file.
func (s *Server) Close() {
	if s.cfg.LIFO {
		close(s.ready)
	} else {
		close(s.queue)
	}
	s.wg.Wait()
}

// Handle traite une requête critique.
func (s *Server) Handle(ctx context.Context) error { return s.HandlePrio(ctx, Critical) }

// HandlePrio traite une requête, ou renvoie une erreur : file pleine, ou délai de ctx dépassé.
func (s *Server) HandlePrio(ctx context.Context, p Priority) error {
	j := job{ctx: ctx, done: make(chan error, 1)}
	if !s.enqueue(j, p) {
		s.Stats.Rejected.Add(1)
		if s.cfg.RejectCost > 0 {
			s.cpu <- struct{}{}
			time.Sleep(s.cfg.RejectCost)
			<-s.cpu
		}
		return ErrOverloaded
	}
	select {
	case err := <-j.done:
		return err
	case <-ctx.Done():
		return ctx.Err() // l'appelant renonce ; la requête, elle, reste dans la file
	}
}

func (s *Server) enqueue(j job, p Priority) bool {
	if s.cfg.LIFO {
		s.mu.Lock()
		if len(s.stack) == s.cfg.QueueCap {
			s.mu.Unlock()
			return false
		}
		s.stack = append(s.stack, j)
		s.mu.Unlock()
		s.ready <- struct{}{} // jamais bloquant : un jeton par élément, au plus QueueCap
		return true
	}
	if s.cfg.ShedBelow && p == BestEffort && len(s.queue) >= s.cfg.QueueCap/2 {
		return false
	}
	select {
	case s.queue <- j:
		return true
	default:
		return false
	}
}

func (s *Server) work() {
	if s.cfg.LIFO {
		for range s.ready {
			s.mu.Lock()
			j := s.stack[len(s.stack)-1]
			s.stack = s.stack[:len(s.stack)-1]
			s.mu.Unlock()
			s.process(j)
		}
		return
	}
	for j := range s.queue {
		s.process(j)
	}
}

func (s *Server) process(j job) {
	if s.cfg.DropExpired && j.ctx.Err() != nil {
		s.Stats.Dropped.Add(1)
		j.done <- j.ctx.Err()
		return
	}
	s.cpu <- struct{}{}
	time.Sleep(s.cfg.Service) // le travail
	<-s.cpu
	s.Stats.Served.Add(1)
	if j.ctx.Err() != nil {
		s.Stats.Wasted.Add(1)
	}
	j.done <- nil
}
```

```go
// Fichier : overload/client.go
package overload

import (
	"context"
	"math/rand/v2"
	"sync"
	"time"
)

// Client appelle le serveur avec un délai par tentative, et réessaie avec attente exponentielle et jitter.
type Client struct {
	Server  *Server
	Timeout time.Duration // délai d'une tentative
	Retries int           // réessais au plus, après la première tentative
	Budget  *Budget       // nil : réessayer sans limite globale
	// NoBackoff : réessayer tout de suite (un client mal écrit ; défi).
	NoBackoff bool
	Attempts  int64 // tentatives envoyées (lu à la fin, sous mu)
	mu        sync.Mutex
}

// Call renvoie true si la requête a fini par réussir.
func (c *Client) Call(rng *rand.Rand) bool {
	backoff := 50 * time.Millisecond
	for attempt := 0; ; attempt++ {
		c.mu.Lock()
		c.Attempts++
		c.mu.Unlock()
		ctx, cancel := context.WithTimeout(context.Background(), c.Timeout)
		err := c.Server.Handle(ctx)
		cancel()
		if err == nil {
			if c.Budget != nil {
				c.Budget.Success()
			}
			return true
		}
		if attempt == c.Retries || (c.Budget != nil && !c.Budget.AllowRetry()) {
			return false
		}
		if !c.NoBackoff {
			time.Sleep(backoff/2 + rand.N(backoff)) // jitter (N10 L9)
			backoff *= 2
		}
	}
}

// Budget limite les réessais à une fraction des succès : chaque succès crédite Ratio jeton,
// chaque réessai en consomme un. Le solde est plafonné à Max.
type Budget struct {
	Ratio, Max float64
	mu         sync.Mutex
	tokens     float64
}

func (b *Budget) Success() {
	b.mu.Lock()
	b.tokens = min(b.tokens+b.Ratio, b.Max)
	b.mu.Unlock()
}

func (b *Budget) AllowRetry() bool {
	b.mu.Lock()
	defer b.mu.Unlock()
	if b.tokens < 1 {
		return false
	}
	b.tokens--
	return true
}
```

## E1 — FIFO ou LIFO sous surcharge

## E2 — Des priorités

Les deux exercices partagent un test : le scénario du cours, sans réessai, 20 % de requêtes critiques.

```go
// Fichier : overload/e12_test.go
package overload

import (
	"context"
	"fmt"
	"math/rand/v2"
	"sync"
	"testing"
	"testing/synctest"
	"time"
)

// classes : succès et total par fenêtre, pour chaque priorité.
type classes struct{ good, total [2][]int }

// runPrio joue le scénario de l'exemple, sans réessai ; 20 % des requêtes sont critiques.
func runPrio(t *testing.T, cfg Config) (classes, *Stats) {
	var r classes
	var st *Stats
	synctest.Test(t, func(t *testing.T) {
		srv := New(cfg)
		rng := rand.New(rand.NewPCG(1, 2))
		n := 12
		for p := range 2 {
			r.good[p], r.total[p] = make([]int, n), make([]int, n)
		}
		var mu sync.Mutex
		var wg sync.WaitGroup
		start := time.Now()
		for _, ph := range scenario {
			end := time.Since(start) + ph.d
			for time.Since(start) < end {
				time.Sleep(time.Duration(rng.ExpFloat64() / ph.rate * float64(time.Second)))
				w := int(time.Since(start) / window)
				if w >= n {
					break
				}
				p := BestEffort
				if rng.Float64() < 0.2 {
					p = Critical
				}
				wg.Go(func() {
					ctx, cancel := context.WithTimeout(context.Background(), time.Second)
					err := srv.HandlePrio(ctx, p)
					cancel()
					mu.Lock()
					r.total[p][w]++
					if err == nil {
						r.good[p][w]++
					}
					mu.Unlock()
				})
			}
		}
		wg.Wait()
		srv.Close()
		st = &srv.Stats
	})
	return r, st
}

func pct(g, t []int, w int) float64 { return 100 * float64(g[w]) / float64(max(t[w], 1)) }

func header(title string) {
	fmt.Println(title)
	fmt.Print("                          ")
	for w := range 12 {
		fmt.Printf("%4ds", w*5)
	}
}

func TestLIFO(t *testing.T) {
	base := Config{Workers: 4, Service: 25 * time.Millisecond, QueueCap: 1000}
	lifo := base
	lifo.LIFO = true
	header("E1 : taux de succès par fenêtre, sans réessai, file de 1 000")
	fmt.Println("   gaspillé")
	for _, v := range []struct {
		name string
		cfg  Config
	}{{"FIFO", base}, {"LIFO", lifo}} {
		r, st := runPrio(t, v.cfg)
		fmt.Printf("%-26s", v.name)
		for w := range 12 {
			g := r.good[0][w] + r.good[1][w]
			tot := r.total[0][w] + r.total[1][w]
			fmt.Printf("%5.0f", 100*float64(g)/float64(max(tot, 1)))
		}
		fmt.Printf("  %7.0f %%\n", 100*float64(st.Wasted.Load())/float64(max(st.Served.Load(), 1)))
	}
}

func TestPriorities(t *testing.T) {
	base := Config{Workers: 4, Service: 25 * time.Millisecond, QueueCap: 40, DropExpired: true}
	shed := base
	shed.ShedBelow = true
	header("E2 : taux de succès par classe, file de 40, abandon des requêtes expirées")
	fmt.Println()
	for _, v := range []struct {
		name string
		cfg  Config
	}{{"sans règle", base}, {"avec règle", shed}} {
		r, _ := runPrio(t, v.cfg)
		for p, cl := range []string{"critique", "best effort"} {
			fmt.Printf("%-26s", v.name+", "+cl)
			for w := range 12 {
				fmt.Printf("%5.0f", pct(r.good[p], r.total[p], w))
			}
			fmt.Println()
		}
	}
}
```

```
$ go test -run 'LIFO|Priorities' -v ./overload/
E1 : taux de succès par fenêtre, sans réessai, file de 1 000
                             0s   5s  10s  15s  20s  25s  30s  35s  40s  45s  50s  55s   gaspillé
FIFO                        100  100   22    0    0   20  100  100  100  100  100  100       35 %
LIFO                        100  100   56  100  100  100  100  100  100  100  100  100        8 %
E2 : taux de succès par classe, file de 40, abandon des requêtes expirées
                             0s   5s  10s  15s  20s  25s  30s  35s  40s  45s  50s  55s
sans règle, critique        100  100   55  100  100  100  100  100  100  100  100  100
sans règle, best effort     100  100   56   99  100  100  100  100  100  100  100  100
avec règle, critique        100  100  100  100  100  100  100  100  100  100  100  100
avec règle, best effort     100  100   42   99  100  100  100  100  100  100  100  100
```

**E1.**
- **Pendant la pointe, LIFO sert 56 % des requêtes, FIFO 22 %.** En FIFO, le travailleur prend la requête la
  plus ancienne, celle qui a déjà attendu le plus longtemps et qui expirera la première. Dès que la file
  représente plus d'une seconde de travail, **toutes** les requêtes servies arrivent trop tard. En LIFO, le
  travailleur prend la plus récente, qui a toute sa seconde devant elle.
- **Après la pointe, LIFO est sain tout de suite** ; FIFO met 15 s à vider sa file, et ne sert personne
  pendant ce temps.
- **Qui paie ?** Les requêtes du **fond** de la pile. Arrivées au début de la pointe, elles ne sont jamais
  servies à temps, alors que des requêtes plus récentes passent devant elles. LIFO n'est pas équitable : sous
  surcharge, il sacrifie quelques requêtes pour en servir beaucoup d'autres. Les 8 % de travail gaspillé sont
  ces vieilles requêtes, servies après la pointe, quand plus personne ne les attend. Avec l'abandon des requêtes
  expirées, ce gaspillage disparaît.
- **La structure** : un canal est FIFO. La pile est une slice protégée par un verrou. Le canal `ready` sert de
  compteur : un jeton par élément, et un travailleur qui attend un jeton dort sans consommer de processeur.

**E2.**
- **Avec la règle, les requêtes critiques réussissent à 100 %, même pendant la pointe**, contre 55 % sans. Les
  requêtes `BestEffort` passent de 56 à 42 % : elles paient pour les autres, et c'est voulu.
- La règle ne coûte rien hors pointe : la file n'atteint jamais la moitié de sa capacité.
- **Pourquoi un seuil à la moitié de la file, et pas une file par priorité ?** Le seuil garde de la place pour
  les requêtes critiques. Les deux classes partagent les mêmes travailleurs, et les requêtes `BestEffort` n'occupent
  jamais plus de la moitié de la file. Deux files demanderaient un ordonnancement entre elles (qui servir
  quand les deux attendent ?), avec un risque de famine pour la classe basse.

## E3 — Propager le délai

Le serveur du cours, avec l'option `-deadline`, et le générateur de charge, avec l'en-tête `Request-Timeout`.
Le serveur compte les requêtes **traitées pour rien** : la réponse est prête, mais le client a déjà fermé la
connexion (`r.Context().Err() != nil`).

```go
// Fichier : cmd/srv/main.go
// srv : un service limité par sa base de données (4 connexions, 5 ms par requête : 800 req/s au plus).
// E3 : avec -deadline, il respecte le délai envoyé par le client dans l'en-tête Request-Timeout.
package main

import (
	"context"
	"flag"
	"fmt"
	"log"
	"net/http"
	"strconv"
	"sync/atomic"
	"time"

	"example.com/resil/admit"
)

var db = make(chan struct{}, 4) // le pool de connexions (comme SetMaxOpenConns(4), N8 L5)

var served, wasted, aborted atomic.Int64

// query attend une connexion libre, mais pas plus longtemps que ctx ne le permet.
func query(ctx context.Context) error {
	select {
	case db <- struct{}{}:
	case <-ctx.Done():
		return ctx.Err()
	}
	defer func() { <-db }()
	if err := ctx.Err(); err != nil { // le délai a pu expirer pendant le select
		return err
	}
	time.Sleep(5 * time.Millisecond)
	return nil
}

func handler(useDeadline bool) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx := context.Background() // sans -deadline : attendre sans limite, comme dans l'exemple
		if useDeadline {
			ms, err := strconv.Atoi(r.Header.Get("Request-Timeout"))
			if err != nil || ms <= 0 {
				ms = 1000 // pas d'en-tête : un délai par défaut, jamais l'infini
			}
			var cancel context.CancelFunc
			ctx, cancel = context.WithTimeout(r.Context(), time.Duration(ms)*time.Millisecond)
			defer cancel()
		}
		if err := query(ctx); err != nil {
			aborted.Add(1)
			http.Error(w, "délai dépassé", http.StatusGatewayTimeout)
			return
		}
		served.Add(1)
		if r.Context().Err() != nil { // le client a fermé la connexion : personne ne lira la réponse
			wasted.Add(1)
		}
		w.Write([]byte("ok"))
	}
}

func main() {
	addr := flag.String("addr", "127.0.0.1:8090", "adresse d'écoute")
	limit := flag.Int("limit", 0, "requêtes simultanées au plus (0 : pas de limite)")
	deadline := flag.Bool("deadline", false, "respecter l'en-tête Request-Timeout")
	flag.Parse()
	var h http.Handler = handler(*deadline)
	if *limit > 0 {
		h = admit.Limit(*limit, 5*time.Millisecond, h)
	}
	mux := http.NewServeMux()
	mux.Handle("/", h)
	mux.HandleFunc("GET /stats", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, served.Load(), wasted.Load(), aborted.Load())
	})
	log.Fatal(http.ListenAndServe(*addr, mux))
}
```

```go
// Fichier : cmd/load/main.go
// load : un test de charge par paliers, en boucle ouverte (N11 L7).
package main

import (
	"context"
	"flag"
	"fmt"
	"io"
	"net/http"
	"slices"
	"strconv"
	"strings"
	"sync"
	"time"
)

func main() {
	url := flag.String("url", "http://127.0.0.1:8090/", "adresse du service")
	step := flag.Duration("step", 3*time.Second, "durée d'un palier")
	rates := flag.String("rates", "200,400,600,700,800,900,1000,1200", "débits des paliers, en req/s")
	flag.Parse()
	client := &http.Client{Transport: &http.Transport{MaxIdleConnsPerHost: 2000}}
	fmt.Println("  débit demandé   réussi/s   503/s   délai/s     p50      p99   traitées pour rien/s")
	for _, f := range strings.Split(*rates, ",") {
		rate, _ := strconv.Atoi(f)
		before := stats(*url)
		var mu sync.Mutex
		var lat []float64
		var ok, shed, timeout int
		var wg sync.WaitGroup
		interval := time.Second / time.Duration(rate)
		start := time.Now()
		for i := 0; ; i++ {
			planned := start.Add(time.Duration(i) * interval)
			if planned.Sub(start) >= *step {
				break
			}
			time.Sleep(time.Until(planned))
			wg.Go(func() {
				ctx, cancel := context.WithTimeout(context.Background(), time.Second)
				defer cancel()
				req, _ := http.NewRequestWithContext(ctx, "GET", *url, nil)
				req.Header.Set("Request-Timeout", "1000") // E3 : le délai voyage avec la requête
				resp, err := client.Do(req)
				code := 0
				if err == nil {
					io.Copy(io.Discard, resp.Body)
					resp.Body.Close()
					code = resp.StatusCode
				}
				d := float64(time.Since(planned).Microseconds()) / 1000
				mu.Lock()
				defer mu.Unlock()
				switch code {
				case http.StatusOK:
					ok++
					lat = append(lat, d)
				case http.StatusServiceUnavailable:
					shed++
				default:
					timeout++
				}
			})
		}
		wg.Wait()
		time.Sleep(time.Second) // laisser le service finir ce qui reste dans sa file
		after := stats(*url)
		slices.Sort(lat)
		q := func(p float64) float64 {
			if len(lat) == 0 {
				return 0
			}
			return lat[int(p*float64(len(lat)-1))]
		}
		s := step.Seconds()
		fmt.Printf("%10d/s  %9.0f  %6.0f  %8.0f  %6.1f  %7.1f  %10.0f\n",
			rate, float64(ok)/s, float64(shed)/s, float64(timeout)/s, q(0.50), q(0.99), float64(after-before)/s)
	}
}

// stats lit le nombre de requêtes traitées pour rien, sur /stats ; -1 si le service ne l'expose pas.
func stats(base string) int {
	resp, err := http.Get(strings.TrimSuffix(base, "/") + "/stats")
	if err != nil {
		return -1
	}
	defer resp.Body.Close()
	var served, wasted, aborted int
	if _, err := fmt.Fscan(resp.Body, &served, &wasted, &aborted); err != nil {
		return -1
	}
	return wasted
}
```

```
$ taskset -c 0,1 ./srv &
$ taskset -c 2-7 ./load -rates 600,800,1000,1200
  débit demandé   réussi/s   503/s   délai/s     p50      p99   traitées pour rien/s
       600/s        600       0         0     6.6      8.2           0
       800/s        800       0         0   243.9    471.6           0
      1000/s        753       0       247   497.3    989.5         246
      1200/s        579       0       621   497.7    993.0         581
$ taskset -c 0,1 ./srv -deadline &
$ taskset -c 2-7 ./load -rates 600,800,1000,1200
  débit demandé   réussi/s   503/s   délai/s     p50      p99   traitées pour rien/s
       600/s        600       0         0     6.9      9.0           0
       800/s        800       0         0   252.2    516.5           0
      1000/s        716       0       284   505.5    988.7         193
      1200/s        555       0       645   497.5    992.4         370
$ taskset -c 0,1 ./srv -deadline -limit 8 &
$ taskset -c 2-7 ./load -rates 600,800,1000,1200
  débit demandé   réussi/s   503/s   délai/s     p50      p99   traitées pour rien/s
       600/s        599       1         0     7.0     20.5           0
       800/s        691     109         0    15.8     19.8           0
      1000/s        680     320         0    16.3     21.1           0
      1200/s        694     506         0    16.3     20.6           0
```

Les comptes par seconde sont ramenés à la durée du palier (3 s). Le service finit sa file pendant la seconde
de pause qui suit, d'où un total servi un peu supérieur à 700 par seconde de palier.

- **Le délai propagé réduit le travail gaspillé d'environ un tiers** (581 → 370 req/s à 1 200 req/s). Le
  serveur n'attend plus une connexion pour une requête déjà expirée.
- **Il n'augmente pas le débit utile** (579 → 555, dans le bruit de la mesure). La file reste longue d'une
  seconde : chaque requête attend presque tout son délai, et celles qui arrivent au bout de la file passent de
  justesse ou pas du tout. Respecter le délai **évite le gaspillage**, il ne **raccourcit pas la file**.
- **Le reste du gaspillage** : une requête obtient sa connexion à 995 ms, la réponse part à 1 001 ms, et le
  client est parti à 1 000 ms. Une marge règle ce cas : refuser de commencer quand il reste moins que la durée
  du travail (5 ms, ou mieux le p99 mesuré).
- **Avec le contrôle d'admission**, plus rien n'est traité pour rien : la file ne dépasse jamais quelques
  millisecondes. Le délai propagé et le contrôle d'admission sont **complémentaires**. L'un empêche de
  travailler pour un client parti, l'autre empêche la file de grandir.

## E4 — Le contrôle d'admission qui ne voit rien

```go
// Fichier : cmd/srvcpu/main.go
// srvcpu : un service dont chaque requête coûte environ 1 ms de calcul (E4).
package main

import (
	"crypto/sha256"
	"flag"
	"log"
	"net/http"
	_ "net/http/pprof" // /debug/pprof/trace, pour la trace d'exécution
	"time"

	"example.com/resil/admit"
	"example.com/resil/schedshed"
)

func work(w http.ResponseWriter, r *http.Request) {
	sum := sha256.Sum256([]byte(r.URL.Path))
	for range 4500 { // environ 1 ms sur un cœur de la machine de mesure
		sum = sha256.Sum256(sum[:])
	}
	w.Write(sum[:4])
}

func main() {
	addr := flag.String("addr", "127.0.0.1:8090", "adresse d'écoute")
	limit := flag.Int("limit", 0, "requêtes simultanées au plus (0 : pas de limite)")
	sched := flag.Duration("sched", 0, "cible de latence d'ordonnancement (0 : pas de refus)")
	flag.Parse()
	var h http.Handler = http.HandlerFunc(work)
	if *limit > 0 {
		h = admit.Limit(*limit, 5*time.Millisecond, h)
	}
	if *sched > 0 {
		h = schedshed.New(*sched).Wrap(h)
	}
	mux := http.NewServeMux()
	mux.Handle("/", h)
	mux.Handle("/debug/", http.DefaultServeMux)
	go func() { log.Fatal(http.ListenAndServe("127.0.0.1:8091", nil)) }() // administration (N12 L9)
	log.Fatal(http.ListenAndServe(*addr, mux))
}
```

Le calcul de 4 500 tours de SHA-256 coûte environ 1 ms : un benchmark de 2 500 tours mesure 0,57 ms par
opération sur un cœur. Le générateur est celui d'E3 ; la colonne « traitées pour rien » est retirée (ce service
n'a pas de `/stats`).

```
$ GOMAXPROCS=1 taskset -c 0 ./srvcpu &
$ taskset -c 2-7 ./load -rates 400,600,800,900,1000,1200,1500
  débit demandé   réussi/s   503/s   délai/s     p50      p99
       400/s        400       0         0     1.9      5.8
       600/s        600       0         0     1.9     10.7
       800/s        800       0         0     2.1     21.0
       900/s        900       0         0     2.8     19.4
      1000/s       1000       0         0   186.6    385.6
      1200/s        952       0       249   372.7    980.0
      1500/s        638       0       862   305.5    696.6
$ GOMAXPROCS=1 taskset -c 0 ./srvcpu -limit 4 &
  débit demandé   réussi/s   503/s   délai/s     p50      p99
       400/s        400       0         0     2.1      8.5
       600/s        600       0         0     1.9     12.4
       800/s        800       0         0     2.1     24.4
       900/s        900       0         0    96.8    121.9
      1000/s       1000       0         0   245.8    509.1
      1200/s       1048       0       152   412.8    989.6
      1500/s        709       0       792   353.2    882.4
```

**Le contrôle d'admission ne refuse aucune requête**, et la latence explose comme sans lui. Une trace d'une
seconde, prise pendant la surcharge à 1 200 req/s sur l'adresse d'administration :

```
$ curl -o cpu.trace 'http://127.0.0.1:8091/debug/pprof/trace?seconds=1'
$ go tool trace -pprof=sched cpu.trace > sched.pprof
$ go tool pprof -top sched.pprof
Type: delay
Showing nodes accounting for 2.45ms, 0.0088% of 27777.56ms total
```

**27,8 s d'attente d'ordonnancement en une seconde** : en moyenne, environ 28 goroutines **prêtes** attendent le
seul processeur. C'est la file. Presque toute cette attente est comptée sans pile d'appels (2,45 ms seulement
sont attribués à une fonction). D'après sa forme, ce sont des goroutines qui n'ont pas encore commencé à
s'exécuter. C'est une interprétation, non vérifiée plus avant.

Le sémaphore, lui, n'est jamais plein. Avec un seul processeur, un gestionnaire calcule sa milliseconde sans
être interrompu (la préemption n'a lieu qu'après 10 ms) : il y a rarement plus d'une requête **dans** le
gestionnaire. Les autres attendent **avant**, dans l'ordonnanceur et dans les tampons du noyau.

**Une protection qui fonctionne** : mesurer l'attente là où elle a lieu. Le *runtime* publie la latence
d'ordonnancement (`/sched/latencies:seconds`, un histogramme de `runtime/metrics`). Quand son p99 dépasse une
cible, on refuse une fraction des requêtes, avant tout travail :

```go
// Fichier : schedshed/schedshed.go
// Paquet schedshed : refuser des requêtes quand les goroutines attendent trop le processeur.
//
// La file d'un service limité par le processeur n'est pas dans le gestionnaire, mais dans
// l'ordonnanceur : des goroutines prêtes qui attendent un P. Le runtime mesure cette attente
// (/sched/latencies:seconds) ; on refuse une fraction des requêtes tant que son p99 dépasse la cible.
package schedshed

import (
	"math/rand/v2"
	"net/http"
	"runtime/metrics"
	"sync/atomic"
	"time"
)

type Shedder struct {
	target time.Duration
	prob   atomic.Uint64 // probabilité de refus, en millièmes
}

// New démarre la surveillance : toutes les 100 ms, le p99 de la latence d'ordonnancement sur
// l'intervalle est comparé à target, et la probabilité de refus monte ou descend.
func New(target time.Duration) *Shedder {
	s := &Shedder{target: target}
	go s.watch()
	return s
}

func (s *Shedder) watch() {
	sample := []metrics.Sample{{Name: "/sched/latencies:seconds"}}
	var prev []uint64
	for range time.Tick(100 * time.Millisecond) {
		metrics.Read(sample)
		h := sample[0].Value.Float64Histogram()
		if prev == nil {
			prev = make([]uint64, len(h.Counts))
			copy(prev, h.Counts)
			continue
		}
		p99 := quantile(h, prev, 0.99)
		copy(prev, h.Counts)
		p := s.prob.Load()
		if p99 > s.target.Seconds() {
			p = min(p+100, 950) // +10 %, au plus 95 % de refus
		} else if p >= 50 {
			p -= 50 // -5 % : redescendre plus lentement qu'on ne monte
		} else {
			p = 0
		}
		s.prob.Store(p)
	}
}

// quantile renvoie la borne supérieure du seau qui contient le quantile q de la différence h - prev.
func quantile(h *metrics.Float64Histogram, prev []uint64, q float64) float64 {
	var total uint64
	for i, c := range h.Counts {
		total += c - prev[i]
	}
	if total == 0 {
		return 0
	}
	want := uint64(q * float64(total))
	var acc uint64
	for i, c := range h.Counts {
		acc += c - prev[i]
		if acc > want {
			return h.Buckets[i+1]
		}
	}
	return h.Buckets[len(h.Buckets)-1]
}

// Prob renvoie la probabilité de refus courante.
func (s *Shedder) Prob() float64 { return float64(s.prob.Load()) / 1000 }

func (s *Shedder) Wrap(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if p := s.prob.Load(); p > 0 && rand.N(uint64(1000)) < p {
			w.Header().Set("Retry-After", "1")
			http.Error(w, "surchargé", http.StatusServiceUnavailable)
			return
		}
		next.ServeHTTP(w, r)
	})
}
```

```
$ GOMAXPROCS=1 taskset -c 0 ./srvcpu -sched 5ms &
  débit demandé   réussi/s   503/s   délai/s     p50      p99
       400/s        399       1         0     2.0      6.6
       600/s        587      13         0     1.9     19.0
       800/s        775      25         0     2.0     26.6
       900/s        600     300         0     2.6     56.4
      1000/s        828     172         0     3.0     24.8
      1200/s        496     705         0     3.0     57.5
      1500/s        619     882         0     3.2    128.0
$ GOMAXPROCS=1 taskset -c 0 ./srvcpu -sched 10ms &
  débit demandé   réussi/s   503/s   délai/s     p50      p99
       400/s        400       0         0     2.4      8.8
       600/s        600       0         0     1.8     10.7
       800/s        786      14         0     1.9     18.0
       900/s        831      69         0     2.4     35.9
      1000/s        833     167         0     4.6     43.0
      1200/s        789     411         0     4.7     59.6
      1500/s        508     993         0    11.6    283.8
```

- **Plus aucun délai dépassé**, et un p50 qui reste sous 12 ms jusqu'à 1 500 req/s, contre 186 à 373 ms sans
  protection au-delà de 1 000 req/s.
- **Mais la protection est imparfaite**, et la mesure le montre :
  - **elle refuse trop** : avec une cible de 5 ms, le service ne sert que 600 req/s à 900 req/s demandées, alors
    qu'il en sert 900 sans protection ;
  - **elle oscille** : la probabilité monte et descend par pas de 100 ms, et les paliers voisins donnent des
    résultats irréguliers (600 puis 828 req/s servies) ;
  - **un refus coûte encore** : il faut accepter la connexion et analyser la requête HTTP avant de refuser. À
    1 500 req/s, ces refus consomment une part du seul processeur, et le débit utile baisse (508 req/s avec la
    cible de 10 ms).
- **La cible** se règle comme la limite du cours : par un test par paliers. 10 ms sert mieux que 5 ms ici, jusqu'à 1 200 req/s.
- **En production**, on préfère protéger un service limité par le processeur **en amont** : un équilibreur de
  charge qui limite les requêtes en cours par instance, et qui refuse sans rien coûter au service. Plus de
  processeurs aident aussi : avec `GOMAXPROCS` > 1, les gestionnaires se chevauchent, et un sémaphore
  redevient visible.

## E5 — Injecter des pannes

```go
// Fichier : chaos/chaos.go
// Paquet chaos : un http.RoundTripper qui injecte des pannes, pour les tester avant qu'elles n'arrivent.
package chaos

import (
	"errors"
	"math/rand/v2"
	"net/http"
	"sync"
	"time"
)

// ErrInjected est l'erreur renvoyée par une panne injectée.
var ErrInjected = errors.New("chaos : panne injectée")

type Transport struct {
	Next      http.RoundTripper
	Delay     time.Duration // retard ajouté…
	DelayProb float64       // … avec cette probabilité
	ErrorProb float64       // probabilité d'une erreur, sans appeler Next

	mu  sync.Mutex
	rng *rand.Rand
}

// New renvoie un Transport déterministe : la même graine donne la même suite de pannes.
func New(next http.RoundTripper, seed uint64) *Transport {
	return &Transport{Next: next, rng: rand.New(rand.NewPCG(seed, seed))}
}

func (t *Transport) RoundTrip(req *http.Request) (*http.Response, error) {
	t.mu.Lock()
	fail, slow := t.rng.Float64() < t.ErrorProb, t.rng.Float64() < t.DelayProb
	t.mu.Unlock()
	if fail {
		return nil, ErrInjected
	}
	if slow {
		timer := time.NewTimer(t.Delay)
		defer timer.Stop()
		select {
		case <-timer.C:
		case <-req.Context().Done(): // un retard injecté respecte l'annulation, comme un vrai réseau
			return nil, req.Context().Err()
		}
	}
	return t.Next.RoundTrip(req)
}
```

```go
// Fichier : page/page.go
// Paquet page : une page produit qui affiche des recommandations, et s'en passe si elles manquent.
package page

import (
	"bufio"
	"context"
	"fmt"
	"net/http"
	"sync/atomic"
	"time"
)

type Page struct {
	Client    *http.Client
	RecoURL   string
	Timeout   time.Duration // délai accordé au service de recommandations
	Fallbacks atomic.Int64  // pages servies sans recommandations
}

func (p *Page) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	recos := p.recos(r.Context())
	fmt.Fprintln(w, "produit 42")
	for _, rec := range recos {
		fmt.Fprintln(w, "voir aussi :", rec)
	}
}

// recos renvoie les recommandations, ou nil si le service est lent ou en panne (mode dégradé).
func (p *Page) recos(ctx context.Context) []string {
	ctx, cancel := context.WithTimeout(ctx, p.Timeout)
	defer cancel()
	req, err := http.NewRequestWithContext(ctx, "GET", p.RecoURL, nil)
	if err != nil {
		p.Fallbacks.Add(1)
		return nil
	}
	resp, err := p.Client.Do(req)
	if err != nil {
		p.Fallbacks.Add(1)
		return nil
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		p.Fallbacks.Add(1)
		return nil
	}
	var out []string
	sc := bufio.NewScanner(resp.Body)
	for sc.Scan() {
		out = append(out, sc.Text())
	}
	if sc.Err() != nil {
		p.Fallbacks.Add(1)
		return nil
	}
	return out
}
```

```go
// Fichier : page/page_test.go
package page

import (
	"io"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"testing/synctest"
	"time"

	"example.com/resil/chaos"
)

// fakeReco répond en 2 ms, sans réseau : tout reste dans la bulle de synctest.
type fakeReco struct{}

func (fakeReco) RoundTrip(req *http.Request) (*http.Response, error) {
	time.Sleep(2 * time.Millisecond)
	return &http.Response{
		StatusCode: http.StatusOK,
		Body:       io.NopCloser(strings.NewReader("livre A\nlivre B\n")),
		Request:    req,
	}, nil
}

func newPage(ct *chaos.Transport) *Page {
	return &Page{Client: &http.Client{Transport: ct}, RecoURL: "http://reco/", Timeout: 50 * time.Millisecond}
}

func get(p *Page) (string, time.Duration) {
	start := time.Now()
	rec := httptest.NewRecorder()
	p.ServeHTTP(rec, httptest.NewRequest("GET", "/produit/42", nil))
	return rec.Body.String(), time.Since(start)
}

func TestRecoSlow(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		ct := chaos.New(fakeReco{}, 1)
		ct.Delay, ct.DelayProb = 200*time.Millisecond, 0.3
		p := newPage(ct)
		var worst time.Duration
		with := 0
		for range 1000 {
			body, d := get(p)
			worst = max(worst, d)
			if strings.Contains(body, "voir aussi") {
				with++
			}
		}
		t.Logf("pire latence %v ; %d pages avec recommandations, %d replis", worst, with, p.Fallbacks.Load())
		if worst >= 60*time.Millisecond {
			t.Errorf("pire latence %v, attendu < 60 ms", worst)
		}
	})
}

func TestRecoDown(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		ct := chaos.New(fakeReco{}, 2)
		ct.ErrorProb = 1
		p := newPage(ct)
		for range 100 {
			body, _ := get(p)
			if !strings.HasPrefix(body, "produit 42") || strings.Contains(body, "voir aussi") {
				t.Fatalf("page inattendue : %q", body)
			}
		}
		if got := p.Fallbacks.Load(); got != 100 {
			t.Errorf("replis : %d, attendu 100", got)
		}
	})
}

func TestDeterministic(t *testing.T) {
	run := func() int64 {
		var n int64
		synctest.Test(t, func(t *testing.T) {
			ct := chaos.New(fakeReco{}, 7)
			ct.Delay, ct.DelayProb, ct.ErrorProb = 200*time.Millisecond, 0.2, 0.1
			p := newPage(ct)
			for range 500 {
				get(p)
			}
			n = p.Fallbacks.Load()
		})
		return n
	}
	a, b := run(), run()
	t.Logf("replis : %d puis %d", a, b)
	if a != b {
		t.Errorf("deux exécutions de même graine diffèrent : %d et %d", a, b)
	}
}
```

```
$ go test -v ./page/
=== RUN   TestRecoSlow
    page_test.go:52: pire latence 50ms ; 718 pages avec recommandations, 282 replis
--- PASS: TestRecoSlow (0.01s)
=== RUN   TestRecoDown
--- PASS: TestRecoDown (0.00s)
=== RUN   TestDeterministic
    page_test.go:91: replis : 149 puis 149
--- PASS: TestDeterministic (0.01s)
PASS
ok  	example.com/resil/page	0.031s
```

- **Le retard injecté respecte l'annulation** : un `select` entre le minuteur et `req.Context().Done()`. Sans
  cela, le délai de 50 ms de la page n'aurait aucun effet sur un retard injecté, et le test mesurerait un
  défaut de l'outil, pas du service.
- **Pire latence : 50 ms tout juste**, c'est-à-dire le délai accordé à `reco`. 282 pages sur 1 000 sont servies
  sans recommandations, pour 30 % d'appels retardés (le tirage donne 28,2 %).
- **Tout reste dans la bulle de `synctest`** : `fakeReco` répond sans réseau, et `httptest.NewRecorder` évite le
  serveur. Les 1 000 pages, dont 282 attendent 50 ms, se testent en 10 ms de temps réel. Un vrai réseau ne
  bloque pas « durablement » au sens de `synctest`, et le temps virtuel n'avancerait pas correctement.
- **Le déterminisme** : deux exécutions de même graine donnent 149 replis. Un test de pannes qui échoue doit se
  rejouer à l'identique ; sinon on ne sait pas si la correction a fonctionné.
- **Le compteur `Fallbacks`** est ce qu'on exporterait comme métrique (N12 L7). Un repli invisible pour
  l'utilisateur doit rester visible pour l'exploitation.

---

## Exercice intermédiaire — Une limite de concurrence adaptative

```go
// Fichier : adaptive/adaptive.go
// Paquet adaptive : un contrôle d'admission dont la limite se règle seule (AIMD).
package adaptive

import (
	"net/http"
	"sync"
	"time"
)

type Limiter struct {
	Target   time.Duration // latence visée pour une requête
	Min, Max float64       // bornes de la limite

	mu           sync.Mutex
	limit        float64
	inflight     int
	lastDecrease time.Time
}

func New(target time.Duration, initial, lo, hi float64) *Limiter {
	return &Limiter{Target: target, Min: lo, Max: hi, limit: initial}
}

// Limit renvoie la limite courante.
func (l *Limiter) Limit() float64 {
	l.mu.Lock()
	defer l.mu.Unlock()
	return l.limit
}

func (l *Limiter) acquire() bool {
	l.mu.Lock()
	defer l.mu.Unlock()
	if l.inflight >= int(l.limit) {
		return false
	}
	l.inflight++
	return true
}

// release ajuste la limite selon la latence observée : +1/limite si elle est bonne (environ +1 par
// fenêtre de requêtes), ×0,9 si elle dépasse la cible, au plus une fois par Target.
func (l *Limiter) release(d time.Duration) {
	l.mu.Lock()
	defer l.mu.Unlock()
	l.inflight--
	if d <= l.Target {
		l.limit = min(l.Max, l.limit+1/l.limit)
		return
	}
	if now := time.Now(); now.Sub(l.lastDecrease) >= l.Target {
		l.limit = max(l.Min, l.limit*0.9)
		l.lastDecrease = now
	}
}

func (l *Limiter) Wrap(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if !l.acquire() {
			w.Header().Set("Retry-After", "1")
			http.Error(w, "surchargé", http.StatusServiceUnavailable)
			return
		}
		start := time.Now()
		defer func() { l.release(time.Since(start)) }()
		next.ServeHTTP(w, r)
	})
}
```

Le serveur dont le pool change de taille : un canal de 64 places, dont une goroutine occupe une partie pour
simuler un pool plus petit.

```go
// Fichier : cmd/adapt/main.go
// adapt : le service de l'exemple, dont le pool de connexions change de taille en cours de route.
package main

import (
	"flag"
	"fmt"
	"log"
	"net/http"
	"strconv"
	"strings"
	"time"

	"example.com/resil/adaptive"
	"example.com/resil/admit"
)

const maxConns = 64

// db : un pool de maxConns places, dont une partie est occupée par « une autre application ».
var db = make(chan struct{}, maxConns)

// resize fait passer le pool à n connexions libres. occupied est le nombre de places prises.
func resize(occupied *int, n int) {
	want := maxConns - n
	for *occupied < want {
		db <- struct{}{} // prendre une place (attend qu'une requête la libère)
		*occupied++
	}
	for *occupied > want {
		<-db
		*occupied--
	}
}

func work(w http.ResponseWriter, r *http.Request) {
	db <- struct{}{}
	time.Sleep(5 * time.Millisecond)
	<-db
	w.Write([]byte("ok"))
}

func main() {
	mode := flag.String("mode", "fixed", "fixed ou aimd")
	limit := flag.Int("limit", 8, "limite fixe (mode fixed)")
	pools := flag.String("pools", "4:10s,16:10s,2:10s", "tailles successives du pool, et leur durée")
	flag.Parse()

	occupied := 0
	resize(&occupied, 4)
	go func() {
		for _, f := range strings.Split(*pools, ",") {
			n, d, _ := strings.Cut(f, ":")
			size, _ := strconv.Atoi(n)
			dur, _ := time.ParseDuration(d)
			resize(&occupied, size)
			time.Sleep(dur)
		}
	}()

	var h http.Handler = http.HandlerFunc(work)
	current := func() float64 { return float64(*limit) }
	if *mode == "aimd" {
		l := adaptive.New(20*time.Millisecond, 8, 1, 200)
		h, current = l.Wrap(h), l.Limit
	} else {
		h = admit.Limit(*limit, 5*time.Millisecond, h) // comme dans l'exemple
	}
	mux := http.NewServeMux()
	mux.Handle("/", h)
	mux.HandleFunc("GET /limit", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "%.1f\n", current())
	})
	log.Fatal(http.ListenAndServe("127.0.0.1:8090", mux))
}
```

Le générateur affiche une ligne par seconde. Il servira aussi au défi :

```go
// Fichier : cmd/timeline/main.go
// timeline : une charge en boucle ouverte, par phases, avec réessais ; une ligne par seconde.
package main

import (
	"context"
	"flag"
	"fmt"
	"io"
	"math/rand/v2"
	"net/http"
	"slices"
	"strconv"
	"strings"
	"sync"
	"sync/atomic"
	"time"
)

type phase struct {
	rate int
	d    time.Duration
}

func parse(s string) (ps []phase) {
	for _, f := range strings.Split(s, ",") {
		r, d, _ := strings.Cut(f, ":")
		rate, _ := strconv.Atoi(r)
		dur, _ := time.ParseDuration(d)
		ps = append(ps, phase{rate, dur})
	}
	return ps
}

// budget : un seau à jetons alimenté par les succès (voir overload.Budget).
type budget struct {
	ratio, max float64
	mu         sync.Mutex
	tokens     float64
}

func (b *budget) success() {
	if b == nil {
		return
	}
	b.mu.Lock()
	b.tokens = min(b.tokens+b.ratio, b.max)
	b.mu.Unlock()
}

func (b *budget) allow() bool {
	if b == nil {
		return true
	}
	b.mu.Lock()
	defer b.mu.Unlock()
	if b.tokens < 1 {
		return false
	}
	b.tokens--
	return true
}

type second struct {
	sent, ok, shed int
	lat            []float64
}

func main() {
	url := flag.String("url", "http://127.0.0.1:8090/", "adresse du service")
	phases := flag.String("phases", "1200:30s", "débit:durée, séparés par des virgules")
	retries := flag.Int("retries", 0, "réessais au plus par requête")
	ratio := flag.Float64("budget", 0, "budget de réessais (fraction des succès ; 0 : pas de budget)")
	showLimit := flag.Bool("limitcol", false, "lire et afficher la limite du serveur (GET /limit)")
	flag.Parse()

	var b *budget
	if *ratio > 0 {
		b = &budget{ratio: *ratio, max: 10}
	}
	client := &http.Client{Transport: &http.Transport{MaxIdleConnsPerHost: 4000}}
	var total time.Duration
	for _, p := range parse(*phases) {
		total += p.d
	}
	n := int(total / time.Second)
	secs := make([]second, n)
	attempts := make([]atomic.Int64, n+60)
	var mu sync.Mutex
	var wg sync.WaitGroup
	start := time.Now()

	// La limite du serveur, lue chaque seconde si le service l'expose.
	limits := make([]string, n)
	go func() {
		if !*showLimit {
			return
		}
		for i := range n {
			time.Sleep(time.Until(start.Add(time.Duration(i)*time.Second + 500*time.Millisecond)))
			limits[i] = "-"
			if resp, err := http.Get(strings.TrimSuffix(*url, "/") + "/limit"); err == nil {
				if resp.StatusCode == http.StatusOK {
					body, _ := io.ReadAll(resp.Body)
					limits[i] = strings.TrimSpace(string(body))
				}
				resp.Body.Close()
			}
		}
	}()

	call := func(planned time.Time, sec int) {
		backoff := 50 * time.Millisecond
		for attempt := 0; ; attempt++ {
			if a := int(time.Since(start) / time.Second); a < len(attempts) {
				attempts[a].Add(1)
			}
			ctx, cancel := context.WithTimeout(context.Background(), time.Second)
			req, _ := http.NewRequestWithContext(ctx, "GET", *url, nil)
			resp, err := client.Do(req)
			code := 0
			if err == nil {
				io.Copy(io.Discard, resp.Body)
				resp.Body.Close()
				code = resp.StatusCode
			}
			cancel()
			if code == http.StatusOK {
				b.success()
				d := float64(time.Since(planned).Microseconds()) / 1000
				mu.Lock()
				secs[sec].ok++
				secs[sec].lat = append(secs[sec].lat, d)
				mu.Unlock()
				return
			}
			if code == http.StatusServiceUnavailable && attempt == 0 {
				mu.Lock()
				secs[sec].shed++
				mu.Unlock()
			}
			if attempt == *retries || !b.allow() {
				return
			}
			time.Sleep(backoff/2 + rand.N(backoff))
			backoff *= 2
		}
	}

	elapsed := time.Duration(0)
	for _, p := range parse(*phases) {
		interval := time.Second / time.Duration(p.rate)
		phaseStart := start.Add(elapsed)
		for k := 0; ; k++ {
			planned := phaseStart.Add(time.Duration(k) * interval)
			if planned.Sub(phaseStart) >= p.d {
				break
			}
			time.Sleep(time.Until(planned))
			sec := int(planned.Sub(start) / time.Second)
			mu.Lock()
			secs[sec].sent++
			mu.Unlock()
			wg.Go(func() { call(planned, sec) })
		}
		elapsed += p.d
	}
	wg.Wait()

	head := "   s  demandé  tentatives  succès  refusées au 1er essai   p99 (ms)"
	if *showLimit {
		head += "  limite"
	}
	fmt.Println(head)
	for s := range n {
		x := secs[s]
		slices.Sort(x.lat)
		p99 := 0.0
		if len(x.lat) > 0 {
			p99 = x.lat[int(0.99*float64(len(x.lat)-1))]
		}
		fmt.Printf("%4d  %7d  %10d  %5.0f %%  %21d  %9.1f",
			s, x.sent, attempts[s].Load(), 100*float64(x.ok)/float64(max(x.sent, 1)), x.shed, p99)
		if *showLimit {
			fmt.Printf("  %6s", limits[s])
		}
		fmt.Println()
	}
}
```

```
$ taskset -c 0,1 ./adapt -mode fixed &
$ taskset -c 2-7 ./timeline -phases 2000:30s -limitcol      # quelques secondes choisies
   s  demandé  tentatives  succès  refusées au 1er essai   p99 (ms)  limite
   0     2000        2000     35 %                   1296       20.1     8.0
   5     2000        2000     35 %                   1306       21.3     8.0
   9     2000        2000     52 %                    963       19.1     8.0
  10     2000        2000     69 %                    624       14.3     8.0
  11     2000        2000     67 %                    659       18.8     8.0
  14     2000        2000     66 %                    670       14.6     8.0
  19     2000        2001     42 %                   1170       31.1     8.0
  20     2000        2000     18 %                   1648       33.2     8.0
  21     2000        1999     18 %                   1650       32.1     8.0
  24     2000        2000     17 %                   1652       32.0     8.0
  29     2000        2000     17 %                   1652       33.4     8.0
$ taskset -c 0,1 ./adapt -mode aimd &
$ taskset -c 2-7 ./timeline -phases 2000:30s -limitcol
   s  demandé  tentatives  succès  refusées au 1er essai   p99 (ms)  limite
   0     2000        2000     36 %                   1277       24.4    13.6
   5     2000        2000     35 %                   1304       30.8    14.9
   9     2000        2001     68 %                    650       23.1    13.3
  10     2000        2001    100 %                      0        9.1    64.6
  11     2000        1999    100 %                      0        8.9    90.4
  14     2000        2001    100 %                      0        9.7   142.0
  19     2000        2000     64 %                    713      242.1   200.0
  20     2000        2000     18 %                   1649       53.9     6.9
  21     2000        2000     18 %                   1639       25.1     7.3
  24     2000        1999     18 %                   1633       25.1     7.2
  29     2000        1999     18 %                   1642       28.6     7.1
```

Le serveur démarre une demi-seconde avant le générateur : les changements de pool tombent pendant les
secondes 9 et 19.

| Pool | Capacité (5,3 ms par requête) | Limite fixe à 8 | AIMD |
|---|---|---|---|
| 4 connexions | ≈ 750 req/s | 35 % de succès (700 req/s), p99 20 ms | 36 %, p99 24 ms, limite ≈ 14 |
| 16 connexions | ≈ 3 000 req/s | **67 %** : la limite plafonne le débit vers 1 350 req/s (8 / 5,3 ms ≈ 1 500 en théorie) | **100 %**, p99 9 ms |
| 2 connexions | ≈ 375 req/s | 17 %, p99 33 ms | 18 %, p99 25 ms, limite ≈ 7 |

3. **La limite adaptative suit la capacité** : environ 14 avec 4 connexions, 7 avec 2. Avec 16 connexions, elle
   sert toutes les requêtes, là où la limite fixe en refuse un tiers. Avec un petit pool, les deux font aussi
   bien : la limite fixe avait été réglée pour ce cas.

4. **Les inconvénients**, tous visibles dans le tableau :
   - **La limite dérive quand rien ne sature.** Avec 16 connexions, aucune requête ne dépasse 20 ms, et la
     limite monte sans arrêt jusqu'à son maximum (200). Quand le pool retombe à 2, la seconde 19 subit un p99
     de 242 ms et 36 % d'échecs : la limite de 200 laisse entrer une avalanche, le temps que les baisses de
     10 % la ramènent vers 7. Il faut un maximum raisonnable, ou ne pas augmenter la limite quand elle n'est pas
     atteinte.
   - **`Target` trop bas** (sous la latence à vide, 5,3 ms) : chaque requête est « lente », et la limite tombe
     au minimum. Le service refuse presque tout, même sans charge.
   - **`Target` trop haut** (500 ms) : la limite monte jusqu'à laisser une file d'une demi-seconde. On retrouve
     le service sans protection, avec une latence élevée.
   - **Une requête lente pour une autre raison** (un gros résultat, une pause du GC) fait baisser la limite à
     tort. La baisse est bornée (une fois par `Target`, ×0,9), mais une requête lente sur cent, au débit de
     l'exemple, suffit à empêcher la limite de monter. Les limiteurs réels comparent plutôt une latence récente
     à une latence de référence (la meilleure observée), ou utilisent un percentile plutôt qu'une requête
     isolée.
   - **Plusieurs instances** ajustent chacune leur limite, sans se coordonner, devant la même base. Elles
     convergent ensemble vers un partage, mais pas forcément équitable.

---

## Défi — La panne métastable, en vrai

Le serveur est celui d'E3 (sans `-deadline`), le générateur celui de l'exercice intermédiaire.

1-3. **Sans contrôle d'admission** (lignes choisies) :

```
$ taskset -c 0,1 ./srv &
$ taskset -c 2-7 ./timeline -phases 500:10s,1200:5s,500:30s -retries 3
   s  demandé  tentatives  succès  refusées au 1er essai   p99 (ms)
   0      500         500    100 %                      0        8.5
   8      500         500    100 %                      0        8.2
   9      500         500    100 %                      0        8.2
  10     1201        1200    100 %                      0      708.1
  11     1200        1200     36 %                      0      996.3
  12     1200        1898      0 %                      0        0.0
  13     1200        3001      0 %                      0        0.0
  14     1200        3934      0 %                      0        0.0
  15      500        4101      0 %                      0        0.0
  16      500        3422      0 %                      0        0.0
  17      500        2045      0 %                      0        0.0
  18      500        2058      0 %                      0        0.0
  24      500        2675      0 %                      0        0.0
  34      500        2602      0 %                      0        0.0
  44      500         940      0 %                      0        0.0
```

**Le service ne se rétablit pas.** Trente secondes après la fin de la pointe, le taux de succès est toujours nul.
Avec la loi de Little : la capacité est d'environ 700 req/s. Après la pointe, la file contient plusieurs milliers
de requêtes, soit plusieurs secondes d'attente. Chaque nouvelle tentative attend plus qu'une seconde, échoue, et
en provoque une autre : le flux de tentatives (2 000 à 4 000 par seconde) dépasse la capacité, et `L = λ × W`
ne peut que croître. Le serveur, lui, sert à pleine vitesse des requêtes dont les clients sont partis. Le nombre
de tentatives ne retombe qu'à la fin, quand les dernières requêtes ont épuisé leurs réessais.

4. **Avec le contrôle d'admission, sans puis avec budget :**

```
$ taskset -c 0,1 ./srv -limit 8 &
$ taskset -c 2-7 ./timeline -phases 500:10s,1200:5s,500:30s -retries 3
   s  demandé  tentatives  succès  refusées au 1er essai   p99 (ms)
   0      500         500    100 %                      0        8.2
   8      500         500    100 %                      0        8.2
   9      500         500    100 %                      0       15.3
  10     1201        2878     63 %                    868      490.9
  11     1200        3555     56 %                    956      486.3
  12     1200        3603     57 %                    967      492.4
  13     1200        3560     56 %                    948      499.2
  14     1200        3559     60 %                    951      485.7
  15      500        1160     98 %                    137      440.0
  16      500         500    100 %                      0        8.1
  17      500         500    100 %                      0        8.0
  18      500         500    100 %                      0        8.2
  24      500         500    100 %                      0        8.2
  34      500         500    100 %                      0       15.0
  44      500         500    100 %                      0        9.6
$ taskset -c 2-7 ./timeline -phases 500:10s,1200:5s,500:30s -retries 3 -budget 0.1
   s  demandé  tentatives  succès  refusées au 1er essai   p99 (ms)
   0      500         500    100 %                      0        8.6
   8      500         500    100 %                      0        8.8
   9      500         500    100 %                      0        8.8
  10     1201        1274     58 %                    530       78.6
  11     1200        1269     57 %                    545       91.1
  12     1200        1268     57 %                    555       90.8
  13     1200        1269     57 %                    548       93.4
  14     1200        1268     57 %                    553       91.8
  15      500         505    100 %                      1       12.2
  16      500         500    100 %                      0        9.2
  17      500         500    100 %                      0        8.7
  18      500         500    100 %                      0        8.6
  24      500         500    100 %                      0        9.8
  34      500         500    100 %                      0        8.8
  44      500         500    100 %                      0        9.6
```

- **Les clients réessaient les 503** : le générateur traite un 503 comme tout autre échec. Pendant la pointe, il
  envoie ainsi trois fois plus de tentatives que de requêtes (3 560 par seconde pour 1 200).
- **Le contrôle d'admission suffit à éviter l'effondrement** : le service se rétablit dès la seconde 16. Les
  refus coûtent si peu (5 ms d'attente, aucun accès à la base) que même 3 560 tentatives par seconde ne
  saturent pas le serveur.
- **Le budget divise les tentatives par 2,8** (1 270 par seconde pendant la pointe) et ramène le p99 des requêtes
  réussies de 490 à 90 ms. Sans budget, une partie des succès n'arrive qu'au troisième ou quatrième essai,
  après des centaines de millisecondes d'attente. Le taux de succès est un peu plus bas avec le budget (57 % au
  lieu de 56 à 63 %) : ces succès tardifs ne sont plus tentés.
- Un client bien écrit respecte aussi `Retry-After`, ce que ce générateur ne fait pas.

5. **Quand le budget devient nécessaire : un refus qui coûte.** Le vrai réseau local ne permet pas de rendre le
   refus cher sans changer tout le service (TLS, authentification). Le simulateur le modélise : un refus
   occupe un travailleur pendant 5 ms, un cinquième d'un traitement.

```go
// Fichier : overload/defi_test.go
package overload

import (
	"fmt"
	"testing"
	"time"
)

// Défi, point 5 : quand un refus coûte (ici 5 ms de « processeur », un cinquième d'un traitement),
// les réessais refusés volent de la capacité au travail utile.
func TestCostlyReject(t *testing.T) {
	cfg := Config{Workers: 4, Service: 25 * time.Millisecond, QueueCap: 40, DropExpired: true, RejectCost: 5 * time.Millisecond}
	fmt.Println("défi : refus à 5 ms, file de 40, abandon des requêtes expirées")
	fmt.Print("                          ")
	for w := range 12 {
		fmt.Printf("%4ds", w*5)
	}
	fmt.Println("   tentatives")
	for _, v := range []struct {
		name string
		mk   func(*Server) *Client
	}{
		{"C'. 3 réessais", func(s *Server) *Client {
			return &Client{Server: s, Timeout: time.Second, Retries: 3}
		}},
		{"C'. sans attente", func(s *Server) *Client {
			return &Client{Server: s, Timeout: time.Second, Retries: 3, NoBackoff: true}
		}},
		{"D'. + budget 10 %", func(s *Server) *Client {
			return &Client{Server: s, Timeout: time.Second, Retries: 3, NoBackoff: true, Budget: &Budget{Ratio: 0.1, Max: 10}}
		}},
	} {
		r := run(t, cfg, v.mk)
		fmt.Printf("%-26s", v.name)
		for w := range r.good {
			fmt.Printf("%5.0f", 100*float64(r.good[w])/float64(max(r.total[w], 1)))
		}
		fmt.Printf("  %10d\n", r.attempt)
	}
}
```

```
$ go test -run CostlyReject -v ./overload/
défi : refus à 5 ms, file de 40, abandon des requêtes expirées
                             0s   5s  10s  15s  20s  25s  30s  35s  40s  45s  50s  55s   tentatives
C'. 3 réessais              100  100    6    5   78  100  100  100  100  100  100  100       14781
C'. sans attente            100  100    5    3   77  100  100  100  100  100  100  100       14848
D'. + budget 10 %           100  100   43   99  100  100  100  100  100  100  100  100        8260
```

Sans budget, les réessais refusés volent du temps de travail. Pendant la pointe et la fenêtre qui suit, **5 %
des requêtes seulement réussissent**, et il faut 10 s de plus pour se rétablir. L'attente exponentielle n'y
change presque rien (6 % contre 5 %). Avec le budget, le taux de succès revient à 43 % pendant la pointe, et le
service est sain dès la fenêtre suivante. En pratique, un refus coûte dès qu'il vient **après** une poignée de
main TLS, une vérification de jeton ou la lecture du corps : c'est là que le budget devient indispensable.

---

## Quiz

1. Le débit compte les requêtes traitées, le débit utile celles dont la réponse arrive à temps chez un client qui
   l'attend encore.
2. Un déclencheur passager et un effet d'entretien. Par exemple : les réessais, le cache froid, le basculement
   de charge sur les instances restantes.
3. Le déclencheur a disparu depuis longtemps : c'est l'effet d'entretien qui maintient la surcharge, et le
   système reste dans son état effondré.
4. Elle transforme une surcharge en latence infinie, et tout le travail finit par être gaspillé. On la
   dimensionne par la loi de Little : débit de vidage × attente acceptable (160 req/s × 250 ms = 40).
5. Que l'appelant attend encore : `ctx.Err() == nil`, et qu'il reste assez de temps pour faire le travail.
6. Elle étale les réessais sans réduire leur nombre. Le budget limite les réessais à une fraction des succès :
   quand tout échoue, il n'y a plus de réessais.
7. La file se forme avant le gestionnaire, dans le noyau et dans l'ordonnanceur : le sémaphore n'est jamais plein
   (E4).
8. La requête la plus récente a le plus de chances d'être encore attendue ; la plus ancienne est la plus proche
   de son délai. Le prix : les requêtes du fond de la pile ne sont pas servies.
9. Par un test par paliers en boucle ouverte, jusqu'au dernier palier qui tient l'objectif de latence. On vise
   60 à 70 % de la capacité à la pointe, avec N+1.
10. Le goulot est souvent une ressource partagée, comme la base de données. Plus d'instances, c'est plus de
    connexions vers elle, et donc plus de contention.
