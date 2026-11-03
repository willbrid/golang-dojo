# Leçon 9 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Les exercices E1 à E3 étendent le paquet `deploy` :

```go
// Fichier : internal/deploy/fix.go
package deploy

import (
	"context"
	"errors"
	"net"
	"net/http"
	"time"
)

// Options d'une instance (E1).
type Options struct {
	Work     time.Duration // durée d'une requête
	Warmup   time.Duration // temps de « chargement du cache » au démarrage
	LieReady bool          // /readyz à 200 dès le lancement, même pendant le chargement (la faute)
}

// StartWith démarre une instance qui n'est prête qu'après son chargement. Les requêtes reçues
// pendant le chargement échouent (503) : le cache n'est pas là.
func StartWith(o Options) (*Backend, error) {
	ln, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		return nil, err
	}
	b := &Backend{ln: ln, URL: "http://" + ln.Addr().String()}
	loaded := make(chan struct{})
	go func() { time.Sleep(o.Warmup); close(loaded) }()
	isLoaded := func() bool {
		select {
		case <-loaded:
			return true
		default:
			return false
		}
	}
	mux := http.NewServeMux()
	mux.HandleFunc("GET /livez", func(w http.ResponseWriter, r *http.Request) { w.Write([]byte("ok")) })
	mux.HandleFunc("GET /readyz", func(w http.ResponseWriter, r *http.Request) {
		if b.draining.Load() || (!o.LieReady && !isLoaded()) {
			http.Error(w, "pas prêt", http.StatusServiceUnavailable)
			return
		}
		w.Write([]byte("ok"))
	})
	mux.HandleFunc("GET /work", func(w http.ResponseWriter, r *http.Request) {
		if !isLoaded() {
			http.Error(w, "cache non chargé", http.StatusServiceUnavailable)
			return
		}
		time.Sleep(o.Work)
		w.Write([]byte("ok"))
	})
	b.srv = &http.Server{Handler: mux, ReadHeaderTimeout: 5 * time.Second}
	go b.srv.Serve(ln)
	return b, nil
}

// Ready indique si /readyz répond 200 (ce que verrait le répartiteur).
func (b *Backend) Ready() bool {
	resp, err := http.Get(b.URL + "/readyz")
	if err != nil {
		return false
	}
	resp.Body.Close()
	return resp.StatusCode == http.StatusOK
}

// StopWithin (E3) : arrêt propre, mais jamais plus long que budget. Au-delà, Close coupe ce qui reste.
func (b *Backend) StopWithin(budget, propagation time.Duration) (forced bool, err error) {
	ctx, cancel := context.WithTimeout(context.Background(), budget)
	defer cancel()
	err = b.Stop(ctx, propagation)
	if errors.Is(err, context.DeadlineExceeded) {
		return true, b.srv.Close()
	}
	return false, err
}
```

