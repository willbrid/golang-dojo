# Leçon 7 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Les simulations sont déterministes : tu dois retrouver exactement ces chiffres. Les mesures sur un vrai
serveur, elles, varient d'une exécution à l'autre. Le code des exercices faciles, ajouté au paquet `lat` :

```go
// Fichier : fix.go
package lat

import (
	"math/rand/v2"
	"net/http"
	"slices"
	"sync"
	"time"
)

// E3 : Simulate avec une loi de service quelconque (service renvoie une durée tirée au hasard).
func SimulateWith(seed uint64, lambda float64, c, n int, shared bool, service func(*rand.Rand) float64) Sim {
	rng := rand.New(rand.NewPCG(seed, 1))
	free := make([]float64, c)
	s := Sim{Arrive: make([]float64, n), Depart: make([]float64, n)}
	t := 0.0
	for i := range n {
		t += rng.ExpFloat64() / lambda
		w := rng.IntN(c)
		if shared {
			w = slices.Index(free, slices.Min(free))
		}
		start := max(t, free[w])
		free[w] = start + service(rng)
		s.Arrive[i], s.Depart[i] = t, free[w]
	}
	return s
}

// E4 : boucle fermée « au calendrier » : chaque worker vise une requête toutes les every, et mesure
// depuis l'instant prévu. S'il est en retard, il part aussitôt (sans rattraper les requêtes manquées).
func ClosedPaced(c *http.Client, url string, workers int, every, d time.Duration) []float64 {
	var mu sync.Mutex
	var all []float64
	start := time.Now()
	var wg sync.WaitGroup
	for w := range workers {
		wg.Go(func() {
			var local []float64
			planned := start.Add(time.Duration(w) * every / time.Duration(workers))
			for planned.Before(start.Add(d)) {
				time.Sleep(time.Until(planned))
				if get(c, url) == nil {
					local = append(local, ms(time.Since(planned)))
				}
				planned = planned.Add(every)
			}
			mu.Lock()
			all = append(all, local...)
			mu.Unlock()
		})
	}
	wg.Wait()
	slices.Sort(all)
	return all
}
```

```go
// Fichier : fix_test.go
package lat

import (
	"fmt"
	"math/rand/v2"
	"net/http"
	"net/http/httptest"
	"slices"
	"testing"
	"time"
)

// E1 : la page attend le plus lent de n services.
func TestFanOut(t *testing.T) {
	one := Simulate(1, 0.7, 1, 1, 1_000_000, true).Sojourns()
	rng := rand.New(rand.NewPCG(3, 4))
	fmt.Printf("un service : p50 %.1f ms, p99 %.1f ms\n", Percentile(one, 50), Percentile(one, 99))
	for _, n := range []int{1, 10, 100} {
		page := make([]float64, 100_000)
		for i := range page {
			for range n {
				page[i] = max(page[i], one[rng.IntN(len(one))])
			}
		}
		slices.Sort(page)
		fmt.Printf("n = %3d : page p50 %5.1f ms, p99 %5.1f ms\n", n, Percentile(page, 50), Percentile(page, 99))
	}
}

// E2 : 2 requêtes par ms, 5 ms de connexion en moyenne : combien de connexions ?
func TestPoolSize(t *testing.T) {
	for c := 11; c <= 20; c++ {
		s := Simulate(1, 2, 0.2, c, 500_000, true).Sojourns()
		fmt.Printf("c = %2d : occupation %3.0f %%  moyenne %5.1f ms  p99 %6.1f ms\n",
			c, 100*2/(0.2*float64(c)), Mean(s), Percentile(s, 99))
	}
}

// E3 : service constant ou exponentiel, même moyenne (1 ms), ρ = 0,9.
func TestVariability(t *testing.T) {
	for _, tc := range []struct {
		name    string
		service func(*rand.Rand) float64
	}{
		{"constant", func(*rand.Rand) float64 { return 1 }},
		{"exponentiel", func(r *rand.Rand) float64 { return r.ExpFloat64() }},
		{"1 % de requêtes ×100", func(r *rand.Rand) float64 {
			if r.IntN(100) == 0 {
				return 100 / 1.99 // moyenne totale toujours 1 ms
			}
			return 1 / 1.99
		}},
	} {
		s := SimulateWith(1, 0.9, 1, 2_000_000, true, tc.service).Sojourns()
		fmt.Printf("%-22s attente moyenne %6.1f ms  séjour p99 %6.1f ms\n", tc.name, Mean(s)-1, Percentile(s, 99))
	}
}

// E4 : la boucle fermée au calendrier, face à la même panne.
func TestClosedPaced(t *testing.T) {
	const dur = 3 * time.Second
	c := &http.Client{Transport: &http.Transport{MaxIdleConnsPerHost: 1000}}
	srv := httptest.NewServer(StallHandler(time.Now(), time.Second, 500*time.Millisecond))
	defer srv.Close()
	report("fermée+calendrier", ClosedPaced(c, srv.URL, 8, 8*time.Millisecond, dur), dur)
}

// E5 : 9 serveurs à 50 %, un à 95 %.
func TestPercentileOfPercentiles(t *testing.T) {
	var all []float64
	sum := 0.0
	for i := range 10 {
		rho := 0.5
		if i == 9 {
			rho = 0.95
		}
		s := Simulate(uint64(i), rho, 1, 1, 200_000, true).Sojourns()
		sum += Percentile(s, 99)
		all = append(all, s...)
	}
	slices.Sort(all)
	fmt.Printf("moyenne des 10 p99 : %.1f ms ; p99 global : %.1f ms\n", sum/10, Percentile(all, 99))
}
```

