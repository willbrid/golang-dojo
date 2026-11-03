# Leçon 5 — pprof : mémoire, blocage, verrous, fuites

> Le profil CPU (leçon 4) ne voit que le temps où le processeur travaille. Or beaucoup de programmes lents
> ne travaillent pas : ils **allouent** au point d'étouffer le ramasse-miettes, ou ils **attendent** — un
> canal, un verrou, une goroutine qui ne viendra jamais. Go fournit un profil pour chacun de ces cas. Cette
> leçon apprend à les collecter et à les lire, sur quatre défauts typiques, et montre une limite mesurée du
> profil de fuites de Go 1.27.

## Objectifs

1. Lire le profil mémoire : `alloc_space`, `alloc_objects`, `inuse_space`.
2. Activer et lire le profil de **blocage** (attentes sur canaux, `select`, `WaitGroup`…).
3. Activer et lire le profil de **contention** des verrous.
4. Utiliser le profil `goroutineleak` (Go 1.27), et connaître sa limite.
5. Choisir le bon profil selon le symptôme.

## Explication

### Quel profil pour quel symptôme ?

| Symptôme | Profil | Activation |
|---|---|---|
| processeur saturé | CPU (`profile`) | toujours disponible (leçon 4) |
| GC très actif, beaucoup d'allocations | mémoire (`allocs`, `heap`) | toujours actif (échantillonné) |
| mémoire qui grossit | `heap` (`inuse_space`) | toujours actif |
| lent mais processeur calme : attentes | blocage (`block`) | `runtime.SetBlockProfileRate` ou `go test -blockprofile` |
| lent sous charge, verrou suspect | contention (`mutex`) | `runtime.SetMutexProfileFraction` ou `go test -mutexprofile` |
| goroutines qui s'accumulent | `goroutine`, `goroutineleak` (1.27) | toujours disponibles |

Tous sont servis par `net/http/pprof` (`/debug/pprof/heap`, `/debug/pprof/block`, `/debug/pprof/mutex`,
`/debug/pprof/goroutineleak`…), sur l'adresse d'administration (leçon 4).

### La mémoire

Le profil mémoire échantillonne les allocations (une tous les 512 Kio alloués en moyenne) et enregistre la
pile de chacune. Le même fichier se lit de quatre façons (`-sample_index=…`) :

| Index | Mesure | Pour |
|---|---|---|
| `alloc_space` | octets alloués depuis le début | trouver ce qui **fait travailler le GC** |
| `alloc_objects` | nombre d'allocations depuis le début | idem, en nombre (souvent plus parlant) |
| `inuse_space` | octets encore **vivants** au dernier cycle du GC | trouver ce qui **occupe** la mémoire (une fuite) |
| `inuse_objects` | nombre d'objets vivants | idem |

Pour un problème de **performance**, on regarde `alloc_*` ; pour un problème de **mémoire**, `inuse_*`.
Mesuré sur l'exemple (`go test -bench Report -memprofile mem.out`, puis
`go tool pprof -sample_index=alloc_space -list prof.Report`) :

```
    3.38GB     3.44GB     15:		s += fmt.Sprintf("ligne %d : %s\n", i, strings.Repeat("x", 20))
```

3,4 Go alloués pendant le benchmark, presque tous par la **concaténation** `s +=` (la leçon 1 : chaque tour
recopie toute la chaîne). En `inuse_space`, en revanche, le profil ne montre presque rien de `Report` : ces
chaînes sont toutes mortes. Les deux vues répondent à deux questions différentes.

### Le blocage

Le profil de **blocage** enregistre le temps que les goroutines passent **à attendre** : envoi ou réception
sur un canal, `select`, `sync.WaitGroup.Wait`, `sync.Cond`… Il est **désactivé** par défaut (il a un coût) :
`runtime.SetBlockProfileRate(n)` l'active (n = 1 : tous les événements ; plus grand : échantillonnage), et
`go test -blockprofile` le fait pour la durée des benchmarks.

```
     1.21s 51.21% 51.21%      1.21s 51.21%  runtime.chanrecv1     ← main.main (le cadre de test)
     1.15s 48.78%   100%      1.15s 48.78%  runtime.chansend1     ← prof.Pipeline
         .      1.15s     55:		ch <- i // le producteur attend ici
```

La moitié du temps mesuré est l'attente du **cadre de test** lui-même (`main.main` attend la fin du
benchmark) : du bruit, qu'on écarte avec `-focus` ou en lisant `list`. L'autre moitié est la ligne 55 : le
producteur attend que le consommateur lent soit prêt. Le profil CPU de ce même code serait presque vide —
le programme est lent sans travailler.

### La contention des verrous

