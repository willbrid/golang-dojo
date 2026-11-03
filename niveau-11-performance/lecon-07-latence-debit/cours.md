# Leçon 7 — Latence et débit

> Un service qui répond en 2 ms au repos peut répondre en 100 ms à 99 % de sa capacité, sans que son code
> ait changé d'une ligne. La latence d'un service ne dépend pas seulement de ce qu'il fait, mais de **combien
> de travail attend devant lui**. Cette leçon donne les trois outils pour raisonner sur un service sous
> charge. Les **percentiles** décrivent la latence. La **loi de Little** relie débit, latence et
> concurrence. La **théorie des files d'attente** explique pourquoi la latence explose près de la saturation.
> La leçon montre enfin comment écrire un test de charge qui ne ment pas.

## Objectifs

- Distinguer débit et latence, et décrire une latence par ses percentiles.
- Comprendre pourquoi la queue de distribution compte davantage dès qu'une requête en déclenche plusieurs.
- Appliquer la loi de Little pour dimensionner un pool, une file, une concurrence.
- Savoir pourquoi la latence croît en 1/(1 − ρ) avec le taux d'occupation ρ.
- Savoir pourquoi une file commune vaut mieux qu'une file par worker.
- Écrire un générateur de charge en boucle ouverte, et reconnaître l'**omission coordonnée**.

## Explication

### Débit et latence

- Le **débit** (*throughput*), noté X ou λ, est le nombre de requêtes traitées par seconde.
- La **latence**, notée W, est la durée d'une requête vue par le client : attente plus traitement.

Ce sont deux grandeurs **différentes**. Ajouter des workers peut augmenter le débit sans changer la latence.
Un cache peut réduire la latence sans changer le débit maximal.

Une latence n'est pas un nombre, c'est une **distribution**. La **moyenne** la résume mal, car quelques
requêtes très lentes la déplacent à peine (N10 L10). On donne donc des **percentiles** : le p50 (la médiane),
le p99 (99 % des requêtes sont plus rapides) et le p99,9.

Le p99 n'est pas un détail réservé aux 1 % de malchanceux. Une page qui appelle 10 services indépendants est
lente dès que **l'un** d'eux est lent. Si chacun dépasse son p99 une fois sur cent, la page le fait avec une
probabilité 1 − 0,99¹⁰ ≈ 10 %. Avec 100 appels, c'est 63 %. **Plus un système est découpé, plus la queue de
distribution de chaque composant devient la latence typique de l'ensemble.**

### La loi de Little

Pour tout système stable, en moyenne sur une durée suffisante :

```
L = λ × W
nombre moyen de requêtes présentes = débit × temps moyen passé dans le système
```

Cette loi ne suppose **rien** sur la distribution des arrivées ou des durées. C'est presque une
tautologie, mais elle sert sans cesse.

- **Dimensionner un pool.** 2 000 requêtes/s qui gardent chacune une connexion à la base 5 ms occupent en
  moyenne 2 000 × 0,005 = **10 connexions**. Un pool de 10 est donc saturé en permanence. Il faut de la
  marge pour les rafales, par exemple 20 à 30 (N8 L5).
- **Dimensionner une file.** Une file de 1 000 requêtes vidée à 500 requêtes/s fait attendre la dernière
  1 000 / 500 = **2 s** (N6 L11).
- **Comprendre un générateur de charge.** Un générateur à N goroutines qui attendent chacune leur réponse
  a toujours exactement N requêtes en cours : L = N. Son débit vaut donc **X = N / W**. Si le serveur
  ralentit, le générateur ralentit aussi. On y revient plus bas.

### Le taux d'occupation et l'explosion de la latence

Soit un worker qui traite μ requêtes/s et en reçoit λ. Son **taux d'occupation** est ρ = λ / μ.

Le cas le plus simple a des arrivées aléatoires et indépendantes, et des durées de service aléatoires de
moyenne S = 1/μ. Pour ce cas (la file « M/M/1 »), la théorie donne :

```
temps de séjour moyen  W = S / (1 − ρ)
p99 du temps de séjour = W × ln(100) ≈ 4,6 × W
```

