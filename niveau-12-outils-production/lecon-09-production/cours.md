# Leçon 9 — Production

> Le service est versionné, construit, emballé, intégré, configuré, observable et durci. Il reste à le faire
> **vivre** : le déployer sans couper les utilisateurs, dire à l'orchestrateur s'il est vivant et prêt,
> revenir en arrière quand une version se comporte mal, et comprendre ce qui se passe dans un processus en
> marche sans l'arrêter. Cette leçon mesure ce qu'un déploiement fait perdre comme requêtes, selon la façon
> dont l'ancienne instance s'arrête. Elle rassemble ensuite les gestes du diagnostic en production.

## Objectifs

- Déployer une nouvelle version instance par instance, sans erreur visible des clients.
- Distinguer les sondes de vie (*liveness*), de disponibilité (*readiness*) et de démarrage (*startup*).
- Arrêter une instance dans le bon ordre, et dimensionner les délais.
- Rendre une version réversible : migrations compatibles, retour arrière, drapeaux.
- Diagnostiquer un service en marche sans l'interrompre.

## Explication

### Le déploiement progressif

Un service tourne en plusieurs **instances** derrière un répartiteur de charge : un *load balancer*, ou le
`Service` d'un orchestrateur comme Kubernetes. Un **déploiement progressif** (*rolling update*) remplace les
instances une par une :

1. démarrer une instance de la nouvelle version ;
2. attendre qu'elle soit **prête**, et que le répartiteur lui envoie du trafic ;
3. arrêter une instance de l'ancienne version ;
4. recommencer jusqu'à la dernière.

Pendant tout le déploiement, **deux versions tournent en même temps**. D'où une contrainte qui dépasse le
code Go : la nouvelle version doit fonctionner avec le schéma de base de l'ancienne, et inversement. C'est la
migration en deux temps « étendre, puis contracter » de N8 L6.

### Trois sondes

L'orchestrateur interroge chaque instance pour décider quoi en faire.

| Sonde | Question | Si elle échoue | Ce qu'elle vérifie |
|---|---|---|---|
| **vie** (`/livez`) | le processus est-il encore capable de répondre ? | l'instance est **redémarrée** | presque rien : que le serveur HTTP répond. **Jamais** une dépendance |
| **disponibilité** (`/readyz`) | l'instance accepte-t-elle du trafic **maintenant** ? | elle est **retirée** du répartiteur, sans être tuée | le démarrage terminé (caches chargés, connexions ouvertes), et pas d'arrêt en cours |
| **démarrage** (*startup*) | le démarrage est-il fini ? | on attend encore, dans une limite | les deux autres sondes sont suspendues tant qu'elle n'a pas réussi |

Deux erreurs classiques :

- **Une sonde de vie qui vérifie la base de données.** La base a un incident de 30 secondes : **toutes** les
  instances échouent leur sonde de vie et sont redémarrées en même temps. La panne de la base devient une
  panne du service, puis une tempête de redémarrages qui ouvrent toutes leurs connexions à la base au moment
  où elle revient.
- **Une sonde de disponibilité qui suit exactement une dépendance partagée.** Si la base tombe, toutes les
  instances se déclarent non prêtes, et le répartiteur n'a plus personne à qui envoyer les requêtes. Les
  clients reçoivent « aucune instance » au lieu d'un message d'erreur utile. La disponibilité exprime
  l'état **local** de l'instance. Une dépendance en panne se traite par des erreurs claires et un disjoncteur
  (N10 L9).

### L'ordre de l'arrêt

Quand l'orchestrateur arrête une instance, il lui envoie `SIGTERM` (leçon 3) **et**, en parallèle, la
retire du répartiteur. Mais ce retrait prend du temps : le répartiteur ne le voit qu'à sa prochaine sonde ou
mise à jour. Si l'instance cesse aussitôt d'accepter des connexions, les requêtes qui arrivent pendant ce
délai échouent. L'exemple compare trois façons de s'arrêter, pendant un déploiement sous charge :