Le profil de **contention** enregistre, pour chaque verrou disputé, le temps pendant lequel d'autres
goroutines ont attendu, et l'attribue à la goroutine qui **tenait** le verrou, au moment où elle l'a
**relâché** (`Unlock`). Il est désactivé par défaut : `runtime.SetMutexProfileFraction(n)` (une contention
sur n enregistrée) ou `go test -mutexprofile`.

```
    10.29s   100%   100%     10.29s   100%  sync.(*Mutex).Unlock
         0     0%   100%     10.29s   100%  prof.(*Cache).Get
         .     10.18s     34:	return v          ← le defer c.mu.Unlock() s'exécute ici
```

10 secondes d'attente cumulée pour un benchmark d'environ une seconde : huit goroutines qui se relaient sur
un seul verrou. La ligne désignée est le `return` (c'est là que le `defer Unlock` s'exécute) ; la cause est
ce que la fonction fait **pendant** qu'elle tient le verrou : `slowSquare`, un calcul de 10 µs sous verrou.
Corrections typiques : sortir le calcul de la section critique, `sync.RWMutex` pour les lectures,
découper le verrou (un par fragment de la map), ou `sync.Map` pour certains profils d'accès.

### Les fuites de goroutines : `goroutineleak` (Go 1.27)

Le profil `goroutine` liste **toutes** les goroutines et leur pile ; utile, mais dans un service qui en
compte des milliers, retrouver celles qui sont bloquées pour toujours est fastidieux. Le profil
**`goroutineleak`** (N6 L10) ne liste que celles que le ramasse-miettes peut **prouver** fuies : bloquées
sur un canal ou une primitive de synchronisation que plus aucune goroutine vivante ne peut atteindre. Il
déclenche lui-même un cycle de détection à chaque lecture.

```
goroutineleak profile: total 20
#	0x546484	prof.FirstOf3.func1+0x24	…/prof.go:66
```

Dix appels à `FirstOf3`, deux perdantes à chaque fois : 20 fuites, toutes à la ligne de l'envoi.

**Une limite mesurée.** En écrivant cet exemple, la première version de `FirstOf3` capturait la variable de
boucle (`go func() { ch <- i }()`) au lieu de la passer en paramètre (`go func(v int) { ch <- v }(i)`).
Avec la capture, le profil indiquait **`total 0`**, alors que les goroutines fuyaient exactement de la même
façon. Mesuré sur des variantes : `ch <- 7` (constante) → fuites détectées ; `ch <- v` (paramètre) →
détectées ; `ch <- i` (variable de boucle capturée) → **non détectées**. Le profil ne produit pas de faux
positif (une goroutine listée est vraiment perdue), mais il peut produire des **faux négatifs** : son
silence ne prouve pas l'absence de fuite. Les autres outils restent nécessaires : le nombre de goroutines
dans le temps (`runtime.NumGoroutine`, une métrique), le profil `goroutine` complet, `testing/synctest` en
test (N6 L10).

### Quand ne pas les activer

Les profils de blocage et de contention ont un coût : en production, on les active avec un
**échantillonnage** (`SetBlockProfileRate(1_000_000)` : un événement par milliseconde d'attente environ ;
`SetMutexProfileFraction(100)`), ou seulement le temps d'une enquête. Le profil mémoire est actif en
permanence à un taux qui ne coûte presque rien.

## Exemple

Quatre défauts, et un benchmark ou un test qui permet de collecter le profil correspondant.

```go
// Fichier : prof.go
// Package prof : quatre défauts, quatre profils.
package prof

import (
	"fmt"
	"strings"
	"sync"
	"time"
)

// 1. Mémoire : une ligne de rapport construite à coups de Sprintf et de concaténations.
func Report(rows int) string {
	s := ""
	for i := range rows {
		s += fmt.Sprintf("ligne %d : %s\n", i, strings.Repeat("x", 20))
	}
	return s
}

// 2. Contention : un cache global, un seul verrou, tenu pendant un calcul.
type Cache struct {
	mu sync.Mutex
	m  map[int]int
}

func (c *Cache) Get(k int) int {
	c.mu.Lock()
	defer c.mu.Unlock()
	if v, ok := c.m[k]; ok {
		return v
	}
	v := slowSquare(k) // calcul fait SOUS le verrou
	c.m[k] = v
	return v
}

func slowSquare(k int) int {
	time.Sleep(10 * time.Microsecond)
	return k * k
}

// 3. Blocage : un producteur rapide, un consommateur lent, un canal non bufferisé.
func Pipeline(n int) int {
	ch := make(chan int)
	done := make(chan int)
	go func() {
		sum := 0
		for v := range ch {
			time.Sleep(20 * time.Microsecond) // consommateur lent
			sum += v
		}
		done <- sum
	}()
	for i := range n {
		ch <- i // le producteur attend ici
	}
	close(ch)
	return <-done
}

// 4. Fuite : on n'attend qu'une réponse sur trois ; les deux autres goroutines restent bloquées.
// (i est passé en PARAMÈTRE : voir dans le cours pourquoi la capture de i masque la fuite.)
func FirstOf3() int {
	ch := make(chan int) // non bufferisé : les perdants ne peuvent jamais envoyer
	for i := range 3 {
		go func(v int) { ch <- v }(i)
	}
	return <-ch
}
```