À 50 % d'occupation, une requête de 1 ms en prend 2 en moyenne. À 90 %, 10. À 99 %, 100. Le service
n'est pas plus lent : la requête attend derrière les autres. Les arrivées aléatoires forment des **rafales**,
et pendant une rafale la file se remplit plus vite que le worker ne la vide. Plus la marge 1 − ρ est petite,
plus il faut de temps pour rattraper le retard.

L'exemple vérifie ces formules par **simulation** : on tire les arrivées et les durées au hasard, et on
calcule pour chaque requête quand elle commence et quand elle finit. Les conséquences pratiques :

- **Ne pas viser 100 % d'utilisation.** Un service dont le processeur est à 95 % n'est pas « bien utilisé ».
  Ses clients attendent 20 fois la durée de service. Une cible courante se situe vers 60 à 70 % aux heures
  de pointe.
- **Réduire la variabilité aide autant qu'accélérer.** Des durées de service régulières font moins
  attendre que des durées très dispersées, à moyenne égale (exercice E3).
- **Une file commune vaut mieux qu'une file par worker.** Avec une file par worker, une requête peut attendre
  derrière un worker occupé alors qu'un autre est libre. Avec une file commune, le premier worker libre la
  prend. C'est ce que fait un canal partagé entre plusieurs goroutines (N6 L9). L'exemple mesure l'écart.

### Les tests de charge, et l'omission coordonnée

Il existe deux façons d'envoyer de la charge.

- **En boucle fermée.** N goroutines envoient une requête, attendent la réponse, recommencent. C'est le
  générateur de la leçon 1, et celui de la plupart des outils simples.
- **En boucle ouverte.** Les requêtes partent selon un **calendrier** (par exemple 1 000 par seconde),
  qu'elles aient ou non reçu les réponses précédentes. C'est ainsi que se comportent de vrais utilisateurs :
  ils ne se concertent pas pour attendre que le serveur aille mieux.

En boucle fermée, quand le serveur se bloque, les N goroutines se bloquent avec lui et **n'envoient plus
rien**. Pendant une panne de 500 ms, le générateur n'enregistre que N requêtes lentes, au lieu des centaines
que de vrais clients auraient subies. Le générateur s'est **coordonné** avec le serveur et a **omis** de
mesurer les requêtes qui auraient dû partir. C'est l'**omission coordonnée**. L'exemple la mesure : la même
panne donne un p99 de 3 ms en boucle fermée et de 488 ms en boucle ouverte.

Le remède a deux parties :

1. un générateur en **boucle ouverte** ;
2. une latence mesurée depuis l'instant où la requête **devait** partir, pas depuis celui où elle est
   partie.

Les autres règles d'un test de charge :

- Monter **par paliers** (50 %, 70 %, 90 %, 110 % de la capacité estimée), et relever débit, p50, p99 et taux
  d'erreurs à chaque palier. Le point où le p99 décolle est la vraie capacité utile.
- Faire tourner le générateur sur **une autre machine** que le serveur, ou au moins vérifier qu'il n'est pas
  lui-même saturé. Un générateur à 100 % de processeur mesure sa propre lenteur.
- Durer assez longtemps pour voir les cycles du GC, les remplissages de cache, les rotations de connexions :
  des minutes, pas des secondes.
- Garder les **durées brutes**, ou un histogramme, et jamais une moyenne de moyennes : les percentiles ne
  s'additionnent pas et ne se moyennent pas.

### Quand ne pas s'en soucier

Pour un outil interne à 3 requêtes par minute, la latence se résume à la durée de traitement, et
aucune file ne se forme. Ces raisonnements comptent dès qu'un service partage une ressource (un processeur,
un pool, un verrou) entre des requêtes qui arrivent sans se concerter. C'est le cas de presque tous les
services en production.

## Exemple

Un paquet `lat` en deux parties. La première est une **simulation** de files d'attente, déterministe et
vérifiable contre la théorie. La seconde, un **vrai serveur** HTTP qui tombe en panne 500 ms, attaqué par
deux générateurs de charge.

