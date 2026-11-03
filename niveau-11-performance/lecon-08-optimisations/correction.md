# Leçon 8 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Le code des exercices faciles, ajouté au paquet `opt` :

```go
// Fichier : fix.go
package opt

import (
	"bufio"
	"bytes"
	"crypto/sha256"
	"encoding/binary"
	"io"
	"sync"
)

// E2 : trois façons de compter les lignes.

type CountingReader struct {
	R     io.Reader
	Calls int
}

func (c *CountingReader) Read(p []byte) (int, error) {
	c.Calls++
	return c.R.Read(p)
}

func LinesByte(r io.Reader) (int, error) {
	var b [1]byte
	n := 0
	for {
		_, err := r.Read(b[:])
		if err == io.EOF {
			return n, nil
		}
		if err != nil {
			return n, err
		}
		if b[0] == '\n' {
			n++
		}
	}
}

func LinesBufio(r io.Reader) (int, error) {
	br := bufio.NewReader(r)
	n := 0
	for {
		c, err := br.ReadByte()
		if err == io.EOF {
			return n, nil
		}
		if err != nil {
			return n, err
		}
		if c == '\n' {
			n++
		}
	}
}

func LinesScanner(r io.Reader) (int, error) {
	sc := bufio.NewScanner(r)
	n := 0
	for sc.Scan() {
		n++
	}
	return n, sc.Err()
}

// E3 : LE BUG. Le tampon est rendu au pool, puis ses octets sont renvoyés à l'appelant.
func RenderBad(recs []Rec) []byte {
	buf := bufPool.Get().(*bytes.Buffer)
	buf.Reset()
	var line [64]byte
	for _, r := range recs {
		buf.Write(AppendRec(line[:0], r))
	}
	bufPool.Put(buf)   // une autre goroutine peut le reprendre dès maintenant…
	return buf.Bytes() // …et écrire dans les octets qu'on renvoie
}

// E4 : le même rendu, avec ou sans plafond.
var giantPool = sync.Pool{New: func() any { return new(bytes.Buffer) }}

func RenderPooledCap(recs []Rec, limit int, send func([]byte)) {
	buf := giantPool.Get().(*bytes.Buffer)
	buf.Reset()
	Render(buf, recs, send)
	if limit == 0 || buf.Cap() <= limit {
		giantPool.Put(buf)
	}
}

// E5 (b) : un travail de calcul par élément.
func HashSum(s []int64) (t uint64) {
	var b [8]byte
	for _, x := range s {
		binary.LittleEndian.PutUint64(b[:], uint64(x))
		h := sha256.Sum256(b[:])
		t += binary.LittleEndian.Uint64(h[:8])
	}
	return
}

func HashSumParallel(s []int64, parts int) uint64 {
	res := make([]uint64, parts)
	var wg sync.WaitGroup
	size := (len(s) + parts - 1) / parts
	for p := range parts {
		lo, hi := min(p*size, len(s)), min((p+1)*size, len(s))
		wg.Go(func() { res[p] = HashSum(s[lo:hi]) })
	}
	wg.Wait()
	var t uint64
	for _, x := range res {
		t += x
	}
	return t
}
```

