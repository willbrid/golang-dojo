# Module 4 — Debugging complexe

> Un bug qu'on reproduit en local se corrige en une heure. Les autres sont plus coriaces : une panique en
> production à 3 h du matin, un pic de latence de 150 ms toutes les deux secondes, invisible au p99, une
> donnée fausse une fois sur mille. Ce module donne les instruments, et surtout la façon de choisir le bon :
> - le débogueur `delve`, sur un programme vivant ou sur un **vidage mémoire** après un plantage ;
> - les **traces d'exécution**, et leur enregistreur de vol qui capture les secondes qui précèdent un
>   incident ;
> - le **détecteur de races**, et ce qu'il coûte.
>
> Chaque instrument est essayé sur un vrai défaut, et ce qu'on y voit est recopié tel quel.

## Objectifs

- Mener une enquête avec méthode : reproduire, observer, formuler une hypothèse, la vérifier.
- Utiliser `delve` : points d'arrêt conditionnels, goroutines, piles, variables, pas à pas.
- Obtenir un vidage mémoire d'un programme Go qui plante, et l'analyser avec `dlv core`.
- Capturer une trace d'exécution au moment d'un incident rare, avec l'enregistreur de vol, et l'analyser.
- Connaître le coût du détecteur de races et ses réglages.

## Explication

### La méthode avant l'outil

Un débogage efficace est une démarche expérimentale :

1. **Reproduire** : rendre le défaut observable, même rarement. Un test, une charge, une graine (module 5).
2. **Observer** sans hypothèse préalable : journaux et traces (`trace_id`, N12 L6–L7), métriques, profils,
   vidage.
3. **Formuler une hypothèse** qui explique **toutes** les observations, pas seulement la plus visible.
4. **La vérifier** par une expérience qui pourrait la réfuter : changer une seule chose, et prédire le
   résultat.
5. **Corriger**, puis écrire le test qui aurait attrapé le défaut.

Le choix de l'outil découle de la question :

| Question | Outil |
|---|---|
| que vaut cette variable à ce moment-là ? | `delve` (programme vivant ou vidage) |
| pourquoi ce programme a-t-il planté ? | la trace de pile de la panique ; le vidage mémoire pour l'état complet |
| où passe le temps ? | le profil CPU (N11 L4) |
| pourquoi cette requête-**là** a-t-elle été lente ? | une trace d'exécution, ou une trace répartie (N12 L7) |
| qui bloque qui ? | les profils de blocage et de contention (N11 L5), la trace d'exécution |
| est-ce une course de données ? | `-race` |

### `delve`

`dlv` est le débogueur de Go. Il comprend les goroutines, les interfaces et les types du langage, ce que
`gdb` fait mal.

| Commande | Usage |
|---|---|
| `dlv debug ./cmd/x` | compile (sans optimisations) et lance sous le débogueur |
| `dlv test ./paquet -- -test.run TestX` | idem pour un test |
| `dlv exec ./binaire` | un binaire déjà construit, de préférence avec `-gcflags=all="-N -l"` (sans optimisations ni *inlining*) |
| `dlv attach PID` | se greffer sur un processus en marche |
| `dlv core ./binaire core` | analyser un vidage mémoire |

Dans la session : `break fichier:ligne` ou `break fonction` ; `condition N expression` (s'arrêter
seulement si…) ; `continue`, `next`, `step` ; `print`, `locals`, `args` ; `goroutines` ;
`goroutine N` ; `stack` ; `frame N`.

Le **point d'arrêt conditionnel** est l'outil des programmes concurrents. Dans l'exemple, huit goroutines
calculent mille totaux, et un seul nous intéresse. `condition 1 o.ID == 637` s'arrête exactement sur lui, dans
la bonne goroutine (la 10), avec sa pile et ses variables. Un `fmt.Println` ajouté au code aurait produit
mille lignes.

Deux contraintes de terrain :

- **`dlv attach`** demande le droit de tracer le processus (`ptrace`). Sur cette machine,
  `/proc/sys/kernel/yama/ptrace_scope` vaut 1 : seul un parent peut tracer son enfant, et `dlv attach` est
  **refusé** (« Could not attach to pid … try writing "0" to /proc/sys/kernel/yama/ptrace_scope »). En
  conteneur, il faut la capacité `SYS_PTRACE`. En production, on s'attache rarement : on capture un vidage
  ou une trace, et on analyse ailleurs.