| Arrêt de l'ancienne instance | Requêtes en erreur (3 exécutions) |
|---|---|
| `srv.Close()` : brutal | 7, 6, 41 |
| `srv.Shutdown()` immédiat | 38, 32, 13 |
| `/readyz` → 503, attente de 3 périodes de sonde, **puis** `Shutdown` | **0, 0, 0** |

`Shutdown` seul ne suffit pas. Il termine bien les requêtes **en cours**, mais il ferme aussitôt l'écoute, alors
que le répartiteur envoie encore du trafic vers cette instance. La séquence correcte :

1. **ne plus être prêt** : `/readyz` répond 503 ;
2. **attendre** que le répartiteur l'ait vu : au moins une période de sonde, plus une marge. En Kubernetes,
   c'est souvent un `preStop` qui attend quelques secondes ;
3. **`Shutdown(ctx)`** : plus de nouvelles connexions, et attente des requêtes en cours dans la limite de
   `ctx` ;
4. **fermer le reste** : pool de base de données, `TracerProvider.Shutdown` (leçon 7), journaux.

Et les délais doivent s'emboîter :

```
attente de propagation + plus longue requête légitime + fermeture  <  délai de l'orchestrateur
```

Le délai de l'orchestrateur est de 30 s par défaut dans Kubernetes (`terminationGracePeriodSeconds`), et de
10 s pour `docker stop`. La leçon 3 (E4) a montré ce qui arrive au-delà : `SIGKILL`.

Un détail mesuré dans l'exemple : `Shutdown` attend **jusqu'à 5 secondes** une connexion ouverte sur
laquelle aucune requête n'est encore arrivée (état `StateNew`), avant de l'abandonner. Sans une telle
connexion, l'arrêt est immédiat ; avec, il prend 5,33 s. Un répartiteur ou un client qui ouvre des connexions
à l'avance peut donc allonger chaque arrêt de 5 s. Il faut le compter dans le budget.

### Revenir en arrière

Une version qui se comporte mal en production doit pouvoir être **retirée en minutes**, sans réfléchir.

- **Le retour arrière est un déploiement comme un autre**, de l'image précédente. Il suppose que l'image
  précédente existe encore, désignée par une étiquette immuable (`1.3.0`) ou par son empreinte, et jamais
  par `latest`.
- **Il suppose aussi un schéma compatible** : une migration qui supprime une colonne dont l'ancienne version a
  besoin rend le retour impossible. On ne supprime qu'**après** que plus aucune version en service ne
  l'utilise (N8 L6).
- **Un drapeau de fonctionnalité** (leçon 5) permet de désactiver une nouveauté **sans** redéployer.
- **Un déploiement « canari »** envoie d'abord une petite part du trafic (5 %) à la nouvelle version. On
  compare ses métriques RED (leçon 7) à celles de l'ancienne avant d'aller plus loin.

En incident, on **rétablit d'abord** (retour arrière, drapeau), et on **comprend ensuite**. Diagnostiquer
pendant que les utilisateurs subissent la panne est une perte de temps pour eux.

### Diagnostiquer un service en marche

Tout ce que le niveau a construit sert ici :

| Question | Outil | Leçon |
|---|---|---|
| quelle version tourne ? | `/version`, journal de démarrage, `go version -m` sur le binaire | 1 |
| les utilisateurs souffrent-ils ? | métriques RED, p99, taux d'erreurs, alertes | 7 |
| où passe le temps d'une requête lente ? | une trace, puis ses journaux (`trace_id`) | 6, 7 |
| où passe le processeur ? la mémoire ? | `go tool pprof http://…:6060/debug/pprof/profile` (et `heap`) | N11 L4–L5 |
| des goroutines bloquées ou en fuite ? | `/debug/pprof/goroutine?debug=2`, profil `goroutineleak` | N11 L5 |
| plus de détail sur un composant ? | niveau de journal relevé à chaud, puis rabaissé | 5 (défi) |

