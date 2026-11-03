# Leçon 3 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Lire `gctrace`

Cycles 5 à 10 (extrait mesuré, `GOGC=100`) :

```
gc 5   0.021+7.4+0.030 ms   61->64->37 MB, 72 MB goal
gc 6   0.019+9.0+0.043 ms   63->67->37 MB, 74 MB goal
gc 7   0.024+10+0.020 ms    65->69->38 MB, 76 MB goal
gc 8   0.032+17+0.016 ms    65->69->39 MB, 76 MB goal
gc 9   0.036+22+0.10 ms     66->72->39 MB, 78 MB goal
gc 10  0.021+20+0.019 ms    67->73->40 MB, 79 MB goal
```

L'objectif d'un cycle vaut environ **deux fois** le tas vivant du cycle **précédent** : vivant 37 Mio au cycle
5 → objectif 74 Mio au cycle 6 ; 38 → 76 ; 39 → 78. La formule est vérifiée (à l'arrondi près ; le
*runtime* y ajoute aussi la taille des piles et des globales, ici nulles). Les pauses (0,02 à 0,1 ms) sont
**cent à mille fois** plus courtes que le marquage concurrent (7 à 22 ms) : c'est le marquage qui coûte.

## E2 — Moins de pointeurs

```
pointeurs (cours)      durée=1.85s   cycles=33   tas-pic≈ 93 Mio  CPU-GC=42.7 %
valeurs (E2)           durée=740ms   cycles=43   tas-pic≈ 62 Mio  CPU-GC= 0.7 %
```

Plus de cycles (43, le tas est plus petit donc l'objectif est atteint plus vite), mais un GC presque
**gratuit** : un `[]vnode` sans pointeur est **un seul** objet que le GC n'a pas besoin de parcourir (il sait,
par le type, qu'il ne contient aucun pointeur). Il ne marque que les quelques objets éphémères encore
atteignables. Deux fois et demie plus rapide, pour le même travail. C'est l'une des optimisations les plus
efficaces pour les gros ensembles en mémoire : des slices de valeurs, des indices plutôt que des pointeurs.

## E3 — Moins d'allocations

```
réutilise (E3)         durée=600ms   cycles=0    tas-pic≈ 35 Mio
```

**Aucun** cycle : rien n'est alloué dans la boucle (on recopie les valeurs dans un objet existant). Le GC
n'a rien à faire. (La part du GC s'affiche `NaN` : les métriques `/cpu/classes/…` ne sont mises à jour
qu'à chaque cycle, et il n'y en a eu aucun.) Ne pas allouer reste le meilleur réglage du GC.

## E4 — La limite dans un conteneur

Mesuré dans un conteneur limité à 100 Mio (`docker run --memory=100m`) :

```
GOGC=400                      → tué par le noyau (code de sortie 137, manque de mémoire)
GOGC=400 GOMEMLIMIT=85MiB     → durée=4.35s cycles=119 tas-pic≈74 Mio CPU-GC=54.7 %
```

Sans limite, le *runtime* ignore la taille du conteneur : avec `GOGC=400`, le tas vise 5 fois le tas vivant,
dépasse 100 Mio, et le noyau tue le processus **sans** avertissement (aucune erreur Go, aucun `defer`). Avec
`GOMEMLIMIT`, le GC travaille plus (55 % du CPU) mais le programme termine. Dans un conteneur, `GOMEMLIMIT`
n'est pas une optimisation : c'est une condition de survie.

## E5 — `runtime.GC()`

```
runtime.GC() toutes les 100 000 itérations   durée=6.68s   cycles=200   CPU-GC=55.5 %
```