```
$ go test -v -run 'FanOut|PoolSize|Variability|ClosedPaced|PercentileOf'
un service : p50 2.3 ms, p99 15.7 ms
n =   1 : page p50   2.3 ms, p99  15.5 ms
n =  10 : page p50   9.1 ms, p99  23.8 ms
n = 100 : page p50  17.0 ms, p99  31.2 ms
c = 11 : occupation  91 %  moyenne   8.3 ms  p99   30.2 ms
c = 12 : occupation  83 %  moyenne   6.1 ms  p99   24.9 ms
c = 13 : occupation  77 %  moyenne   5.5 ms  p99   23.7 ms
c = 14 : occupation  71 %  moyenne   5.2 ms  p99   23.3 ms
c = 15 : occupation  67 %  moyenne   5.1 ms  p99   23.1 ms
c = 16 : occupation  62 %  moyenne   5.0 ms  p99   23.1 ms
c = 17 : occupation  59 %  moyenne   5.0 ms  p99   23.0 ms
c = 18 : occupation  56 %  moyenne   5.0 ms  p99   23.0 ms
c = 19 : occupation  53 %  moyenne   5.0 ms  p99   23.0 ms
c = 20 : occupation  50 %  moyenne   5.0 ms  p99   23.0 ms
constant               attente moyenne    4.5 ms  séjour p99   23.5 ms
exponentiel            attente moyenne    8.9 ms  séjour p99   46.4 ms
1 % de requêtes ×100   attente moyenne  112.5 ms  séjour p99  542.1 ms
fermée+calendrier  3000 req. (1000/s)  moyenne  55.7  p50  2.3  p99 479.3  p99,9 500.2  max 502.5 (ms)
moyenne des 10 p99 : 15.7 ms ; p99 global : 44.1 ms
```

## E1 — L'amplification par la queue

Le p50 d'une page qui appelle 10 services (9,1 ms) vaut déjà 4 fois le p50 d'un service (2,3 ms). Avec 100
services, le p50 de la page (17,0 ms) **dépasse le p99** d'un service seul (15,7 ms). La requête typique de la
page est aussi lente que la pire requête sur cent d'un service.

C'est pourquoi les systèmes très découpés surveillent le p99, voire le p99,9, de chaque composant. On
utilise aussi des parades propres à l'appel multiple : envoyer une seconde requête si la première tarde
(*hedging*), ou répondre sans les services les plus lents, en mode dégradé.

## E2 — Little pour dimensionner

(a) L = λ × W = 2 000 × 0,005 = **10 connexions** occupées en moyenne.

(b) **Aucun** nombre de connexions ne donne un p99 sous 20 ms : le p99 plafonne à 23,0 ms, même avec 20
connexions. Ce plancher, c'est le **service lui-même**. Une durée exponentielle de moyenne 5 ms a un p99
de 5 × ln(100) ≈ 23 ms. Ajouter des connexions supprime l'**attente**, pas la durée du travail.