```go
// Fichier : internal/deploy/fix_test.go
package deploy

import (
	"context"
	"fmt"
	"net"
	"net/http"
	"net/http/httptest"
	"sync"
	"sync/atomic"
	"testing"
	"time"
)

const period = 100 * time.Millisecond

// load lance 8 clients en boucle sur front jusqu'à stop, et compte succès et erreurs.
func load(front string, stop <-chan struct{}) func() (int64, int64) {
	var ok, bad atomic.Int64
	var wg sync.WaitGroup
	c := &http.Client{Timeout: 2 * time.Second}
	for range 8 {
		wg.Go(func() {
			for {
				select {
				case <-stop:
					return
				default:
				}
				resp, err := c.Get(front + "/work")
				if err == nil && resp.StatusCode == http.StatusOK {
					ok.Add(1)
				} else {
					bad.Add(1)
				}
				if err == nil {
					resp.Body.Close()
				}
			}
		})
	}
	return func() (int64, int64) { wg.Wait(); return ok.Load(), bad.Load() }
}

// replace remplace une instance par une nouvelle, sous charge ; la nouvelle est ajoutée tout de suite,
// et l'ancienne arrêtée proprement dès que la nouvelle se DÉCLARE prête.
func replace(t *testing.T, o Options, propagation time.Duration) (int64, int64) {
	lb := NewBalancer(period)
	front := httptest.NewServer(lb)
	defer front.Close()
	old, _ := StartWith(Options{Work: o.Work})
	lb.Add(old.URL)
	time.Sleep(3 * period)
	stop := make(chan struct{})
	wait := load(front.URL, stop)
	time.Sleep(300 * time.Millisecond)
	next, _ := StartWith(o)
	lb.Add(next.URL)
	for !next.Ready() {
		time.Sleep(10 * time.Millisecond)
	}
	time.Sleep(2 * period) // le répartiteur l'a vue prête
	old.Stop(context.Background(), propagation)
	lb.Remove(old.URL)
	time.Sleep(time.Second)
	close(stop)
	defer next.Kill()
	return wait()
}

func TestE1Warmup(t *testing.T) {
	for _, lie := range []bool{false, true} {
		ok, bad := replace(t, Options{Work: 20 * time.Millisecond, Warmup: 500 * time.Millisecond, LieReady: lie}, 3*period)
		fmt.Printf("readyz menteur=%-5v %5d réussies, %4d en erreur\n", lie, ok, bad)
	}
}

func TestE2Propagation(t *testing.T) {
	for _, p := range []time.Duration{0, 50 * time.Millisecond, 100 * time.Millisecond, 150 * time.Millisecond, 300 * time.Millisecond} {
		var errs []int64
		for range 3 {
			_, bad := replace(t, Options{Work: 20 * time.Millisecond}, p)
			errs = append(errs, bad)
		}
		fmt.Printf("attente %-6v erreurs %v\n", p, errs)
	}
}

func TestE3Bounded(t *testing.T) {
	b, _ := Start(time.Millisecond)
	c, err := net.Dial("tcp", b.ln.Addr().String()) // la connexion neuve, jamais utilisée
	if err != nil {
		t.Fatal(err)
	}
	defer c.Close()
	start := time.Now()
	forced, err := b.StopWithin(time.Second, 100*time.Millisecond)
	fmt.Printf("arrêt borné à 1 s : %v (Close forcé : %v, err : %v)\n", time.Since(start).Round(10*time.Millisecond), forced, err)
}
```

```
$ go test -v -run 'TestE' ./internal/deploy/
readyz menteur=false   852 réussies,    0 en erreur
readyz menteur=true    584 réussies, 1456 en erreur
--- PASS: TestE1Warmup (4.75s)
attente 0s     erreurs [36 32 33]
attente 50ms   erreurs [16 15 15]
attente 100ms  erreurs [0 0 0]
attente 150ms  erreurs [0 0 0]
attente 300ms  erreurs [0 0 0]
--- PASS: TestE2Propagation (29.24s)
arrêt borné à 1 s : 1s (Close forcé : true, err : <nil>)
--- PASS: TestE3Bounded (1.00s)
```

## E1 — Prêt seulement quand c'est vrai

- **Avec un `/readyz` honnête**, la nouvelle instance est ajoutée au répartiteur tout de suite, mais elle ne
  reçoit **aucun** trafic pendant son chargement de 500 ms. Le répartiteur la voit non prête. L'ancienne
  n'est arrêtée qu'une fois la nouvelle déclarée prête : **0 erreur**.
- **Avec un `/readyz` qui ment** (200 dès le lancement) : le script croit la nouvelle instance prête, arrête
  l'ancienne aussitôt, et **tout** le trafic part vers une instance dont le cache n'est pas chargé. Elle
  répond 503 à tout pendant près d'une demi-seconde : **1 456 erreurs**.

C'est l'erreur la plus coûteuse de cette leçon. Une sonde de disponibilité qui dit « prêt » trop tôt
transforme chaque déploiement en coupure, et c'est l'orchestrateur lui-même qui la provoque, en suivant
fidèlement la sonde.

## E2 — Combien attendre ?

Avec une période de sonde de 100 ms :
- sans attente, 32 à 36 erreurs ;
- 50 ms d'attente divisent les erreurs par deux ;
- à partir de **100 ms**, plus aucune erreur sur ces trois exécutions.

Le seuil est la **période de sonde** : il faut que le répartiteur ait sondé au moins une fois après le
passage à 503. Mais 100 ms tout juste est fragile :
- la sonde peut être en vol au moment du passage à 503, et elle aura alors vu « prêt » ;
- une sonde peut prendre du retard ;
- un vrai répartiteur doit souvent échouer **plusieurs** sondes avant de retirer une instance (le paramètre
  `failureThreshold` des orchestrateurs).

On prend donc une marge : plusieurs périodes, et le seuil d'échecs × la période, plus une seconde ou deux.
L'exemple du cours prenait 3 périodes.

## E3 — Un arrêt borné