Trois fois et demie plus lent que sans (1,85 s) : 200 cycles **complets** et **bloquants** (`runtime.GC()`
attend la fin du cycle), au lieu de 33 cycles concurrents. Un appel explicite se justifie rarement : dans un
test qui mesure la mémoire vivante, juste avant une mesure ; ou après un pic ponctuel connu (le chargement
d'un gros fichier à libérer), suivi de `debug.FreeOSMemory()` si l'on veut rendre la mémoire au système.

## Exercice intermédiaire — Un tableau de bord du GC

```go
// Fichier : gcstats/gcstats.go
// Exercice intermédiaire : lire le GC avec runtime/metrics, sans pause.
package gcstats

import (
	"context"
	"fmt"
	"io"
	"math"
	"runtime/metrics"
	"sync"
	"time"
)

type Snapshot struct {
	Live, Goal uint64
	Cycles     uint64
	GCShare    float64 // part du CPU consacrée au GC depuis la lecture précédente (0 à 1)
	PauseMax   time.Duration
	PauseP99   time.Duration
}

var names = []string{
	"/gc/heap/live:bytes",
	"/gc/heap/goal:bytes",
	"/gc/cycles/total:gc-cycles",
	"/cpu/classes/gc/total:cpu-seconds",
	"/cpu/classes/total:cpu-seconds",
	"/sched/pauses/total/gc:seconds",
}

type Collector struct {
	mu       sync.Mutex
	samples  []metrics.Sample
	prevGC   float64
	prevAll  float64
	last     Snapshot
	high     int // nombre de lectures consécutives au-dessus du seuil
	Log      io.Writer
	Treshold float64
	Sustain  int
}

func New(log io.Writer) *Collector {
	c := &Collector{Log: log, Treshold: 0.25, Sustain: 10}
	for _, n := range names {
		c.samples = append(c.samples, metrics.Sample{Name: n})
	}
	return c
}

// Read lit les métriques ; à appeler à intervalle régulier.
func (c *Collector) Read() Snapshot {
	c.mu.Lock()
	defer c.mu.Unlock()
	metrics.Read(c.samples)
	s := Snapshot{
		Live:   c.samples[0].Value.Uint64(),
		Goal:   c.samples[1].Value.Uint64(),
		Cycles: c.samples[2].Value.Uint64(),
	}
	gc, all := c.samples[3].Value.Float64(), c.samples[4].Value.Float64()
	if d := all - c.prevAll; d > 0 {
		s.GCShare = (gc - c.prevGC) / d
	}
	c.prevGC, c.prevAll = gc, all
	s.PauseMax, s.PauseP99 = quantiles(c.samples[5].Value.Float64Histogram())
	if s.GCShare > c.Treshold {
		if c.high++; c.high == c.Sustain && c.Log != nil {
			fmt.Fprintf(c.Log, "alerte : GC à %.0f %% du CPU depuis %d lectures\n", s.GCShare*100, c.Sustain)
		}
	} else {
		c.high = 0
	}
	c.last = s
	return s
}

// quantiles : maximum et p99 d'un histogramme de runtime/metrics (bornes supérieures des cases).
func quantiles(h *metrics.Float64Histogram) (maxD, p99 time.Duration) {
	var total uint64
	for _, n := range h.Counts {
		total += n
	}
	if total == 0 {
		return 0, 0
	}
	var seen uint64
	target := uint64(math.Ceil(float64(total) * 0.99))
	for i, n := range h.Counts {
		if n == 0 {
			continue
		}
		upper := h.Buckets[i+1]
		if math.IsInf(upper, 1) {
			upper = h.Buckets[i]
		}
		seen += n
		if p99 == 0 && seen >= target {
			p99 = time.Duration(upper * float64(time.Second))
		}
		maxD = time.Duration(upper * float64(time.Second))
	}
	return maxD, p99
}

// Run lit toutes les period jusqu'à l'annulation.
func (c *Collector) Run(ctx context.Context, period time.Duration) {
	t := time.NewTicker(period)
	defer t.Stop()
	for {
		select {
		case <-ctx.Done():
			return
		case <-t.C:
			c.Read()
		}
	}
}

func (c *Collector) WriteText(w io.Writer) {
	c.mu.Lock()
	s := c.last
	c.mu.Unlock()
	fmt.Fprintf(w, "tas vivant %d Mio, objectif %d Mio, cycles %d, GC %.1f %% du CPU, pauses max %v / p99 %v\n",
		s.Live>>20, s.Goal>>20, s.Cycles, s.GCShare*100, s.PauseMax, s.PauseP99)
}
```

```go
// Fichier : gcstats/gcstats_test.go
package gcstats

import (
	"bytes"
	"strings"
	"testing"
)

var sink []byte

func TestLowAndHighAllocation(t *testing.T) {
	var log bytes.Buffer
	c := New(&log)
	c.Sustain = 3
	c.Read()
	quiet := c.Read() // aucune allocation entre les deux lectures
	keep := make([][]byte, 0, 4096)
	maxShare := 0.0
	for range 5 {
		for i := range 200_000 { // beaucoup d'allocations, un peu de tas vivant
			sink = make([]byte, 256)
			if i%100 == 0 {
				keep = append(keep, sink)
				if len(keep) == cap(keep) {
					keep = keep[:0]
				}
			}
		}
		maxShare = max(maxShare, c.Read().GCShare)
	}
	busy := c.Read()
	var out bytes.Buffer
	c.WriteText(&out)
	t.Logf("calme : GC %.1f %% ; chargé : GC jusqu'à %.0f %% ; %s", quiet.GCShare*100, maxShare*100, strings.TrimSpace(out.String()))
	if maxShare < 0.05 {
		t.Fatalf("part du GC sous charge : %.1f %%", maxShare*100)
	}
	if busy.Cycles <= quiet.Cycles {
		t.Fatal("aucun cycle pendant la charge")
	}
	if busy.PauseMax <= 0 || busy.PauseP99 > busy.PauseMax {
		t.Fatalf("pauses : max %v p99 %v", busy.PauseMax, busy.PauseP99)
	}
	t.Log("journal :", log.String())
}
```

`go test ./...` passe. Mesuré : à l'arrêt, part du GC 0 % ; sous charge, jusqu'à 7 %, 54 cycles, pauses
maximales ≈ 0,2 ms, p99 ≈ 0,1 ms.

Points notables :

- la part du GC est une **différence** entre deux lectures des compteurs cumulés (même principe que
  `db.Stats()`, N8 L5) ; et ces compteurs ne bougent qu'aux cycles : une lecture sans cycle depuis la
  précédente donne 0 ;
- les pauses arrivent sous forme d'**histogramme** (`Float64Histogram`) : on en tire le maximum et le p99 en
  cumulant les cases (bornes supérieures) ;
- l'alerte n'est émise que si le seuil est dépassé pendant `Sustain` lectures **consécutives** (une pointe
  isolée ne compte pas, N8 L5 défi) ;
- **pourquoi `runtime/metrics`** : `runtime.ReadMemStats` arrête brièvement le monde à chaque appel ; le lire
  chaque seconde ajoute des pauses au programme qu'on observe. `runtime/metrics` est conçu pour une lecture
  périodique, sans pause.

## Défi — Trouver le bon réglage

Reproduit à l'échelle ¼ (conteneur de 128 Mio au lieu de 512 ; tas vivant de 30 à 50 Mio au lieu de 120 à
200), avec ce programme :

