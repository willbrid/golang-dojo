# Leçon 3 — Le ramasse-miettes

> Go libère la mémoire tout seul : un ramasse-miettes (*garbage collector*, GC) parcourt régulièrement
> les objets encore utilisés et récupère les autres. Ce confort a un prix, payé en **temps processeur** et
> en **mémoire**, et ce prix se règle. Avec deux variables d'environnement (`GOGC`, `GOMEMLIMIT`), le même
> programme mesuré ici passe de 3 s à 0,9 s — ou s'effondre à 10 s. Cette leçon explique le fonctionnement
> du GC de Go, ce qu'il coûte, comment le lire et comment le régler.

## Objectifs

1. Comprendre le principe du GC de Go : concurrent, marquage tricolore, sans compactage.
2. Savoir quand un cycle se déclenche (`GOGC`) et ce que fait une limite mémoire (`GOMEMLIMIT`).
3. Lire `GODEBUG=gctrace=1` et mesurer avec `runtime/metrics`.
4. Connaître le vrai coût du GC : le temps processeur, pas les pauses.
5. Situer les nouveautés activées par défaut en Go 1.27, et mesurer leur effet.

## Explication

### Ce que fait le GC

Le GC de Go est un collecteur **par marquage et balayage**, **concurrent** :

1. **Marquage.** En partant des **racines** (variables globales, piles de toutes les goroutines), il suit
   chaque pointeur et marque les objets atteignables. L'algorithme est dit **tricolore** : blanc (pas
   encore vu), gris (vu, mais ses pointeurs restent à suivre), noir (vu, et tous ses pointeurs suivis). À la
   fin, les objets restés blancs sont inatteignables.
2. **Balayage.** La mémoire des objets blancs est rendue à l'allocateur.

Le marquage se fait **pendant** que le programme continue de tourner. Pour que le programme ne cache pas
un objet au collecteur en déplaçant un pointeur pendant le marquage, une **barrière d'écriture** est
activée : chaque écriture de pointeur prévient le GC. Il n'y a que deux **pauses** très courtes (*stop the
world*), au début et à la fin du marquage : mesurées ici, de l'ordre de **0,1 à 0,4 ms**.

Le GC de Go **ne déplace pas** les objets (pas de compactage) et n'est **pas générationnel** (il ne traite
pas à part les objets jeunes). D'où l'importance de l'analyse d'échappement (leçon 2) : un objet qui reste
sur la pile n'est jamais vu par le GC.

### Le coût réel : le processeur, pas les pauses

Les pauses font peur, mais elles sont minuscules. Le vrai coût, c'est le **travail** de marquage : le GC
réquisitionne environ un quart des processeurs pendant un cycle, et quand le programme alloue plus vite
que le GC ne marque, il fait **aider** les goroutines qui allouent (*mark assist*), qui ralentissent
d'autant. Ce coût est proportionnel :

- au **nombre de cycles**, donc à la **vitesse d'allocation** ;
- à la **taille du tas vivant** (ce qu'il faut parcourir à chaque cycle) et au nombre de **pointeurs**.

Mesuré sur l'exemple (500 000 objets vivants, 20 millions d'objets éphémères), avec `GOGC=100` : 33
cycles, **2,2 ms** de pauses au total… et **43 %** du temps processeur consacré au GC.

### Quand un cycle commence : `GOGC`

Le GC démarre un cycle quand le tas atteint un **objectif** calculé à partir du tas vivant au cycle
précédent :

> objectif = tas vivant × (1 + `GOGC`/100)

Avec `GOGC=100` (la valeur par défaut), on laisse le tas **doubler** avant le cycle suivant. Plus `GOGC` est
grand, moins il y a de cycles, plus le programme consomme de mémoire. Mesuré :

