# Leçon 4 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Reproduire le diagnostic

`top` (trié par temps propre) commence par `regexp/syntax.appendRange`, `regexp/syntax.ranges.Less`,
`regexp/syntax.appendTable`, puis `runtime.memmove` : des fonctions **internes** de la compilation
d'expressions régulières et du *runtime*. Elles disent que le processeur travaille dans `regexp/syntax`,
mais pas **quelle ligne de notre code** l'y envoie, ni s'il faut compiler moins, ou appliquer moins, ou
autrement. `top -cum` répond : `words.TopSlow` (75 % cumulé), dont `regexp.MustCompile` 62 % ; et
`list TopSlow` désigne la ligne 22 (860 ms sur 1,05 s).

## E2 — Le *flamegraph*

Le rectangle `regexp.MustCompile` occupe environ **62 %** de la largeur sous `TopSlow` : c'est la même
valeur que la colonne *cum* de `top -cum` (la largeur d'un rectangle **est** le temps cumulé). Le
*flamegraph* a l'avantage de montrer d'un coup **tous** les chemins, et d'éviter de lire une liste triée
qui mélange fonctions de niveaux différents.

## E3 — Pas assez d'échantillons

```
benchtime=1x  Total samples = 40ms   ToLower 50 %, mapassign 50 %
benchtime=1x  Total samples = 50ms   ToLower 20 %, mapassign 60 %
benchtime=3s  Total samples = 3.14s  ToLower 47 %, mapassign 35 %
benchtime=3s  Total samples = 3.46s  ToLower 51 %, mapassign 36 %
```

Avec une seule itération, **4 à 5 échantillons** : chaque échantillon pèse 20 à 25 %, et les proportions
changent du tout au tout d'une exécution à l'autre. Avec 3 secondes, plus de 300 échantillons : les deux
profils concordent à quelques points près. Règle : quelques secondes de travail au minimum, ou plusieurs
profils fusionnés (`go tool pprof` accepte plusieurs fichiers).

## E4 — Le poste suivant

```go
// Ajouté à words.go
// TopLower : étape 3 — convertir le texte entier UNE fois, puis découper sans allouer par mot.
func TopLower(text string, k int) []Count {
	return TopScanLowered(strings.ToLower(text), k)
}

// TopScanLowered compte les mots d'un texte déjà en minuscules : les clés sont des sous-chaînes.
func TopScanLowered(text string, k int) []Count {
	counts := map[string]int{}
	start := -1
	for i, r := range text {
		if unicode.IsLetter(r) || r == '\'' {
			if start < 0 {
				start = i
			}
			continue
		}
		if start >= 0 {
			counts[text[start:i]]++
			start = -1
		}
	}
	if start >= 0 {
		counts[text[start:]]++
	}
	return top(counts, k)
}
```

Mesuré (médiane de 5) :

```
TopScan    37,3 ms   4 131 711 B/op   240 030 allocs/op
TopLower   23,2 ms   2 413 434 B/op        31 allocs/op
```

Une seule conversion pour tout le texte, et des **sous-chaînes** comme clés de map (elles ne copient rien) :
les 240 000 allocations par mot disparaissent. Nouveau profil : `strings.ToLower` du texte entier (49 %,
séquentiel, inévitable tant qu'on veut des minuscules), puis le découpage et `mapassign_faststr` (31 %).
On approche de ce que ce programme peut faire sur un cœur. Le résultat est identique (`TestLowerSame`).

Attention à un effet de bord de ce choix : les clés de la map **retiennent** le texte converti entier en
mémoire (une sous-chaîne garde vivant tout le tableau dont elle fait partie). Pour un résultat gardé
longtemps, on recopierait les mots retenus (`strings.Clone`).

## E5 — PGO

Programme `cmd/top` (lecture d'un fichier de 118 Mo, `TopScan`) ; profil collecté sur ce même fichier et
copié en `cmd/top/default.pgo` ; dix exécutions de chaque binaire, **alternées** :

```
sans PGO  3.26 3.44 3.48 3.50 3.54 3.55 3.57 3.66 3.67 3.90   médiane 3,545 s
avec PGO  3.12 3.18 3.35 3.39 3.45 3.45 3.45 3.53 3.59 3.61   médiane 3,450 s
```

Gain médian de **2,7 %**, mais test de rang : **p = 0,075** : pas de différence démontrée avec dix
exécutions sur cette machine (bruit de ±8 %). Un gain de cet ordre est cohérent avec ce qu'annonce la
documentation de Go (2 à 14 % selon les programmes), mais il faut plus de mesures, ou une machine plus
stable, pour l'établir. PGO est utile et gratuit ; il ne remplace pas les corrections des étapes 1 à 3
(×24 au total).

## Exercice intermédiaire — Profiler un service

Le service mesuré (avant correction) :

```go
// Fichier : cmd/svc/main.go
// Un service qui rend une liste de livres en JSON ; pprof sur une adresse d'administration.
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	_ "net/http/pprof" // routes /debug/pprof/ sur http.DefaultServeMux
	"strings"
	"time"
)

type Book struct {
	ID      int       `json:"id"`
	Title   string    `json:"title"`
	Authors []string  `json:"authors"`
	Added   time.Time `json:"added"`
}

func books(w http.ResponseWriter, r *http.Request) {
	out := make([]Book, 0, 200)
	for i := range 200 {
		out = append(out, Book{
			ID:      i,
			Title:   strings.ToUpper(fmt.Sprintf("titre numéro %d", i)),
			Authors: []string{fmt.Sprintf("auteur %d", i%17)},
			Added:   time.Date(2026, 1, 1, 0, 0, i, 0, time.UTC),
		})
	}
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(out)
}

func main() {
	go func() { log.Fatal(http.ListenAndServe("127.0.0.1:6061", nil)) }() // administration
	public := http.NewServeMux() // PAS le DefaultServeMux : /debug/pprof n'y est pas
	public.HandleFunc("GET /books", books)
	log.Fatal(http.ListenAndServe("127.0.0.1:8081", public))
}
```

```go
// Fichier : cmd/load/main.go
package main

import (
	"fmt"
	"io"
	"net/http"
	"sync"
	"sync/atomic"
	"time"
)

func main() {
	var n atomic.Int64
	stop := time.Now().Add(15 * time.Second)
	var wg sync.WaitGroup
	for range 8 {
		wg.Go(func() {
			for time.Now().Before(stop) {
				resp, err := http.Get("http://127.0.0.1:8081/books")
				if err == nil {
					io.Copy(io.Discard, resp.Body)
					resp.Body.Close()
					n.Add(1)
				}
			}
		})
	}
	wg.Wait()
	fmt.Println("requêtes :", n.Load())
}
```

1. **pprof sur l'adresse d'administration seulement** : le service public utilise son propre `ServeMux`,
   pas `http.DefaultServeMux` sur lequel `net/http/pprof` s'enregistre. Vérifié :
   `public /debug/pprof/ → 404`. (Erreur fréquente : `http.ListenAndServe(":8080", nil)` pour le service
   public, qui expose alors `/debug/pprof` à tout Internet.)
2. **Profil de 10 s sous charge** (`go tool pprof -top -cum http://127.0.0.1:6061/debug/pprof/profile?seconds=10`) :

   ```
   Duration: 10s, Total samples = 43.18s (431.75%)
     37.70s 87.31%  net/http.(*conn).serve
     33.80s 78.28%  main.books
     19.33s 44.77%  encoding/json.(*Encoder).Encode
      6.95s 16.10%  fmt.Sprintf
      4.59s 10.63%  strings.ToUpper
      4.56s 10.56%  syscall.Syscall
      3.47s  8.04%  runtime.mallocgc
   ```

   (431 % : en moyenne, 4,3 processeurs occupés pendant les 10 s.)
3. **Répartition** : le handler 78 %, dont JSON 45 % et la construction des titres (`Sprintf` +
   `ToUpper`) 27 % ; les appels système (écriture sur le réseau) 11 % ; le reste dans `net/http` et le
   *runtime*.
4. **Correction d'un poste** : la construction des données — titres déjà en majuscules par construction,
   `strconv.Itoa` au lieu de `fmt.Sprintf`, 17 noms d'auteurs calculés une fois :

```go
// Fichier : cmd/svc/main.go
// Un service qui rend une liste de livres en JSON ; pprof sur une adresse d'administration.
package main

import (
	"encoding/json"
	"log"
	"net/http"
	_ "net/http/pprof" // routes /debug/pprof/ sur http.DefaultServeMux
	"strconv"
	"time"
)

type Book struct {
	ID      int       `json:"id"`
	Title   string    `json:"title"`
	Authors []string  `json:"authors"`
	Added   time.Time `json:"added"`
}

var authors = func() []string {
	a := make([]string, 17)
	for i := range a {
		a[i] = "auteur " + strconv.Itoa(i)
	}
	return a
}()

func books(w http.ResponseWriter, r *http.Request) {
	out := make([]Book, 0, 200)
	for i := range 200 {
		out = append(out, Book{
			ID:      i,
			Title:   "TITRE NUMÉRO " + strconv.Itoa(i), // la casse est déjà la bonne : rien à convertir
			Authors: []string{authors[i%17]},           // 17 valeurs possibles : calculées une fois
			Added:   time.Date(2026, 1, 1, 0, 0, i, 0, time.UTC),
		})
	}
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(out)
}

func main() {
	go func() { log.Fatal(http.ListenAndServe("127.0.0.1:6061", nil)) }() // administration
	public := http.NewServeMux()                                          // PAS le DefaultServeMux : /debug/pprof n'y est pas
	public.HandleFunc("GET /books", books)
	log.Fatal(http.ListenAndServe("127.0.0.1:8081", public))
}
```

   Débit mesuré, trois charges de 15 s alternées : **avant** 84 077, 100 599, 92 433 requêtes ;
   **après** 130 798, 130 613, 132 588. Environ **+40 %**, pour un poste qui pesait 27 % du profil.

   Ce que le profil a révélé : le soupçon naturel portait sur le JSON (et il est bien le premier poste),
   mais il est coûteux à remplacer ; la construction des titres, que personne ne regardait, pesait un quart du
   temps et se corrigeait en trois lignes. Le profil ne dit pas seulement **où** ; il dit ce qui est **bon
   marché** à corriger.

## Défi — Un profil sous concurrence

```go
// Fichier : par.go
package words

import (
	"runtime"
	"strings"
	"sync"
	"unicode"
)

// countInto ajoute à counts les mots (déjà en minuscules) de text.
func countInto(text string, counts map[string]int) {
	start := -1
	for i, r := range text {
		if unicode.IsLetter(r) || r == '\'' {
			if start < 0 {
				start = i
			}
			continue
		}
		if start >= 0 {
			counts[text[start:i]]++
			start = -1
		}
	}
	if start >= 0 {
		counts[text[start:]]++
	}
}

// chunks découpe text en n morceaux, coupés aux fins de ligne.
func chunks(text string, n int) []string {
	var out []string
	for n > 1 && len(text) > 0 {
		cut := len(text) / n
		if j := strings.IndexByte(text[cut:], '\n'); j >= 0 {
			cut += j + 1
		} else {
			cut = len(text)
		}
		out = append(out, text[:cut])
		text = text[cut:]
		n--
	}
	if len(text) > 0 {
		out = append(out, text)
	}
	return out
}

// TopParallel : une map par goroutine, fusionnées à la fin.
func TopParallel(text string, k int) []Count {
	text = strings.ToLower(text)
	parts := chunks(text, runtime.GOMAXPROCS(0))
	maps := make([]map[string]int, len(parts))
	var wg sync.WaitGroup
	for i, p := range parts {
		wg.Go(func() {
			maps[i] = map[string]int{}
			countInto(p, maps[i])
		})
	}
	wg.Wait()
	total := maps[0]
	for _, m := range maps[1:] {
		for w, n := range m {
			total[w] += n
		}
	}
	return top(total, k)
}

// TopShared : une seule map partagée, sous verrou, à chaque mot.
func TopShared(text string, k int) []Count {
	text = strings.ToLower(text)
	parts := chunks(text, runtime.GOMAXPROCS(0))
	var mu sync.Mutex
	total := map[string]int{}
	var wg sync.WaitGroup
	for _, p := range parts {
		wg.Go(func() {
			start := -1
			add := func(w string) { mu.Lock(); total[w]++; mu.Unlock() }
			for i, r := range p {
				if unicode.IsLetter(r) || r == '\'' {
					if start < 0 {
						start = i
					}
					continue
				}
				if start >= 0 {
					add(p[start:i])
					start = -1
				}
			}
			if start >= 0 {
				add(p[start:])
			}
		})
	}
	wg.Wait()
	return top(total, k)
}
```

```go
// Fichier : par_test.go
package words

import (
	"slices"
	"testing"
)

func TestParallelSame(t *testing.T) {
	want := TopScan(text, 10)
	for name, f := range map[string]func(string, int) []Count{"par": TopParallel, "shared": TopShared} {
		if got := f(text, 10); !slices.Equal(got, want) {
			t.Errorf("%s : %v", name, got)
		}
	}
}

func BenchmarkTopParallel(b *testing.B) {
	for b.Loop() {
		TopParallel(text, 10)
	}
}

func BenchmarkTopShared(b *testing.B) {
	for b.Loop() {
		TopShared(text, 10)
	}
}
```

Mesuré (médiane de 3, `-cpu 1,2,4,8`) :

| Version | 1 cœur | 2 | 4 | 8 |
|---|---|---|---|---|
| `TopLower` (séquentielle) | 23,6 ms | 22,4 | 22,2 | 22,1 |
| `TopParallel` (une map par goroutine) | 23,4 ms | 18,1 | 17,2 | 15,8 |
| `TopShared` (une map, un verrou) | 27,3 ms | 29,4 | 34,4 | **41,4** |

- **Pas linéaire** : ×1,4 seulement à 8 cœurs. Le profil (`Total samples = 7.47s` pour `Duration: 3.60s`,
  soit 207 % : à peine deux cœurs occupés en moyenne) montre pourquoi : `strings.ToLower` du texte entier
  (34 %) reste **séquentiel**, comme la fusion et le tri final. La loi d'Amdahl : la partie séquentielle
  borne le gain, quel que soit le nombre de cœurs. Paralléliser aussi la conversion (chaque goroutine
  convertit son morceau) serait l'étape suivante.
- **La map partagée** devient **plus lente** à mesure qu'on ajoute des cœurs : chaque mot prend le verrou.
  Le profil CPU montre `sync.(*Mutex).Lock` à 42 % (le temps passé à **tenter** de prendre le verrou, à
  tourner, à se garer) — mais il ne montre pas le temps pendant lequel les goroutines sont **endormies** en
  attente : celui-là n'utilise pas le processeur. Pour le voir, il faut le **profil de contention** des
  verrous (`-mutexprofile`, leçon 5).

## Quiz

1. Il échantillonne les piles d'appels environ 100 fois par seconde et compte où elles se trouvent.
2. `go test -cpuprofile`, `runtime/pprof.StartCPUProfile`, `net/http/pprof`.
3. *flat* : temps dans la fonction elle-même ; *cum* : avec tout ce qu'elle appelle.
4. Parce qu'il désigne les fonctions de notre code qui, directement ou non, coûtent le plus.
5. Le source d'une fonction avec le temps de chaque ligne.
6. Son temps cumulé.
7. Pour savoir ce qui a compté, et parce que la cible suivante n'est plus la même.
8. Il révèle l'intérieur du service et permet de le charger ; on le place sur une adresse d'administration.
9. Le temps passé à attendre (réseau, base, verrous endormis).
10. L'optimisation guidée par profil : un profil de production nommé `default.pgo` dans le paquet
    principal, que `go build` utilise.
