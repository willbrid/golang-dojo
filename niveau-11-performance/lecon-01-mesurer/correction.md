# Leçon 1 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Ton bruit

Les valeurs dépendent de la machine. Sur celle de la leçon, au repos, `Join` varie de ±17 à ±18 %
autour de la médiane. Sous charge concurrente, on doit s'attendre à une médiane plus haute et une
étendue plus large — c'est ce que l'exercice demande de constater sur ta machine, et le test de rang le
dira s'il conclut à une différence (la charge **change** réellement les conditions de mesure). Leçon : on
ne compare jamais une série mesurée au repos à une série mesurée sous charge.

## E2 — Le compilateur malin

Mesuré :

```
BenchmarkSquareOld-8    0.3011 ns/op    (boucle b.N, résultat ignoré)
BenchmarkSquareLoop-8   1.478 ns/op     (b.Loop)
BenchmarkSquareSink-8   0.3700 ns/op    (boucle b.N, résultat dans une globale)
```

0,3 ns, c'est moins d'un cycle d'horloge : `Square` a été *inlinée*, et son résultat inutilisé **supprimé** ;
on mesure une boucle vide. Avec une globale, la multiplication reste, mais l'appel est toujours *inliné*.
`b.Loop` empêche l'élimination **et** l'*inlining* de l'appel dans la boucle : on mesure un vrai appel de
fonction (≈ 1,5 ns). Aucune des trois n'est « la bonne » dans l'absolu : pour une fonction aussi petite,
la question elle-même (« combien coûte `Square` ? ») n'a de sens que dans son contexte d'appel. C'est la
limite des micro-benchmarks de fonctions minuscules.

## E3 — Les tailles

```
Concat-10     957 ns       1 280 o     18 allocs     Builder-10     254 ns      240 o    4    Join-10     158 ns      128 o   1
Concat-100    48 473 ns    135 328 o   198 allocs    Builder-100    1 934 ns    3 312 o  8    Join-100    1 546 ns    1 408 o 1
Concat-1000   4 431 972 ns 14 559 814 o 1 998 allocs Builder-1000   23 958 ns   62 960 o 16   Join-1000   12 889 ns   14 336 o 1
                                                     Builder-10000  313 793 ns  686 834 o 24   Join-10000  161 027 ns  155 648 o 1
```

`Concat` est **quadratique** : à chaque tour, il recopie toute la chaîne déjà construite. Dix fois plus
d'éléments, cent fois plus de temps (957 ns → 48 µs → 4,4 ms) et de mémoire. `Builder` et `Join` sont
linéaires. `Builder` n'est jamais indiscernable de `Join` dans ces mesures : il reste environ deux fois plus
lent aux grandes tailles, à cause des réallocations du tampon (24 allocations pour 10 000 éléments) et de la
copie qui les accompagne.

## E4 — `Builder` préalloué

```
Join-8   ns/op   1157 → 1012   p=0.000   -12.58 %
Join-8   B/op    1408 → 1408   p=1.000   ~
```

(Comparaison faite avec l'outil de l'exercice intermédiaire.) Avec `Grow`, `Builder` alloue exactement
comme `Join` (une allocation de 1 408 octets), et il est même **12,6 %** plus rapide, différence démontrée.
Explication probable : notre version écrit un séparateur d'**un** octet (`WriteByte`), alors que
`strings.Join` gère un séparateur de longueur quelconque. Pour un code général, `strings.Join` reste le bon
choix ; le gain de 12 % n'a d'intérêt que si cette fonction pèse dans un profil.

## E5 — Une p-valeur trompeuse

Mesuré : 400 mesures du **même** `Join`, découpées en 20 paires de séries de 10 :

```
20 comparaisons du même code : 7 avec p < 0,05 (p min = 0.0003)
```