| Réglage | Durée | Cycles | Pauses (total) | Mémoire (pic) | CPU du GC |
|---|---|---|---|---|---|
| `GOGC=50` | 3,07 s | 69 | 4,9 ms | 63 Mio | 51 % |
| `GOGC=100` (défaut) | 1,85 s | 33 | 2,2 ms | 93 Mio | 43 % |
| `GOGC=200` | 1,28 s | 17 | 1,3 ms | 123 Mio | 31 % |
| `GOGC=400` | 0,92 s | 8 | 0,6 ms | 214 Mio | 21 % |

C'est un **échange** mémoire contre processeur, presque linéaire : doubler `GOGC` divise à peu près par
deux le nombre de cycles. `GOGC=off` désactive le GC (utile seulement avec une limite mémoire).

### Une limite de mémoire : `GOMEMLIMIT`

`GOMEMLIMIT` (Go 1.19, ou `debug.SetMemoryLimit`) fixe une limite **souple** à la mémoire du *runtime* :
quand on s'en approche, le GC se déclenche plus tôt que ne le dirait `GOGC`. Deux usages :

- **`GOGC=off` + `GOMEMLIMIT`** : pas de cycle tant qu'on est loin de la limite, beaucoup de cycles près
  d'elle. Utile pour un programme qui doit tenir dans un conteneur de taille connue ;
- **`GOGC` normal + `GOMEMLIMIT`** : la limite comme garde-fou, pour ne pas dépasser la mémoire du
  conteneur et être tué (*OOM killer*).

Le danger : une limite **proche du tas vivant**. Le GC tourne alors presque sans arrêt pour rester sous la
limite. Mesuré, avec 32 Mio d'objets vivants :

| Réglage | Durée | Cycles | CPU du GC |
|---|---|---|---|
| `GOGC=off`, `GOMEMLIMIT=80MiB` | 6,4 s | 203 | 57 % |
| `GOGC=100`, `GOMEMLIMIT=40MiB` | **9,9 s** | **326** | 61 % |

Le programme est cinq fois plus lent qu'avec les réglages par défaut. Le *runtime* plafonne le temps du GC
(environ la moitié du processeur) pour ne pas bloquer complètement le programme, ce qui évite l'arrêt total
mais pas la lenteur. Règle : une limite à **une marge confortable au-dessus** du tas vivant (par exemple
80 à 90 % de la mémoire du conteneur), jamais au ras.

### Lire le GC : `GODEBUG=gctrace=1`

Avec `GODEBUG=gctrace=1`, chaque cycle écrit une ligne sur la sortie d'erreur :

```
gc 11 @0.263s 10%: 0.027+27+0.022 ms clock, 0.21+3.6/45/100+0.18 ms cpu, 67->73->40 MB, 78 MB goal, 0 MB stacks, 0 MB globals, 8 P
```

| Champ | Sens |
|---|---|
| `gc 11 @0.263s` | 11ᵉ cycle, 0,263 s après le démarrage |
| `10%` | part du temps passé dans le GC depuis le démarrage |
| `0.027+27+0.022 ms clock` | pause de début, marquage concurrent, pause de fin (temps réel) |
| `0.21+3.6/45/100+0.18 ms cpu` | temps processeur : pause, puis marquage par les goroutines qui allouent (*assist*) / en arrière-plan / en temps libre, puis pause |
| `67->73->40 MB` | tas au début du cycle → à la fin → **vivant** après marquage |
| `78 MB goal` | l'objectif de ce cycle (≈ 39 × 2, avec `GOGC=100`) |
| `8 P` | `GOMAXPROCS` |

Le chiffre le plus utile est le **tas vivant** (ici 40 Mio) : c'est la mémoire réellement utilisée par le
programme ; le reste est l'espace laissé au GC.

Pour mesurer dans le programme lui-même : `runtime.ReadMemStats` (simple, mais elle marque une courte
pause) et `runtime/metrics` (le paquet moderne, sans pause : `/cpu/classes/gc/total:cpu-seconds`,
`/gc/heap/live:bytes`…). L'exemple utilise les deux.