`StopWithin(1 s, 100 ms)` fait l'arrêt propre, mais passe la main à `Close` si `Shutdown` n'a pas fini dans le
budget. Avec la connexion neuve inutilisée, il s'arrête en **1 s** au lieu de 5,3, en signalant `Close
forcé : true`.

Ce qui est perdu en passant à `Close` : les requêtes **encore en cours** au moment du délai. Dans le cas de
la connexion neuve, rien, puisqu'aucune requête n'y est arrivée. Mais `Close` ne fait pas la différence, et
une vraie requête lente serait coupée. Le budget doit donc rester supérieur à la plus longue requête
légitime (leçon 3, E4). `StopWithin` garantit seulement qu'on finit **avant** l'orchestrateur, qui, lui,
enverrait `SIGKILL` sans aucun nettoyage.

## E4 — Les sondes et la base en panne

```go
// Fichier : internal/probes/probes.go
// Package probes : les sondes d'un service qui dépend d'une base de données (E4).
package probes

import (
	"context"
	"net/http"
	"sync/atomic"
	"time"
)

type Pinger interface{ PingContext(context.Context) error }

type Service struct {
	DB       Pinger
	started  atomic.Bool // démarrage terminé (base jointe UNE fois)
	draining atomic.Bool
}

func (s *Service) MarkStarted()  { s.started.Store(true) }
func (s *Service) MarkDraining() { s.draining.Store(true) }

func (s *Service) Handler() http.Handler {
	mux := http.NewServeMux()
	// Vie : rien d'autre que le processus.
	mux.HandleFunc("GET /livez", func(w http.ResponseWriter, r *http.Request) { w.Write([]byte("ok")) })
	// Disponibilité : l'état LOCAL. La base en panne ne retire pas l'instance du répartiteur.
	mux.HandleFunc("GET /readyz", func(w http.ResponseWriter, r *http.Request) {
		if !s.started.Load() || s.draining.Load() {
			http.Error(w, "pas prêt", http.StatusServiceUnavailable)
			return
		}
		w.Write([]byte("ok"))
	})
	mux.HandleFunc("GET /books/{id}", func(w http.ResponseWriter, r *http.Request) {
		ctx, cancel := context.WithTimeout(r.Context(), time.Second)
		defer cancel()
		if err := s.DB.PingContext(ctx); err != nil {
			http.Error(w, "base de données indisponible, réessayer plus tard", http.StatusServiceUnavailable)
			return
		}
		w.Write([]byte("livre"))
	})
	return mux
}
```

```go
// Fichier : internal/probes/probes_test.go
package probes

import (
	"context"
	"errors"
	"net/http"
	"net/http/httptest"
	"testing"
)

type fakeDB struct{ down bool }

func (f *fakeDB) PingContext(context.Context) error {
	if f.down {
		return errors.New("connexion refusée")
	}
	return nil
}

func TestProbes(t *testing.T) {
	for _, tc := range []struct {
		name                      string
		started, draining, dbDown bool
		livez, readyz, books      int
	}{
		{"démarrage", false, false, false, 200, 503, 200},
		{"normal", true, false, false, 200, 200, 200},
		{"base en panne", true, false, true, 200, 200, 503},
		{"arrêt en cours", true, true, false, 200, 503, 200},
	} {
		t.Run(tc.name, func(t *testing.T) {
			s := &Service{DB: &fakeDB{down: tc.dbDown}}
			if tc.started {
				s.MarkStarted()
			}
			if tc.draining {
				s.MarkDraining()
			}
			h := s.Handler()
			for path, want := range map[string]int{"/livez": tc.livez, "/readyz": tc.readyz, "/books/1": tc.books} {
				rec := httptest.NewRecorder()
				h.ServeHTTP(rec, httptest.NewRequest(http.MethodGet, path, nil))
				if rec.Code != want {
					t.Errorf("%s : %d, attendu %d", path, rec.Code, want)
				}
			}
		})
	}
}
```

```
$ go test ./internal/probes/
ok  	example.com/svc/internal/probes	0.004s
```

Avec la base en panne :
- `/livez` répond 200 : l'instance n'est **pas** redémarrée ;
- `/readyz` répond 200 : elle reste dans le répartiteur ;
- la route métier répond un 503 **explicite**.

Le client reçoit « base indisponible, réessayer plus tard », et non un « aucune instance » du répartiteur.

**La version « mauvaise »** (`/livez` qui fait `PingContext`), pendant une panne de 30 s de la base :
- après quelques sondes de vie en échec (3 par défaut dans Kubernetes, toutes les 10 s), **toutes** les
  instances sont tuées et redémarrées, en même temps ;
- au redémarrage, la base est toujours en panne : les instances ne deviennent pas prêtes et ne servent rien,
  même pas le message d'erreur ;
- quand la base revient, toutes les instances ouvrent leurs connexions au même moment, ce qui peut la faire
  retomber.

Une panne de 30 s de la base devient plusieurs minutes de panne totale du service.

## E5 — Les goroutines sans `SIGQUIT`

```go
// Fichier : cmd/goroutines/main.go
// Commande goroutines : résume /debug/pprof/goroutine?debug=2 (état et fonction de départ).
//
//	goroutines http://127.0.0.1:6060/debug/pprof/goroutine?debug=2
package main