```go
// Fichier : varload/main.go
// Défi : un tas vivant qui varie (30 → 50 → 30 Mio), des allocations soutenues.
package main

import (
	"fmt"
	"runtime"
	"runtime/metrics"
	"time"
)

type obj struct {
	next *obj
	pad  [56]byte // 64 octets
}

var sink *obj

func main() {
	const mib = 1 << 20
	per := 64
	var live []*obj
	samples := []metrics.Sample{{Name: "/cpu/classes/gc/total:cpu-seconds"}, {Name: "/cpu/classes/total:cpu-seconds"}}
	start := time.Now()
	var peak uint64
	for step := range 40 { // 40 étapes : le tas vivant monte puis redescend
		phase := step % 20
		if phase > 10 {
			phase = 20 - phase
		}
		target := (30 + 2*phase) * mib / per // 30 à 50 Mio d'objets vivants
		for len(live) < target {
			live = append(live, &obj{})
		}
		live = live[:target]
		for range 1_000_000 { // allocations éphémères soutenues
			sink = &obj{}
		}
		var m runtime.MemStats
		runtime.ReadMemStats(&m)
		peak = max(peak, m.Sys-m.HeapReleased)
	}
	metrics.Read(samples)
	fmt.Printf("durée=%-6v mémoire-pic≈%3d Mio  CPU-GC=%4.1f %%\n", time.Since(start).Round(10*time.Millisecond),
		peak/mib, samples[0].Value.Float64()/samples[1].Value.Float64()*100)
	runtime.KeepAlive(live)
}
```