La question piège a donc une réponse utile : avant de dimensionner, vérifier que l'objectif est atteignable
**sans aucune file**. Ici, il faudrait raccourcir les requêtes elles-mêmes, ou fixer un objectif réaliste.

Avec un objectif de p99 sous 25 ms, il faut c = 12, soit 83 % d'occupation par connexion. Vers c = 14 (71 %),
on est à 1 % du plancher : au-delà, des connexions de plus ne servent plus à rien.

Les 10 connexions de Little sont le **minimum** absolu. Avec c = 10, le système est à 100 % et la file
grandit sans fin.

## E3 — La variabilité

Durées constantes : 4,5 ms d'attente. Exponentielles : 8,9 ms. Le rapport est de **½**, à moyenne de service
égale. La théorie le prévoit. Pour une file à un worker et des arrivées aléatoires, l'attente moyenne vaut
λ × E[S²] / (2(1 − ρ)), où E[S²] est la moyenne des carrés des durées de service. L'attente est donc
proportionnelle à E[S²], qui vaut S² pour des durées constantes et 2S² pour des durées exponentielles.

Le troisième cas a une moyenne toujours égale à 1 ms : 1 % de requêtes de 50 ms et 99 % de requêtes de
0,5 ms. Il donne **112,5 ms** d'attente moyenne et un p99 de 542 ms. La formule prévoit
0,9 × 25,5 / 0,2 ≈ 115 ms. Quelques requêtes très longues font attendre **toutes** les autres.

D'où une règle pratique : **séparer les requêtes longues** (exports, rapports) des requêtes courtes, sur des
workers ou des pools distincts. Les requêtes longues ont alors leur propre file et n'empoisonnent plus celle
des autres.

## E4 — Corriger la boucle fermée

Avec un calendrier et une mesure depuis l'instant prévu, le p99 passe de 3 ms à **479 ms**. La mesure se
rapproche beaucoup de la boucle ouverte (488 ms). Ce n'est pourtant pas la même expérience.

- **Pendant la panne, le serveur n'a reçu que 8 requêtes**, une par worker bloqué, au lieu de 500. Le
  calendrier ne fait que **noter** le retard.
- **À la fin de la panne**, chaque worker envoie ses requêtes en retard **à la suite** : une rafale que de
  vrais clients n'auraient pas produite sous cette forme. D'où une moyenne différente (55,7 ms contre
  46,5 ms).

Corriger la mesure rend les **chiffres** honnêtes. Seule une vraie boucle ouverte soumet le serveur à la
**charge** réelle d'une panne : des centaines de connexions simultanées, la mémoire qu'elles occupent, les
délais d'attente qui expirent. Les deux corrections sont utiles. Seule la seconde teste le comportement du
serveur.

## E5 — Les percentiles ne se moyennent pas

La moyenne des 10 p99 donne 15,7 ms ; le p99 de l'ensemble des durées, **44,1 ms**. Le premier chiffre laisse
croire que tout va bien.

Le second dit qu'une requête sur cent attend 44 ms. Les plus lentes viennent presque toutes du serveur
chargé, et il ne pèse que pour un dixième dans la moyenne des p99.

Pour agréger, il faut agréger les **données** : des histogrammes aux mêmes bornes, qu'on additionne (N10 L10),
puis calculer le percentile sur le total.

---

## Exercice intermédiaire — Un test de charge par paliers