```go
// Fichier : prof_test.go
package prof

import (
	"bytes"
	"runtime/pprof"
	"strings"
	"testing"
	"time"
)

var sink string

func BenchmarkReport(b *testing.B) {
	for b.Loop() {
		sink = Report(200)
	}
}

func BenchmarkCache(b *testing.B) {
	c := &Cache{m: map[int]int{}}
	b.RunParallel(func(pb *testing.PB) {
		i := 0
		for pb.Next() {
			c.Get(i % 5000) // beaucoup de clés nouvelles au début
			i++
		}
	})
}

func BenchmarkPipeline(b *testing.B) {
	for b.Loop() {
		Pipeline(100)
	}
}

func TestLeak(t *testing.T) {
	for range 10 {
		FirstOf3()
	}
	time.Sleep(50 * time.Millisecond)
	var buf bytes.Buffer
	pprof.Lookup("goroutineleak").WriteTo(&buf, 1)
	for _, l := range strings.Split(buf.String(), "\n") {
		if strings.Contains(l, "goroutineleak profile") || strings.Contains(l, "prof.go") {
			t.Log(strings.TrimSpace(l))
		}
	}
}
```

```sh
go test -run XXX -bench Report   -memprofile mem.out
go tool pprof -sample_index=alloc_space -list prof.Report prof.test mem.out

go test -run XXX -bench Cache    -mutexprofile mtx.out
go tool pprof -list 'Cache..Get' prof.test mtx.out

go test -run XXX -bench Pipeline -blockprofile blk.out
go tool pprof -list prof.Pipeline prof.test blk.out

go test -run Leak -v
```

Les extraits de profils de l'explication proviennent de ces commandes.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `Report` et `s +=` | 3,4 Go alloués : un problème pour le GC, invisible en `inuse_space`. |
| `Cache.Get` et `slowSquare` sous verrou | La contention n'est pas dans le verrou, mais dans ce qu'on fait en le tenant. |
| `b.RunParallel` | Huit goroutines au lieu d'une : sans parallélisme, aucune contention à mesurer. |
| `Pipeline` et le consommateur lent | Le producteur attend sur `ch <- i` : visible dans le profil de blocage, pas dans le profil CPU. |
| `FirstOf3` et le canal non bufferisé | Deux goroutines bloquées pour toujours à chaque appel (correction : `make(chan int, 3)`). |
| `go func(v int) { … }(i)` | Le paramètre plutôt que la capture : sinon le profil de fuites ne les voit pas (mesuré). |
| `pprof.Lookup("goroutineleak").WriteTo(&buf, 1)` | Lance la détection et écrit les piles des goroutines prouvées fuies. |

## Erreurs fréquentes

1. **Chercher une attente dans le profil CPU.**
2. **Confondre `alloc_space` et `inuse_space`** : chercher une fuite dans les allocations passées, ou un
   problème de GC dans la mémoire vivante.
3. **Oublier d'activer** les profils de blocage et de contention (ils sont vides par défaut).
4. **Les activer à 100 % en production** en permanence.
5. **Accuser le verrou** au lieu de ce qui est fait pendant qu'on le tient.
6. **Prendre l'attente du cadre de test** (`main.main`) pour un problème du code.
7. **Conclure qu'il n'y a pas de fuite** parce que `goroutineleak` affiche `total 0`.

## Bonnes pratiques Go

- Choisir le profil d'après le symptôme (tableau ci-dessus).
- `alloc_*` pour la performance, `inuse_*` pour la mémoire.
- Blocage et contention échantillonnés en production, activés le temps d'une enquête.
- Sections critiques courtes : rien de lent sous un verrou.
- `goroutineleak` + nombre de goroutines en métrique + `synctest` en test : aucun ne suffit seul.
- Passer les valeurs aux goroutines en paramètre quand on veut que leurs fuites soient détectables.

## Ce que je dois retenir

- Mémoire : `alloc_*` (ce qui fait travailler le GC) et `inuse_*` (ce qui occupe la mémoire).
- Blocage : le temps passé à attendre ; contention : le temps perdu sur les verrous, attribué au `Unlock`.
- Ces deux profils sont désactivés par défaut.
- `goroutineleak` (1.27) prouve les fuites qu'il trouve, mais peut en manquer.
- Le profil CPU ne voit que le travail ; les autres voient l'attente et la mémoire.

---

➡️ [Exercices](exercices.md)