Mesuré dans le conteneur limité :

| `GOGC` | `GOMEMLIMIT` | Durée | Mémoire (pic) | CPU du GC |
|---|---|---|---|---|
| 100 | aucune | 1,60 s | **127 Mio** | 21 % |
| 100 | 100 Mio | 2,11 s | 96 Mio | 31 % |
| 100 | 112 Mio | 1,86 s | 106 Mio | 27 % |
| 200 | aucune | **tué** | — | — |
| 200 | 112 Mio | 1,84 s | 106 Mio | 26 % |
| 400 | aucune | **tué** | — | — |
| 400 | 112 Mio | 1,81 s | 107 Mio | 27 % |
| off | aucune | **tué** | — | — |
| off | 112 Mio | 1,82 s | 107 Mio | 26 % |

**Choix : `GOMEMLIMIT` à ≈ 87 % du conteneur (112 Mio pour 128, soit ≈ 450 Mio pour 512), `GOGC=100` ou
`off`.**

1. Le plus rapide, `GOGC=100` sans limite, est passé **à 1 Mio** de la limite du conteneur : au moindre pic,
   il est tué. Aucun réglage sans limite n'est acceptable.
2. Avec la limite à 112 Mio, `GOGC` ne change presque plus rien (1,81 à 1,86 s) : quand le tas vivant atteint
   50 Mio, l'objectif de `GOGC=100` (100 Mio) est déjà proche de la limite, c'est elle qui décide.
3. Une limite à 100 Mio (78 %) coûte 15 % de temps en plus, sans gain de sécurité utile.
4. **La fuite** : si le tas vivant monte vers 120 Mio (480 Mio à l'échelle réelle), la limite ne peut plus être
   respectée ; le GC tourne presque en continu (son temps est plafonné à environ la moitié du CPU), le
   service devient très lent, puis la mémoire dépasse et le conteneur est tué. `GOMEMLIMIT` ne protège pas
   d'une fuite : il la rend **visible** (part du GC qui grimpe, à surveiller avec l'exercice intermédiaire)
   avant la mort du processus.

**Green Tea sur cette charge** (`GOGC=off`, `GOMEMLIMIT=112MiB`, trois exécutions) :

```
défaut          1.79s  1.80s  1.88s   CPU-GC 25–27 %
nogreenteagc    1.99s  2.27s  1.98s   CPU-GC 31–34 %
```

Ici, Green Tea fait gagner environ **10 %** et réduit la part du GC d'un cinquième — alors qu'il n'apportait
rien sur la charge du cours. Cette charge-ci (des millions de petits objets à pointeurs, des cycles très
fréquents) correspond à ce pour quoi il a été conçu. C'est exactement la leçon : une évolution du *runtime*
se mesure sur **sa** charge.

## Quiz

1. Blanc (non vu), gris (vu, pointeurs à suivre), noir (vu et parcouru) ; les blancs restants sont des déchets.
2. Le programme modifie des pointeurs pendant le marquage ; la barrière prévient le GC pour qu'aucun objet
   atteignable ne reste blanc.
3. Non, et non.
4. Le processeur (marquage, assistance des goroutines qui allouent) ; les pauses sont de l'ordre de 0,1 ms.
5. Quand le tas atteint le double du tas vivant mesuré au cycle précédent.
6. Elle déclenche le GC plus tôt à l'approche de la limite ; au ras du tas vivant, le GC tourne sans arrêt.
7. Tas au début du cycle, à la fin, et tas vivant après marquage.
8. Allouer moins, et avoir moins de pointeurs dans le tas vivant.
9. Le GC Green Tea et l'allocateur spécialisé par taille.
10. Construire avec `GOEXPERIMENT=no…` et comparer sur sa propre charge, plusieurs exécutions.