Avec un seuil de 5 %, on attendrait environ **une** fausse alerte sur vingt. On en obtient **sept**. Les
deux séries d'une paire ont été mesurées l'une **après** l'autre, et les conditions de la machine
**dérivent** (fréquence, température) : les séries ne sont pas indépendantes, ce que le test suppose. Deux
leçons :

- **alterner** les mesures (A, B, A, B…) ou les entrelacer, plutôt que toutes les mesures de A puis toutes
  celles de B ;
- quand on compare **beaucoup** de benchmarks, quelques « différences significatives » sont attendues par
  hasard. Une variation isolée de 2 % sur un benchmark parmi cinquante ne prouve rien ; il faut la
  reproduire.

## Exercice intermédiaire — Un mini `benchstat`

```go
// Fichier : cmpbench/stats.go
// Package cmpbench : comparer deux sorties de go test -bench (le principe de benchstat).
package cmpbench

import (
	"cmp"
	"math"
	"slices"
)

func Median(xs []float64) float64 {
	s := slices.Sorted(slices.Values(xs))
	n := len(s)
	if n == 0 {
		return math.NaN()
	}
	if n%2 == 1 {
		return s[n/2]
	}
	return (s[n/2-1] + s[n/2]) / 2
}

// MannWhitney : p-valeur bilatérale (approximation normale, correction de continuité et
// correction pour les égalités). Rend 1 si toutes les valeurs sont égales.
func MannWhitney(a, b []float64) float64 {
	type obs struct {
		v float64
		a bool
	}
	all := make([]obs, 0, len(a)+len(b))
	for _, v := range a {
		all = append(all, obs{v, true})
	}
	for _, v := range b {
		all = append(all, obs{v, false})
	}
	slices.SortFunc(all, func(x, y obs) int { return cmp.Compare(x.v, y.v) })
	n1, n2 := float64(len(a)), float64(len(b))
	n := n1 + n2
	var ra, ties float64
	for i := 0; i < len(all); {
		j := i
		for j < len(all) && all[j].v == all[i].v {
			j++
		}
		rank := float64(i+j+1) / 2
		t := float64(j - i)
		ties += t*t*t - t
		for k := i; k < j; k++ {
			if all[k].a {
				ra += rank
			}
		}
		i = j
	}
	u := ra - n1*(n1+1)/2
	variance := n1 * n2 / 12 * ((n + 1) - ties/(n*(n-1)))
	if variance <= 0 {
		return 1 // aucune variabilité : aucune différence démontrable
	}
	z := (math.Abs(u-n1*n2/2) - 0.5) / math.Sqrt(variance)
	if z < 0 {
		z = 0
	}
	return math.Erfc(z / math.Sqrt2)
}
```

```go
// Fichier : cmpbench/parse.go
package cmpbench

import (
	"bufio"
	"io"
	"strconv"
	"strings"
)

// Samples : nom du benchmark → unité → mesures.
type Samples map[string]map[string][]float64

// Parse lit la sortie de go test -bench ; les autres lignes sont ignorées.
func Parse(r io.Reader) (Samples, error) {
	s := Samples{}
	sc := bufio.NewScanner(r)
	for sc.Scan() {
		f := strings.Fields(sc.Text())
		if len(f) < 4 || !strings.HasPrefix(f[0], "Benchmark") {
			continue
		}
		if _, err := strconv.Atoi(f[1]); err != nil { // le nombre d'itérations
			continue
		}
		name := f[0]
		for i := 2; i+1 < len(f); i += 2 { // paires « valeur unité »
			v, err := strconv.ParseFloat(f[i], 64)
			if err != nil {
				break
			}
			if s[name] == nil {
				s[name] = map[string][]float64{}
			}
			s[name][f[i+1]] = append(s[name][f[i+1]], v)
		}
	}
	return s, sc.Err()
}
```