```go
// Fichier : lat.go
// Package lat : latence, débit, files d'attente et tests de charge.
package lat

import (
	"math"
	"math/rand/v2"
	"slices"
)

// Percentile renvoie le p-ième percentile (méthode du rang le plus proche) d'une slice triée.
func Percentile(sorted []float64, p float64) float64 {
	if len(sorted) == 0 {
		return math.NaN()
	}
	i := int(math.Ceil(p/100*float64(len(sorted)))) - 1
	return sorted[min(max(i, 0), len(sorted)-1)]
}

func Mean(s []float64) float64 {
	t := 0.0
	for _, x := range s {
		t += x
	}
	return t / float64(len(s))
}

// Sim est le résultat d'une simulation : instants d'arrivée et de départ de chaque requête.
type Sim struct {
	Arrive, Depart []float64
}

// Simulate fait passer n requêtes dans une file FIFO servie par c workers. Les arrivées sont
// aléatoires (λ par unité de temps en moyenne), les durées de service aussi (1/μ en moyenne).
// shared : une seule file pour tous les workers ; sinon, chaque requête est envoyée au hasard
// dans la file d'un worker.
func Simulate(seed uint64, lambda, mu float64, c, n int, shared bool) Sim {
	rng := rand.New(rand.NewPCG(seed, 1))
	free := make([]float64, c) // instant où chaque worker aura fini son travail
	s := Sim{Arrive: make([]float64, n), Depart: make([]float64, n)}
	t := 0.0
	for i := range n {
		t += rng.ExpFloat64() / lambda
		w := rng.IntN(c)
		if shared {
			w = slices.Index(free, slices.Min(free)) // le premier worker libre
		}
		start := max(t, free[w])
		free[w] = start + rng.ExpFloat64()/mu
		s.Arrive[i], s.Depart[i] = t, free[w]
	}
	return s
}

// Sojourns renvoie les temps de séjour (attente + service), triés.
func (s Sim) Sojourns() []float64 {
	d := make([]float64, len(s.Arrive))
	for i := range d {
		d[i] = s.Depart[i] - s.Arrive[i]
	}
	slices.Sort(d)
	return d
}

// InSystem mesure le nombre moyen de requêtes présentes (en file ou en service) en l'observant à k
// instants tirés au hasard : arrivées avant t moins départs avant t.
func (s Sim) InSystem(seed uint64, k int) float64 {
	rng := rand.New(rand.NewPCG(seed, 2))
	dep := slices.Sorted(slices.Values(s.Depart))
	end := s.Arrive[len(s.Arrive)-1]
	total := 0
	for range k {
		t := rng.Float64() * end
		a, _ := slices.BinarySearch(s.Arrive, t)
		d, _ := slices.BinarySearch(dep, t)
		total += a - d
	}
	return float64(total) / float64(k)
}
```

```go
// Fichier : load.go
package lat

import (
	"io"
	"net/http"
	"slices"
	"sync"
	"time"
)

// StallHandler répond en 1 ms environ, sauf pendant une panne de pauseFor qui commence pauseAt après
// start : les requêtes qui arrivent pendant la panne attendent qu'elle se termine (un verrou tenu,
// un disque saturé, une base qui bascule…).
func StallHandler(start time.Time, pauseAt, pauseFor time.Duration) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		since := time.Since(start)
		if since >= pauseAt && since < pauseAt+pauseFor {
			time.Sleep(pauseAt + pauseFor - since)
		}
		time.Sleep(time.Millisecond)
		io.WriteString(w, "ok")
	})
}

func get(c *http.Client, url string) error {
	resp, err := c.Get(url)
	if err != nil {
		return err
	}
	_, err = io.Copy(io.Discard, resp.Body)
	resp.Body.Close()
	return err
}

// ms convertit une durée en millisecondes.
func ms(d time.Duration) float64 { return float64(d) / float64(time.Millisecond) }

// Closed est un générateur en boucle FERMÉE : workers goroutines envoient chacune une requête, attendent
// la réponse, recommencent. Chaque durée est mesurée depuis l'envoi effectif. Renvoie les durées triées
// (en ms).
func Closed(c *http.Client, url string, workers int, d time.Duration) []float64 {
	var mu sync.Mutex
	var all []float64
	deadline := time.Now().Add(d)
	var wg sync.WaitGroup
	for range workers {
		wg.Go(func() {
			var local []float64
			for time.Now().Before(deadline) {
				t0 := time.Now()
				if get(c, url) == nil {
					local = append(local, ms(time.Since(t0)))
				}
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

// Open est un générateur en boucle OUVERTE : une requête est lancée toutes les 1/rate secondes, sans
// attendre les précédentes. Chaque durée est mesurée depuis l'instant où la requête DEVAIT partir.
func Open(c *http.Client, url string, rate float64, d time.Duration) []float64 {
	interval := time.Duration(float64(time.Second) / rate)
	n := int(d / interval)
	out := make([]float64, n)
	ok := make([]bool, n)
	start := time.Now()
	var wg sync.WaitGroup
	for i := range n {
		planned := start.Add(time.Duration(i) * interval)
		time.Sleep(time.Until(planned))
		wg.Go(func() {
			if get(c, url) == nil {
				out[i], ok[i] = ms(time.Since(planned)), true
			}
		})
	}
	wg.Wait()
	var res []float64
	for i, v := range out {
		if ok[i] {
			res = append(res, v)
		}
	}
	slices.Sort(res)
	return res
}
```