```go
// Fichier : paliers.go
package lat

import (
	"io"
	"math/rand/v2"
	"net/http"
	"slices"
	"sync"
	"sync/atomic"
	"time"
)

// Limited simule un service de capacité limitée : 4 traitements simultanés au plus, de 2 ms en
// moyenne (durée exponentielle). maxWaiting > 0 active le contrôle d'admission : au-delà de
// maxWaiting requêtes en attente d'une place, réponse 503 immédiate.
type Limited struct {
	sem        chan struct{}
	waiting    atomic.Int64
	maxWaiting int64
}

func NewLimited(maxWaiting int64) *Limited {
	return &Limited{sem: make(chan struct{}, 4), maxWaiting: maxWaiting}
}

func (l *Limited) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if n := l.waiting.Add(1); l.maxWaiting > 0 && n > l.maxWaiting {
		l.waiting.Add(-1)
		http.Error(w, "surcharge", http.StatusServiceUnavailable)
		return
	}
	l.sem <- struct{}{}
	l.waiting.Add(-1)
	time.Sleep(time.Duration(rand.ExpFloat64() * float64(2*time.Millisecond)))
	<-l.sem
	io.WriteString(w, "ok")
}

// Stage est le résultat d'un palier.
type Stage struct {
	OK       []float64 // durées des réponses 200, triées (ms)
	Rejected int       // réponses 503
}

// OpenStatus : comme Open, en distinguant les réponses acceptées et refusées.
func OpenStatus(c *http.Client, url string, rate float64, d time.Duration) Stage {
	interval := time.Duration(float64(time.Second) / rate)
	n := int(d / interval)
	var mu sync.Mutex
	var st Stage
	start := time.Now()
	var wg sync.WaitGroup
	for i := range n {
		planned := start.Add(time.Duration(i) * interval)
		time.Sleep(time.Until(planned))
		wg.Go(func() {
			resp, err := c.Get(url)
			if err != nil {
				return
			}
			io.Copy(io.Discard, resp.Body)
			resp.Body.Close()
			el := ms(time.Since(planned))
			mu.Lock()
			if resp.StatusCode == http.StatusOK {
				st.OK = append(st.OK, el)
			} else {
				st.Rejected++
			}
			mu.Unlock()
		})
	}
	wg.Wait()
	slices.Sort(st.OK)
	return st
}
```

```go
// Fichier : paliers_test.go
package lat

import (
	"fmt"
	"net/http"
	"net/http/httptest"
	"os"
	"testing"
	"time"
)

// Long (≈ 2 min) : lancé seulement avec PALIERS=1.
func TestStages(t *testing.T) {
	if os.Getenv("PALIERS") == "" {
		t.Skip("PALIERS=1 pour lancer le test de charge par paliers")
	}
	const dur = 10 * time.Second
	c := &http.Client{Transport: &http.Transport{MaxIdleConnsPerHost: 4000}}
	for _, admission := range []int64{0, 16} {
		for _, pct := range []float64{50, 70, 90, 100, 110} {
			if admission > 0 && pct < 90 {
				continue
			}
			srv := httptest.NewServer(NewLimited(admission))
			st := OpenStatus(c, srv.URL, 2000*pct/100, dur)
			srv.Close()
			sim := Simulate(1, 2*pct/100, 0.5, 4, 500_000, true).Sojourns() // instable si pct >= 100
			fmt.Printf("admission=%-2d %3.0f %% : %4.0f ok/s, refus %4.1f %%, p50 %6.1f  p99 %7.1f  L %5.1f | simulé p50 %4.1f p99 %5.1f (ms)\n",
				admission, pct, float64(len(st.OK))/dur.Seconds(),
				100*float64(st.Rejected)/float64(st.Rejected+len(st.OK)),
				Percentile(st.OK, 50), Percentile(st.OK, 99),
				float64(len(st.OK))/dur.Seconds()*Mean(st.OK)/1000,
				Percentile(sim, 50), Percentile(sim, 99))
		}
	}
}
```

```
$ PALIERS=1 go test -v -run Stages -timeout 10m
admission=0   50 % : 1000 ok/s, refus  0.0 %, p50    3.0  p99    10.8  L   3.6 | simulé p50  1.6 p99   9.6 (ms)
admission=0   70 % : 1400 ok/s, refus  0.0 %, p50    4.5  p99    14.5  L   7.2 | simulé p50  2.1 p99  11.2 (ms)
admission=0   90 % : 1800 ok/s, refus  0.0 %, p50  705.2  p99  1524.8  L 1322.2 | simulé p50  4.4 p99  23.4 (ms)
admission=0  100 % : 2000 ok/s, refus  0.0 %, p50 1343.1  p99  2749.1  L 2682.1 | simulé p50 154.1 p99 348.7 (ms)
admission=0  110 % : 2200 ok/s, refus  0.0 %, p50 2034.2  p99  3991.0  L 4417.4 | simulé p50 10729.7 p99 21627.8 (ms)
admission=16  90 % : 1554 ok/s, refus 13.7 %, p50   12.0  p99    23.2  L  19.2 | simulé p50  4.4 p99  23.4 (ms)
admission=16 100 % : 1598 ok/s, refus 20.1 %, p50   12.2  p99    21.4  L  19.9 | simulé p50 154.1 p99 348.7 (ms)
admission=16 110 % : 1594 ok/s, refus 27.5 %, p50   12.4  p99    21.9  L  20.3 | simulé p50 10729.7 p99 21627.8 (ms)
```

