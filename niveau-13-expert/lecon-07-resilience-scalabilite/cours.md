# Module 7 — Résilience et scalabilité

> Le niveau 10 (leçon 9) a donné les outils de base de la résilience : délais, réessais avec *jitter*,
> disjoncteur, idempotence, cloisons. Le niveau 11 (leçon 7) a montré comment la latence explose près de la
> saturation, et pourquoi un test de charge en boucle fermée ment. Ce module traite de ce qui arrive **au-delà**
> de la saturation :
> - un service surchargé peut faire beaucoup de travail et ne rendre presque aucun service utile ;
> - il peut rester effondré **après** la disparition de la cause, entretenu par ses propres clients ;
> - il se protège en refusant tôt, et en dégradant ce qui peut l'être ;
> - sa capacité se mesure, et se planifie avec une marge.
>
> L'exemple reproduit une panne **métastable** en temps virtuel, puis mesure sur un vrai serveur HTTP ce que
> change un contrôle d'admission.

## Objectifs

- Distinguer le débit du **débit utile** (*goodput*), et mesurer le travail gaspillé d'un service surchargé.
- Reconnaître une panne métastable : un déclencheur, et un effet qui l'entretient.
- Protéger un service : file bornée, abandon des requêtes expirées, refus rapide, budget de réessais.
- Dégrader proprement : priorités, fonctions coupées sous charge, valeurs de repli.
- Mesurer une capacité par un test de charge par paliers, et en déduire une flotte.
- Tester les pannes volontairement, avant qu'elles n'arrivent.

## Explication

### La surcharge, une panne à part

Un service est **surchargé** quand on lui demande plus de travail qu'il ne peut en faire. Ce n'est pas une
panne au sens habituel : rien n'est cassé, chaque composant fonctionne, et le service traite autant de
requêtes qu'il le peut. Pourtant il peut ne **rendre presque aucun service**, parce que :
- les requêtes attendent dans une file de plus en plus longue (N11 L7) ;
- les clients abandonnent après leur délai (une seconde, par exemple), mais le serveur, lui, ne le sait pas ;
- il traite donc des requêtes dont plus personne n'attend la réponse : du **travail gaspillé**.

Ce qui compte n'est pas le **débit** (requêtes traitées par seconde) mais le **débit utile** : les requêtes
dont la réponse arrive **à temps** chez quelqu'un qui l'attend encore. Dans l'exemple, un serveur naïf
pendant une pointe traite des requêtes à pleine vitesse, mais 95 % de ce travail est gaspillé.

### Les pannes métastables

Une panne **métastable** a deux ingrédients :
1. un **déclencheur** : une pointe de trafic, une dépendance lente pendant quelques secondes, un
   redémarrage ;
2. un **effet d'entretien** : un mécanisme qui maintient la surcharge **après** la disparition du
   déclencheur.

Le système a alors deux états stables pour la même charge : sain, ou effondré. Une perturbation passagère
le fait passer de l'un à l'autre, et il n'en revient pas seul. Corriger le déclencheur ne sert à rien : la
pointe est finie depuis longtemps.

Les effets d'entretien courants :
- **Les réessais.** Une requête qui échoue par délai dépassé est réessayée. La charge offerte est multipliée
  par le nombre de tentatives, juste au moment où le service est déjà saturé. Avec 3 réessais, une charge
  normale de 75 % devient 300 %.
- **Le cache froid.** Après un redémarrage ou une purge, toutes les requêtes vont à la base. La base
  ralentit, les requêtes dépassent leur délai avant d'avoir rempli le cache, qui reste froid.
- **Le basculement.** Une instance tombe, sa charge passe sur les autres, qui saturent à leur tour.

Le point commun : le système **fait plus de travail** quand il va mal. Dans l'exemple, la variante B reçoit la
même pointe que la variante A, et ne s'en remet jamais, à cause des réessais seuls.

### Refuser tôt, et pour pas cher

Un service surchargé doit **refuser** une partie des requêtes, pour bien servir les autres. Refuser n'est
pas un échec : c'est le seul moyen de garder un débit utile. Les protections, de la plus simple à la plus fine :

- **Une file bornée** (N6 L11). Une file sans limite transforme une surcharge en latence infinie. Avec une limite,
  le surplus est refusé immédiatement. La bonne taille se déduit du délai : une file qu'on vide à 160 req/s et
  qui ne doit pas faire attendre plus de 250 ms tient 40 requêtes (loi de Little, N11 L7).