Quelques règles :

- **L'adresse d'administration n'est pas exposée** : on y accède par un tunnel (`ssh -L`, `kubectl
  port-forward`), jamais en la publiant (leçon 8).
- **Pas de `SIGQUIT` pour voir les goroutines.** Il affiche bien la pile de toutes les goroutines, mais il
  **arrête** le processus (code de sortie 2, mesuré). En production, on interroge
  `/debug/pprof/goroutine?debug=2`, qui ne coûte qu'un instant d'arrêt du monde.
- **Une image sans shell** (leçon 3) se diagnostique de l'extérieur. On lance un conteneur d'outils qui
  partage son espace de processus et son réseau (`docker run --pid container:X --network container:X …`, ou
  `kubectl debug`). On n'installe rien dans le conteneur du service.
- **On ne modifie jamais une instance à la main** (« je redémarre juste celle-là », « je change ce
  réglage »). La correction passe par le dépôt, la CI et le déploiement. Sinon, l'instance ne correspond plus
  à aucune version, et le prochain déploiement efface la correction.

### Après l'incident

Un incident se termine par une **analyse sans recherche de coupable** (*postmortem*) : la chronologie,
l'impact mesuré (métriques), la cause, et surtout ce qui l'a rendu possible et ce qui a ralenti sa détection.
Chaque action qui en sort est un ticket. Un nouveau test, une nouvelle alerte ou un point ajouté à la liste
de durcissement (leçon 8) empêchent la même panne de revenir.

## Exemple

Un paquet `deploy` : une instance (`Backend`) avec ses sondes et ses deux façons de s'arrêter, et un
répartiteur (`Balancer`) qui ne route que vers les instances prêtes, sondées toutes les 100 ms, comme un
équilibreur de charge. Le programme `rolling` remplace deux instances une par une sous la charge de 8 clients,
avec chacune des trois méthodes d'arrêt.

```go.mod
// Fichier : go.mod
module example.com/svc

go 1.27
```

```go
// Fichier : internal/deploy/backend.go
// Package deploy : une instance de service qui sait s'arrêter sans perdre de requêtes, et un
// répartiteur qui, comme un équilibreur de charge, n'envoie du trafic qu'aux instances prêtes.
package deploy

import (
	"context"
	"errors"
	"net"
	"net/http"
	"sync/atomic"
	"time"
)

// Backend est une instance du service.
type Backend struct {
	srv      *http.Server
	ln       net.Listener
	draining atomic.Bool
	URL      string
}

// Start démarre une instance sur un port libre. work est la durée de traitement d'une requête.
func Start(work time.Duration) (*Backend, error) {
	ln, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		return nil, err
	}
	b := &Backend{ln: ln, URL: "http://" + ln.Addr().String()}
	mux := http.NewServeMux()
	// Vivant : le processus répond. Ne dépend de RIEN d'autre (ni base, ni dépendance) :
	// un échec ici fait redémarrer l'instance.
	mux.HandleFunc("GET /livez", func(w http.ResponseWriter, r *http.Request) { w.Write([]byte("ok")) })
	// Prêt : l'instance accepte du trafic. Faux pendant le démarrage et dès le début de l'arrêt.
	mux.HandleFunc("GET /readyz", func(w http.ResponseWriter, r *http.Request) {
		if b.draining.Load() {
			http.Error(w, "arrêt en cours", http.StatusServiceUnavailable)
			return
		}
		w.Write([]byte("ok"))
	})
	mux.HandleFunc("GET /work", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(work)
		w.Write([]byte("ok"))
	})
	b.srv = &http.Server{Handler: mux, ReadHeaderTimeout: 5 * time.Second}
	go b.srv.Serve(ln)
	return b, nil
}

// Kill arrête brutalement : les connexions ouvertes sont coupées, les requêtes en cours perdues.
func (b *Backend) Kill() error { return b.srv.Close() }