Les paliers durent 10 s au lieu de 30 pour garder un test de 90 s. Les colonnes « simulé » sont instables à
100 % et 110 % : la file simulée grandit sans fin, et ses chiffres dépendent de la durée de la simulation.

**1. Le service n'a pas la capacité annoncée.** À « 90 % » (1 800 requêtes/s), la latence explose déjà :
p50 de 705 ms. La simulation, elle, prévoit 4,4 ms. Le contrôle d'admission révèle le vrai plafond : le débit
accepté sature vers **1 600 requêtes/s**, et non 2 000. On a donc mesuré `time.Sleep` :

```go
// Fichier : sleep_test.go
package lat

import (
	"math/rand/v2"
	"testing"
	"time"
)

// Que dure vraiment time.Sleep(d) pour d exponentielle de moyenne 2 ms ?
func TestSleepOverhead(t *testing.T) {
	var asked, got time.Duration
	for range 2000 {
		d := time.Duration(rand.ExpFloat64() * float64(2*time.Millisecond))
		t0 := time.Now()
		time.Sleep(d)
		asked += d
		got += time.Since(t0)
	}
	t.Logf("demandé en moyenne %v, obtenu %v", asked/2000, got/2000)
}
```

```
sleep_test.go:19: demandé en moyenne 1.97613ms, obtenu 2.594899ms
```

Un `time.Sleep` de 2 ms dure en moyenne 2,6 ms sur cette machine. Le réveil d'une goroutine endormie a un
coût. La capacité réelle est donc d'environ 4 / 2,6 ms ≈ 1 540 requêtes/s. Les paliers « 90 % », « 100 % » et
« 110 % » sont tous **au-dessus** de la capacité, d'où l'explosion. C'est le résultat le plus important de
l'exercice : **la capacité se mesure, elle ne se calcule pas**. Une hypothèse fausse de 25 % sur la durée de
service suffit à transformer « 90 % de charge » en surcharge.

**Débits affichés sans admission.** Les colonnes « ok/s » à 90–110 % affichent le débit **envoyé** (1 800,
2 000, 2 200), pas le débit traité. Le test attend la fin de toutes les requêtes, et la file se vide bien après
les 10 s de charge : on a divisé par 10 s un travail qui en a pris davantage.

**2. Mesure contre simulation.**

- À 50 et 70 %, le p99 mesuré (10,8 et 14,5 ms) reste proche du p99 simulé (9,6 et 11,2 ms).
- Le p50 mesuré est environ le double du p50 simulé. Le service réel dure plus longtemps, comme on vient de
  le voir, et chaque requête paie en plus le passage par HTTP en local. Dans l'exemple du cours, un traitement de 1 ms donnait un p50 de 1,6 à 2 ms, ce qui donne l'ordre de grandeur de ce surcoût.
- La loi de Little tient sur la mesure elle-même : à 50 %, 1 000/s × 3,6 ms ≈ 3,6 requêtes en cours.

**3. Le contrôle d'admission.** À « 110 % », 27,5 % des requêtes sont refusées tout de suite. Les requêtes
acceptées gardent un p99 de **22 ms**, contre 4 s sans admission.

Le nombre moyen de requêtes en cours reste borné vers 20 : 4 en traitement et au plus 16 en attente. Little
donne alors la latence maximale : 20 / 1 600 par seconde ≈ 12,5 ms en moyenne, ce qui est mesuré (p50 de
12,4 ms).

**4. Conclusion possible.**

> Capacité utile mesurée : environ 1 550 requêtes/s, pas 2 000. Viser au plus 1 100 requêtes/s par instance
> aux heures de pointe (70 %), où le p99 reste sous 15 ms. Contrôle d'admission à 16 requêtes en attente : en
> surcharge, les requêtes acceptées gardent un p99 d'environ 22 ms et les autres reçoivent un 503 immédiat
> avec `Retry-After`. Ajouter des instances quand le taux de refus dépasse 1 %. Refaire la mesure à chaque
> changement notable du traitement.