```go
// Fichier : lat_test.go
package lat

import (
	"fmt"
	"math"
	"testing"
)

func TestPercentile(t *testing.T) {
	s := []float64{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	for _, tc := range []struct{ p, want float64 }{{50, 5}, {90, 9}, {99, 10}, {100, 10}, {0, 1}} {
		if got := Percentile(s, tc.p); got != tc.want {
			t.Errorf("p%v = %v, attendu %v", tc.p, got, tc.want)
		}
	}
}

// Loi de Little : nombre moyen présent = débit × temps moyen de séjour, mesurés indépendamment.
func TestLittle(t *testing.T) {
	for _, rho := range []float64{0.5, 0.9} {
		s := Simulate(1, rho, 1, 1, 200_000, true)
		w := Mean(s.Sojourns())
		throughput := float64(len(s.Arrive)) / s.Arrive[len(s.Arrive)-1]
		l := s.InSystem(1, 100_000)
		t.Logf("ρ=%.1f : L observé = %.2f ; λ×W = %.2f × %.2f = %.2f", rho, l, throughput, w, throughput*w)
		if math.Abs(l-throughput*w)/l > 0.1 {
			t.Errorf("Little non vérifiée : %.2f contre %.2f", l, throughput*w)
		}
	}
}

// Un worker, service moyen 1 ms : temps de séjour selon le taux d'occupation ρ = λ/μ.
func TestUtilization(t *testing.T) {
	fmt.Println("  ρ   moyenne (théorie)   p99 (théorie)")
	for _, rho := range []float64{0.5, 0.7, 0.8, 0.9, 0.95, 0.99} {
		s := Simulate(1, rho, 1, 1, 1_000_000, true).Sojourns()
		th := 1 / (1 - rho) // file M/M/1 : séjour exponentiel de taux μ−λ
		fmt.Printf("%.2f  %6.1f ms (%5.1f)   %6.1f ms (%5.1f)\n",
			rho, Mean(s), th, Percentile(s, 99), th*math.Log(100))
	}
}

// Quatre workers à 90 % : une file commune, ou une file par worker.
func TestPooling(t *testing.T) {
	for _, shared := range []bool{true, false} {
		s := Simulate(1, 3.6, 1, 4, 1_000_000, shared).Sojourns()
		fmt.Printf("file commune=%-5v moyenne %5.1f ms  p99 %5.1f ms\n", shared, Mean(s), Percentile(s, 99))
	}
}
```