// Stop arrête proprement, en trois temps :
//  1. ne plus être prêt (readyz → 503) : le répartiteur cesse d'envoyer du trafic ;
//  2. attendre que le répartiteur l'ait remarqué (au moins une période de sonde) ;
//  3. Shutdown : attendre la fin des requêtes en cours, dans la limite du délai.
func (b *Backend) Stop(ctx context.Context, propagation time.Duration) error {
	b.draining.Store(true)
	select {
	case <-time.After(propagation):
	case <-ctx.Done():
		return ctx.Err()
	}
	if err := b.srv.Shutdown(ctx); err != nil && !errors.Is(err, http.ErrServerClosed) {
		return err
	}
	return nil
}
```

```go
// Fichier : internal/deploy/balancer.go
package deploy

import (
	"net/http"
	"net/http/httputil"
	"net/url"
	"sync"
	"sync/atomic"
	"time"
)

// Balancer répartit les requêtes entre les instances PRÊTES, sondées toutes les period.
type Balancer struct {
	mu       sync.Mutex
	backends map[string]*target
	ready    atomic.Pointer[[]*target]
	next     atomic.Uint64
	client   *http.Client
}

type target struct {
	url   string
	proxy *httputil.ReverseProxy
}

func NewBalancer(period time.Duration) *Balancer {
	lb := &Balancer{backends: map[string]*target{}, client: &http.Client{Timeout: period}}
	lb.ready.Store(&[]*target{})
	go func() {
		for range time.Tick(period) {
			lb.probe()
		}
	}()
	return lb
}

func (lb *Balancer) Add(rawURL string) {
	u, _ := url.Parse(rawURL)
	p := httputil.NewSingleHostReverseProxy(u)
	p.ErrorHandler = func(w http.ResponseWriter, r *http.Request, err error) {
		http.Error(w, "instance injoignable", http.StatusBadGateway)
	}
	lb.mu.Lock()
	lb.backends[rawURL] = &target{rawURL, p}
	lb.mu.Unlock()
}

func (lb *Balancer) Remove(rawURL string) {
	lb.mu.Lock()
	delete(lb.backends, rawURL)
	lb.mu.Unlock()
}

// probe interroge /readyz de chaque instance connue et publie la liste des prêtes.
func (lb *Balancer) probe() {
	lb.mu.Lock()
	all := make([]*target, 0, len(lb.backends))
	for _, t := range lb.backends {
		all = append(all, t)
	}
	lb.mu.Unlock()
	var ready []*target
	for _, t := range all {
		resp, err := lb.client.Get(t.url + "/readyz")
		if err == nil {
			resp.Body.Close()
			if resp.StatusCode == http.StatusOK {
				ready = append(ready, t)
			}
		}
	}
	lb.ready.Store(&ready)
}

func (lb *Balancer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	ready := *lb.ready.Load()
	if len(ready) == 0 {
		http.Error(w, "aucune instance prête", http.StatusServiceUnavailable)
		return
	}
	ready[lb.next.Add(1)%uint64(len(ready))].proxy.ServeHTTP(w, r)
}
```

```go
// Fichier : internal/deploy/backend_test.go
package deploy

import (
	"context"
	"net/http"
	"testing"
	"time"
)

func get(t *testing.T, url string) int {
	t.Helper()
	resp, err := http.Get(url)
	if err != nil {
		return 0 // connexion refusée ou coupée
	}
	resp.Body.Close()
	return resp.StatusCode
}