---

## Défi — L'effondrement par les relances

La simulation n'a qu'un worker FIFO, mais elle doit gérer des **relances** qui arrivent plus tard. Les
arrivées sont donc rangées dans un tas (`container/heap`), qui rend toujours la plus proche dans le temps.

```go
// Fichier : retry/retry.go
// Package retry simule l'effondrement d'un service par les relances de ses clients.
package retry

import (
	"container/heap"
	"math/rand/v2"
)

// Config décrit le scénario. Les durées sont en millisecondes.
type Config struct {
	Lambda           float64 // arrivées nouvelles par ms, hors pic
	PeakLambda       float64 // arrivées pendant le pic
	PeakFrom, PeakTo float64
	Horizon          float64 // durée simulée
	Timeout          float64 // délai du client
	DropStale        bool    // remède 1 : le serveur jette ce qui a dépassé le délai
	MaxAttempts      int     // remède 2 : nombre d'essais au plus (0 = illimité)
	Backoff          float64 // remède 2 : attente avant relance, doublée à chaque essai (avec gigue)
}

// Result donne les compteurs (Useful : réponses reçues dans le délai ; Accepted : requêtes traitées
// ou à traiter ; Dropped : jetées sans traitement ; GaveUp : clients qui ont renoncé) et la longueur de la file (en ms de travail) dans le temps.
type Result struct {
	Useful, Accepted, Dropped, GaveUp int
	Backlog                           []float64 // travail en attente, échantillonné toutes les 100 ms
}

type arrival struct {
	t       float64
	attempt int
}

type queue []arrival

func (q queue) Len() int           { return len(q) }
func (q queue) Less(i, j int) bool { return q[i].t < q[j].t }
func (q queue) Swap(i, j int)      { q[i], q[j] = q[j], q[i] }
func (q *queue) Push(x any)        { *q = append(*q, x.(arrival)) }
func (q *queue) Pop() any {
	old := *q
	x := old[len(old)-1]
	*q = old[:len(old)-1]
	return x
}

// Run simule un worker FIFO, service exponentiel de moyenne 1 ms.
func Run(cfg Config, seed uint64) Result {
	rng := rand.New(rand.NewPCG(seed, 1))
	var q queue
	// Les arrivées nouvelles, tirées d'avance (avec le pic).
	for t := 0.0; ; {
		l := cfg.Lambda
		if t >= cfg.PeakFrom && t < cfg.PeakTo {
			l = cfg.PeakLambda
		}
		t += rng.ExpFloat64() / l
		if t >= cfg.Horizon {
			break
		}
		q = append(q, arrival{t: t, attempt: 1})
	}
	heap.Init(&q)
	var r Result
	free := 0.0 // instant où le worker aura fini tout ce qu'il a accepté
	next := 0.0
	for q.Len() > 0 {
		a := heap.Pop(&q).(arrival)
		for next <= a.t && next < cfg.Horizon {
			r.Backlog = append(r.Backlog, max(0, free-next))
			next += 100
		}
		start := max(a.t, free)
		late := start-a.t > cfg.Timeout
		if !(late && cfg.DropStale) {
			free = start + rng.ExpFloat64()
			r.Accepted++
			if free-a.t <= cfg.Timeout {
				r.Useful++
				continue
			}
		} else {
			r.Dropped++
		}
		// Le client a abandonné au bout de Timeout : il réessaie (ou renonce).
		if cfg.MaxAttempts > 0 && a.attempt >= cfg.MaxAttempts {
			r.GaveUp++
			continue
		}
		wait := 0.0
		if cfg.Backoff > 0 {
			wait = rng.Float64() * cfg.Backoff * float64(int(1)<<(a.attempt-1)) // gigue complète
		}
		if t := a.t + cfg.Timeout + wait; t < cfg.Horizon {
			heap.Push(&q, arrival{t: t, attempt: a.attempt + 1})
		}
	}
	return r
}
```