- **Un binaire optimisé** se débogue mal : variables « optimisées » absentes, lignes qui sautent. Et un
  binaire construit avec `-s -w` n'a plus de quoi afficher les variables (N12 L2, E4). On garde dans la CI le
  binaire non réduit de chaque version.

### Le vidage mémoire après un plantage

Par défaut, une panique non récupérée affiche la pile de la goroutine fautive et arrête le programme. Avec
**`GOTRACEBACK=crash`** (ou `debug.SetTraceback("crash")`), le *runtime* affiche **toutes** les goroutines,
puis se termine par un signal `SIGABRT` qui fait produire au noyau un **vidage mémoire** (*core dump*) : une
copie de toute la mémoire du processus, à l'instant de la panne.

Il faut que le système les accepte :
- `ulimit -c unlimited` pour le processus ;
- `/proc/sys/kernel/core_pattern` dit où ils vont. Ici, il est confié à `apport`, qui les range dans
  `/var/lib/apport/coredump/`. Ailleurs, ce peut être `systemd-coredump` (`coredumpctl`), ou un simple
  fichier `core`.

`dlv core binaire vidage` ouvre alors l'état exact du programme mort. Dans l'exemple, une commande `nil`
dans une liste :

- `goroutines` montre la goroutine 1 arrêtée dans `main.total` (`main.go:10`) ;
- `frame 10` puis `args` donnent `o = *main.order nil` ;
- `frame 11` remonte dans `main` : `orders` a 3 éléments, et le troisième est `nil`.

Tout cela **sans avoir reproduit** le plantage. Le vidage remplace la reproduction quand elle est impossible.

Un détail que le vidage a révélé : la panique a lieu dans `total(o)`, et **non** sur `o.id`, écrit plus tôt
dans la même ligne (`fmt.Println(o.id, total(o))`). La spécification de Go ne fixe l'ordre d'évaluation
qu'entre les appels de fonctions et de méthodes, les réceptions sur un canal et les opérations logiques
binaires. Entre un appel
(`total(o)`) et un accès à un champ (`o.id`), l'ordre n'est pas défini, et le compilateur a évalué l'appel
d'abord. La trace de pile désigne donc `total`, alors que l'erreur est dans la liste. Il faut lire la ligne
**appelante**, pas seulement la ligne fautive.

Un vidage mémoire contient **toute** la mémoire : mots de passe, jetons, données personnelles. Il se traite
comme un secret (N12 L8) : accès restreint, durée de conservation courte, jamais joint à un ticket public.

### Les traces d'exécution

Le profil CPU dit **où**, en moyenne, le temps est passé. Il ne dit pas pourquoi **cette** requête a attendu
150 ms. La **trace d'exécution** (`runtime/trace`) enregistre les événements du *runtime* :
- les débuts et fins d'exécution de chaque goroutine ;
- les blocages (verrou, canal, réseau, appel système), avec leur pile ;
- les cycles du GC ;
- l'activité de chaque processeur logique.

On la lit dans un navigateur avec `go tool trace trace.out`, où l'on voit qui tourne, qui attend, et quand.
On peut aussi en tirer des profils sans interface : `go tool trace -pprof=sync|sched|net|syscall`.

On l'obtient avec `go test -trace=trace.out`, avec `trace.Start`/`trace.Stop`, ou par
`/debug/pprof/trace?seconds=5` (N11 L4).

Le problème d'un incident **rare** est qu'il faut tracer **au moment** où il se produit. Depuis Go 1.25,
l'**enregistreur de vol** (`trace.FlightRecorder`) trace en permanence dans un tampon circulaire en mémoire, qui
garde les dernières secondes. Le programme appelle `WriteTo` quand il **détecte** l'anomalie, par exemple une
requête de plus de 50 ms, et obtient la trace des secondes **qui l'ont précédée**.

L'exemple simule un service dont les requêtes durent environ 1 ms. Une tâche de fond reconstruit un cache
toutes les 2 s, **en tenant** le verrou partagé :

```
pic de 151ms : 469044 octets de trace écrits (err <nil>)
enregistreur=true : 63360 requêtes, p50 1.18ms, p99 1.45ms, p99,9 2.09ms, max 153ms
```