```go
// Fichier : fix_test.go
package opt

import (
	"fmt"
	"io"
	"os"
	"path/filepath"
	"runtime"
	"sync"
	"testing"
	"time"
)

func csvFile(t testing.TB) string {
	p := filepath.Join(t.TempDir(), "in.csv")
	f, err := os.Create(p)
	if err != nil {
		t.Fatal(err)
	}
	if err := WriteAppend(f, recs(10_000)); err != nil {
		t.Fatal(err)
	}
	if err := f.Close(); err != nil {
		t.Fatal(err)
	}
	return p
}

// E2
func TestReaders(t *testing.T) {
	p := csvFile(t)
	for _, tc := range []struct {
		name string
		f    func(r io.Reader) (int, error)
	}{{"octet par octet", LinesByte}, {"bufio.ReadByte", LinesBufio}, {"bufio.Scanner", LinesScanner}} {
		f, err := os.Open(p)
		if err != nil {
			t.Fatal(err)
		}
		cr := &CountingReader{R: f}
		start := time.Now()
		n, err := tc.f(cr)
		el := time.Since(start)
		f.Close()
		if err != nil || n != 10_000 {
			t.Fatalf("%s : %d lignes, %v", tc.name, n, err)
		}
		fmt.Printf("%-16s %7d appels à Read  %v\n", tc.name, cr.Calls, el.Round(10*time.Microsecond))
	}
}

// E3 : 8 goroutines, chacune ses propres données ; chacune vérifie ce qu'elle reçoit.
func TestRenderBad(t *testing.T) {
	var wg sync.WaitGroup
	var mu sync.Mutex
	bad := 0
	for g := range 8 {
		mine := []Rec{{ID: int64(g), Name: fmt.Sprintf("goroutine-%d", g), Price: 100}}
		want := string(AppendRec(nil, mine[0]))
		wg.Go(func() {
			for range 20_000 {
				got := RenderBad(mine)
				runtime.Gosched() // laisser les autres travailler avant de lire
				if string(got) != want {
					mu.Lock()
					bad++
					mu.Unlock()
				}
			}
		})
	}
	wg.Wait()
	t.Logf("%d réponses corrompues sur %d", bad, 8*20_000)
}

// E4 : une réponse sur 1 000 fait environ 4 Mio.
func TestGiant(t *testing.T) {
	small, big := recs(80), recs(170_000) // ≈ 2 Kio et ≈ 4 Mio
	for _, limit := range []int{64 << 10, 0} {
		runtime.GC()
		var peak uint64
		stop := make(chan struct{})
		done := make(chan struct{})
		go func() { // échantillonne le tas pendant la charge
			defer close(done)
			var m runtime.MemStats
			for {
				select {
				case <-stop:
					return
				case <-time.After(20 * time.Millisecond):
					runtime.ReadMemStats(&m)
					peak = max(peak, m.HeapInuse)
				}
			}
		}()
		start := time.Now()
		var wg sync.WaitGroup
		for g := range 8 {
			wg.Go(func() {
				total := 0
				send := func(p []byte) { total += len(p) }
				for i := range 100_000 / 8 {
					r := small
					if (i*8+g)%1000 == 0 {
						r = big
					}
					RenderPooledCap(r, limit, send)
				}
			})
		}
		wg.Wait()
		el := time.Since(start)
		close(stop)
		<-done
		var m runtime.MemStats
		inuse := func() uint64 { runtime.ReadMemStats(&m); return m.HeapInuse >> 20 }
		end := inuse()
		runtime.GC()
		gc1 := inuse()
		runtime.GC()
		gc2 := inuse()
		name := "plafond 64 Kio"
		if limit == 0 {
			name = "sans plafond"
		}
		fmt.Printf("%-15s %v  HeapInuse (Mio) : max %d, fin %d, après 1 GC %d, après 2 GC %d\n",
			name, el.Round(time.Millisecond), peak>>20, end, gc1, gc2)
		giantPool = sync.Pool{New: giantPool.New} // repartir d'un pool vide
	}
}

// E5
func BenchmarkSumThreshold(b *testing.B) {
	for _, n := range []int{5_000, 10_000, 20_000, 50_000} {
		s := make([]int64, n)
		b.Run(fmt.Sprintf("seq-%d", n), func(b *testing.B) {
			for b.Loop() {
				sinkI = Sum(s)
			}
		})
		b.Run(fmt.Sprintf("par8-%d", n), func(b *testing.B) {
			for b.Loop() {
				sinkI = SumParallel(s, 8)
			}
		})
	}
}

var sinkU uint64

func BenchmarkHash(b *testing.B) {
	for _, n := range []int{1_000, 100_000} {
		s := make([]int64, n)
		for i := range s {
			s[i] = int64(i)
		}
		if HashSum(s) != HashSumParallel(s, 8) {
			b.Fatal("résultats différents")
		}
		b.Run(fmt.Sprintf("seq-%d", n), func(b *testing.B) {
			for b.Loop() {
				sinkU = HashSum(s)
			}
		})
		b.Run(fmt.Sprintf("par8-%d", n), func(b *testing.B) {
			for b.Loop() {
				sinkU = HashSumParallel(s, 8)
			}
		})
	}
}
```

## E1 — Amdahl avant d'écrire

```
$ go test -run XXX -bench 'Write/bufio$' -benchtime 3s -cpuprofile cpu.out -o opt.test
$ go tool pprof -top -cum opt.test cpu.out
      flat  flat%   sum%        cum   cum%
         0     0%     0%      3.64s 95.79%  exemple.WriteBuffered
     0.16s  4.21%  4.21%      3.64s 95.79%  exemple.WriteNaive
     0.04s  1.05%  5.26%      2.76s 72.63%  fmt.Fprintf
     0.54s 14.21% 19.47%      2.05s 53.95%  fmt.(*pp).doPrintf
…
     0.09s  2.37% 43.42%      0.38s 10.00%  runtime.convT64
     0.07s  1.84% 45.26%      0.34s  8.95%  bufio.(*Writer).Write
     0.04s  1.05% 50.00%      0.34s  8.95%  runtime.convTstring
     0.01s  0.26% 62.37%      0.20s  5.26%  bufio.(*Writer).Flush
```

Ce qui revient à `fmt` se compose de deux parties :

- `fmt.Fprintf` lui-même : 72,6 % ;
- les conversions des arguments en `any`, faites dans `WriteNaive` avant l'appel (`convT64`,
  `convTstring`) : 19 %.

`fmt` compte donc pour p ≈ 92 % du temps. En supprimant `fmt` entièrement (s = ∞), Amdahl donne un gain
maximal de 1 / 0,08 ≈ **×12**. Le gain mesuré avec `AppendRec` est de ×5,2 (2,86 → 0,55 ms).

L'écart vient de ce que le travail de mise en forme n'a pas disparu : il est fait plus simplement, par
`AppendRec` et `strconv.AppendInt`. Le temps restant se répartit ainsi :

- la construction des lignes ;
- la copie dans le tampon de `bufio` ;
- les 58 appels système (`Flush` en prend à lui seul 5 %).

Amdahl donne un **plafond**, jamais une promesse. Ici, il disait que l'étape valait la peine (×12 au mieux),
et elle a donné ×5.

## E2 — Lire aussi

```
octet par octet   236601 appels à Read  109.09ms
bufio.ReadByte        59 appels à Read  1.19ms
bufio.Scanner         59 appels à Read  380µs
```

- **Un `Read` d'un octet par appel système** : 236 601 appels, un par octet du fichier, plus le dernier qui
  renvoie `io.EOF`. Il faut 109 ms.
- **`bufio.Reader`** lit par blocs de 4 Kio : 59 appels, **×90** plus rapide.
- **`bufio.Scanner`** gagne encore ×3. Il cherche les fins de ligne avec `bytes.IndexByte` (optimisé en
  assembleur, leçon 4) au lieu d'examiner un octet à la fois dans le code Go.

Les durées viennent d'une seule exécution (`time.Since`) : elles donnent l'ordre de grandeur, pas une mesure
fine (leçon 1).

