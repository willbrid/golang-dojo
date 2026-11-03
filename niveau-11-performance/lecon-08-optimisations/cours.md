# Leçon 8 — Optimisations qui valent le coup

> Les leçons précédentes ont appris à mesurer et à trouver le goulot. Reste à décider **quoi faire**. Toutes
> les optimisations ne se valent pas. Certaines divisent le temps par dix pour trois lignes changées, d'autres
> compliquent le code pour un gain invisible. Cette leçon les range par rendement et mesure chacune sur un
> même exemple : écrire des lignes dans un fichier, construire une réponse. Elle se termine par ce qu'il ne
> faut pas faire, et par la question qui clôt le niveau : **quand s'arrêter**.

## Objectifs

- Estimer le gain maximal d'une optimisation avant de l'écrire (loi d'Amdahl).
- Essayer les optimisations dans l'ordre de leur rendement : E/S, allocations, réutilisation, parallélisme.
- Utiliser `bufio`, les fonctions `Append*` et la préallocation.
- Utiliser `sync.Pool` correctement, et savoir quand il ne sert à rien.
- Reconnaître les fausses optimisations, et savoir quand s'arrêter.

## Explication

### Avant d'optimiser : combien peut-on gagner ?

Si une partie du programme prend une fraction p du temps total, et qu'on la rend s fois plus rapide, le gain
global est de :

```
gain = 1 / ((1 − p) + p / s)
```

C'est la **loi d'Amdahl**. Quelques exemples :

| Ce qu'on accélère | Gain global |
|---|---|
| ×10 sur une partie qui fait 10 % du temps | ×1,1 |
| ×2 sur une partie qui fait 80 % du temps | ×1,7 |
| une partie qui fait 10 % du temps, rendue infiniment rapide | ×1,11 au mieux |

Le profil (leçons 4 et 5) donne p. Avant toute optimisation, faire le calcul. Si le gain maximal ne se
verrait pas de bout en bout (leçon 7), ne pas l'écrire.

### L'ordre des optimisations

Du meilleur rendement au plus faible :

1. **Ne pas faire le travail.** Un meilleur algorithme, un index, un cache bien borné, une requête SQL qui
   ne charge que ce qu'il faut (N8 L8). Rien ne bat un travail supprimé.
2. **Les entrées-sorties.** Un appel système coûte de l'ordre de la microseconde, un aller-retour réseau
   des centaines de microsecondes à des millisecondes. On regroupe :
   - un `bufio.Writer` devant un fichier ou une connexion ;
   - des insertions par lots (N8 L8) ;
   - une seule requête au lieu de N (le N+1, N8 L8).
3. **Les allocations.** On préalloue (leçon 6), on construit les octets avec `strconv.Append*` au lieu de
   `fmt` (leçon 2), et on **réutilise** un tampon dans une boucle (`line = AppendRec(line[:0], r)`).
4. **La réutilisation entre appels** avec `sync.Pool`, pour de **gros** objets temporaires dans du code
   concurrent.
5. **Le parallélisme**, pour un travail de calcul **assez gros** et indépendant.
6. **Les micro-optimisations** : ordre des champs, faux partage (leçon 6). Seulement là où le profil
   les désigne.

### Les entrées-sorties

`fmt.Fprintf(f, …)` sur un `*os.File` fait **un appel système par ligne**. Placer un `bufio.Writer` (tampon
de 4 Kio par défaut) devant le fichier divise ce nombre par ~170 dans l'exemple. Le code d'écriture ne change
pas. Il faut seulement penser au `Flush` final et vérifier son erreur.

C'est presque toujours la première optimisation à faire quand un programme écrit beaucoup. Elle est aussi la
plus oubliée, car le code « a l'air » efficace.

### `sync.Pool`

Un `sync.Pool` garde des objets temporaires pour les réutiliser au lieu de les réallouer (N6 L7).
Ses règles :

- **Remettre l'objet dans son état initial** après `Get` (`buf.Reset()`). Le pool rend un objet déjà
  utilisé.
- **Ne plus toucher à l'objet après `Put`.** Une autre goroutine peut l'avoir déjà repris. Un `[]byte` rendu
  à l'appelant qui pointe dans un tampon remis au pool est un bug grave : les données changent sous ses
  yeux. Dans l'exemple, `send` doit avoir fini d'utiliser les octets avant de rendre la main.
- **Ne pas garder les objets géants.** Si une requête sur mille construit une réponse de 10 Mo et remet le
  tampon au pool, ce tampon de 10 Mo resservira pour des réponses de 1 Kio. La mémoire reste occupée pour
  rien. On jette les objets trop gros au lieu de les rendre (`if buf.Cap() <= 64<<10`). C'est ce que fait `fmt` lui-même pour ses tampons internes, avec la même limite de 64 Kio (voir `free` dans `$(go env GOROOT)/src/fmt/print.go`). Le prix à payer : les rares grosses réponses sont réallouées à chaque fois. L'exercice E4 et le défi mesurent ce compromis.
- **Mettre des pointeurs dans le pool** (`*bytes.Buffer`, pas `bytes.Buffer`). Sinon, la conversion en `any`
  alloue à chaque `Put` (leçon 2).
- **Le pool n'est pas un cache.** Le GC peut le vider : un objet survit au plus deux cycles s'il n'est pas
  repris. On ne s'en sert jamais pour garder des données.

Le pool est utile pour les objets **gros** (des tampons de plusieurs Kio) et **fréquents**, dans du code
**concurrent** : il garde un cache par processeur, et `Get` ne prend aucun verrou dans le cas courant (le
défi de la leçon 6 l'a vérifié). Pour un petit objet, le gain mesuré est de quelques nanosecondes.

### Le parallélisme

Lancer une goroutine et attendre sa fin coûte de l'ordre de quelques centaines de nanosecondes : l'exemple mesure environ 3 µs pour 8 goroutines. Paralléliser
n'est donc rentable que si chaque part de travail dure **bien plus** que ce coût. De plus, un calcul limité
par la **bande passante mémoire** (parcourir un grand tableau) ne va pas plus vite avec 8 cœurs, qui se
partagent le même bus mémoire. L'exemple le mesure.

### Ce qu'il ne faut pas faire

- **Optimiser sans profil**, ou garder une optimisation qu'aucune mesure ne justifie : c'est la règle du
  niveau.
- **`unsafe` pour éviter une copie** `string` ↔ `[]byte`. Une `string` construite ainsi peut changer si les
  octets changent, ce qui casse une garantie du langage, et le gain est rarement mesurable. Le compilateur
  évite déjà la copie dans les cas courants : `m[string(b)]`, une comparaison `string(b) == "x"`, une boucle
  `for … range string(b)`.
- **Mettre en pool de petits objets** « par principe ».
- **Des caches sans borne ni expiration** : c'est la fuite mémoire de la leçon 5 (E5).
- **Une goroutine par petit morceau de travail.**
- **Dérouler les boucles à la main, deviner l'*inlining*, remplacer les interfaces par des `switch`** :
  le compilateur fait déjà une partie de ce travail, et PGO (leçon 4) s'occupe du reste là où le profil le
  justifie.
- **Sacrifier la lisibilité d'un code froid.** Un code appelé une fois par requête et qui prend 1 % du temps
  doit rester simple.

### Quand s'arrêter

On s'arrête quand l'**objectif** est atteint : un p99 sous 50 ms, un traitement de nuit sous une heure, une
facture de machines divisée par deux. On s'arrête aussi quand le profil ne montre plus de poste dominant :
le temps est réparti sur cent fonctions qui en prennent chacune 1 %.

Sans objectif chiffré, on ne s'arrête jamais. Sans mesure, on ne sait pas si on l'a atteint.

## Exemple

Le même travail écrit de plusieurs façons. On écrit 10 000 lignes `id;nom;prix` dans un fichier, et on
construit une réponse dans un tampon.

```go
// Fichier : opt.go
// Package opt : les optimisations qui valent le coup, dans l'ordre où les essayer.
package opt

import (
	"bufio"
	"bytes"
	"fmt"
	"io"
	"strconv"
	"sync"
)

type Rec struct {
	ID    int64
	Name  string
	Price int64 // en centimes
}

// --- 1. Les entrées-sorties : un appel système par ligne, ou un tampon ---

// CountingWriter compte les appels à Write (sur un fichier, chacun est un appel système).
type CountingWriter struct {
	W     io.Writer
	Calls int
}

func (c *CountingWriter) Write(p []byte) (int, error) {
	c.Calls++
	return c.W.Write(p)
}

// WriteNaive : fmt.Fprintf directement sur la destination.
func WriteNaive(w io.Writer, recs []Rec) error {
	for _, r := range recs {
		if _, err := fmt.Fprintf(w, "%d;%s;%d.%02d\n", r.ID, r.Name, r.Price/100, r.Price%100); err != nil {
			return err
		}
	}
	return nil
}

// WriteBuffered : le même code derrière un bufio.Writer.
func WriteBuffered(w io.Writer, recs []Rec) error {
	bw := bufio.NewWriter(w)
	if err := WriteNaive(bw, recs); err != nil {
		return err
	}
	return bw.Flush()
}

// AppendRec ajoute la ligne de r à dst, sans fmt ni allocation.
func AppendRec(dst []byte, r Rec) []byte {
	dst = strconv.AppendInt(dst, r.ID, 10)
	dst = append(dst, ';')
	dst = append(dst, r.Name...)
	dst = append(dst, ';')
	dst = strconv.AppendInt(dst, r.Price/100, 10)
	dst = append(dst, '.')
	if c := r.Price % 100; c < 10 {
		dst = append(dst, '0')
	}
	dst = strconv.AppendInt(dst, r.Price%100, 10)
	return append(dst, '\n')
}

// WriteAppend : un tampon, et des lignes construites sans fmt.
func WriteAppend(w io.Writer, recs []Rec) error {
	bw := bufio.NewWriter(w)
	var line []byte
	for _, r := range recs {
		line = AppendRec(line[:0], r)
		if _, err := bw.Write(line); err != nil {
			return err
		}
	}
	return bw.Flush()
}

// --- 2. sync.Pool : réutiliser de gros tampons temporaires ---

// Render construit une réponse de n lignes dans un tampon, et la passe à send.
func Render(buf *bytes.Buffer, recs []Rec, send func([]byte)) {
	var line [64]byte
	for _, r := range recs {
		buf.Write(AppendRec(line[:0], r))
	}
	send(buf.Bytes())
}

func RenderAlloc(recs []Rec, send func([]byte)) {
	var buf bytes.Buffer
	Render(&buf, recs, send)
}

// RenderGrow : sans pool, mais avec une estimation de la taille finale.
func RenderGrow(recs []Rec, send func([]byte)) {
	var buf bytes.Buffer
	buf.Grow(len(recs) * 32)
	Render(&buf, recs, send)
}

var bufPool = sync.Pool{New: func() any { return new(bytes.Buffer) }}

func RenderPooled(recs []Rec, send func([]byte)) {
	buf := bufPool.Get().(*bytes.Buffer)
	buf.Reset()
	Render(buf, recs, send)
	if buf.Cap() <= 64<<10 { // ne pas garder les tampons géants (voir le texte)
		bufPool.Put(buf)
	}
}

// --- 3. Un petit objet : le pool gagne peu ---

type Point struct{ X, Y, Z float64 }

var pointPool = sync.Pool{New: func() any { return new(Point) }}

//go:noinline
func use(p *Point) float64 { return p.X + p.Y + p.Z }

// --- 4. Paralléliser : seulement si le travail est assez gros ---

func Sum(s []int64) (t int64) {
	for _, x := range s {
		t += x
	}
	return
}

// SumParallel découpe s en parts égales, une goroutine par part.
func SumParallel(s []int64, parts int) int64 {
	res := make([]int64, parts)
	var wg sync.WaitGroup
	size := (len(s) + parts - 1) / parts
	for p := range parts {
		lo, hi := min(p*size, len(s)), min((p+1)*size, len(s))
		wg.Go(func() { res[p] = Sum(s[lo:hi]) }) // une écriture par goroutine : pas de faux partage gênant
	}
	wg.Wait()
	return Sum(res)
}
```

```go
// Fichier : opt_test.go
package opt

import (
	"bytes"
	"fmt"
	"os"
	"path/filepath"
	"testing"
)

func recs(n int) []Rec {
	s := make([]Rec, n)
	for i := range s {
		s[i] = Rec{ID: int64(i), Name: fmt.Sprintf("article-%d", i%1000), Price: int64(i * 37 % 100000)}
	}
	return s
}

func TestSameOutput(t *testing.T) {
	r := recs(1000)
	var a, b, c bytes.Buffer
	WriteNaive(&a, r)
	WriteBuffered(&b, r)
	WriteAppend(&c, r)
	if a.String() != b.String() || a.String() != c.String() {
		t.Fatal("sorties différentes")
	}
	if got := string(AppendRec(nil, Rec{7, "x", 5})); got != "7;x;0.05\n" {
		t.Fatalf("AppendRec = %q", got)
	}
}

func TestSyscalls(t *testing.T) {
	r := recs(10_000)
	for _, tc := range []struct {
		name string
		f    func(*CountingWriter, []Rec) error
	}{
		{"naïf", func(w *CountingWriter, r []Rec) error { return WriteNaive(w, r) }},
		{"bufio", func(w *CountingWriter, r []Rec) error { return WriteBuffered(w, r) }},
		{"bufio + append", func(w *CountingWriter, r []Rec) error { return WriteAppend(w, r) }},
	} {
		var buf bytes.Buffer
		cw := &CountingWriter{W: &buf}
		tc.f(cw, r)
		fmt.Printf("%-15s %6d appels à Write pour %d lignes\n", tc.name, cw.Calls, len(r))
	}
}

func BenchmarkWrite(b *testing.B) {
	r := recs(10_000)
	f, err := os.Create(filepath.Join(b.TempDir(), "out.csv"))
	if err != nil {
		b.Fatal(err)
	}
	defer f.Close()
	for _, tc := range []struct {
		name string
		f    func(*os.File, []Rec) error
	}{
		{"naif", func(f *os.File, r []Rec) error { return WriteNaive(f, r) }},
		{"bufio", func(f *os.File, r []Rec) error { return WriteBuffered(f, r) }},
		{"bufio+append", func(f *os.File, r []Rec) error { return WriteAppend(f, r) }},
	} {
		b.Run(tc.name, func(b *testing.B) {
			for b.Loop() {
				f.Seek(0, 0)
				if err := tc.f(f, r); err != nil {
					b.Fatal(err)
				}
			}
		})
	}
}

func BenchmarkRender(b *testing.B) {
	for _, n := range []int{10, 200} {
		r := recs(n)
		b.Run(fmt.Sprintf("alloc-%d", n), func(b *testing.B) {
			b.RunParallel(func(pb *testing.PB) {
				total := 0 // un compteur par goroutine : pas de variable globale partagée (leçon 6)
				send := func(p []byte) { total += len(p) }
				for pb.Next() {
					RenderAlloc(r, send)
				}
			})
		})
		b.Run(fmt.Sprintf("grow-%d", n), func(b *testing.B) {
			b.RunParallel(func(pb *testing.PB) {
				total := 0
				send := func(p []byte) { total += len(p) }
				for pb.Next() {
					RenderGrow(r, send)
				}
			})
		})
		b.Run(fmt.Sprintf("pool-%d", n), func(b *testing.B) {
			b.RunParallel(func(pb *testing.PB) {
				total := 0
				send := func(p []byte) { total += len(p) }
				for pb.Next() {
					RenderPooled(r, send)
				}
			})
		})
	}
}

var (
	sinkF float64
	sinkP *Point
)

func BenchmarkSmall(b *testing.B) {
	b.Run("new", func(b *testing.B) {
		for b.Loop() {
			p := &Point{1, 2, 3}
			sinkF = use(p)
			sinkP = p // l'objet survit à l'itération : il doit vivre sur le tas
		}
	})
	b.Run("pool", func(b *testing.B) {
		for b.Loop() {
			p := pointPool.Get().(*Point)
			*p = Point{1, 2, 3}
			sinkF = use(p)
			pointPool.Put(p)
		}
	})
}

var sinkI int64

func BenchmarkSum(b *testing.B) {
	for _, n := range []int{1_000, 100_000, 10_000_000} {
		s := make([]int64, n)
		for i := range s {
			s[i] = int64(i)
		}
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

func TestSumParallel(t *testing.T) {
	s := []int64{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11}
	for parts := 1; parts <= 16; parts++ {
		if got := SumParallel(s, parts); got != 66 {
			t.Fatalf("parts=%d : %d", parts, got)
		}
	}
}
```

```
$ go test -v -run 'Same|Syscalls|SumParallel'
--- PASS: TestSameOutput (0.00s)
naïf             10000 appels à Write pour 10000 lignes
bufio               58 appels à Write pour 10000 lignes
bufio + append      58 appels à Write pour 10000 lignes
--- PASS: TestSyscalls (0.01s)
--- PASS: TestSumParallel (0.00s)
```

Les benchmarks (`go test -bench . -benchmem -count 5`, médianes) :

| 10 000 lignes dans un fichier | Temps | Allocations |
|---|---|---|
| `fmt.Fprintf` sur le fichier | 9,50 ms | 26 977 |
| `bufio.Writer` + `fmt.Fprintf` | 2,86 ms (×3,3) | 26 979 |
| `bufio.Writer` + `AppendRec` | 0,55 ms (×17) | 4 |

| Construire une réponse (8 goroutines) | 10 lignes | 200 lignes |
|---|---|---|
| `bytes.Buffer` neuf | 281 ns, 3 allocations | 8,96 µs, 8 allocations |
| neuf, avec `Grow` | 304 ns, 1 allocation | 4,43 µs, 1 allocation |
| `sync.Pool` | 142 ns, 0 allocation | 2,77 µs, 0 allocation |

| Un objet de 24 octets | Temps |
|---|---|
| `&Point{…}` sur le tas | 21,2 ns |
| `sync.Pool` | 16,8 ns |

| Somme de n `int64` | séquentielle | 8 goroutines |
|---|---|---|
| n = 1 000 | 0,39 µs | 3,5 µs (×9 plus **lent**) |
| n = 100 000 | 46,5 µs | 32,3 µs (×1,4) |
| n = 10 000 000 | 10,4 ms | 8,9 ms (×1,2) |

**Lecture :**

- **Les E/S d'abord.** Le tampon divise par 170 le nombre d'appels à `Write` et le temps par 3,3, sans
  toucher au code d'écriture.
- **Puis les allocations.** Une fois les appels système regroupés, le premier poste devient `fmt`, qui alloue
  près de trois objets par ligne. `AppendRec` gagne encore ×5. Au total, ×17 et presque plus aucune
  allocation.
- **`Grow` avant le pool.** Pour 200 lignes, estimer la taille finale divise le temps par 2. C'est une ligne de
  code, sans aucun risque. Le pool gagne encore ×1,6 en supprimant la dernière allocation, mais il apporte ses
  règles : remise à zéro, pas d'usage après `Put`, plafond de taille.
- **Pour un petit objet, le pool gagne 4 ns.** C'est réel, mais il faut des dizaines de millions d'objets par
  seconde pour que cela compte. Avec un pool, on risque aussi d'utiliser un objet après l'avoir rendu.
- **Le parallélisme.** Sur 1 000 éléments, lancer et attendre 8 goroutines coûte 9 fois plus que la somme
  elle-même. Sur 10 millions, le gain plafonne à ×1,2 alors qu'il y a 8 fils matériels. Les 80 Mo du tableau
  passent par le même bus mémoire, et une somme attend la mémoire bien plus qu'elle ne calcule (leçon 6). Le
  meilleur gain (×1,4) est atteint à 100 000 éléments : le tableau (800 Kio) tient alors dans le L3 de la machine (6 Mio, leçon 6).

## Explication du code

| Élément | Rôle |
|---|---|
| `CountingWriter` | Compte les appels à `Write`. Sur un `*os.File`, chacun est un appel système `write`. |
| `WriteBuffered` | Le code naïf, derrière un `bufio.Writer`. Le `Flush` final est obligatoire, et son erreur doit être vérifiée : c'est souvent là que l'écriture échoue réellement (disque plein). |
| `AppendRec(dst, r)` | Construit la ligne dans `dst` sans `fmt`. L'appelant fournit le tampon et le réutilise (`line[:0]`). C'est la convention des fonctions `Append*` de la bibliothèque standard. |
| `var line [64]byte` dans `Render` | Un tableau local de taille fixe, qui reste sur la pile (leçon 2) : `AppendRec` n'alloue pas tant que la ligne y tient. |
| `RenderGrow` | `buf.Grow(len(recs) * 32)` : une estimation suffit. Si elle est trop basse, `bytes.Buffer` grandit normalement. |
| `RenderPooled` | `Get`, `Reset`, travail, puis `Put` seulement si le tampon n'est pas devenu géant. `send` doit avoir fini d'utiliser les octets avant de rendre la main. |
| `SumParallel` | Une goroutine par part, et chacune écrit **une seule fois** sa case de `res`. Il n'y a pas de course, et le faux partage est négligeable (8 écritures au total). |
| `total` dans les benchmarks parallèles | Un compteur **par goroutine**. Une variable globale partagée entre les goroutines d'un `RunParallel` fausserait la mesure par du faux partage, et serait une course. |

## Erreurs fréquentes

1. **Optimiser une partie qui fait 5 % du temps** sans faire le calcul d'Amdahl.
2. **Écrire ligne par ligne dans un fichier ou une connexion** sans `bufio`.
3. **Oublier le `Flush`**, ou ignorer son erreur.
4. **Utiliser un objet après l'avoir rendu au pool**, ou rendre à l'appelant des octets qui pointent dans un
   tampon du pool.
5. **Remettre au pool des tampons géants**, qui gardent la mémoire occupée.
6. **Mettre en pool de petits objets**, ou des objets qui ne sont pas temporaires.
7. **Paralléliser un petit travail**, ou un travail limité par la mémoire, en espérant ×8.
8. **`unsafe` pour gagner une copie.**

## Bonnes pratiques Go

- Un objectif chiffré, un profil, le calcul d'Amdahl, puis la modification, puis la mesure.
- `bufio.Writer`/`bufio.Reader` devant toute E/S faite par petits morceaux.
- Des fonctions `AppendX(dst []byte, …) []byte` pour les chemins chauds qui produisent des octets.
- `Grow`/`make(…, 0, n)` dès qu'une taille est estimable : le gain est souvent l'essentiel de celui d'un pool,
  sans ses risques.
- `sync.Pool` pour des tampons gros et fréquents en code concurrent, avec `Reset`, un plafond de taille, et
  des pointeurs.
- Paralléliser des morceaux de travail d'au moins quelques dizaines de microsecondes, et mesurer le gain réel.
- Garder une trace de chaque optimisation : le benchmark qui la justifie, dans le dépôt.

## Ce que je dois retenir

- Amdahl : le gain global est borné par la part du temps qu'on accélère.
- L'ordre : supprimer le travail, regrouper les E/S, réduire les allocations, réutiliser, paralléliser.
- `bufio` : ×3,3 sans changer le code d'écriture ; avec `Append*`, ×17 au total.
- `Grow` d'abord (×2), `sync.Pool` ensuite (×1,6 de plus), avec ses règles.
- Le parallélisme ne paie que pour un gros travail de calcul (×9 plus lent sur 1 000 éléments ; ×1,2 au
  plus sur un parcours limité par la mémoire).
- On s'arrête quand l'objectif est atteint, ou quand le profil est plat.

---

➡️ [Exercices](exercices.md)