- **Le p99,9 vaut 2 ms : les pics sont invisibles dans les percentiles.** Ils ne touchent qu'une requête sur
  2 000 environ, et seul le maximum les montre. Une alerte sur le p99 ne verrait rien. C'est pourtant ce que
  vivent les utilisateurs qui tombent dessus.
- **La trace au moment du pic** (469 Ko) donne, par `go tool trace -pprof=sync`, le blocage des requêtes :
  2,45 s d'attente cumulée dans `sync.(*Mutex).Lock`, appelé par `main.handle`. On sait **qui attend**.
- **Le profil de contention**, écrit au même instant, donne **qui tenait** le verrou : 99,5 % de l'attente est
  attribuée à `sync.(*Mutex).Unlock` dans `main.rebuild`. Le profil compte l'attente au moment où le
  titulaire relâche (N11 L5).

Les deux ensemble donnent le diagnostic complet : `rebuild` garde le verrou pendant sa reconstruction de
150 ms. La correction consiste à construire la nouvelle map **hors** du verrou, et à ne prendre le verrou
que pour l'échange, ou à publier la map par un `atomic.Pointer` (module 2, défi).

**Le coût** : 64 392 requêtes sans l'enregistreur, 63 256 avec, soit environ 1,8 % de débit en moins (un essai
de chaque). C'est assez peu pour le laisser actif en production sur un service sensible à la latence, ce
qui était l'objectif de sa conception.

### Le détecteur de races, et son prix

`-race` instrumente chaque accès mémoire. Mesuré sur une charge concurrente ordinaire (une map sous verrou,
8 goroutines, des canaux), trois exécutions :

| Construction | Durée du travail | Durée du processus | Mémoire maximale |
|---|---|---|---|
| normale | 140 ms | 0,14 s | 2,6 Mio |
| `-race` | 958 ms (×6,8) | 1,99 s | 23 Mio (×9) |
| `-race`, `GORACE=atexit_sleep_ms=0` | 960 ms | **0,97 s** | 23 Mio |

- **×7 en temps et ×9 en mémoire** sur cette charge. La documentation officielle annonce 2 à 20 fois pour le
  processeur et 5 à 10 fois pour la mémoire, selon le programme.
- **Une seconde de plus à la fin.** Par défaut, le programme instrumenté attend 1 s à sa sortie pour laisser
  les autres goroutines signaler leurs courses (`atexit_sleep_ms=1000`). Sur une suite de tests de 500
  binaires, cela fait 8 minutes. `GORACE=atexit_sleep_ms=0` les supprime, au risque de manquer un rapport
  tardif.
- **D'autres réglages de `GORACE`** :
  - `halt_on_error=1` arrête à la première course ;
  - `log_path=/chemin` écrit les rapports dans un fichier ;
  - `history_size=N` garde plus d'historique, pour des rapports complets sur les programmes longs.

Le détecteur ne trouve que les courses qui **se produisent** pendant l'exécution. D'où son usage :
- **toujours** en CI (N12 L4) ;
- dans les tests de stress (module 1) ;
- et, pour les services, une instance de **pré-production** construite avec `-race` sous une charge réaliste
  pendant quelques heures. On l'arrête au premier rapport.

## Exemple

Quatre programmes : `crash` (le vidage mémoire), `orders` (la session `delve`), `spike` (l'enregistreur de
vol), `work` (le coût de `-race`).

```go.mod
// Fichier : go.mod
module example.com/dbg

go 1.27
```

```go
// Fichier : cmd/crash/main.go
package main

import "fmt"

type order struct {
	id    int
	items []string
}

func total(o *order) int { return len(o.items) * 100 }

func main() {
	orders := []*order{{1, []string{"a"}}, {2, nil}, nil}
	for _, o := range orders {
		fmt.Println(o.id, total(o)) // le troisième est nil : panique
	}
}
```