## E3 — Le bug du pool

```
$ go test -v -run RenderBad
    fix_test.go:73: 118420 réponses corrompues sur 160000
$ go test -race -v -run RenderBad
WARNING: DATA RACE
…
    fix_test.go:73: 2 réponses corrompues sur 160000
--- FAIL: TestRenderBad
```

**Sans le détecteur**, 74 % des réponses contiennent les données d'**une autre goroutine**. Dans un service, ce
serait la réponse d'un autre client : une fuite de données entre utilisateurs, pas seulement un bug.

**Avec `-race`**, le détecteur signale la course. Le `Put` ne crée de relation « arrivé-avant » qu'avec le
`Get` qui suit. La lecture de `got`, faite après le `Put`, n'est ordonnée par rien avec l'écriture de la
goroutine qui a repris le tampon.

Remarque : sous `-race`, presque aucune réponse n'est corrompue (2 sur 160 000). En mode `-race`, `sync.Pool`
jette volontairement un objet rendu sur quatre (voir `Put` dans `sync/pool.go`), et les tampons sont donc moins souvent partagés. Le
détecteur trouve tout de même la course. D'où la règle de N6 : **les tests concurrents tournent sous
`-race`**.

Pourquoi ce bug est-il si dangereux ?

- Il ne se voit **pas** dans un test séquentiel : une seule goroutine relit toujours ses propres données.
- Il dépend de la charge, donc il apparaît en production.
- Il produit des réponses **plausibles** : du JSON valide, mais celui d'un autre client.

Le correctif : ne rendre le tampon au pool qu'après la **dernière** utilisation de ses octets. Si l'appelant
doit garder les octets, on lui en donne une **copie** (`bytes.Clone(buf.Bytes())`). On peut aussi passer une
fonction (`send`) qui les consomme avant le `Put`, comme le fait `RenderPooled`.

## E4 — Le tampon géant

```
plafond 64 Kio  1.437s  HeapInuse (Mio) : max 31, fin 23, après 1 GC 9, après 2 GC 9
sans plafond    1.014s  HeapInuse (Mio) : max 31, fin 31, après 1 GC 17, après 2 GC 9
```

La mesure ne donne pas le résultat attendu, et c'est ce qui la rend instructive.

- **Sans plafond, c'est plus rapide** (1,0 s contre 1,4 s). Avec plafond, chacune des 100 réponses géantes
  repart d'un tampon vide et le fait grandir jusqu'à 4 Mio, en une quinzaine de recopies (leçon 6).
- **Le pic de mémoire est le même** (31 Mio). Avec plafond, les tampons géants sont alloués puis abandonnés
  au GC. Sans plafond, ils sont gardés.
- **La différence est la mémoire retenue.** À la fin de la charge, le pool sans plafond garde ses tampons
  géants : 31 Mio contre 23. Il les garde encore après un cycle du GC (17 contre 9), à cause du cache de
  secours de `sync.Pool`. Il ne les lâche qu'après le deuxième.

Que garde le pool sans plafond ? Au pire, un tampon géant par processeur (P) et par emplacement du pool, et ces
tampons resservent pour des réponses de 2 Kio. Ici, cela coûte environ 8 Mio sur 8 processeurs. Sur une
machine à 64 processeurs avec des réponses de 100 Mio, ce serait plusieurs Gio retenus pour rien.

Le plafond n'est donc pas toujours le bon choix. C'est un **compromis** : de la mémoire bornée contre une
réallocation des grosses réponses. Le défi mesure une troisième voie, qui évite les deux défauts.

## E5 — Le seuil du parallélisme

| Somme de n `int64` | séquentielle | 8 goroutines |
|---|---|---|
| 5 000 | 2,0 µs | 6,7 µs |
| 10 000 | 3,9 µs | 9,5 µs |
| 20 000 | 9,5 µs | 16,3 µs |
| 50 000 | 25,4 µs | 28,4 µs |
| 100 000 (cours) | 46,5 µs | 32,3 µs |

(a) Sur cette machine, la version parallèle ne gagne qu'**entre 50 000 et 100 000** éléments, soit plus de
6 000 éléments par goroutine. Son coût fixe n'est pas constant : il augmente avec n, de 3,5 µs à 1 000
éléments jusqu'à 16 µs à 20 000. Une hypothèse, non vérifiée : chaque goroutine démarre sur un autre cœur,
dont les caches ne contiennent pas encore sa part du tableau (leçon 6).

| Somme de `sha256` de n éléments | séquentielle | 8 goroutines | gain |
|---|---|---|---|
| 1 000 | 226 µs | 107 µs | ×2,1 |
| 100 000 | 22,5 ms | 7,2 ms | ×3,1 |

(b) Avec un vrai **calcul** (environ 225 ns par élément), le parallélisme paie dès 1 000 éléments. À 100 000,
il donne ×3,1. Ce n'est pas ×8, car la machine n'a que 4 cœurs physiques : les 8 fils matériels partagent deux
à deux les unités de calcul d'un cœur.

La somme, elle, ne fait presque aucun calcul par élément. Elle passe son temps à attendre la mémoire, que les
cœurs se partagent. **Le parallélisme accélère le calcul, pas l'accès à la mémoire.**

---

## Exercice intermédiaire — Optimiser un programme de bout en bout

Les cinq étapes dans un même paquet. Pour comparer, on mesure par un benchmark plutôt que par `time` : le
fichier de 1 000 000 de lignes (environ 34 Mo) est généré une fois et lu à chaque itération.