```go
// Fichier : load_test.go
package lat

import (
	"fmt"
	"net/http"
	"net/http/httptest"
	"testing"
	"time"
)

func report(name string, d []float64, dur time.Duration) {
	fmt.Printf("%-15s %5d req. (%4.0f/s)  moyenne %5.1f  p50 %4.1f  p99 %5.1f  p99,9 %5.1f  max %5.1f (ms)\n",
		name, len(d), float64(len(d))/dur.Seconds(), Mean(d), Percentile(d, 50), Percentile(d, 99),
		Percentile(d, 99.9), d[len(d)-1])
}

// Le même serveur, la même panne de 500 ms, deux générateurs.
func TestCoordinatedOmission(t *testing.T) {
	const dur = 3 * time.Second
	c := &http.Client{Transport: &http.Transport{MaxIdleConnsPerHost: 1000}}
	for _, mode := range []string{"boucle fermée", "boucle ouverte"} {
		srv := httptest.NewServer(StallHandler(time.Now(), time.Second, 500*time.Millisecond))
		var d []float64
		if mode == "boucle fermée" {
			d = Closed(c, srv.URL, 8, dur)
		} else {
			d = Open(c, srv.URL, 1000, dur)
		}
		srv.Close()
		report(mode, d, dur)
	}
}
```

```
$ go test -v -run 'Percentile|Little|Utilization|Pooling'
--- PASS: TestPercentile (0.00s)
    lat_test.go:25: ρ=0.5 : L observé = 0.98 ; λ×W = 0.50 × 1.98 = 0.99
    lat_test.go:25: ρ=0.9 : L observé = 8.25 ; λ×W = 0.90 × 9.25 = 8.28
--- PASS: TestLittle (0.15s)
  ρ   moyenne (théorie)   p99 (théorie)
0.50     2.0 ms (  2.0)      9.3 ms (  9.2)
0.70     3.3 ms (  3.3)     15.7 ms ( 15.4)
0.80     5.0 ms (  5.0)     23.4 ms ( 23.0)
0.90     9.7 ms ( 10.0)     43.6 ms ( 46.1)
0.95    19.5 ms ( 20.0)     87.2 ms ( 92.1)
0.99    92.5 ms (100.0)    392.5 ms (460.5)
--- PASS: TestUtilization (0.84s)
file commune=true  moyenne   2.9 ms  p99  11.6 ms
file commune=false moyenne   9.8 ms  p99  44.7 ms
--- PASS: TestPooling (0.30s)
```

Ces chiffres sont **déterministes** : la graine est fixée, et on retrouve exactement les mêmes en
relançant. La simulation n'utilise pas d'horloge.

- **La loi de Little est vérifiée.** Le nombre moyen de requêtes présentes, **observé** à 100 000 instants
  tirés au hasard, coïncide avec débit × séjour moyen (0,98 contre 0,99, et 8,25 contre 8,28).
- **La formule de la file M/M/1 est retrouvée** à quelques pour cent près. À 99 %, l'écart grandit : plus le
  système est chargé, plus il lui faut de requêtes pour atteindre son régime moyen. Un million n'y suffit
  pas encore.
- **Passer de 50 % à 90 % d'occupation multiplie le temps de séjour par 5** (2 ms → 9,7 ms), et le p99 par
  4,7. Le code du service, lui, n'a pas changé.
- **Quatre workers à 90 %** : une file commune donne une moyenne de 2,9 ms, quatre files séparées 9,8 ms.
  Quatre files séparées, ce sont quatre files M/M/1 indépendantes, chacune à 90 %.

Le test de charge, sur un vrai serveur (les durées réelles varient d'une exécution à l'autre ; deux
exécutions ont donné des p99 en boucle ouverte de 485 et 492 ms) :

```
$ go test -v -run Coordinated
boucle fermée   11582 req. (3861/s)  moyenne   2.1  p50  1.6  p99   2.9  p99,9   4.2  max 501.6 (ms)
boucle ouverte   3000 req. (1000/s)  moyenne  46.5  p50  2.6  p99 487.8  p99,9 515.7  max 518.6 (ms)
```

Le **même** serveur subit la **même** panne de 500 ms :

- **En boucle fermée**, le p99 vaut 2,9 ms et le p99,9 4,2 ms. La panne n'apparaît que dans le maximum.
  Un rapport de test « p99 < 3 ms » serait vrai et pourtant trompeur.
- **En boucle ouverte**, les 500 requêtes arrivées pendant la panne ont attendu jusqu'à 500 ms, soit 17 % du
  total. Le p99 vaut 488 ms. C'est ce qu'auraient vu de vrais utilisateurs.