```go
// Fichier : retry/retry_test.go
package retry

import (
	"fmt"
	"testing"
)

func TestScenarios(t *testing.T) {
	base := Config{Lambda: 0.8, PeakLambda: 1.5, PeakFrom: 1000, PeakTo: 1200, Horizon: 6000, Timeout: 50}
	for _, sc := range []struct {
		name     string
		peak     bool
		drop     bool
		attempts int
	}{
		{"sans pic", false, false, 0},
		{"pic", true, false, 0},
		{"pic, jeter", true, true, 0},
		{"pic, 3 essais", true, false, 3},
		{"pic, 2 essais", true, false, 2},
		{"pic, jeter + 3 essais", true, true, 3},
		{"pic, jeter + 2 essais", true, true, 2},
	} {
		c := base
		if !sc.peak {
			c.PeakLambda = c.Lambda
		}
		c.DropStale, c.MaxAttempts = sc.drop, sc.attempts
		if sc.attempts > 0 {
			c.Backoff = 100
		}
		r := Run(c, 1)
		fmt.Printf("%-22s utiles %4d  acceptées %6d  jetées %6d  renoncements %4d\n  file :",
			sc.name, r.Useful, r.Accepted, r.Dropped, r.GaveUp)
		for i, b := range r.Backlog {
			if i%5 == 0 {
				fmt.Printf(" %.0f", b)
			}
		}
		fmt.Println()
	}
}
```

```
$ go test -v -run Scenarios ./retry
sans pic               utiles 4810  acceptées   4810  jetées      0  renoncements    0
  file : 0 0 12 0 1 0 0 2 17 5 3 2
pic                    utiles  872  acceptées 208722  jetées      0  renoncements    0
  file : 0 5 9 2220 8540 18696 32934 51112 73571 99862 129806 164682
pic, jeter             utiles 1215  acceptées   5803  jetées 175854  renoncements    0
  file : 0 5 9 50 50 51 51 50 50 50 51 50
pic, 3 essais          utiles  872  acceptées  12807  jetées      0  renoncements 3870
  file : 0 5 9 672 1377 2088 2804 3519 4244 4868 5530 6370
pic, 2 essais          utiles  872  acceptées   8945  jetées      0  renoncements 3997
  file : 0 5 9 450 749 1066 1341 1688 2020 2276 2558 2911
pic, jeter + 3 essais  utiles 3661  acceptées   5763  jetées   3675  renoncements 1192
  file : 0 5 9 50 50 49 51 51 50 49 53 49
pic, jeter + 2 essais  utiles 4792  acceptées   4948  jetées    260  renoncements  156
  file : 0 5 9 43 6 0 3 9 2 6 8 0
```

« file » donne le travail en attente, en ms, toutes les 500 ms sur les 6 s simulées. Le pic a lieu entre
1 000 et 1 200 ms.

**1. Sans pic**, le service est à 80 % et tout va bien. Les 4 810 requêtes sont toutes utiles et la file
reste sous 20 ms. Le délai de 50 ms n'est pratiquement jamais atteint : à ρ = 0,8, la probabilité d'un séjour
de plus de 50 ms vaut e^(−0,2 × 50) ≈ 0,005 %.

**2. Avec un pic de 200 ms**, le système **ne revient jamais** à la normale. Pendant le pic, la file accumule
environ (1,5 − 1) × 200 = 100 ms de travail. Les requêtes attendent alors plus de 50 ms, leurs clients
abandonnent et réessaient. Chaque relance est une arrivée de plus, alors que la requête abandonnée sera tout
de même traitée.

Le débit d'arrivée réel dépasse donc 1 par ms, la file grandit, encore plus de requêtes dépassent le délai…
À la fin, la file contient 165 s de travail, et seules 872 réponses sur 6 s ont été utiles, toutes avant le
pic. Le serveur travaille à 100 % pour des clients partis : c'est l'effondrement de N6 L11, provoqué cette
fois par les clients.

**3. Les remèdes, seuls ou combinés.**