```go
// Fichier : cmpbench/compare.go
package cmpbench

import (
	"fmt"
	"io"
	"math"
	"slices"
	"strings"
)

type Row struct {
	Name, Unit    string
	Before, After float64
	Delta, P      float64
	Significant   bool
	Warning       string
}

const Alpha = 0.05

func Compare(before, after Samples) []Row {
	var rows []Row
	names := make([]string, 0, len(before))
	for n := range before {
		names = append(names, n)
	}
	slices.Sort(names)
	for _, n := range names {
		units := make([]string, 0)
		for u := range before[n] {
			units = append(units, u)
		}
		slices.Sort(units)
		for _, u := range units {
			a, b := before[n][u], after[n][u]
			if len(b) == 0 {
				continue
			}
			r := Row{Name: n, Unit: u, Before: Median(a), After: Median(b), P: MannWhitney(a, b)}
			if r.Before != 0 {
				r.Delta = (r.After - r.Before) / r.Before * 100
			}
			r.Significant = r.P <= Alpha
			if len(a) < 5 || len(b) < 5 {
				r.Significant, r.Warning = false, "moins de 5 mesures"
			}
			rows = append(rows, r)
		}
	}
	return rows
}

func Write(w io.Writer, rows []Row) {
	logSum, k := 0.0, 0
	for _, r := range rows {
		verdict := "~"
		if r.Significant {
			verdict = fmt.Sprintf("%+.2f %%", r.Delta)
			if r.Unit == "ns/op" && r.Before > 0 && r.After > 0 {
				logSum += math.Log(r.After / r.Before)
				k++
			}
		}
		extra := ""
		if r.Warning != "" {
			extra = " (" + r.Warning + ")"
		}
		fmt.Fprintf(w, "%-28s %-9s %12.4g %12.4g   p=%.3f  %s%s\n",
			strings.TrimPrefix(r.Name, "Benchmark"), r.Unit, r.Before, r.After, r.P, verdict, extra)
	}
	if k > 0 {
		fmt.Fprintf(w, "moyenne géométrique (ns/op significatifs) : %+.2f %%\n", (math.Exp(logSum/float64(k))-1)*100)
	}
}
```

```go
// Fichier : cmpbench/cmpbench_test.go
package cmpbench

import (
	"math"
	"strings"
	"testing"
)

const out = `goos: linux
cpu: Intel(R) Core(TM) i5-8265U CPU @ 1.60GHz
BenchmarkJoin-8      	  951049	      1176 ns/op	    1408 B/op	       1 allocs/op
BenchmarkJoin-8      	  982100	      1167 ns/op	    1408 B/op	       1 allocs/op
BenchmarkX-8         	     100	        12.5 ns/op	         3.000 widgets/op
PASS
ok  	join	12.3s
`

func TestParse(t *testing.T) {
	s, err := Parse(strings.NewReader(out))
	if err != nil {
		t.Fatal(err)
	}
	if got := s["BenchmarkJoin-8"]["ns/op"]; len(got) != 2 || got[0] != 1176 {
		t.Fatalf("ns/op = %v", got)
	}
	if got := s["BenchmarkJoin-8"]["allocs/op"]; len(got) != 2 || got[1] != 1 {
		t.Fatalf("allocs/op = %v", got)
	}
	if s["BenchmarkX-8"]["widgets/op"][0] != 3 {
		t.Fatalf("métrique personnalisée : %v", s["BenchmarkX-8"])
	}
}

func TestIdenticalValuesGiveP1(t *testing.T) {
	a := []float64{1, 1, 1, 1, 1, 1}
	if p := MannWhitney(a, a); p != 1 {
		t.Fatalf("p = %v", p)
	}
}

func TestClearDifference(t *testing.T) {
	a := []float64{10, 11, 12, 10, 11, 12, 10, 11, 12, 10}
	b := []float64{20, 21, 22, 20, 21, 22, 20, 21, 22, 23}
	if p := MannWhitney(a, b); p > 0.001 {
		t.Fatalf("p = %v", p)
	}
}

func TestUnequalSizesAndWarning(t *testing.T) {
	before := Samples{"BenchmarkA": {"ns/op": {1, 2, 3}}}
	after := Samples{"BenchmarkA": {"ns/op": {4, 5, 6, 7, 8, 9}}}
	r := Compare(before, after)
	if len(r) != 1 || r[0].Significant || r[0].Warning == "" || math.IsNaN(r[0].P) {
		t.Fatalf("%+v", r)
	}
}
```