```go
// Fichier : ventes/ventes.go
// Package ventes : un rapport de ventes, optimisé étape par étape.
package ventes

import (
	"bufio"
	"bytes"
	"cmp"
	"encoding/csv"
	"errors"
	"fmt"
	"io"
	"math/rand/v2"
	"slices"
	"strconv"
)

// Generate écrit n lignes date,produit,quantité,prix_unitaire sur products produits.
func Generate(w io.Writer, n, products int) error {
	bw := bufio.NewWriter(w)
	rng := rand.New(rand.NewPCG(1, 2))
	for range n {
		fmt.Fprintf(bw, "2026-%02d-%02d,produit-%05d,%d,%d.%02d\n",
			1+rng.IntN(12), 1+rng.IntN(28), rng.IntN(products), 1+rng.IntN(9), rng.IntN(200), rng.IntN(100))
	}
	return bw.Flush()
}

// --- Étape 0 : la version naïve ---

type statsF struct {
	qty int64
	ca  float64
}

func Report0(r io.Reader, w io.Writer) error {
	rows, err := csv.NewReader(r).ReadAll()
	if err != nil {
		return err
	}
	m := map[string]*statsF{}
	for _, row := range rows {
		q, err := strconv.Atoi(row[2])
		if err != nil {
			return err
		}
		p, err := strconv.ParseFloat(row[3], 64)
		if err != nil {
			return err
		}
		s := m[row[1]]
		if s == nil {
			s = &statsF{}
			m[row[1]] = s
		}
		s.qty += int64(q)
		s.ca += float64(q) * p
	}
	keys := make([]string, 0, len(m))
	for k := range m {
		keys = append(keys, k)
	}
	slices.SortFunc(keys, func(a, b string) int { return cmp.Or(cmp.Compare(m[b].ca, m[a].ca), cmp.Compare(a, b)) })
	for _, k := range keys {
		if _, err := fmt.Fprintf(w, "%s;%d;%.2f\n", k, m[k].qty, m[k].ca); err != nil {
			return err
		}
	}
	return nil
}

// --- Étape 1 : lire ligne par ligne (Read + ReuseRecord) au lieu de ReadAll ---

func Report1(r io.Reader, w io.Writer) error {
	cr := csv.NewReader(r)
	cr.ReuseRecord = true
	m := map[string]*statsF{}
	for {
		row, err := cr.Read()
		if err == io.EOF {
			break
		}
		if err != nil {
			return err
		}
		q, err := strconv.Atoi(row[2])
		if err != nil {
			return err
		}
		p, err := strconv.ParseFloat(row[3], 64)
		if err != nil {
			return err
		}
		s := m[row[1]]
		if s == nil {
			s = &statsF{}
			m[row[1]] = s
		}
		s.qty += int64(q)
		s.ca += float64(q) * p
	}
	return writeF(w, m)
}

func writeF(w io.Writer, m map[string]*statsF) error {
	keys := make([]string, 0, len(m))
	for k := range m {
		keys = append(keys, k)
	}
	slices.SortFunc(keys, func(a, b string) int { return cmp.Or(cmp.Compare(m[b].ca, m[a].ca), cmp.Compare(a, b)) })
	for _, k := range keys {
		if _, err := fmt.Fprintf(w, "%s;%d;%.2f\n", k, m[k].qty, m[k].ca); err != nil {
			return err
		}
	}
	return nil
}

// --- Étape 2 : bufio en sortie ---

func Report2(r io.Reader, w io.Writer) error {
	bw := bufio.NewWriter(w)
	if err := Report1(r, bw); err != nil {
		return err
	}
	return bw.Flush()
}

// --- Étape 3 : des centimes entiers au lieu de float64 (exact) ---

type stats struct {
	qty, cents int64
}

var errPrice = errors.New("prix invalide")

// parseCents lit "12.34" (deux décimales exactement) en 1234.
func parseCents(b []byte) (int64, error) {
	i := bytes.IndexByte(b, '.')
	if i < 0 || len(b)-i != 3 {
		return 0, errPrice
	}
	var v int64
	for j, c := range b {
		if j == i {
			continue
		}
		if c < '0' || c > '9' {
			return 0, errPrice
		}
		v = v*10 + int64(c-'0')
	}
	return v, nil
}

func Report3(r io.Reader, w io.Writer) error {
	cr := csv.NewReader(r)
	cr.ReuseRecord = true
	m := map[string]*stats{}
	for {
		row, err := cr.Read()
		if err == io.EOF {
			break
		}
		if err != nil {
			return err
		}
		q, err := strconv.Atoi(row[2])
		if err != nil {
			return err
		}
		c, err := parseCents([]byte(row[3]))
		if err != nil {
			return err
		}
		s := m[row[1]]
		if s == nil {
			s = &stats{}
			m[row[1]] = s
		}
		s.qty += int64(q)
		s.cents += int64(q) * c
	}
	return writeCents(w, m)
}

func writeCents(w io.Writer, m map[string]*stats) error {
	bw := bufio.NewWriter(w)
	keys := make([]string, 0, len(m))
	for k := range m {
		keys = append(keys, k)
	}
	slices.SortFunc(keys, func(a, b string) int { return cmp.Or(cmp.Compare(m[b].cents, m[a].cents), cmp.Compare(a, b)) })
	var line []byte
	for _, k := range keys {
		s := m[k]
		line = append(line[:0], k...)
		line = append(line, ';')
		line = strconv.AppendInt(line, s.qty, 10)
		line = append(line, ';')
		line = strconv.AppendInt(line, s.cents/100, 10)
		line = append(line, '.')
		if s.cents%100 < 10 {
			line = append(line, '0')
		}
		line = strconv.AppendInt(line, s.cents%100, 10)
		line = append(line, '\n')
		if _, err := bw.Write(line); err != nil {
			return err
		}
	}
	return bw.Flush()
}

// --- Étape 4 : découper les lignes soi-même, sans encoding/csv ---
// Le format est simple et maîtrisé (pas de guillemets) : c'est ce qui rend l'étape légitime.

func Report4(r io.Reader, w io.Writer) error {
	sc := bufio.NewScanner(r)
	sc.Buffer(make([]byte, 64<<10), 1<<20)
	m := make(map[string]*stats, 16_384)
	for sc.Scan() {
		line := sc.Bytes()
		var f [4][]byte
		for i := range 3 {
			j := bytes.IndexByte(line, ',')
			if j < 0 {
				return fmt.Errorf("ligne invalide : %q", sc.Text())
			}
			f[i], line = line[:j], line[j+1:]
		}
		f[3] = line
		q, err := strconv.Atoi(string(f[2])) // le compilateur n'alloue pas pour cette conversion
		if err != nil {
			return err
		}
		c, err := parseCents(f[3])
		if err != nil {
			return err
		}
		s := m[string(f[1])] // recherche sans allocation
		if s == nil {
			s = &stats{}
			m[string(f[1])] = s // allocation de la clé, une fois par produit
		}
		s.qty += int64(q)
		s.cents += int64(q) * c
	}
	if err := sc.Err(); err != nil {
		return err
	}
	return writeCents(w, m)
}
```