import (
	"bufio"
	"cmp"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"regexp"
	"slices"
	"strings"
)

var header = regexp.MustCompile(`^goroutine \d+ (?:gp=\S+ m=\S+ )?\[([^\],]+)`)

func main() {
	resp, err := http.Get(os.Args[1])
	if err != nil {
		log.Fatal(err)
	}
	defer resp.Body.Close()
	for _, l := range Summarize(resp.Body) {
		fmt.Println(l)
	}
}

// Summarize compte les goroutines par (état, fonction de départ « created by »).
func Summarize(r io.Reader) []string {
	counts := map[string]int{}
	sc := bufio.NewScanner(r)
	sc.Buffer(make([]byte, 1<<20), 1<<20)
	state, origin := "", ""
	flush := func() {
		if state != "" {
			counts[fmt.Sprintf("%-16s %s", state, origin)]++
		}
	}
	for sc.Scan() {
		line := sc.Text()
		if m := header.FindStringSubmatch(line); m != nil {
			flush()
			state, origin = m[1], "(main)"
			continue
		}
		if after, ok := strings.CutPrefix(line, "created by "); ok {
			origin, _, _ = strings.Cut(after, " in goroutine")
		}
	}
	flush()
	var out []string
	for k, n := range counts {
		out = append(out, fmt.Sprintf("%4d  %s", n, k))
	}
	slices.SortFunc(out, func(a, b string) int { return cmp.Compare(b, a) })
	return out
}
```

Sur le programme `quit` de l'exemple, avec `import _ "net/http/pprof"` :

```
$ goroutines 'http://127.0.0.1:18099/debug/pprof/goroutine?debug=2'
   3  chan receive     main.main
   1  sleep            (main)
   1  running          net/http.(*Server).Serve
   1  IO wait          net/http.(*connReader).startBackgroundRead
   1  IO wait          main.main
processus toujours vivant
```

- **Les trois goroutines bloquées** sur la réception d'un canal, créées par `main.main`, apparaissent
  regroupées. Avec mille goroutines en fuite (N11 L5), c'est la ligne qui saute aux yeux.
- **La goroutine `running`** est celle qui sert la requête de profil elle-même.
- **Le vidage ne montre pas les goroutines du *runtime*** (GC, finaliseurs), contrairement à `SIGQUIT`
  (12 goroutines). Ce sont celles de l'application qui intéressent.
- **Le processus est toujours vivant.**

`import _ "net/http/pprof"` enregistre les routes sur le `ServeMux` **par défaut** : acceptable ici parce que
ce mux n'est servi que sur l'adresse d'administration. Dans un vrai service, on monte les handlers
explicitement sur le mux d'administration (leçon 8).

---

## Exercice intermédiaire — Le cycle de vie complet

Le cœur de `run`, avec les dépendances injectées. La configuration, le logger et la télémétrie des leçons
5 à 7 se branchent dans `Config` et `Deps`.

```go
// Fichier : internal/lifecycle/run.go
// Package lifecycle : le démarrage et l'arrêt complets d'un service.
package lifecycle

import (
	"context"
	"errors"
	"log/slog"
	"net"
	"net/http"
	"net/http/pprof"
	"sync/atomic"
	"time"
)

type Pinger interface{ PingContext(context.Context) error }

type Config struct {
	Addr, AdminAddr string
	Propagation     time.Duration // attente après readyz → 503 (≥ période de sonde du répartiteur)
	Grace           time.Duration // délai laissé aux requêtes en cours
}

type Deps struct {
	DB       Pinger
	CloseDB  func() error
	Log      *slog.Logger
	OnListen func(public, admin string) // pour les tests : les adresses réelles (port 0)
}