Et la commande :

```go
// Fichier : cmd/cmpbench/main.go
package main

import (
	"fmt"
	"os"

	"exemple/cmpbench"
)

func main() {
	if len(os.Args) != 3 {
		fmt.Fprintln(os.Stderr, "usage : cmpbench avant.txt apres.txt")
		os.Exit(2)
	}
	var s [2]cmpbench.Samples
	for i, path := range os.Args[1:] {
		f, err := os.Open(path)
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		s[i], err = cmpbench.Parse(f)
		f.Close()
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
	}
	cmpbench.Write(os.Stdout, cmpbench.Compare(s[0], s[1]))
}
```

Sortie pour E4 :

```
Join-8                       B/op              1408         1408   p=1.000  ~
Join-8                       allocs/op            1            1   p=1.000  ~
Join-8                       ns/op             1157         1012   p=0.000  -12.58 %
moyenne géométrique (ns/op significatifs) : -12.58 %
```

Points notables :

- la **correction pour les égalités** dans la variance : sans elle, des séries entièrement identiques
  (`allocs/op` partout égal à 1) donneraient une division par zéro ; avec elle, la variance est nulle et
  la fonction rend p = 1 (testé) ;
- les **métriques personnalisées** (`b.ReportMetric`) sont lues comme les autres (`widgets/op`, testé) ;
- moins de 5 mesures : pas de conclusion, et un avertissement ;
- la moyenne géométrique résume plusieurs variations **relatives** (une moyenne arithmétique de
  pourcentages serait faussée par les grands écarts).

## Défi — Mesurer de bout en bout

```go
// Fichier : loadgen/loadgen.go
// Défi : un générateur de charge minimal, avec percentiles.
package loadgen

import (
	"context"
	"io"
	"net/http"
	"slices"
	"sync"
	"time"
)

type Result struct {
	Requests, Errors int
	Elapsed          time.Duration
	durations        []time.Duration
}

func (r Result) Throughput() float64 { return float64(r.Requests) / r.Elapsed.Seconds() }

// Percentile p (0 < p ≤ 100) par la méthode du rang le plus proche.
func (r Result) Percentile(p float64) time.Duration {
	if len(r.durations) == 0 {
		return 0
	}
	i := int(float64(len(r.durations))*p/100+0.5) - 1
	return r.durations[min(max(i, 0), len(r.durations)-1)]
}

// Run : workers goroutines envoient des requêtes en boucle pendant d.
func Run(ctx context.Context, client *http.Client, url string, workers int, d time.Duration) Result {
	ctx, cancel := context.WithTimeout(ctx, d)
	defer cancel()
	var mu sync.Mutex
	var res Result
	var wg sync.WaitGroup
	start := time.Now()
	for range workers {
		wg.Go(func() {
			var local []time.Duration
			errs := 0
			for ctx.Err() == nil {
				req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
				t := time.Now()
				resp, err := client.Do(req)
				if err != nil {
					if ctx.Err() == nil {
						errs++
					}
					continue
				}
				io.Copy(io.Discard, resp.Body)
				resp.Body.Close()
				if resp.StatusCode >= 500 {
					errs++
				}
				local = append(local, time.Since(t))
			}
			mu.Lock()
			res.durations = append(res.durations, local...)
			res.Errors += errs
			mu.Unlock()
		})
	}
	wg.Wait()
	res.Elapsed = time.Since(start)
	res.Requests = len(res.durations)
	slices.Sort(res.durations)
	return res
}
```