- **Abandonner les requêtes expirées.** Avant de commencer le travail, vérifier que l'appelant attend
  encore : `ctx.Err() != nil`. Le contexte porte le délai de l'appelant (N6 L8, N7 L7). Entre services, le délai
  restant voyage avec la requête (gRPC le fait par l'en-tête `grpc-timeout`, N7 L12). Chaque service le
  respecte, et le raccourcit pour ses propres appels.
- **Un refus qui ne coûte rien.** Un refus doit coûter beaucoup moins cher qu'un traitement. Sinon, refuser
  sature autant que servir. En HTTP : `503 Service Unavailable` et `Retry-After`, sans lire le corps, sans
  toucher à la base.
- **Le dernier arrivé d'abord** (LIFO) sous surcharge : en file FIFO, la requête servie est la plus vieille,
  la plus proche de son délai. En LIFO, c'est la plus fraîche, celle qui a le plus de chances d'être encore
  attendue. Certains serveurs basculent en LIFO quand la file dépasse un seuil (exercice E1).

### Limiter les réessais : le budget

Le niveau 10 recommandait déjà de ne réessayer qu'à **un** niveau, avec une attente exponentielle et du
*jitter*. Ce n'est pas suffisant : l'attente étale les réessais dans le temps, mais **ne réduit pas leur
nombre**. Pendant une surcharge longue, chaque requête finit par envoyer toutes ses tentatives.

Un **budget de réessais** limite les réessais à une **fraction des succès** : chaque succès crédite
0,1 jeton, chaque réessai en consomme un. Quand tout va bien, les réessais passent. Quand tout échoue, il
n'y a plus de jetons, et la charge ajoutée par les réessais ne dépasse jamais 10 % de la charge utile. Le
budget est tenu par **le client** (une instance de client HTTP partagée), pas par requête.

### Où se forme la file

Un contrôle d'admission ne protège que s'il est placé **devant** la file. L'exemple le montre deux fois :
- **Le service est limité par sa base** (4 connexions). Les requêtes attendent une connexion, dans le
  gestionnaire. Un sémaphore à l'entrée du gestionnaire voit cette file, et refuse le surplus : le p99 reste
  à 20 ms sous une charge de 170 %.
- **Le service est limité par le processeur** (un seul cœur, environ 1 ms de calcul par requête ; essayé
  lors de la rédaction, exercice E4). Le même sémaphore ne refuse **aucune** requête, et la latence explose de la même façon.
  La file ne se forme pas dans le gestionnaire : elle est dans le noyau (les octets non lus des
  connexions), et dans l'ordonnanceur de Go (les goroutines prêtes qui attendent le processeur). Quand une
  requête atteint le sémaphore, elle a déjà attendu. Là, la protection se place **en amont** : un équilibreur
  de charge qui limite les requêtes par instance, ou une limite sur les connexions.

Avant de protéger, il faut donc savoir **quelle ressource sature**. La trace d'exécution (N13 M4) montre si les
goroutines attendent un verrou, un canal ou le processeur.

### Dégrader proprement

Refuser n'est pas tout ou rien. Un service peut **moins bien** servir, plutôt que ne pas servir :
- **Des priorités.** Une requête de paiement passe avant un rafraîchissement de tableau de bord. Sous
  charge, on refuse d'abord les requêtes les moins importantes (exercice E2). La priorité se décide **à
  l'avance**, par type de requête ou par un en-tête posé par l'appelant.
- **Couper une fonction sous charge.** Les recommandations d'une page produit coûtent cher et ne sont pas
  indispensables. Au-delà d'un seuil de charge, on les retire : la page s'affiche sans elles.
- **Une valeur de repli.** Une dépendance ne répond pas : afficher la dernière valeur connue, une valeur par
  défaut, ou rien. Le niveau 10 (leçon 9) le présentait comme une décision métier : elle se prend **avant**
  la panne, et se teste.

### La capacité

La **capacité** d'un service est le débit maximal qu'il sert en respectant son objectif de latence (son SLO,
par exemple « p99 < 50 ms »). Elle ne se calcule pas, elle se **mesure** :
- un **test par paliers** augmente le débit demandé, palier après palier, en boucle ouverte, et mesure le
  p99 et les erreurs à chaque palier. La capacité est le dernier palier qui tient l'objectif ;
- dans l'exemple, la théorie donne 800 req/s (4 connexions, 5 ms par requête). Un `time.Sleep(5 ms)` dure en
  réalité 5,3 ms sur la machine de mesure, ce qui donne 757 req/s. Le test mesure environ 700 req/s. **Le
  calcul surestime toujours** ;
- la loi de Little dimensionne la limite d'admission : 700 req/s × 5,7 ms = 4 requêtes en cours, la taille
  du pool. Une limite de 8 laisse 4 requêtes attendre au plus quelques millisecondes.

**Planifier**, c'est ajouter une marge :
- on vise un taux d'occupation de 60 à 70 % à la pointe attendue, parce que la latence explose au-delà
  (N11 L7) ;
- on prévoit la perte d'une instance (**N+1**), ou d'une zone entière ;
- on vérifie que la ressource **partagée** suit : dix instances avec 4 connexions chacune, c'est 40
  connexions à la base. Ajouter des instances n'augmente pas la capacité quand le goulot est la base, et peut
  même la dégrader.

### Tester la charge et les pannes

Les tests de charge :
- **Par paliers** : trouver la capacité, comme ci-dessus.
- **Pointe** : passer brusquement à 2 ou 3 fois la charge normale, puis revenir. Le service doit se
  rétablir seul, rapidement. C'est le test qui révèle les pannes métastables.
- **Endurance** : une charge normale pendant des heures, pour les fuites et les dérives (N13 M5).
- Toujours **en boucle ouverte**, avec la latence mesurée depuis l'instant prévu (N11 L7). Le générateur ne
  doit pas être le goulot : ici, il tourne sur d'autres cœurs que le serveur (`taskset`).

**L'injection de pannes** provoque volontairement ce qu'on redoute : une dépendance lente, une dépendance
qui renvoie des erreurs, une instance arrêtée. En Go, c'est souvent un simple `http.RoundTripper` qui
enveloppe le vrai (exercice E5). On vérifie ainsi que les délais, le mode dégradé et les alertes
fonctionnent **pour de vrai**. Une **journée d'exercice** (*game day*) le fait en équipe, sur un
environnement proche de la production, avec un scénario écrit et un retour d'expérience.

### Quand ne pas le faire

- **Un outil interne** à dix utilisateurs : un délai par requête suffit. Une file bornée et un budget de
  réessais n'ont de sens que si la surcharge est possible.
- **Une limite d'admission jamais mesurée.** Trop basse, elle refuse des requêtes que le service aurait
  servies ; elle crée la panne qu'elle devait éviter. On la déduit d'un test par paliers, et on la réévalue.
- **Une dégradation jamais testée.** Un mode dégradé qui n'a jamais tourné est un code mort, qui échouera
  le jour où on en aura besoin.
- **Chercher à absorber toute pointe** en surdimensionnant. Au-delà d'un certain point, il est moins cher de
  refuser proprement une pointe rare que de payer en permanence la capacité de la servir.

## Exemple

Deux expériences :
- `overload` : un serveur à file d'attente et ses clients, en **temps virtuel** (`testing/synctest`, N6 L11).
  Une minute de trafic se simule en un quart de seconde, à l'identique d'une exécution à l'autre ;
- `admit`, `srv` et `load` : un vrai serveur HTTP limité par sa base, et un test de charge par paliers.

```go.mod
// Fichier : go.mod
module example.com/resil

go 1.27
```

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
}