```go
// Fichier : ventes/ventes_test.go
package ventes

import (
	"bytes"
	"fmt"
	"os"
	"path/filepath"
	"testing"
)

var reports = []struct {
	name string
	f    func(r *os.File, w *os.File) error
}{
	{"0-naif", func(r, w *os.File) error { return Report0(r, w) }},
	{"1-lecture-en-flux", func(r, w *os.File) error { return Report1(r, w) }},
	{"2-bufio", func(r, w *os.File) error { return Report2(r, w) }},
	{"3-centimes", func(r, w *os.File) error { return Report3(r, w) }},
	{"4-sans-csv", func(r, w *os.File) error { return Report4(r, w) }},
}

const small = `2026-01-01,b,2,1.10
2026-01-02,a,1,3.05
2026-01-03,b,1,0.85
2026-01-04,c,3,1.00
`

// a et b ont le même chiffre d'affaires exact (3,05) : départage par nom, a d'abord.
const want = "a;1;3.05\nb;3;3.05\nc;3;3.00\n"

func TestSmall(t *testing.T) {
	for i, f := range []func(*bytes.Reader, *bytes.Buffer) error{
		func(r *bytes.Reader, w *bytes.Buffer) error { return Report3(r, w) },
		func(r *bytes.Reader, w *bytes.Buffer) error { return Report4(r, w) },
	} {
		var out bytes.Buffer
		if err := f(bytes.NewReader([]byte(small)), &out); err != nil {
			t.Fatal(i, err)
		}
		if out.String() != want {
			t.Errorf("étape %d :\n%s", i+3, out.String())
		}
	}
}

// Les étapes 0 à 2 calculent en float64 : 2 × 1,10 + 0,85 n'y vaut pas exactement 3,05.
func TestFloatBug(t *testing.T) {
	var out bytes.Buffer
	if err := Report0(bytes.NewReader([]byte(small)), &out); err != nil {
		t.Fatal(err)
	}
	q, p1, p2 := 2.0, 1.10, 0.85 // des variables : calcul en float64 à l'exécution
	fmt.Printf("étape 0 :\n%s(b : %.17g)\n", out.String(), q*p1+p2)
}

func bigFile(tb testing.TB) string {
	p := filepath.Join(os.TempDir(), "ventes-1M.csv")
	if _, err := os.Stat(p); err == nil {
		return p
	}
	f, err := os.Create(p)
	if err != nil {
		tb.Fatal(err)
	}
	if err := Generate(f, 1_000_000, 10_000); err != nil {
		tb.Fatal(err)
	}
	if err := f.Close(); err != nil {
		tb.Fatal(err)
	}
	return p
}

func run(tb testing.TB, f func(r, w *os.File) error, out string) {
	r, err := os.Open(bigFile(tb))
	if err != nil {
		tb.Fatal(err)
	}
	defer r.Close()
	w, err := os.Create(out)
	if err != nil {
		tb.Fatal(err)
	}
	defer w.Close()
	if err := f(r, w); err != nil {
		tb.Fatal(err)
	}
}

// Les sorties des étapes sur le gros fichier : identiques ?
func TestBigSame(t *testing.T) {
	if testing.Short() {
		t.Skip()
	}
	dir := t.TempDir()
	var outs [][]byte
	for _, rp := range reports {
		p := filepath.Join(dir, rp.name)
		run(t, rp.f, p)
		b, _ := os.ReadFile(p)
		outs = append(outs, b)
	}
	for i := 1; i < len(outs); i++ {
		same := bytes.Equal(outs[0], outs[i])
		diff := 0
		la, lb := bytes.Split(outs[0], []byte("\n")), bytes.Split(outs[i], []byte("\n"))
		for j := range min(len(la), len(lb)) {
			if !bytes.Equal(la[j], lb[j]) {
				diff++
			}
		}
		fmt.Printf("étape %d identique à l'étape 0 : %v (%d lignes différentes)\n", i, same, diff)
	}
}

func BenchmarkReport(b *testing.B) {
	bigFile(b)
	out := filepath.Join(b.TempDir(), "out")
	for _, rp := range reports {
		b.Run(rp.name, func(b *testing.B) {
			for b.Loop() {
				run(b, rp.f, out)
			}
		})
	}
}
```