// Run démarre le service et le fait tourner jusqu'à l'annulation de ctx (SIGTERM dans main).
func Run(ctx context.Context, cfg Config, d Deps) error {
	log := d.Log
	var ready atomic.Bool

	// 1. Le démarrage : la base doit répondre avant qu'on se déclare prêt.
	pctx, cancel := context.WithTimeout(ctx, 10*time.Second)
	defer cancel()
	for {
		if err := d.DB.PingContext(pctx); err == nil {
			break
		} else if pctx.Err() != nil {
			return errors.Join(errors.New("base injoignable au démarrage"), err)
		}
		time.Sleep(100 * time.Millisecond)
	}

	admin := http.NewServeMux()
	admin.HandleFunc("/debug/pprof/", pprof.Index)
	public := http.NewServeMux()
	public.HandleFunc("GET /livez", func(w http.ResponseWriter, r *http.Request) { w.Write([]byte("ok")) })
	public.HandleFunc("GET /readyz", func(w http.ResponseWriter, r *http.Request) {
		if !ready.Load() {
			http.Error(w, "pas prêt", http.StatusServiceUnavailable)
			return
		}
		w.Write([]byte("ok"))
	})
	pubLn, err := net.Listen("tcp", cfg.Addr)
	if err != nil {
		return err
	}
	admLn, err := net.Listen("tcp", cfg.AdminAddr)
	if err != nil {
		pubLn.Close()
		return err
	}
	pubSrv := &http.Server{Handler: public, ReadHeaderTimeout: 5 * time.Second}
	admSrv := &http.Server{Handler: admin, ReadHeaderTimeout: 5 * time.Second}
	errc := make(chan error, 2)
	go func() { errc <- pubSrv.Serve(pubLn) }()
	go func() { errc <- admSrv.Serve(admLn) }()
	if d.OnListen != nil {
		d.OnListen(pubLn.Addr().String(), admLn.Addr().String())
	}
	ready.Store(true)
	log.Info("prêt", "addr", pubLn.Addr().String(), "admin", admLn.Addr().String())

	// 2. Attendre l'arrêt (ou la mort d'un serveur).
	select {
	case <-ctx.Done():
	case err := <-errc:
		return err
	}

	// 3. L'arrêt, dans l'ordre, chaque étape mesurée.
	step := func(name string, f func() error) error {
		start := time.Now()
		err := f()
		log.Info("arrêt : "+name, "durée", time.Since(start).Round(time.Millisecond), "err", err)
		return err
	}
	ready.Store(false)
	step("plus prêt, attente de propagation", func() error { time.Sleep(cfg.Propagation); return nil })
	sctx, scancel := context.WithTimeout(context.Background(), cfg.Grace)
	defer scancel()
	errShut := step("serveur public vidé", func() error { return pubSrv.Shutdown(sctx) })
	step("serveur d'administration arrêté", func() error { return admSrv.Shutdown(sctx) })
	errDB := step("base fermée", d.CloseDB)
	log.Info("arrêt terminé")
	return errors.Join(errShut, errDB)
}
```

```go
// Fichier : internal/lifecycle/run_test.go
package lifecycle

import (
	"bytes"
	"context"
	"log/slog"
	"net/http"
	"strings"
	"sync"
	"testing"
	"time"
)

type db struct{ closed bool }

func (*db) PingContext(context.Context) error { return nil }

// syncBuffer : un tampon partagé entre le service et le test, protégé par un verrou.
type syncBuffer struct {
	mu sync.Mutex
	b  bytes.Buffer
}

func (s *syncBuffer) Write(p []byte) (int, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	return s.b.Write(p)
}
func (s *syncBuffer) String() string { s.mu.Lock(); defer s.mu.Unlock(); return s.b.String() }