type Stats struct {
	Served   atomic.Int64 // requêtes traitées jusqu'au bout
	Wasted   atomic.Int64 // dont : traitées alors que l'appelant avait renoncé
	Dropped  atomic.Int64 // abandonnées sans travail (délai dépassé)
	Rejected atomic.Int64 // refusées à l'entrée (file pleine)
}

type job struct {
	ctx  context.Context
	done chan error
}

type Server struct {
	cfg   Config
	queue chan job
	wg    sync.WaitGroup
	Stats Stats
}

func New(cfg Config) *Server {
	s := &Server{cfg: cfg, queue: make(chan job, cfg.QueueCap)}
	for range cfg.Workers {
		s.wg.Go(s.work)
	}
	return s
}

// Close arrête les travailleurs, et attend qu'ils aient vidé la file.
func (s *Server) Close() {
	close(s.queue)
	s.wg.Wait()
}

// Handle traite une requête, ou renvoie une erreur : file pleine, ou délai de ctx dépassé.
func (s *Server) Handle(ctx context.Context) error {
	j := job{ctx: ctx, done: make(chan error, 1)}
	select {
	case s.queue <- j:
	default:
		s.Stats.Rejected.Add(1)
		return ErrOverloaded
	}
	select {
	case err := <-j.done:
		return err
	case <-ctx.Done():
		return ctx.Err() // l'appelant renonce ; la requête, elle, reste dans la file
	}
}