// Pendant l'arrêt propre : readyz passe à 503, livez reste à 200, et la requête en cours se termine.
func TestGracefulStop(t *testing.T) {
	b, err := Start(300 * time.Millisecond)
	if err != nil {
		t.Fatal(err)
	}
	inFlight := make(chan int)
	go func() { inFlight <- get(t, b.URL+"/work") }()
	time.Sleep(50 * time.Millisecond) // la requête est en cours

	done := make(chan error)
	go func() { done <- b.Stop(context.Background(), 100*time.Millisecond) }()
	time.Sleep(20 * time.Millisecond)
	if got := get(t, b.URL+"/readyz"); got != http.StatusServiceUnavailable {
		t.Errorf("readyz pendant l'arrêt : %d", got)
	}
	if got := get(t, b.URL+"/livez"); got != http.StatusOK {
		t.Errorf("livez pendant l'arrêt : %d (l'instance ne doit pas être tuée)", got)
	}
	if got := <-inFlight; got != http.StatusOK {
		t.Errorf("requête en cours : %d", got)
	}
	if err := <-done; err != nil {
		t.Fatal(err)
	}
	if got := get(t, b.URL+"/livez"); got != 0 {
		t.Errorf("après l'arrêt : %d", got)
	}
}

func TestKillLosesInFlight(t *testing.T) {
	b, err := Start(300 * time.Millisecond)
	if err != nil {
		t.Fatal(err)
	}
	inFlight := make(chan int)
	go func() { inFlight <- get(t, b.URL+"/work") }()
	time.Sleep(50 * time.Millisecond)
	b.Kill()
	if got := <-inFlight; got == http.StatusOK {
		t.Error("la requête en cours a survécu à Close")
	}
}
```

```go
// Fichier : internal/deploy/probe_test.go
package deploy

import (
	"context"
	"fmt"
	"net"
	"testing"
	"time"
)

// Une connexion ouverte mais sur laquelle aucune requête n'est encore arrivée (état « new »).
func TestShutdownWithNewConn(t *testing.T) {
	for _, open := range []bool{false, true} {
		b, _ := Start(time.Millisecond)
		if open {
			c, err := net.Dial("tcp", b.ln.Addr().String())
			if err != nil {
				t.Fatal(err)
			}
			defer c.Close()
			time.Sleep(10 * time.Millisecond)
		}
		start := time.Now()
		b.Stop(context.Background(), 0)
		fmt.Printf("connexion neuve ouverte=%-5v Shutdown en %v\n", open, time.Since(start).Round(10*time.Millisecond))
	}
}
```

```go
// Fichier : cmd/rolling/main.go
// Commande rolling : un déploiement progressif sous charge, avec un arrêt brutal puis propre.
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"net/http/httptest"
	"sync"
	"sync/atomic"
	"time"

	"example.com/svc/internal/deploy"
)

const (
	work   = 20 * time.Millisecond  // durée d'une requête
	period = 100 * time.Millisecond // période de sonde du répartiteur
)

func main() {
	for _, m := range []mode{brutal, shutdownOnly, graceful} {
		ok, errs := scenario(m)
		fmt.Printf("%-42s %5d requêtes réussies, %3d en erreur\n", names[m], ok, errs)
	}
}

type mode int

const (
	brutal       mode = iota // srv.Close()
	shutdownOnly             // srv.Shutdown() tout de suite, sans prévenir le répartiteur
	graceful                 // readyz à 503, attente, puis Shutdown
)

var names = map[mode]string{
	brutal:       "arrêt brutal (Close)",
	shutdownOnly: "Shutdown sans attente",
	graceful:     "arrêt propre (readyz, attente, Shutdown)",
}