func TestRunLifecycle(t *testing.T) {
	var logs syncBuffer
	d := &db{}
	addrs := make(chan string, 1)
	ctx, cancel := context.WithCancel(context.Background())
	done := make(chan error, 1)
	cfg := Config{Addr: "127.0.0.1:0", AdminAddr: "127.0.0.1:0", Propagation: 200 * time.Millisecond, Grace: 2 * time.Second}
	go func() {
		done <- Run(ctx, cfg, Deps{
			DB: d, CloseDB: func() error { d.closed = true; return nil },
			Log:      slog.New(slog.NewTextHandler(&logs, nil)),
			OnListen: func(public, _ string) { addrs <- public },
		})
	}()
	url := "http://" + <-addrs
	status := func() int {
		resp, err := http.Get(url + "/readyz")
		if err != nil {
			return 0
		}
		resp.Body.Close()
		return resp.StatusCode
	}
	if got := status(); got != http.StatusOK {
		t.Fatalf("readyz au démarrage : %d", got)
	}

	start := time.Now()
	cancel() // SIGTERM
	time.Sleep(50 * time.Millisecond)
	if got := status(); got != http.StatusServiceUnavailable {
		t.Errorf("readyz pendant la propagation : %d, attendu 503", got)
	}
	if err := <-done; err != nil {
		t.Fatal(err)
	}
	if el := time.Since(start); el > cfg.Propagation+cfg.Grace {
		t.Errorf("arrêt en %v, au-delà du budget", el)
	}
	if !d.closed {
		t.Error("base non fermée")
	}
	out := logs.String()
	order := []string{"prêt", "plus prêt", "serveur public vidé", "serveur d'administration", "base fermée", "arrêt terminé"}
	pos := 0
	for _, s := range order {
		i := strings.Index(out[pos:], s)
		if i < 0 {
			t.Fatalf("étape %q absente ou dans le désordre :\n%s", s, out)
		}
		pos += i
	}
	t.Log("\n" + out)
}
```

```
$ go test -race -v ./internal/lifecycle/
    run_test.go:79:
        level=INFO msg=prêt addr=127.0.0.1:40979 admin=127.0.0.1:34101
        level=INFO msg="arrêt : plus prêt, attente de propagation" durée=200ms err=<nil>
        level=INFO msg="arrêt : serveur public vidé" durée=1ms err=<nil>
        level=INFO msg="arrêt : serveur d'administration arrêté" durée=0s err=<nil>
        level=INFO msg="arrêt : base fermée" durée=0s err=<nil>
        level=INFO msg="arrêt terminé"
--- PASS: TestRunLifecycle (0.20s)
```

- **Le démarrage** : `/readyz` ne passe à 200 qu'après un `PingContext` réussi, avec 10 s au plus. Au-delà,
  `Run` échoue, et le service ne démarre pas sur une base injoignable. **Après** le démarrage, une panne de la
  base ne change plus la disponibilité (E4).
- **`net.Listen` avant `Serve`** : les adresses sont connues avant qu'une requête puisse arriver. Avec le port
  0, le test les reçoit par `OnListen`. Un service ordinaire n'utilise pas ce crochet.
- **L'arrêt** : chaque étape est journalisée avec sa durée. Pendant un incident d'arrêt (« le déploiement
  prend 30 s »), ces lignes disent laquelle traîne.
- **L'ordre** : le serveur public d'abord, qui ne reçoit plus de requêtes, puis l'administration, gardée
  jusqu'au bout pour pouvoir encore observer, puis la base, dont plus aucune requête n'a besoin.
- **Le test** vérifie le comportement **observable** (503 pendant la propagation), l'ordre des journaux, le
  budget de temps et la fermeture de la base. `syncBuffer` est nécessaire, car le service écrit ses journaux
  depuis sa goroutine pendant que le test lit : sans verrou, `-race` le signalerait.

---

## Défi — Le canari et le retour arrière automatique

```go
// Fichier : internal/canary/canary.go
// Package canary : envoyer une part du trafic à la nouvelle version, et la retirer si elle échoue
// plus que l'ancienne.
package canary

import (
	"math/rand/v2"
	"net/http"
	"sync/atomic"
	"time"
)

// Version : un handler et ses compteurs (comme les métriques RED d'une instance).
type Version struct {
	Handler  http.Handler
	ok, errs atomic.Int64
}

func (v *Version) record(status int) {
	if status >= 500 {
		v.errs.Add(1)
	} else {
		v.ok.Add(1)
	}
}

// Counts renvoie le nombre de requêtes et d'erreurs depuis le début.
func (v *Version) Counts() (n, errs int64) { e := v.errs.Load(); return v.ok.Load() + e, e }

// Router envoie une fraction Weight (en millièmes) du trafic au canari.
type Router struct {
	Stable, Canary *Version
	weight         atomic.Int64 // millièmes ; 0 = canari retiré
}

func (r *Router) SetWeight(perMille int64) { r.weight.Store(perMille) }
func (r *Router) Weight() int64            { return r.weight.Load() }

type recorder struct {
	http.ResponseWriter
	status int
}

func (r *recorder) WriteHeader(c int) { r.status = c; r.ResponseWriter.WriteHeader(c) }

func (r *Router) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	v := r.Stable
	if rand.Int64N(1000) < r.weight.Load() {
		v = r.Canary
	}
	rec := &recorder{ResponseWriter: w, status: http.StatusOK}
	v.Handler.ServeHTTP(rec, req)
	v.record(rec.status)
}

// Rule : retirer le canari quand il a servi au moins MinRequests requêtes et que son taux d'erreurs
// dépasse celui de la version stable de plus de MaxDelta.
type Rule struct {
	MinRequests int64
	MaxDelta    float64
}