- **La loi de Little explique le débit de la boucle fermée** : 8 requêtes toujours en cours, une durée moyenne
  de 2,1 ms, donc 8 / 0,0021 ≈ 3 800 requêtes/s (mesuré : 3 861). Ce débit mesure le **générateur**, pas la
  capacité du serveur. En boucle ouverte, c'est l'inverse : le débit est imposé (1 000/s), et la loi donne le
  nombre moyen de requêtes en cours : 1 000 × 0,0465 ≈ 46.

## Explication du code

| Élément | Rôle |
|---|---|
| `Percentile` | Rang le plus proche : l'élément d'indice ⌈p/100 × n⌉ − 1 d'une slice **triée**. Sur des durées brutes, pas besoin d'interpolation. |
| `Simulate` | Pour chaque requête : instant d'arrivée (écart exponentiel de moyenne 1/λ, des arrivées « au hasard »), worker choisi, début = max(arrivée, fin du worker), fin = début + service. Une simulation **à événements** sans horloge réelle, donc rapide et reproductible. |
| `shared` | File commune : la requête va au worker qui se libère le premier. Files séparées : à un worker tiré au hasard, même s'il est occupé. |
| `InSystem` | Mesure **indépendante** de L. Le nombre présent à l'instant t vaut les arrivées avant t moins les départs avant t, obtenus par deux recherches dichotomiques. |
| `StallHandler` | Un serveur de 1 ms qui « gèle » de 1 s à 1,5 s après son démarrage. On simule une panne réaliste : un verrou tenu, une bascule de base de données, une pause. |
| `Closed` | N goroutines qui envoient, attendent, recommencent. Durée mesurée depuis l'envoi **effectif**. |
| `Open` | Une goroutine par requête, lancée à l'instant prévu `planned`. La durée est mesurée depuis `planned` : si le générateur lui-même prend du retard, ce retard est compté. |
| `MaxIdleConnsPerHost: 1000` | Par défaut, le client ne garde que 2 connexions inactives par hôte. En boucle ouverte, des centaines de requêtes simultanées rouvriraient sans cesse des connexions (N7 L6). |

## Erreurs fréquentes

1. **Ne regarder que la moyenne**, ou seulement le p50.
2. **Moyenner des percentiles** (le p99 moyen de 10 serveurs n'est pas le p99 global). Il faut agréger les
   histogrammes, puis calculer le percentile.
3. **Viser 90–100 % d'utilisation** pour « rentabiliser » les machines, et découvrir la latence au premier pic.
4. **Mesurer avec un générateur en boucle fermée** et conclure que le service tient sa latence pendant les
   pannes.
5. **Mesurer la durée depuis l'envoi effectif** dans un générateur qui prend du retard.
6. **Dimensionner un pool ou une file au hasard**, au lieu de faire le calcul de Little avec le débit et la
   durée réels.
7. **Une file par worker** (un canal par goroutine, avec répartition circulaire) là où une file commune suffit.

## Bonnes pratiques Go

- Des histogrammes pour les durées (N10 L10) ; p50, p99, p99,9 dans les tableaux de bord et les objectifs.
- L = λ × W pour dimensionner `SetMaxOpenConns`, la taille des canaux, les sémaphores de concurrence.
- Une marge de capacité : la cible d'utilisation aux heures de pointe se fixe d'après le p99 mesuré par
  paliers, pas d'après le débit maximal.
- Un seul canal partagé par un pool de workers, plutôt qu'un canal par worker.
- Des tests de charge en boucle ouverte, avec une latence mesurée depuis l'instant prévu, et un générateur
  qui n'est pas saturé.

## Ce que je dois retenir

- La latence est une distribution ; le p99 de chaque composant devient la latence typique d'un appel qui
  en touche beaucoup.
- Loi de Little : L = λ × W, toujours vraie en moyenne, et l'outil de dimensionnement de base.
- La latence croît en 1 / (1 − ρ) : ×5 entre 50 % et 90 % d'occupation, ×50 à 99 %.
- Une file commune bat des files séparées (×3,4 mesuré sur 4 workers à 90 %).
- Un générateur en boucle fermée cache les pannes (omission coordonnée) : p99 de 3 ms au lieu de 488 ms.

---

➡️ [Exercices](exercices.md)