func scenario(m mode) (ok, errs int64) {
	lb := deploy.NewBalancer(period)
	front := httptest.NewServer(lb)
	defer front.Close()
	var live []*deploy.Backend
	for range 2 {
		b := must(deploy.Start(work))
		lb.Add(b.URL)
		live = append(live, b)
	}
	time.Sleep(3 * period) // les deux instances sont vues prêtes

	var nOK, nErr atomic.Int64
	stop := make(chan struct{})
	var wg sync.WaitGroup
	client := &http.Client{Timeout: 2 * time.Second}
	for range 8 { // 8 clients en boucle
		wg.Go(func() {
			for {
				select {
				case <-stop:
					return
				default:
				}
				resp, err := client.Get(front.URL + "/work")
				if err == nil && resp.StatusCode == http.StatusOK {
					nOK.Add(1)
				} else {
					nErr.Add(1)
				}
				if err == nil {
					resp.Body.Close()
				}
			}
		})
	}
	// Le déploiement : on remplace les instances une par une.
	for i := range live {
		time.Sleep(time.Second)
		next := must(deploy.Start(work)) // la nouvelle version
		lb.Add(next.URL)
		time.Sleep(3 * period) // elle est prête et reçoit du trafic
		old := live[i]
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		switch m {
		case brutal:
			old.Kill()
		case shutdownOnly:
			if err := old.Stop(ctx, 0); err != nil {
				log.Fatal(err)
			}
		case graceful:
			if err := old.Stop(ctx, 3*period); err != nil {
				log.Fatal(err)
			}
		}
		cancel()
		lb.Remove(old.URL)
		live[i] = next
	}
	time.Sleep(time.Second)
	close(stop)
	wg.Wait()
	for _, b := range live {
		b.Kill()
	}
	return nOK.Load(), nErr.Load()
}