```go
// Fichier : loadgen/loadgen_test.go
package loadgen

import (
	"fmt"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"
)

func concat(parts []string) string {
	s := ""
	for i, p := range parts {
		if i > 0 {
			s += ","
		}
		s += p
	}
	return s
}

func TestEndToEnd(t *testing.T) {
	if testing.Short() {
		t.Skip("mesure longue")
	}
	for _, n := range []int{10, 100, 1000} {
		parts := make([]string, n)
		for i := range parts {
			parts[i] = fmt.Sprintf("élément-%d", i)
		}
		for name, join := range map[string]func([]string) string{
			"Join": func(p []string) string { return strings.Join(p, ",") }, "+=": concat,
		} {
			srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
				w.Write([]byte(join(parts)))
			}))
			res := Run(t.Context(), srv.Client(), srv.URL, 8, time.Second)
			srv.Close()
			t.Logf("n=%-5d %-4s %7.0f req/s  p50=%-8v p99=%-8v erreurs=%d", n, name, res.Throughput(),
				res.Percentile(50).Round(time.Microsecond), res.Percentile(99).Round(time.Microsecond), res.Errors)
		}
	}
}
```

Mesuré (8 workers, 1 s par mesure, serveur `httptest` sur la même machine) :

```
n=10    Join   42418 req/s  p50=125µs    p99=1.167ms
n=10    +=     39126 req/s  p50=128µs    p99=1.449ms
n=100   Join   37384 req/s  p50=132µs    p99=1.543ms
n=100   +=      6857 req/s  p50=831µs    p99=4.802ms
n=1000  Join   21358 req/s  p50=214µs    p99=2.202ms
n=1000  +=       178 req/s  p50=43.971ms p99=53.578ms
```

Conclusion :

1. À 10 éléments, le micro-benchmark annonçait ×6 sur la fonction ; de bout en bout, l'écart (≈ 8 %) est
   dans le bruit : la fonction est noyée dans le coût de HTTP.
2. À 100 éléments, l'écart de bout en bout (×5 en débit) est **bien plus grand** que ce que le temps de la
   fonction seule laissait prévoir : les 135 Ko alloués par requête font travailler le ramasse-miettes,
   qui ralentit **toutes** les requêtes (leçon 3) — un coût que le micro-benchmark ne pouvait pas montrer.
3. À 1 000 éléments, le service s'effondre (178 req/s, p50 à 44 ms). Le micro-benchmark dit **combien**,
   la mesure de bout en bout dit **si ça compte**, et ici les allocations comptaient plus que le temps.

## Quiz

1. Il exclut la préparation du chronomètre, empêche l'élimination et l'*inlining* de l'appel, et règle
   seul le nombre d'itérations.
2. Temps par itération, octets et nombre d'allocations sur le tas par itération ; les deux derniers sont
   déterministes.
3. Fréquence du processeur, autres programmes, ramasse-miettes (aussi : caches, alignement du code).
4. Elle n'est pas déformée par les valeurs extrêmes, et les temps ne suivent pas une loi normale.
5. Si les mesures d'une série sont systématiquement mieux classées que celles de l'autre ; la p-valeur est
   la probabilité d'un tel écart si les deux versions étaient identiques.
6. Que la différence n'est pas démontrée : le bruit pourrait l'expliquer.
7. Il calcule la taille totale d'abord, puis alloue une seule fois.
8. L'outil officiel de comparaison de benchmarks (`golang.org/x/perf/cmd/benchstat`) : `benchstat
   avant.txt apres.txt` sur deux sorties de `go test -bench -count 10`.
9. Si la fonction compte dans le temps total, et les effets globaux (ramasse-miettes, contention).
10. Parce qu'avec un seuil de 5 %, quelques fausses alertes sont attendues par hasard, et davantage quand
    les mesures dérivent dans le temps.