### Go 1.27 : ce qui est activé par défaut

Deux évolutions du *runtime*, apparues comme expériences dans les versions précédentes, sont **activées par
défaut** dans Go 1.27 (on le vérifie dans les sources de la distribution, `internal/buildcfg`) :

- **le GC « Green Tea »**, qui marque les petits objets **par page mémoire** plutôt qu'objet par objet,
  pour mieux profiter des caches du processeur ;
- **l'allocateur spécialisé par taille** (`sizespecializedmalloc`), avec des chemins d'allocation
  compilés pour chaque classe de taille.

On peut les désactiver à la compilation pour comparer :
`GOEXPERIMENT=nogreenteagc,nosizespecializedmalloc go build`. Mesuré sur l'exemple, trois exécutions
chacune :

| Construction | Durée |
|---|---|
| défaut (les deux activés) | 1,97 – 2,07 s |
| sans Green Tea | 1,93 – 1,99 s |
| sans l'allocateur spécialisé | 2,31 – 2,34 s |
| sans les deux | 2,20 – 2,21 s |

Sur cette charge, l'allocateur spécialisé fait gagner environ **12 %** ; Green Tea n'apporte **rien de
mesurable** (ses gains annoncés portent sur des tas de petits objets à forte densité de pointeurs, sur
beaucoup de cœurs). Une nouveauté du *runtime* n'améliore pas tous les programmes : on mesure le sien.

### Ce qu'on règle, et ce qu'on change dans le code

- **Régler** : `GOMEMLIMIT` dans tout conteneur (à 80–90 % de sa limite mémoire), `GOGC` plus haut si la
  mémoire est abondante et le processeur rare. Mesurer avant et après.
- **Changer le code** : le levier le plus puissant est d'**allouer moins** (leçon 2 : moins d'échappements ;
  leçon 6 : préallouer ; leçon 8 : réutiliser). Moins d'allocations = moins de cycles. Et un tas vivant
  avec **moins de pointeurs** (des slices de valeurs plutôt que de pointeurs, des identifiants plutôt que
  des références) se marque plus vite.

## Exemple

Une charge typique : un ensemble d'objets vivants qui se renouvelle, et beaucoup d'objets éphémères. Le
programme mesure le nombre de cycles, les pauses, le pic de mémoire et la part du processeur consacrée au GC.
On le lance avec différents réglages : `GOGC=200 ./gcload`, `GOMEMLIMIT=80MiB GOGC=off ./gcload`,
`GODEBUG=gctrace=1 ./gcload`.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"runtime/metrics"
	"time"
)

type node struct {
	id   int64
	next *node
	data [6]int64 // 64 octets au total
}

var sink *node

func main() {
	rng := rand.New(rand.NewPCG(1, 2))
	live := make([]*node, 500_000) // ≈ 32 Mio vivants en permanence
	for i := range live {
		live[i] = &node{id: int64(i)}
	}
	var before runtime.MemStats
	runtime.ReadMemStats(&before)
	samples := []metrics.Sample{{Name: "/cpu/classes/gc/total:cpu-seconds"}, {Name: "/cpu/classes/total:cpu-seconds"}}
	metrics.Read(samples)
	gc0, tot0 := samples[0].Value.Float64(), samples[1].Value.Float64()

	start := time.Now()
	var peak uint64
	for i := range 20_000_000 { // 20 millions d'objets éphémères…
		n := &node{id: int64(i)}
		sink = n
		if i%10 == 0 { // … dont un sur dix remplace un objet vivant
			live[rng.IntN(len(live))] = n
		}
		if i%1_000_000 == 0 {
			var m runtime.MemStats
			runtime.ReadMemStats(&m)
			peak = max(peak, m.HeapSys-m.HeapReleased)
		}
	}
	elapsed := time.Since(start)
	var after runtime.MemStats
	runtime.ReadMemStats(&after)
	metrics.Read(samples)
	gcCPU := samples[0].Value.Float64() - gc0
	totCPU := samples[1].Value.Float64() - tot0
	fmt.Printf("durée=%-7v cycles=%-4d pause-totale=%-9v pause-max=%-9v tas-pic≈%3d Mio  CPU-GC=%4.1f %%\n",
		elapsed.Round(10*time.Millisecond), after.NumGC-before.NumGC,
		time.Duration(after.PauseTotalNs-before.PauseTotalNs).Round(10*time.Microsecond),
		time.Duration(slicesMax(after.PauseNs[:])).Round(time.Microsecond),
		peak>>20, gcCPU/totCPU*100)
	runtime.KeepAlive(live)
}