func rate(n, e int64) float64 {
	if n == 0 {
		return 0
	}
	return float64(e) / float64(n)
}

// Watch vérifie la règle toutes les every, jusqu'à stop ; il renvoie le délai avant le retrait,
// ou 0 si le canari n'a pas été retiré.
func Watch(r *Router, rule Rule, every time.Duration, stop <-chan struct{}) time.Duration {
	start := time.Now()
	t := time.NewTicker(every)
	defer t.Stop()
	for {
		select {
		case <-stop:
			return 0
		case <-t.C:
			cn, ce := r.Canary.Counts()
			sn, se := r.Stable.Counts()
			if cn >= rule.MinRequests && rate(cn, ce)-rate(sn, se) > rule.MaxDelta {
				r.SetWeight(0) // retour arrière : plus aucun trafic vers le canari
				return time.Since(start)
			}
		}
	}
}

// Faulty renvoie un handler qui échoue (500) avec la probabilité p.
func Faulty(p float64) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if rand.Float64() < p {
			http.Error(w, "panne", http.StatusInternalServerError)
			return
		}
		w.Write([]byte("ok"))
	})
}
```

```go
// Fichier : internal/canary/canary_test.go
package canary

import (
	"fmt"
	"net/http"
	"net/http/httptest"
	"sync"
	"testing"
	"time"
)

var rule = Rule{MinRequests: 300, MaxDelta: 0.025}

// run envoie du trafic pendant d, avec un canari de poids w, et la surveillance si watch.
func run(stableP, canaryP float64, w int64, d time.Duration, watch bool) (detect time.Duration, clientErrs, total int64) {
	r := &Router{Stable: &Version{Handler: Faulty(stableP)}, Canary: &Version{Handler: Faulty(canaryP)}}
	r.SetWeight(w)
	stop := make(chan struct{})
	res := make(chan time.Duration, 1)
	if watch {
		go func() { res <- Watch(r, rule, 100*time.Millisecond, stop) }()
	}
	var wg sync.WaitGroup
	for range 8 {
		wg.Go(func() {
			req := httptest.NewRequest(http.MethodGet, "/", nil)
			for {
				select {
				case <-stop:
					return
				default:
				}
				r.ServeHTTP(httptest.NewRecorder(), req)
				time.Sleep(time.Millisecond) // environ 8 000 requêtes par seconde au total
			}
		})
	}
	time.Sleep(d)
	close(stop)
	wg.Wait()
	if watch {
		detect = <-res
	}
	sn, se := r.Stable.Counts()
	cn, ce := r.Canary.Counts()
	return detect, se + ce, sn + cn
}

func TestCanaryDetects(t *testing.T) {
	for i := range 5 {
		d, errs, total := run(0.01, 0.06, 100, 2*time.Second, true)
		fmt.Printf("essai %d : canari retiré au bout de %v ; erreurs vues %d / %d requêtes\n", i+1, d.Round(10*time.Millisecond), errs, total)
		if d == 0 {
			t.Error("canari défectueux non détecté")
		}
	}
}

func TestCanaryFalseAlarms(t *testing.T) {
	alarms := 0
	for range 20 {
		if d, _, _ := run(0.01, 0.01, 100, 2*time.Second, true); d != 0 {
			alarms++
		}
	}
	fmt.Printf("fausses alertes : %d sur 20 essais d'un canari sain\n", alarms)
}