| Remède | Ce qui se passe |
|---|---|
| Jeter les requêtes trop vieilles | La file reste bloquée à 50 ms : tout ce qui attendrait plus est jeté. Mais les requêtes servies attendent presque 50 ms, leur séjour dépasse le délai, et leurs clients réessaient. Seulement 1 215 réponses utiles. |
| 3 ou 2 essais, attente exponentielle | Moins de relances, mais chaque essai abandonné **est encore traité**. La file grandit plus lentement, sans jamais revenir. |
| Jeter + 3 essais | Mieux (3 661 utiles), mais toujours bloqué à 50 ms. |
| **Jeter + 2 essais** | **Le système se rétablit** : la file revient à quelques ms dès 1 500 ms. 4 792 réponses utiles sur environ 4 950 requêtes nouvelles. |

Sur ce scénario, aucun remède seul ne suffit. Il faut à la fois que le serveur **ne gaspille plus** son
travail pour des clients partis, et que les clients **n'amplifient plus** la charge. Le chiffre exact (2
essais et non 3) dépend du scénario : ce sont les mesures qui le fixent.

La file bornée, troisième piste, n'a pas été mesurée ici. Elle agit comme « jeter », mais plus tôt, à
l'entrée : le refus est immédiat au lieu d'arriver après 50 ms d'attente. Le client est alors prévenu plus
vite et peut patienter avant de réessayer.

**4. Deux régimes stables.** Pour le même λ = 0,8 par ms, le système a deux états qui se maintiennent
d'eux-mêmes :

- **Le régime sain.** La file est courte, les requêtes passent sous le délai, il n'y a pas de relances.
  La charge reste de 0,8 et le service est à 80 %.
- **Le régime effondré.** La file est longue et les requêtes dépassent le délai. Les relances
  multiplient la charge au-delà de la capacité, ce qui entretient la file longue.

Un état **métastable** est stable tant qu'on ne le perturbe pas. Le régime sain l'est : un pic passager suffit
à basculer dans le régime effondré, et le retour de la charge normale ne suffit pas à revenir. On a vu
qu'il faut **agir** : jeter le travail inutile, limiter les relances, ou couper le trafic un moment.

D'où les règles de N10 L9 : des relances **bornées** et **espacées**, un budget de relances, et des serveurs
qui vérifient `ctx.Err()` avant de commencer un travail.

---

## Quiz

1. La latence est une distribution, souvent très asymétrique : quelques requêtes très lentes changent peu la
   moyenne mais touchent de vrais utilisateurs. Les percentiles (p50, p99, p99,9) décrivent ce que vivent les
   clients, la moyenne ne le dit pas.
2. 1 − 0,99⁵⁰ ≈ **39 %** des pages.
3. L = λ × W : nombre moyen présent = débit × temps moyen passé dans le système. Elle ne suppose **rien** sur
   les distributions des arrivées et des durées. Il suffit que le système soit stable et que la mesure porte
   sur une durée assez longue.
4. 500 × 0,020 = **10 connexions** en moyenne. Le pool doit en avoir davantage, pour la marge.
5. Pour des arrivées et des durées aléatoires, le séjour moyen vaut S / (1 − ρ). Il double à 50 %, est
   multiplié par 10 à 90 % et par 100 à 99 %. La croissance est très rapide près de la saturation.
6. Avec des files séparées, une requête peut attendre derrière un worker occupé alors qu'un autre est libre.
   Avec une file commune, aucun worker n'est inactif tant qu'il y a du travail. Mesuré : 2,9 ms contre 9,8 ms
   de moyenne, pour 4 workers à 90 %.
7. Un générateur qui attend ses réponses cesse d'envoyer quand le serveur se bloque. Il omet donc de
   mesurer les requêtes que de vrais clients auraient envoyées pendant la panne. Ses N goroutines sont
   « coordonnées » avec le serveur. Résultat : un p99 de 3 ms au lieu de 488 ms dans l'exemple.
8. Depuis l'instant où la requête **devait** partir selon le calendrier. Si le générateur lui-même prend du
   retard, ce retard fait partie de la latence vue par le client.
9. Son débit vaut N / W (Little, avec L = N). Il dépend de la latence du serveur et du nombre de goroutines :
   il mesure le générateur, pas la capacité du serveur.
10. Un percentile ne se déduit pas des percentiles de sous-ensembles. La moyenne des p99 donne le même poids
    à chaque serveur et masque la queue du serveur le plus lent : 15,7 ms contre 44,1 ms dans E5. On agrège
    les histogrammes, puis on calcule le percentile.