func (s *Server) work() {
	for j := range s.queue {
		if s.cfg.DropExpired && j.ctx.Err() != nil {
			s.Stats.Dropped.Add(1)
			j.done <- j.ctx.Err()
			continue
		}
		time.Sleep(s.cfg.Service) // le travail
		s.Stats.Served.Add(1)
		if j.ctx.Err() != nil {
			s.Stats.Wasted.Add(1)
		}
		j.done <- nil
	}
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
	Server   *Server
	Timeout  time.Duration // délai d'une tentative
	Retries  int           // réessais au plus, après la première tentative
	Budget   *Budget       // nil : réessayer sans limite globale
	Attempts int64         // tentatives envoyées (lu à la fin, sous mu)
	mu       sync.Mutex
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
		time.Sleep(backoff/2 + rand.N(backoff)) // jitter (N10 L9)
		backoff *= 2
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

Le scénario : 120 req/s (75 % de la capacité) pendant 10 s, une pointe à 300 req/s pendant 5 s, puis 120 req/s
pendant 45 s. Quatre variantes du serveur et des clients reçoivent exactement les mêmes arrivées :

```go
// Fichier : overload/meta_test.go
package overload

import (
	"fmt"
	"math/rand/v2"
	"sync"
	"testing"
	"testing/synctest"
	"time"
)

// phase : un débit d'arrivée (requêtes par seconde) pendant une durée.
type phase struct {
	rate float64
	d    time.Duration
}

// Le scénario : 75 % de la capacité, une pointe de 5 s à 190 %, puis 75 % à nouveau.
var scenario = []phase{{120, 10 * time.Second}, {300, 5 * time.Second}, {120, 45 * time.Second}}

const window = 5 * time.Second

type result struct {
	good    []int // requêtes réussies, par fenêtre de 5 s (selon l'instant d'arrivée)
	total   []int
	stats   *Stats
	attempt int64
}

// run joue le scénario en temps virtuel contre un serveur de 4 travailleurs à 25 ms (160 req/s).
func run(t *testing.T, cfg Config, mk func(*Server) *Client) result {
	var r result
	synctest.Test(t, func(t *testing.T) {
		srv := New(cfg)
		cl := mk(srv)
		rng := rand.New(rand.NewPCG(1, 2))
		var total time.Duration
		for _, p := range scenario {
			total += p.d
		}
		n := int(total / window)
		r.good, r.total = make([]int, n), make([]int, n)
		var mu sync.Mutex
		var wg sync.WaitGroup
		start := time.Now()
		for _, p := range scenario {
			end := time.Since(start) + p.d
			for time.Since(start) < end {
				time.Sleep(time.Duration(rng.ExpFloat64() / p.rate * float64(time.Second)))
				w := int(time.Since(start) / window)
				if w >= n {
					break
				}
				seed := rng.Uint64()
				wg.Go(func() {
					ok := cl.Call(rand.New(rand.NewPCG(seed, 0)))
					mu.Lock()
					r.total[w]++
					if ok {
						r.good[w]++
					}
					mu.Unlock()
				})
			}
		}
		wg.Wait()
		srv.Close()
		r.stats = &srv.Stats
		r.attempt = cl.Attempts
	})
	return r
}

func TestMetastable(t *testing.T) {
	base := Config{Workers: 4, Service: 25 * time.Millisecond}
	unbounded := base
	unbounded.QueueCap = 100_000
	protected := base
	protected.QueueCap, protected.DropExpired = 40, true

	variants := []struct {
		name string
		cfg  Config
		mk   func(*Server) *Client
	}{
		{"A. naïf, sans réessai", unbounded, func(s *Server) *Client {
			return &Client{Server: s, Timeout: time.Second}
		}},
		{"B. naïf, 3 réessais", unbounded, func(s *Server) *Client {
			return &Client{Server: s, Timeout: time.Second, Retries: 3}
		}},
		{"C. + abandon, file 40", protected, func(s *Server) *Client {
			return &Client{Server: s, Timeout: time.Second, Retries: 3}
		}},
		{"D. + budget 10 %", protected, func(s *Server) *Client {
			return &Client{Server: s, Timeout: time.Second, Retries: 3, Budget: &Budget{Ratio: 0.1, Max: 10}}
		}},
	}
	fmt.Println("taux de succès par fenêtre de 5 s (pointe entre 10 s et 15 s)")
	fmt.Print("                          ")
	for w := range 12 {
		fmt.Printf("%4ds", w*5)
	}
	fmt.Println("   tentatives  gaspillé")
	for _, v := range variants {
		r := run(t, v.cfg, v.mk)
		fmt.Printf("%-26s", v.name)
		for w := range r.good {
			fmt.Printf("%5.0f", 100*float64(r.good[w])/float64(max(r.total[w], 1)))
		}
		fmt.Printf("  %10d  %7.0f %%\n", r.attempt, 100*float64(r.stats.Wasted.Load())/float64(max(r.stats.Served.Load(), 1)))
	}
}
```

```
$ go test -run TestMetastable -v ./overload/
taux de succès par fenêtre de 5 s (pointe entre 10 s et 15 s)
                             0s   5s  10s  15s  20s  25s  30s  35s  40s  45s  50s  55s   tentatives  gaspillé
A. naïf, sans réessai       100  100   22    0    0   20  100  100  100  100  100  100        8191       35 %
B. naïf, 3 réessais         100  100   22    0    0    0    0    0    0    0    0    0       28162       95 %
C. + abandon, file 40       100  100   58   99  100  100  100  100  100  100  100  100       11224        0 %
D. + budget 10 %            100  100   56   99  100  100  100  100  100  100  100  100        8278        0 %
```

Le taux de succès compte les requêtes **arrivées** dans chaque fenêtre de 5 s qui ont fini par réussir.
Trois exécutions donnent les mêmes résultats pour A et B ; pour C et D, à un point de pourcentage et quelques dizaines de tentatives près.

- **A, sans réessai** : la pointe remplit la file, qui met une quinzaine de secondes à se vider. Pendant ce
  temps, toutes les requêtes échouent, même celles arrivées après la pointe. Le système se rétablit à 30 s.
  35 % du travail est gaspillé.
- **B, avec 3 réessais** : la même pointe, et le système **ne se rétablit jamais**. À 55 s, quarante secondes
  après la fin de la pointe, le taux de succès est toujours nul. Il y a eu 28 162 tentatives pour 8 191
  requêtes, et 95 % du travail est gaspillé : le serveur tourne à plein pour des clients partis. C'est la panne
  métastable.
- **C, file bornée et abandon des requêtes expirées** : le serveur refuse le surplus immédiatement, et ne
  traite plus que des requêtes encore attendues (0 % de gaspillage). Pendant la pointe, 58 % des requêtes
  réussissent, et le système est sain dès la fenêtre suivante.
- **D, plus un budget de réessais** : le même comportement, avec 8 278 tentatives au lieu de 11 224. Les
  réessais n'ajoutent plus qu'un pour cent de charge. Sur ce scénario, C suffit à éviter l'effondrement. Le
  budget protège le **serveur** quand les refus eux-mêmes coûtent (exercice du défi), et les autres
  clients de la même dépendance.

Le serveur HTTP, limité par sa base :

```go
// Fichier : admit/admit.go
// Paquet admit : le contrôle d'admission d'un serveur HTTP.
package admit

import (
	"net/http"
	"time"
)

// Limit laisse passer au plus maxInFlight requêtes à la fois. Une requête attend une place au plus
// maxWait ; au-delà, elle est refusée tout de suite (503), ce qui ne coûte presque rien.
func Limit(maxInFlight int, maxWait time.Duration, next http.Handler) http.Handler {
	sem := make(chan struct{}, maxInFlight)
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		t := time.NewTimer(maxWait)
		defer t.Stop()
		select {
		case sem <- struct{}{}:
		case <-t.C:
			w.Header().Set("Retry-After", "1")
			http.Error(w, "surchargé", http.StatusServiceUnavailable)
			return
		case <-r.Context().Done():
			return
		}
		defer func() { <-sem }()
		next.ServeHTTP(w, r)
	})
}
```

```go
// Fichier : cmd/srv/main.go
// srv : un service limité par sa base de données (4 connexions, 5 ms par requête : 800 req/s au plus).
package main

import (
	"flag"
	"log"
	"net/http"
	"time"

	"example.com/resil/admit"
)

var db = make(chan struct{}, 4) // le pool de connexions (comme SetMaxOpenConns(4), N8 L5)

func query() {
	db <- struct{}{} // attendre une connexion libre, sans limite de temps
	time.Sleep(5 * time.Millisecond)
	<-db
}

func work(w http.ResponseWriter, r *http.Request) {
	query()
	w.Write([]byte("ok"))
}

func main() {
	addr := flag.String("addr", "127.0.0.1:8090", "adresse d'écoute")
	limit := flag.Int("limit", 0, "requêtes simultanées au plus (0 : pas de limite)")
	flag.Parse()
	var h http.Handler = http.HandlerFunc(work)
	if *limit > 0 {
		h = admit.Limit(*limit, 5*time.Millisecond, h)
	}
	log.Fatal(http.ListenAndServe(*addr, h))
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
	"sync"
	"time"
)

func main() {
	url := flag.String("url", "http://127.0.0.1:8090/", "adresse du service")
	step := flag.Duration("step", 3*time.Second, "durée d'un palier")
	flag.Parse()
	client := &http.Client{Transport: &http.Transport{MaxIdleConnsPerHost: 2000}}
	fmt.Println("  débit demandé   réussi/s   503/s   délai/s     p50      p99   (ms, depuis l'instant prévu)")
	for _, rate := range []int{200, 400, 600, 700, 800, 900, 1000, 1200} {
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
		slices.Sort(lat)
		q := func(p float64) float64 {
			if len(lat) == 0 {
				return 0
			}
			return lat[int(p*float64(len(lat)-1))]
		}
		s := step.Seconds()
		fmt.Printf("%10d/s  %9.0f  %6.0f  %8.0f  %6.1f  %7.1f\n",
			rate, float64(ok)/s, float64(shed)/s, float64(timeout)/s, q(0.50), q(0.99))
		time.Sleep(time.Second) // laisser le service se vider entre deux paliers
	}
}
```

```
$ go build -o srv ./cmd/srv && go build -o load ./cmd/load
$ taskset -c 0,1 ./srv &              # le serveur sur deux cœurs, le générateur sur les six autres
$ taskset -c 2-7 ./load
  débit demandé   réussi/s   503/s   délai/s     p50      p99   (ms, depuis l'instant prévu)
       200/s        200       0         0     7.0      8.5
       400/s        400       0         0     7.0      9.7
       600/s        600       0         0     6.8      8.8
       700/s        700       0         0    19.2     32.7
       800/s        800       0         0   254.6    498.5
       900/s        900       0         0   454.1    929.1
      1000/s        730       0       270   499.6    989.3
      1200/s        567       0       634   489.1    989.2
$ taskset -c 0,1 ./srv -limit 8 &     # même service, avec le contrôle d'admission
$ taskset -c 2-7 ./load
  débit demandé   réussi/s   503/s   délai/s     p50      p99   (ms, depuis l'instant prévu)
       200/s        200       0         0     7.1      8.8
       400/s        400       0         0     7.0      8.8
       600/s        600       0         0     6.9      9.1
       700/s        693       8         0    14.1     18.5
       800/s        688     112         0    15.6     21.1
       900/s        694     207         0    15.9     19.9
      1000/s        693     307         0    16.2     19.9
      1200/s        697     504         0    16.4     20.4
```

- **La capacité** est d'environ 700 req/s : c'est le dernier palier où le p99 reste sous 50 ms, et le
  plateau des réponses réussies avec le contrôle d'admission.
- **Sans contrôle d'admission**, au-delà de la capacité, les requêtes attendent une connexion à la base
  sans limite. À 800 req/s, le p99 passe à 499 ms ; à 1 000 req/s, un quart des requêtes dépassent leur délai
  d'une seconde, et le **débit utile baisse** quand la charge augmente (730, puis 567 req/s). Le palier ne
  dure que 3 s : plus long, il s'effondrerait davantage, la file n'ayant pas le temps de se vider.
- **Avec le contrôle d'admission**, le débit utile reste entre 688 et 697 req/s quelle que soit la charge, avec un p99
  autour de 20 ms. Le surplus reçoit un 503 en 5 ms. Les clients refusés peuvent réessayer ailleurs, ou plus
  tard ; ceux qui sont servis le sont bien.

## Explication du code

| Élément | Rôle |
|---|---|
| `select { case s.queue <- j: … default: … }` | L'envoi non bloquant : si la file est pleine, refus immédiat (N6 L4). Le refus ne coûte qu'une tentative d'envoi. |
| `case <-ctx.Done(): return ctx.Err()` dans `Handle` | L'appelant renonce à son délai. La requête, elle, **reste dans la file** : c'est ce qui produit le travail gaspillé. |
| `if s.cfg.DropExpired && j.ctx.Err() != nil` | L'abandon : vérifier, juste avant de travailler, que quelqu'un attend encore. Une ligne, et 0 % de gaspillage. |
| `Budget` | Un seau à jetons alimenté par les succès. Il est partagé par toutes les requêtes du client, sous un verrou. |
| `synctest.Test` et `time.Sleep` | En temps virtuel, le temps n'avance que quand toutes les goroutines de la bulle sont bloquées. Le travail de 25 ms est un sommeil, la minute de trafic dure une fraction de seconde, et les mêmes graines donnent les mêmes arrivées. |
| `wg.Go` (Go 1.25) | Lance une goroutine comptée par le `WaitGroup` ; `Close` attend que les travailleurs aient vidé la file avant la fin de la bulle. |
| `admit.Limit` | Un sémaphore (canal tamponné, N6 L11) et une attente bornée par un minuteur. Au-delà de `maxWait`, `503` et `Retry-After`. |
| `var db = make(chan struct{}, 4)` | Le pool de connexions simulé : une attente **sans limite de temps** d'une connexion, comme `database/sql` quand toutes sont prises. |
| `time.Sleep(time.Until(planned))` et `time.Since(planned)` | La boucle ouverte : les requêtes partent selon le calendrier, et la latence se mesure depuis l'instant prévu (N11 L7). |

## Erreurs fréquentes

1. **Mesurer le débit au lieu du débit utile** : un serveur qui traite 160 req/s à pleine charge peut n'en
   servir aucune à temps.
2. **Une file sans limite**, par défaut : un canal immense, un pool de connexions sans délai d'attente.
3. **Traiter une requête dont l'appelant est parti**, faute de vérifier `ctx.Err()` avant le travail.
4. **Croire que l'attente exponentielle limite les réessais** : elle les étale, elle ne les réduit pas.
5. **Corriger le déclencheur d'une panne métastable** (« la pointe était anormale ») et laisser l'effet
   d'entretien en place.
6. **Placer un contrôle d'admission derrière la file**, quand la ressource saturée est le processeur.
7. **Calculer la capacité au lieu de la mesurer** : 800 req/s en théorie, 700 mesurées.
8. **Ajouter des instances devant une base saturée.**

## Bonnes pratiques Go

- Le contexte de la requête jusqu'au bout, et `ctx.Err()` vérifié avant chaque travail coûteux.
- Des files bornées : canaux tamponnés de taille choisie, `select` avec `default` pour refuser.
- Un refus rapide et explicite : `503` et `Retry-After`, avant toute lecture du corps ou de la base.
- Un budget de réessais par client, en plus du délai et de l'attente exponentielle avec *jitter*.
- Des tests en temps virtuel (`synctest`) pour les scénarios de surcharge : déterministes et rapides.
- Un test par paliers en boucle ouverte pour chaque service, refait à chaque changement important.

## Ce que je dois retenir

- Un service surchargé peut gaspiller 95 % de son travail : seul compte le débit utile.
- Une panne métastable a un déclencheur et un effet d'entretien. Ici, 3 réessais suffisent pour que le
  système ne se remette jamais d'une pointe de 5 s.
- Une file bornée et l'abandon des requêtes expirées ont suffi à le rétablir dès la fin de la pointe.
- Un contrôle d'admission tient un p99 de 20 ms à 170 % de charge, s'il est placé devant la ressource qui
  sature.
- La capacité se mesure par paliers, puis on planifie avec une marge (60-70 %, N+1).

---

➡️ [Exercices](exercices.md)