```go
// Fichier : cmd/orders/main.go
// Commande orders : 8 goroutines traitent des commandes ; l'une d'elles a un total faux.
package main

import (
	"fmt"
	"sync"
)

type Order struct {
	ID    int
	Lines []int // prix en centimes
}

func total(o Order) int {
	t := 0
	for i := 1; i < len(o.Lines); i++ { // le défaut : la première ligne est oubliée
		t += o.Lines[i]
	}
	return t
}

func main() {
	var wg sync.WaitGroup
	results := make([]int, 1000)
	for w := range 8 {
		wg.Go(func() {
			for id := w; id < 1000; id += 8 {
				o := Order{ID: id, Lines: []int{id, 100, 200}}
				results[id] = total(o)
			}
		})
	}
	wg.Wait()
	fmt.Println("commande 637 :", results[637], "(attendu", 637+100+200, ")")
}
```

```go
// Fichier : cmd/spike/main.go
// Commande spike : des requêtes de 1 ms, et de rares pics de latence. L'enregistreur de vol garde les
// dernières secondes de trace en mémoire, et les écrit dans un fichier au premier pic détecté.
package main

import (
	"flag"
	"fmt"
	"log"
	"os"
	"runtime"
	"runtime/pprof"
	"runtime/trace"
	"slices"
	"sync"
	"sync/atomic"
	"time"
)

var (
	mu    sync.Mutex // protège le cache partagé
	cache = map[int]int{}
)

// handle : une requête ordinaire, qui lit le cache sous le verrou.
func handle(k int) {
	mu.Lock()
	_ = cache[k]
	mu.Unlock()
	time.Sleep(time.Millisecond) // le reste du travail
}

// rebuild : une tâche de fond qui reconstruit le cache EN TENANT le verrou (le défaut à trouver).
func rebuild() {
	mu.Lock()
	defer mu.Unlock()
	next := map[int]int{}
	for i := range 2_000_000 {
		next[i%1000] += i
	}
	time.Sleep(100 * time.Millisecond) // un appel lent « pour charger les données »
	cache = next
}

func main() {
	record := flag.Bool("record", true, "enregistreur de vol actif")
	flag.Parse()
	runtime.SetMutexProfileFraction(1) // profil de contention : QUI tenait le verrou (N11 L5)
	var fr *trace.FlightRecorder
	if *record {
		fr = trace.NewFlightRecorder(trace.FlightRecorderConfig{MinAge: 2 * time.Second})
		if err := fr.Start(); err != nil {
			log.Fatal(err)
		}
		defer fr.Stop()
	}
	go func() { // toutes les 2 s, la reconstruction
		for range time.Tick(2 * time.Second) {
			rebuild()
		}
	}()
	var once sync.Once
	var lat []time.Duration
	var lmu sync.Mutex
	var n atomic.Int64
	deadline := time.Now().Add(5 * time.Second)
	var wg sync.WaitGroup
	for w := range 16 {
		wg.Go(func() {
			for i := 0; time.Now().Before(deadline); i++ {
				t0 := time.Now()
				handle(w*1000 + i)
				d := time.Since(t0)
				n.Add(1)
				lmu.Lock()
				lat = append(lat, d)
				lmu.Unlock()
				if d > 50*time.Millisecond && fr != nil {
					once.Do(func() { // un instantané au premier pic, pas un par requête lente
						f, _ := os.Create("pic.trace")
						defer f.Close()
						nb, err := fr.WriteTo(f)
						fmt.Printf("pic de %v : %d octets de trace écrits (err %v)\n", d.Round(time.Millisecond), nb, err)
						m, _ := os.Create("pic.mutex")
						defer m.Close()
						pprof.Lookup("mutex").WriteTo(m, 0)
					})
				}
			}
		})
	}
	wg.Wait()
	slices.Sort(lat)
	p := func(q float64) time.Duration { return lat[int(q*float64(len(lat)-1))].Round(10 * time.Microsecond) }
	fmt.Printf("enregistreur=%v : %d requêtes, p50 %v, p99 %v, p99,9 %v, max %v\n", *record, n.Load(), p(0.5), p(0.99), p(0.999), lat[len(lat)-1].Round(time.Millisecond))
}
```

```go
// Fichier : cmd/work/main.go
// Commande work : une charge concurrente ordinaire (map sous verrou, canaux), pour mesurer -race.
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	start := time.Now()
	var mu sync.Mutex
	m := map[int]int{}
	ch := make(chan int, 64)
	var wg sync.WaitGroup
	for w := range 8 {
		wg.Go(func() {
			for i := range 200_000 {
				mu.Lock()
				m[(w*200_000+i)%10_000]++
				mu.Unlock()
				if i%100 == 0 {
					ch <- i
					<-ch
				}
			}
		})
	}
	wg.Wait()
	fmt.Printf("%v\n", time.Since(start).Round(time.Millisecond))
}
```