func TestWithoutCanary(t *testing.T) {
	// La même version défectueuse déployée à 100 %, pendant la même durée, sans surveillance.
	_, errs, total := run(0.06, 0.06, 0, 2*time.Second, false)
	_, base, total2 := run(0.01, 0.01, 0, 2*time.Second, false)
	fmt.Printf("sans canari : %d erreurs / %d requêtes (référence saine : %d / %d)\n", errs, total, base, total2)
}
```

```
$ go test -v ./internal/canary/
essai 1 : canari retiré au bout de 500ms ; erreurs vues 158 / 13275 requêtes
essai 2 : canari retiré au bout de 500ms ; erreurs vues 160 / 13203 requêtes
essai 3 : canari retiré au bout de 500ms ; erreurs vues 152 / 13324 requêtes
essai 4 : canari retiré au bout de 500ms ; erreurs vues 155 / 13052 requêtes
essai 5 : canari retiré au bout de 500ms ; erreurs vues 145 / 13383 requêtes
--- PASS: TestCanaryDetects (10.01s)
fausses alertes : 0 sur 20 essais d'un canari sain
--- PASS: TestCanaryFalseAlarms (40.03s)
sans canari : 772 erreurs / 13042 requêtes (référence saine : 145 / 13253)
--- PASS: TestWithoutCanary (4.00s)
```

Pour rendre les fausses alertes **possibles**, la version stable elle aussi échoue à 1 % des requêtes. Sinon,
un canari sain (0 % d'erreurs) ne pourrait jamais dépasser la version stable.

**La règle.**

- On retire le canari après **300 requêtes** au moins, s'il échoue **2,5 points** de plus que la version
  stable.
- Avec 300 requêtes et un vrai taux de 6 %, on attend 18 erreurs. Un canari **sain** à 1 % en donne 3 en
  moyenne. Il lui faudrait 11 erreurs ou plus (3,5 %) pour déclencher, ce qui est très improbable : 0 fausse
  alerte sur 20 essais. Mesurer est la seule façon de le vérifier pour **ses** taux.
- Le minimum de requêtes est essentiel. Sur 20 requêtes, une seule erreur vaut 5 % : on déclencherait au
  hasard.

**Les mesures.**

- **Le canari défectueux est retiré en 500 ms**, aux cinq essais : le temps d'atteindre 300 requêtes à
  10 % du trafic, puis la vérification suivante.
- **Les erreurs vues par les clients** : environ 155 sur 13 200, alors que la référence saine en donne
  environ 145. Le canari n'a causé qu'**une vingtaine d'erreurs**.
- **Sans canari**, la même version défectueuse déployée à 100 % pendant les mêmes 2 s donne 772 erreurs, soit
  plus de 600 de trop, et elle n'aurait **jamais** été retirée automatiquement. Le canari divise
  l'exposition par 30 environ.

**Les limites de cette version.**

- **Les compteurs sont cumulés depuis le début.** Un vrai contrôleur compare des **taux sur une fenêtre**
  glissante, calculés par Prometheus avec `rate(…[1m])` (leçon 7), pour qu'un incident ancien ne pèse pas
  indéfiniment.
- **Un seul indicateur.** On compare aussi la latence (p99), et pas seulement les erreurs.
- **Le hasard.** Un taux d'erreurs très faible (0,1 %) demande des milliers de requêtes avant de conclure :
  le canari doit alors durer plus longtemps, ou recevoir plus de trafic.

---

## Quiz

1. Remplacer les instances une par une, chaque nouvelle étant prête avant l'arrêt d'une ancienne. Deux
   versions tournent en même temps : le schéma doit convenir aux deux (étendre, puis contracter, N8 L6).
2. La vie dit « je fonctionne » : si elle échoue, l'instance est redémarrée. La disponibilité dit
   « envoyez-moi du trafic » : si elle échoue, l'instance est retirée du répartiteur sans être tuée.
3. Une panne de la dépendance ferait redémarrer **toutes** les instances ensemble : la panne d'une
   dépendance devient une panne totale, et une tempête de reconnexions au retour.
4. Ne plus être prêt, attendre que le répartiteur l'ait vu, `Shutdown` avec un délai, fermer les ressources.
   `Shutdown` seul ferme aussitôt l'écoute, alors que le répartiteur envoie encore du trafic : 13 à 38
   erreurs mesurées.
5. Attente de propagation + plus longue requête légitime + fermeture < délai de l'orchestrateur (30 s dans
   Kubernetes par défaut, 10 s pour `docker stop`).
6. `Shutdown` attend jusqu'à 5 s une connexion ouverte sur laquelle aucune requête n'est encore arrivée
   (`StateNew`). On borne l'arrêt (`Shutdown` avec délai, puis `Close`), ou on compte ces 5 s dans le budget.
7. L'image précédente, désignée par une étiquette immuable ou son empreinte. Un schéma de base encore
   compatible avec elle. Un déploiement automatisé. Idéalement, des drapeaux pour désactiver sans
   redéployer.
8. Envoyer d'abord une petite part du trafic à la nouvelle version, comparer ses métriques à celles de
   l'ancienne, et la retirer automatiquement si elle est moins bonne. Mesuré : 20 erreurs au lieu de 600.
9. Par `/debug/pprof/goroutine?debug=2`, sur l'adresse d'administration, par un tunnel. `SIGQUIT` affiche les
   goroutines, mais arrête le processus.
10. L'instance ne correspondrait plus à aucune version du dépôt, la correction serait perdue au prochain
    déploiement, et personne ne pourrait la reproduire. On corrige par le dépôt, la CI et un déploiement.