func slicesMax(xs []uint64) uint64 {
	var m uint64
	for _, x := range xs {
		m = max(m, x)
	}
	return m
}
```

Les tableaux de l'explication sont les mesures de ce programme (une ligne par exécution, les valeurs
varient légèrement d'une fois à l'autre). Par exemple, avec `GOGC=100` :

```
durée=1.85s   cycles=33   pause-totale=2.16ms    pause-max=138µs     tas-pic≈ 93 Mio  CPU-GC=42.7 %
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `live` : 500 000 `*node` | Le tas **vivant** que chaque cycle doit parcourir (≈ 32 Mio de données). |
| `node` de 64 octets avec un pointeur | Des objets avec pointeurs : le GC doit les suivre. |
| 20 millions de `&node{…}` dans `sink` | Le flux d'allocations qui déclenche les cycles ; `sink` empêche l'analyse d'échappement de les garder sur la pile. |
| Un sur dix remplace un objet vivant | Le tas vivant se renouvelle : les anciens objets deviennent des déchets. |
| `runtime.ReadMemStats` | Cycles, pauses, pic (`HeapSys - HeapReleased`) ; pause brève à chaque appel, donc rarement. |
| `runtime/metrics` | La part de CPU du GC, sans pause. |
| `runtime.KeepAlive(live)` | Garantit que `live` reste vivant jusqu'à la fin de la mesure. |

## Erreurs fréquentes

1. **Craindre les pauses** et ignorer le temps processeur du GC.
2. **Régler `GOGC` sans mesurer** l'effet sur la mémoire **et** sur le temps.
3. **Pas de `GOMEMLIMIT` dans un conteneur** : le programme dépasse la mémoire et se fait tuer.
4. **Un `GOMEMLIMIT` au ras du tas vivant** : le GC tourne sans arrêt.
5. **Appeler `runtime.GC()` « pour aider »** : un cycle de plus, rarement utile.
6. **Juger la mémoire sur la taille totale du tas** plutôt que sur le tas **vivant**.
7. **Supposer qu'une nouvelle version du *runtime* accélère tout** sans mesure.

## Bonnes pratiques Go

- `GOMEMLIMIT` à 80–90 % de la mémoire du conteneur ; `GOGC` par défaut sauf mesure contraire.
- Lire `gctrace` ou `runtime/metrics` pour connaître le tas vivant et la part de CPU du GC.
- Réduire les allocations sur les chemins chauds : c'est le levier principal.
- Préférer des structures avec peu de pointeurs pour les grands ensembles vivants.
- Comparer les expériences du *runtime* (`GOEXPERIMENT`) sur sa propre charge.

## Ce que je dois retenir

- GC concurrent, tricolore, sans compactage ni générations ; pauses de l'ordre de 0,1 ms.
- Le coût réel est le processeur, proportionnel aux allocations et au tas vivant.
- `GOGC` : le tas peut croître de `GOGC` % avant le cycle suivant ; mémoire contre processeur.
- `GOMEMLIMIT` : une limite souple ; jamais au ras du tas vivant.
- Go 1.27 active Green Tea et l'allocateur spécialisé par taille ; leur effet se mesure, charge par charge.

---

➡️ [Exercices](exercices.md)