**Le vidage mémoire** de `crash`, construit sans optimisations :

```
$ go build -gcflags=all="-N -l" -o crash ./cmd/crash
$ ulimit -c unlimited; GOTRACEBACK=crash ./crash
1 100
2 0
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x4b6395]
…
Abandon (core dumped)
$ ls /var/lib/apport/coredump/
core._tmp_…_crash.1000.907302cf-….851628.4610213
$ cat dlv-core.txt
goroutine 1
frame 10
args
frame 11
locals
print orders[2]
print len(orders)
exit
$ dlv core ./crash ./core.crash --init dlv-core.txt
Frame 10: ./main.go:10 (PC: 4b6395)
=>  10:	func total(o *order) int { return len(o.items) * 100 }
o = *main.order nil
~r0 = 0
Frame 11: ./main.go:15 (PC: 4b6611)
=>  15:			fmt.Println(o.id, total(o)) // le troisième est nil : panique
orders = []*main.order len: 3, cap: 3, [...]
o = *main.order nil
*main.order nil
3
```

**La session `delve`** sur `orders`, avec un point d'arrêt conditionnel :

```
$ cat dlv-orders.txt
break total
condition 1 o.ID == 637
continue
print o
goroutine
stack 1
next
next
next
print i
print t
exit
$ dlv debug ./cmd/orders --init dlv-orders.txt
Breakpoint 1 set at 0x4b7044 for main.total() ./cmd/orders/main.go:14
> [Breakpoint 1] main.total() ./cmd/orders/main.go:14 (hits goroutine(10):1 total:1) (PC: 0x4b7044)
main.Order {
	ID: 637,
	Lines: []int len: 3, cap: 3, [637,100,200],}
Goroutine 10:
	Runtime: ./cmd/orders/main.go:14 main.total (0x4b7044)
	…
	Start: /usr/local/go/src/sync/waitgroup.go:238 sync.(*WaitGroup).Go.func1 (0x494bc0)
0  0x00000000004b7044 in main.total
   at ./cmd/orders/main.go:14
1  0x00000000004b7514 in main.main.func1
   at ./cmd/orders/main.go:29
> main.total() ./cmd/orders/main.go:15 (PC: 0x4b7064)
> main.total() ./cmd/orders/main.go:16 (PC: 0x4b706d)
=>  16:		for i := 1; i < len(o.Lines); i++ { // le défaut : la première ligne est oubliée
> main.total() ./cmd/orders/main.go:17 (PC: 0x4b708b)
1
0
```

Au premier passage dans la boucle, `i` vaut **1** et `t` vaut 0 : la première ligne (637) ne sera jamais
ajoutée.

**L'enregistreur de vol** de `spike`, puis l'analyse du pic :

```
$ ./spike
pic de 151ms : 469044 octets de trace écrits (err <nil>)
enregistreur=true : 63360 requêtes, p50 1.18ms, p99 1.45ms, p99,9 2.09ms, max 153ms
$ go tool trace -pprof=sync pic.trace > sync.pprof && go tool pprof -top sync.pprof
      flat  flat%   sum%        cum   cum%
     4.11s 47.94% 47.94%      4.11s 47.94%  runtime.chanrecv1
     2.47s 28.73% 76.67%      2.47s 28.73%  sync.(*Mutex).Lock
        2s 23.31%   100%         2s 23.31%  runtime.chanrecv2
         0     0%   100%      2.45s 28.60%  main.handle
…
$ go tool pprof -top pic.mutex
      flat  flat%   sum%        cum   cum%
     2.40s   100%   100%      2.40s   100%  sync.(*Mutex).Unlock
         0     0%   100%      2.39s 99.47%  main.main.func1
         0     0%   100%      2.39s 99.47%  main.rebuild
$ ./spike -record=false
enregistreur=false : 64392 requêtes, p50 1.16ms, p99 1.39ms, p99,9 2.36ms, max 154ms
```