func must[T any](v T, err error) T {
	if err != nil {
		log.Fatal(err)
	}
	return v
}
```

```
$ for i in 1 2 3; do go run ./cmd/rolling; done
arrêt brutal (Close)                        1316 requêtes réussies,   7 en erreur
Shutdown sans attente                       1343 requêtes réussies,  38 en erreur
arrêt propre (readyz, attente, Shutdown)    1525 requêtes réussies,   0 en erreur
arrêt brutal (Close)                        1285 requêtes réussies,   6 en erreur
Shutdown sans attente                       1288 requêtes réussies,  32 en erreur
arrêt propre (readyz, attente, Shutdown)    1504 requêtes réussies,   0 en erreur
arrêt brutal (Close)                        1295 requêtes réussies,  41 en erreur
Shutdown sans attente                       1311 requêtes réussies,  13 en erreur
arrêt propre (readyz, attente, Shutdown)    1498 requêtes réussies,   0 en erreur
$ go test -race -v ./internal/deploy/
--- PASS: TestGracefulStop (5.43s)
--- PASS: TestKillLosesInFlight (0.05s)
connexion neuve ouverte=false Shutdown en 0s
connexion neuve ouverte=true  Shutdown en 5.33s
--- PASS: TestShutdownWithNewConn (5.34s)
PASS
ok  	example.com/svc/internal/deploy	11.849s
```

**Lecture :**

- **Seul l'arrêt propre ne perd aucune requête**, aux trois exécutions. Les deux autres méthodes perdent les
  requêtes envoyées pendant les 100 à 300 ms où le répartiteur croit encore l'instance prête.
- **Les arrêts fautifs traitent aussi moins de requêtes** (environ 1 300 contre 1 500 en 4 s). Une hypothèse,
  non vérifiée : des clients restent bloqués sur des connexions coupées avant de recevoir leur erreur.
- **`TestGracefulStop` prend 5,43 s**, et non 0,3 s. Le client du test a ouvert une connexion qu'il n'a pas
  encore utilisée, et `Shutdown` l'attend 5 s : c'est le détail `StateNew` du cours, isolé par
  `TestShutdownWithNewConn`.

Le `SIGQUIT`, sur un petit programme qui a trois goroutines bloquées :

```
$ ./quit & sleep 1; kill -QUIT %1; wait %1; echo "code de sortie $?"
SIGQUIT: quit
goroutine 1 gp=… m=nil [sleep]:
main.main()
…
goroutine 8 gp=… m=nil [chan receive]:
main.main.func1()
…
code de sortie 2
```

Le vidage est complet (12 goroutines, celles du *runtime* comprises), mais le programme est mort.

## Explication du code

| Élément | Rôle |
|---|---|
| `net.Listen("tcp", "127.0.0.1:0")` | Un port libre choisi par le système : plusieurs instances dans le même processus, comme dans les tests. |
| `/livez` | Ne vérifie rien d'autre que le serveur HTTP : un échec signifie « redémarre-moi ». |
| `/readyz` et `draining` | Un `atomic.Bool` : le gestionnaire de `SIGTERM` le passe à vrai, et les sondes le lisent sans verrou. |
| `Stop(ctx, propagation)` | Les trois temps de l'arrêt. `propagation` doit couvrir au moins une période de sonde du répartiteur. |
| `Kill` | `srv.Close()` : ferme écoute **et** connexions en cours. Les requêtes en cours sont perdues (`TestKillLosesInFlight`). |
| `Balancer.probe` | Toutes les `period`, interroge `/readyz` de chaque instance et publie la liste des prêtes (`atomic.Pointer`, N6 L7). |
| `ReverseProxy.ErrorHandler` | Une instance injoignable donne 502 au client : c'est ce que comptent les « erreurs ». |
| `lb.Add(next)` puis `time.Sleep(3 * period)` | La nouvelle version reçoit du trafic **avant** qu'on arrête l'ancienne : la capacité ne baisse jamais. |
| `TestShutdownWithNewConn` | Ouvre une connexion TCP sans y envoyer de requête, et mesure la durée de `Shutdown`. |

## Erreurs fréquentes

1. **S'arrêter dès `SIGTERM`** (`Close`, ou `Shutdown` immédiat) : les requêtes arrivées pendant le retrait
   du répartiteur échouent.
2. **Une sonde de vie qui vérifie la base** : une panne de la base fait redémarrer toutes les instances.
3. **Une seule sonde pour tout** (`/health`), qui sert à la fois à la vie et à la disponibilité.
4. **Un délai d'arrêt du service plus long que celui de l'orchestrateur** : `SIGKILL` au milieu de la
   fermeture.
5. **Une migration qui casse l'ancienne version** encore en service pendant le déploiement, et rend le
   retour arrière impossible.
6. **Déployer `latest`** : on ne sait plus revenir à « la version d'avant ».
7. **`SIGQUIT` en production** pour voir les goroutines : le service s'arrête.
8. **Corriger une instance à la main** pendant un incident.
9. **Diagnostiquer avant de rétablir.**

## Bonnes pratiques Go

- `/livez` trivial, `/readyz` qui reflète l'état local (démarrage, arrêt en cours), sur l'adresse publique
  ou celle de l'orchestrateur, mais sans détails internes.
- Sur `SIGTERM` : `/readyz` → 503, attente de propagation, `Shutdown(ctx)` avec un délai, fermeture des
  ressources, sortie.
- Délais emboîtés et notés dans la configuration (leçon 5) : propagation + plus longue requête < délai de
  l'orchestrateur.
- Images désignées par une étiquette immuable ; migrations en deux temps ; drapeaux pour désactiver sans
  redéployer.
- Diagnostic par l'adresse d'administration (profils, goroutines), les métriques, les traces et les journaux,
  jamais par une modification à la main.
- Une analyse après chaque incident, dont les actions deviennent des tests et des alertes.

## Ce que je dois retenir

- Un déploiement progressif fait tourner deux versions en même temps : code et schéma doivent le supporter.
- Vie : « redémarre-moi » ; disponibilité : « ne m'envoie plus de trafic ». Jamais une dépendance dans la vie.
- Arrêt : ne plus être prêt, attendre, `Shutdown`, fermer. Mesuré : 0 erreur, contre 6 à 41 sinon.
- `Shutdown` peut attendre 5 s une connexion neuve inutilisée : à compter dans le budget d'arrêt.
- En incident : rétablir d'abord (retour arrière, drapeau), comprendre ensuite, sans toucher aux instances.

---

➡️ [Exercices](exercices.md)