```
$ go test -v -run 'Small|FloatBug|BigSame'
--- PASS: TestSmall (0.00s)
étape 0 :
b;3;3.05
a;1;3.05
c;3;3.00
(b : 3.0500000000000003)
--- PASS: TestFloatBug (0.00s)
étape 1 identique à l'étape 0 : true (0 lignes différentes)
étape 2 identique à l'étape 0 : true (0 lignes différentes)
étape 3 identique à l'étape 0 : false (26 lignes différentes)
étape 4 identique à l'étape 0 : false (26 lignes différentes)
```

| Étape | Prédiction (profil de l'étape précédente) | Temps mesuré (médiane de 5) | Allocations | Lignes ajoutées |
|---|---|---|---|---|
| 0 — naïve | — | 546 ms | 2 040 142 | — |
| 1 — `Read` + `ReuseRecord` | `ReadAll` : 41 %, plus une partie des 30 % de GC dus aux 225 Mo retenus | 420 ms (×1,30) | 1 040 106 | ≈ 10 |
| 2 — `bufio` en sortie | `Fprintf` absent du haut du profil : ≈ 0 | 415 ms (rien de mesurable) | 1 040 107 | 4 |
| 3 — centimes | `ParseFloat` : 10 %, donc ×1,11 au mieux | 369 ms (×1,12) | 1 010 103 | ≈ 45 |
| 4 — découpe manuelle | `encoding/csv` : 40 %, donc ×1,67 au mieux | 178 ms (×2,07) | 20 079 | ≈ 30 |

Au total, ×3,1, et 100 fois moins d'allocations.

**Lecture du tableau :**

- **Étape 1.** `ReadAll` construit les 1 000 000 de lignes en mémoire (225 Mo alloués) avant d'en lire une.
  Lire en flux, avec `ReuseRecord` pour réutiliser la slice des champs, divise les allocations par deux.
- **Étape 2.** Le rapport ne fait que 10 000 lignes : écrire sans tampon n'était pas un problème. La règle du
  niveau voudrait qu'on retire cette étape. On la garde, car l'écriture de l'étape 3 la contient de toute
  façon.
- **Étape 3.** La prédiction est exacte : ×1,11 prévu, ×1,12 mesuré.
- **Étape 4.** Le gain **dépasse** la prédiction : ×2,07 pour ×1,67 au mieux. Le profil répartit les
  échantillons sur **tous** les fils, y compris ceux où le GC travaille en parallèle. Or supprimer un million
  d'allocations allège aussi l'aide au marquage que le GC demande au code (leçon 3), et les caches sont moins
  encombrés. Amdahl s'applique à la durée **de bout en bout**, et le profil n'en donne qu'une approximation.

  Cette étape n'est légitime que parce que le format est **maîtrisé** : pas de guillemets, pas de virgules
  dans les noms. Sur un CSV venu de l'extérieur, on garde `encoding/csv`.

**Où s'arrêter ?** Le profil de l'étape 4 :

```
     0.02s  0.57% 11.75%      2.04s 58.45%  runtime.mapaccess1_faststr
     0.49s 14.04% 25.79%      2.02s 57.88%  runtime.mapaccess2_faststr
     0.69s 19.77% 45.56%      0.69s 19.77%  internal/runtime/maps.ctrlGroup.matchH2 (inline)
     0.56s 16.05% 61.60%      0.56s 16.05%  memeqbody
     0.10s  2.87% 64.47%      0.39s 11.17%  bufio.(*Scanner).Scan
```

Le premier poste est maintenant la **map** (58 %) : un hachage, puis la comparaison de la clé (`memeqbody`).
C'est le travail lui-même, regrouper par produit.

On pourrait aller plus loin en exploitant le format des noms (`produit-01234` → un indice dans une slice),
mais ce serait lier le code à un détail des données de test. Le profil n'est pas plat, mais **sans objectif qui
l'exige, on s'arrête là**. C'est le second critère de la leçon.

**5. Le résultat a changé à l'étape 3, et c'est l'étape 3 qui a raison.** En `float64`, 2 × 1,10 + 0,85 vaut
3,0500000000000003 : le produit b passe devant le produit a, alors que leurs chiffres d'affaires sont égaux.
Sur le gros fichier, **26 lignes** du rapport sont dans un ordre différent. Les sommes de millions de
`float64` accumulent ces erreurs d'arrondi, et deux montants égaux ne le sont plus.

Les centimes entiers (`int64`) sont **exacts**. Pour de l'argent, c'est la seule représentation correcte
(N8 L2 : `numeric` en base). L'optimisation a ici corrigé un bug : il faut le dire dans le message de commit,
et ajouter le test `TestSmall` qui le fixe.

---

## Défi — Un pool par classe de taille

```go
// Fichier : tiered/tiered.go
// Package tiered : des tampons réutilisés, rangés par classe de taille.
package tiered

import "sync"

// BufPool est l'interface commune aux stratégies comparées.
type BufPool interface {
	Get(n int) *[]byte // un tampon de longueur n
	Put(b *[]byte)
}

// None alloue à chaque fois.
type None struct{}

func (None) Get(n int) *[]byte { b := make([]byte, n); return &b }
func (None) Put(*[]byte)       {}

// Single : un seul sync.Pool ; limit > 0 jette les tampons plus grands que limit.
// On met des *[]byte dans le pool : un []byte converti en any allouerait à chaque Put.
type Single struct {
	p     sync.Pool
	limit int
}

func NewSingle(limit int) *Single { return &Single{limit: limit} }

func (s *Single) Get(n int) *[]byte {
	if v := s.p.Get(); v != nil {
		b := v.(*[]byte)
		if cap(*b) >= n {
			*b = (*b)[:n]
			return b
		}
		// trop petit : on le jette, et on alloue à la bonne taille
	}
	b := make([]byte, n)
	return &b
}

func (s *Single) Put(b *[]byte) {
	if s.limit > 0 && cap(*b) > s.limit {
		return
	}
	s.p.Put(b)
}

// Tiered : un pool par classe (4 Kio, 64 Kio, 1 Mio). Un tampon d'une classe a exactement sa capacité.
type Tiered struct {
	pools [3]sync.Pool
}

var classes = [3]int{4 << 10, 64 << 10, 1 << 20}

func class(n int) int {
	for i, c := range classes {
		if n <= c {
			return i
		}
	}
	return -1
}

func (t *Tiered) Get(n int) *[]byte {
	i := class(n)
	if i < 0 { // hors classe : allocation simple, jamais rendue au pool
		b := make([]byte, n)
		return &b
	}
	if v := t.pools[i].Get(); v != nil {
		b := v.(*[]byte)
		*b = (*b)[:n]
		return b
	}
	b := make([]byte, n, classes[i])
	return &b
}

func (t *Tiered) Put(b *[]byte) {
	i := class(cap(*b))
	if i < 0 || cap(*b) != classes[i] {
		return // pas un tampon de classe : on le laisse au GC
	}
	t.pools[i].Put(b)
}
```

```go
// Fichier : tiered/tiered_test.go
package tiered

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"runtime/metrics"
	"sync"
	"testing"
	"time"
)

// size tire une taille : 90 % sous 4 Kio, 9 % de 4 à 64 Kio, 1 % de 64 Kio à 1 Mio.
func size(r *rand.Rand) int {
	switch x := r.IntN(100); {
	case x < 90:
		return 256 + r.IntN(4<<10-256)
	case x < 99:
		return 4<<10 + r.IntN(60<<10)
	default:
		return 64<<10 + r.IntN(960<<10)
	}
}

// work simule l'écriture d'une réponse : on remplit le tampon.
func work(b []byte) {
	for i := range b {
		b[i] = byte(i)
	}
}

func strategies() []struct {
	name string
	p    BufPool
} {
	return []struct {
		name string
		p    BufPool
	}{
		{"sans-pool", None{}},
		{"pool-sans-plafond", NewSingle(0)},
		{"pool-plafond-64K", NewSingle(64 << 10)},
		{"pool-par-classe", &Tiered{}},
	}
}

func TestTiered(t *testing.T) {
	var p Tiered
	for _, n := range []int{1, 4 << 10, 4<<10 + 1, 1 << 20, 2 << 20} {
		b := p.Get(n)
		if len(*b) != n {
			t.Fatalf("Get(%d) : longueur %d", n, len(*b))
		}
		p.Put(b)
	}
}

func BenchmarkPools(b *testing.B) {
	for _, s := range strategies() {
		b.Run(s.name, func(b *testing.B) {
			b.RunParallel(func(pb *testing.PB) {
				r := rand.New(rand.NewPCG(rand.Uint64(), 1))
				for pb.Next() {
					buf := s.p.Get(size(r))
					work(*buf)
					s.p.Put(buf)
				}
			})
		})
	}
}

// Mémoire, allocations et cycles du GC pour 400 000 réponses sur 8 goroutines.
func TestMemory(t *testing.T) {
	read := func() (alloc, cycles uint64) {
		m := []metrics.Sample{{Name: "/gc/heap/allocs:bytes"}, {Name: "/gc/cycles/total:gc-cycles"}}
		metrics.Read(m)
		return m[0].Value.Uint64(), m[1].Value.Uint64()
	}
	for _, s := range strategies() {
		runtime.GC()
		a0, c0 := read()
		var peak uint64
		stop, done := make(chan struct{}), make(chan struct{})
		go func() {
			defer close(done)
			var m runtime.MemStats
			for {
				select {
				case <-stop:
					return
				case <-time.After(10 * time.Millisecond):
					runtime.ReadMemStats(&m)
					peak = max(peak, m.HeapInuse)
				}
			}
		}()
		start := time.Now()
		var wg sync.WaitGroup
		for g := range 8 {
			wg.Go(func() {
				r := rand.New(rand.NewPCG(uint64(g), 1))
				for range 50_000 {
					buf := s.p.Get(size(r))
					work(*buf)
					s.p.Put(buf)
				}
			})
		}
		wg.Wait()
		el := time.Since(start)
		close(stop)
		<-done
		a1, c1 := read()
		fmt.Printf("%-18s %6v  HeapInuse max %3d Mio  alloué %6d Mio  cycles du GC %4d\n",
			s.name, el.Round(time.Millisecond), peak>>20, (a1-a0)>>20, c1-c0)
	}
}
```

**1. Éviter l'allocation du `Put`.** Un `[]byte` est un en-tête de trois mots (pointeur, longueur, capacité).
Le convertir en `any` le recopie sur le tas : une allocation à chaque `Put`. On met donc des `*[]byte` dans
le pool. Un pointeur tient dans un `any` sans allocation, et on modifie la slice à travers lui
(`*b = (*b)[:n]`).

**2. Mesures** (8 goroutines ; benchmark : médianes de 5 ; test : 400 000 réponses, deux exécutions
concordantes) :

| Stratégie | ns/réponse | B/op | Temps (400 000) | `HeapInuse` max | Alloué | Cycles du GC |
|---|---|---|---|---|---|---|
| sans pool | 3 530 | 11 159 | 1,43 s | 38–43 Mio | 4 155 Mio | 872 |
| un pool, sans plafond | 1 376 | 3 | 0,52 s | 19–20 Mio | 55–63 Mio | 9 |
| un pool, plafond 64 Kio | 2 670 | 6 693 | 1,0 s | 18–39 Mio | 2 510 Mio | 516–537 |
| pool par classe | 1 399 | 0 | 0,52–0,73 s | **12 Mio** | **11 Mio** | **2** |

- **Le plafond de 64 Kio coûte cher avec cette distribution.** Le 1 % de réponses de plus de 64 Kio
  (544 Kio en moyenne) est réalloué à chaque fois : 2,5 Gio alloués et 500 cycles du GC. Pour ce 1 %, c'est
  comme s'il n'y avait pas de pool.
- **Le pool sans plafond est rapide**, mais ses tampons dérivent vers la plus grande taille. Quand un tampon
  trop petit sort du pool, on le jette et on en alloue un plus grand. Au bout d'un moment, le pool ne
  contient plus que des tampons proches de 1 Mio, prêtés pour des réponses de 2 Kio. La mémoire
  retenue grandit avec le nombre de processeurs, comme dans E4.
- **Le pool par classe a les deux qualités** : aussi rapide que le pool sans plafond, et 40 % de mémoire
  retenue en moins (12 contre 19 Mio). Un tampon de 4 Kio sert aux réponses de 4 Kio, et un tampon de 1 Mio
  n'est prêté qu'aux réponses qui en ont besoin.

**3. Dans la bibliothèque standard** (`grep -rn sync.Pool $(go env GOROOT)/src/net/http $(go env GOROOT)/src/fmt`) :

- `net/http` (`server.go`) a des pools **par taille** pour ses `bufio.Writer` : `bufioWriter2kPool` et
  `bufioWriter4kPool`, choisis par `bufioWriterPool(size)`. Il a aussi un pool de tampons de copie de taille
  fixe (`copyBufPool`).
- `fmt` (`print.go`) a un seul pool, **avec plafond**. Son commentaire dans `free` explique pourquoi : « Proper
  usage of a sync.Pool requires each entry to have approximately the same memory cost » (chaque entrée doit
  avoir à peu près le même coût en mémoire). Les tampons de plus de 64 Kio ne sont pas rendus (ticket Go 23199).

Les deux réponses au même problème coexistent donc dans la bibliothèque standard : des pools par taille quand
les tailles sont connues (2 et 4 Kio), un plafond quand elles sont quelconques.

**4. Conclusion.** Face au pool simple **avec plafond**, le pool par classe est ×1,9 plus rapide et alloue
200 fois moins, **pour cette distribution**. Face au pool sans plafond, il ne gagne que de la mémoire
retenue.

Il ajoute 50 lignes et un risque : un tampon rendu à la mauvaise classe. Ce risque est écarté par la
vérification `cap(*b) != classes[i]`.

Il se justifie si le profil montre des allocations de tampons parmi les premiers postes, et si les tailles
sont **étalées**. Si elles sont régulières, un seul pool avec plafond suffit. Comme toujours, c'est la
mesure sur **la vraie** distribution des tailles qui décide.

---

## Quiz

1. gain = 1 / ((1 − p) + p / s). Pour p = 20 %, même avec s infini : 1 / 0,8 = **×1,25** au mieux.
2. Ne pas faire le travail (algorithme, index, requêtes), regrouper les E/S, réduire les allocations,
   réutiliser (`sync.Pool`), paralléliser, et les micro-optimisations en dernier.
3. Un appel système par petite écriture coûte cher, et le tampon les regroupe sans changer le code d'écriture
   (×3,3 ici, et ×90 en lecture dans E2). Le piège : oublier le `Flush` final, ou ignorer son erreur. C'est
   souvent lui qui révèle l'échec réel de l'écriture.
4. `AppendX(dst []byte, …) []byte` ajoute au tampon fourni et renvoie le tampon agrandi. L'appelant réutilise
   le même tampon (`dst[:0]`). Tant que sa capacité suffit, il n'y a aucune allocation, et `dst` peut même
   être un tableau sur la pile (leçon 2).
5. Remettre l'objet à zéro après `Get`. Ne plus le toucher après `Put`. Ne pas rendre d'objets géants
   (plafond, ou pools par taille). Mettre des pointeurs, pas des valeurs. Ne s'en servir que pour des objets
   temporaires, jamais comme cache.
6. Il resservirait pour de petites utilisations et garderait sa mémoire. Au pire, le pool retient un objet
   géant par processeur. Les entrées d'un pool doivent avoir à peu près le même coût, comme le dit le code
   de `fmt`.
7. Le GC le vide : un objet non repris disparaît en deux cycles. On ne peut pas y retrouver une donnée
   précise, seulement « un objet libre ».
8. La somme est limitée par la **bande passante mémoire**, partagée par tous les cœurs. Huit goroutines
   attendent la mémoire ensemble. Le parallélisme accélère le calcul, pas les transferts.
9. Une `string` doit être immuable. Si les octets d'origine changent, la `string` construite par `unsafe`
   change aussi : une garantie du langage est cassée (clés de map, comparaisons). Le gain est rarement
   mesurable, et le compilateur évite déjà la copie dans les cas courants (`m[string(b)]`, les
   comparaisons).
10. Quand l'objectif chiffré est atteint, ou quand le profil est plat. Sans objectif ni mesure, on ne sait pas
    s'arrêter, et on complique le code pour rien.