(Les attentes sur `chanrecv` du profil de synchronisation sont celles du *runtime* et de la minuterie de la
reconstruction, et non des requêtes. On lit la ligne de `main.handle`. Ce premier profil a été tiré d'une
exécution précédente du même programme, avant l'ajout du profil de contention.)

**Le coût de `-race`** :

```
$ /usr/bin/time -f "%e s  %M Kio max" ./work          # trois fois
141ms  0.14 s  2560 Kio max
$ /usr/bin/time -f "%e s  %M Kio max" ./work-race     # trois fois
950ms  1.96 s  22912 Kio max
$ GORACE=atexit_sleep_ms=0 /usr/bin/time -f "%e s  %M Kio max" ./work-race
965ms  0.97 s  23808 Kio max
```

## Explication du code

| Élément | Rôle |
|---|---|
| `-gcflags=all="-N -l"` | Sans optimisations ni *inlining*, pour **tous** les paquets : les variables existent et les lignes se suivent. |
| `GOTRACEBACK=crash` | Toutes les goroutines dans la trace de pile, puis `SIGABRT` : le noyau écrit un vidage. |
| `--init fichier` et `--allow-non-terminal-interactive` | Une session `delve` scriptée, rejouable, et dont la sortie se recopie dans un ticket. |
| `condition 1 o.ID == 637` | Le point d'arrêt 1 ne s'arrête que pour cette commande, quelle que soit la goroutine. |
| `trace.NewFlightRecorder(…{MinAge: 2 * time.Second})` | Garder au moins les 2 dernières secondes. `MaxBytes` borne la mémoire. |
| `once.Do(…)` autour de `WriteTo` | Un instantané au premier pic. Un instantané par requête lente noierait le disque. Un seul `WriteTo` à la fois est permis. |
| `runtime.SetMutexProfileFraction(1)` | Tous les événements de contention, pour l'exemple. En production, un échantillonnage (N11 L5). |
| `pprof.Lookup("mutex").WriteTo` | Le profil de contention, écrit au même instant que la trace. |
| `rebuild` sous `mu` | Le défaut à trouver : un travail de 150 ms dans la section critique. |

## Erreurs fréquentes

1. **Changer plusieurs choses à la fois** pendant l'enquête, et ne plus savoir laquelle a agi.
2. **Des `fmt.Println` dans du code concurrent** au lieu d'un point d'arrêt conditionnel.
3. **Déboguer un binaire optimisé**, et croire que les variables « disparues » sont un bug.
4. **Jeter le binaire non réduit** de la version déployée : le vidage mémoire devient illisible.
5. **Lire la ligne de la panique et pas la ligne appelante.**
6. **Partager un vidage mémoire** comme un fichier ordinaire.
7. **Juger la latence au p99** quand les incidents touchent moins d'une requête sur mille.
8. **Attendre de reproduire** un incident rare pour le tracer, au lieu d'un enregistreur de vol actif.
9. **`-race` seulement en local**, ou jamais sous une charge réaliste.

## Bonnes pratiques Go

- Pour chaque incident : reproduire, observer, une hypothèse, une expérience qui pourrait la réfuter, puis un
  test.
- Sessions `delve` scriptées (`--init`) pour les enquêtes à refaire ; points d'arrêt conditionnels pour le code
  concurrent.
- `GOTRACEBACK=crash` et des vidages activés sur les machines de pré-production ; le binaire non réduit de
  chaque version archivé.
- Un enregistreur de vol actif sur les services sensibles à la latence, avec un déclenchement sur le symptôme
  (latence, erreur) et un instantané à la fois.
- Trace et profil de contention capturés ensemble : qui attend, et qui fait attendre.
- `-race` en CI, en tests de stress, et sur une instance de pré-production sous charge.

## Ce que je dois retenir

- On choisit l'outil d'après la question : variable, plantage, temps, requête précise, blocage, course.
- `delve` : points d'arrêt conditionnels, goroutines, piles ; `dlv core` analyse un plantage sans le
  reproduire.
- `GOTRACEBACK=crash` donne un vidage mémoire, qui se garde comme un secret.
- L'enregistreur de vol capture les secondes avant un incident rare, pour 1,8 % du débit.
- `-race` : ×7 en temps, ×9 en mémoire, et 1 s à la sortie par défaut.

---

➡️ [Exercices](exercices.md)
