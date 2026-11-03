# Leçon 6 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Les mesures viennent d'une machine à 4 cœurs (8 fils), avec `lscpu` : L1d 128 Kio (4 × 32 Kio), L2 1 Mio
(4 × 256 Kio), L3 6 Mio partagé ; go1.27.3. Les chiffres sont des médianes de 3 à 5 exécutions. Tes valeurs
seront différentes ; ce sont les **rapports** et les **seuils** qu'il faut comparer.

Le code des exercices faciles, ajouté au paquet `mem` du cours :

```go
// Fichier : fix.go
package mem

import "sync"

// E3 : les mêmes pointeurs, sans mélange (ordre d'allocation).
func PointersOrdered(n int) []*Book {
	s := make([]*Book, n)
	for i := range s {
		s[i] = &Book{ID: int64(i), Price: int64(i % 100)}
	}
	return s
}

// E4 : matrice de taille quelconque (une seule allocation, découpée en lignes).
func NewMatrix(n int) [][]int32 {
	flat := make([]int32, n*n)
	m := make([][]int32, n)
	for i := range m {
		m[i] = flat[i*n : (i+1)*n]
	}
	return m
}

func SumRowsN(m [][]int32) (t int64) {
	for i := range m {
		for j := range m[i] {
			t += int64(m[i][j])
		}
	}
	return
}

func SumColsN(m [][]int32) (t int64) {
	for j := range len(m) {
		for i := range m {
			t += int64(m[i][j])
		}
	}
	return
}

// E5 : chaque goroutine compte dans une variable locale, écrite une seule fois à la fin.
func CountLocal(workers, per int) int64 {
	var c counters
	var wg sync.WaitGroup
	for w := range workers {
		wg.Go(func() {
			local := int64(0)
			for range per {
				local++
			}
			c.n[w] = local
		})
	}
	wg.Wait()
	var t int64
	for _, x := range c.n {
		t += x
	}
	return t
}
```

```go
// Fichier : fix_test.go
package mem

import (
	"fmt"
	"testing"
)

// E1 : chaque changement de capacité est une réallocation (et une recopie).
func TestGrowth(t *testing.T) {
	var s []int
	last, n := 0, 0
	for i := range 1_000_000 {
		s = append(s, i)
		if cap(s) != last {
			if cap(s) <= 10_000 {
				t.Logf("len %6d : cap %6d (×%.2f)", len(s), cap(s), float64(cap(s))/float64(max(last, 1)))
			}
			last = cap(s)
			n++
		}
	}
	t.Logf("1 000 000 éléments : %d réallocations", n)
}

// E2 : clés entières, élément cherché en première ou en dernière position.
type kvi struct{ k, v int }

func FindInt(s []kvi, k int) int {
	for _, e := range s {
		if e.k == k {
			return e.v
		}
	}
	return -1
}

func BenchmarkLookupInt(b *testing.B) {
	for _, n := range []int{4, 8, 16, 32, 64, 128} {
		s := make([]kvi, n)
		m := make(map[int]int, n)
		for i := range n {
			s[i], m[i*7919] = kvi{i * 7919, i}, i
		}
		for _, pos := range []string{"premier", "dernier"} {
			target := s[0].k
			if pos == "dernier" {
				target = s[n-1].k
			}
			b.Run(fmt.Sprintf("slice-%s-%d", pos, n), func(b *testing.B) {
				for b.Loop() {
					sinkN = FindInt(s, target)
				}
			})
			b.Run(fmt.Sprintf("map-%s-%d", pos, n), func(b *testing.B) {
				for b.Loop() {
					sinkN = m[target]
				}
			})
		}
	}
}

// E3
func BenchmarkSumOrdered(b *testing.B) {
	ptrs := PointersOrdered(1_000_000)
	for b.Loop() {
		sinkI = SumPointers(ptrs)
	}
}

// E4
func BenchmarkMatrixSizes(b *testing.B) {
	for _, n := range []int{64, 256, 1000, 1024, 1500, 2000, 2048, 4096} {
		m := NewMatrix(n)
		b.Run(fmt.Sprintf("lignes-%d", n), func(b *testing.B) {
			for b.Loop() {
				sinkI = SumRowsN(m)
			}
		})
		b.Run(fmt.Sprintf("colonnes-%d", n), func(b *testing.B) {
			for b.Loop() {
				sinkI = SumColsN(m)
			}
		})
	}
}

// E5
func TestCountLocal(t *testing.T) {
	if got := CountLocal(8, 1000); got != 8000 {
		t.Fatalf("CountLocal = %d, attendu 8000", got)
	}
}

func BenchmarkCountLocal(b *testing.B) {
	for b.Loop() {
		sinkI = CountLocal(8, 1_000_000)
	}
}
```

## E1 — La croissance d'`append`

```
len      1 : cap      4 (×4.00)
len      5 : cap      8 (×2.00)
…
len    257 : cap    512 (×2.00)
len    513 : cap    848 (×1.66)
len    849 : cap   1280 (×1.51)
len   1281 : cap   1792 (×1.40)
len   1793 : cap   2560 (×1.43)
len   2561 : cap   3408 (×1.33)
len   3409 : cap   5120 (×1.50)
len   5121 : cap   7168 (×1.40)
len   7169 : cap   9216 (×1.29)
1 000 000 éléments : 36 réallocations
```

- **Petites slices** : la capacité **double** jusqu'à 256 éléments.
- **Au-delà** : le facteur décroît progressivement vers ×1,25 pour les grandes slices. Les dernières
  capacités relevées pour 1 000 000 d'éléments sont 539 648 → 674 816 → 843 776 → 1 055 744, soit ×1,25 à
  chaque fois.
- **Facteurs irréguliers** (×1,43 après ×1,40, ×1,50 après ×1,33) : la capacité calculée est **arrondie
  à la classe de taille** de l'allocateur (leçon 3). On obtient autant d'éléments que le bloc en contient,
  gratuitement.
- **Première capacité** : elle vaut 4 pour des `int` et 32 pour des `byte`, soit 32 octets dans les deux
  cas. C'est la mesure sur go1.27.3. Ne pas en faire une règle : ce détail de l'implémentation a déjà changé
  d'une version à l'autre.

**36 réallocations, mais le benchmark en compte 38 : d'où viennent les deux autres ?** On a mesuré directement
`runtime.MemStats.Mallocs` autour d'un appel. Une fonction non *inlinée* donne 36 allocations, toujours.
Le benchmark, lui, donne 54 allocations par opération avec `-benchtime=1x`, 40 avec `10x`, 38 avec `100x`. Les allocations en trop
viennent donc du *runtime* : chaque opération alloue 41 Mo, ce qui déclenche des cycles du GC, et ceux-ci allouent un peu pour leur propre compte.
Une opération qui alloue beaucoup « hérite » de ces allocations dans `allocs/op`. Ce qu'il faut retenir :
`allocs/op` est une **moyenne de tout le processus** pendant la mesure, pas un comptage de ta fonction.

## E2 — Slice ou map, avec des clés entières

| Taille | slice, 1re position | map, 1re | slice, dernière | map, dernière |
|---|---|---|---|---|
| 4 | 1,5 ns | 4,6 ns | 3,3 ns | 5,9 ns |
| 8 | 1,5 ns | 4,6 ns | 4,2 ns | 7,7 ns |
| 16 | 1,5 ns | 9,1 ns | 7,1 ns | 8,8 ns |
| 32 | 1,5 ns | 9,1 ns | 13,1 ns | 8,7 ns |
| 64 | 1,5 ns | 8,7 ns | 25,4 ns | 8,6 ns |
| 128 | 1,5 ns | 8,6 ns | 56,3 ns | 8,6 ns |

L'hypothèse du cours tient pour des clés `int` : la recherche linéaire gagne jusqu'à **16 éléments** dans le
pire cas (élément en dernière position). Elle perd à 32. En moyenne (élément au milieu), le seuil se situe
vers 32. Quand l'élément cherché est presque toujours en tête, elle gagne quelle que soit la taille.

Avec des `string`, au contraire, la map gagnait dès 4 clés. Comparer deux `int` coûte une instruction ;
comparer deux `string` demande de comparer les longueurs puis les octets. Le hachage de la map, lui,
coûte à peu près la même chose dans les deux cas.

La map montre aussi deux paliers : environ 5 ns jusqu'à 8 éléments, environ 9 ns au-delà. Depuis Go 1.24, une
petite map tient dans un seul **groupe** de 8 cases parcouru d'un bloc. À partir de 9 éléments, elle passe à
une vraie table, avec un niveau d'indirection de plus.

La leçon : le seuil dépend du **type de clé**, de la **distribution** des recherches et de la version de Go.
Seul un benchmark avec tes vraies données le donne.

## E3 — Les pointeurs dans l'ordre

| Somme sur 1 000 000 livres | Temps |
|---|---|
| `[]Book` (valeurs) | 3,9 ms |
| `[]*Book` alloués à la suite, dans l'ordre | 4,7 ms |
| `[]*Book` mélangés (cours) | 15,8 ms |

Sans le mélange, l'écart tombe de ×4 à ×1,2. Les livres alloués à la suite sont **contigus** dans la mémoire :
l'allocateur les range côte à côte dans le même bloc de sa classe de taille (32 octets). Le parcours des
pointeurs dans l'ordre lit donc deux flux séquentiels, les pointeurs et les livres. Le préchargeur suit
les deux.

L'écart restant vient de ce qu'on lit 8 Mo de pointeurs en plus des 32 Mo de livres, et d'une indirection par
élément.

Ce n'est pas le pointeur qui coûte cher, c'est la **dispersion**. Mais dans un vrai programme, des objets
alloués au fil de l'eau, entre d'autres allocations et après des cycles du GC, ne restent pas dans l'ordre.
Les valeurs contiguës le **garantissent**.

## E4 — La taille de la matrice

| N | Taille (int32) | Lignes | Colonnes | Rapport |
|---|---|---|---|---|
| 64 | 16 Kio | 1,9 µs | 3,7 µs | ×2 |
| 256 | 256 Kio | 32 µs | 60 µs | ×1,9 |
| 1 000 | 3,8 Mio | 466 µs | 950 µs | ×2 |
| 1 024 | 4 Mio | 490 µs | 977 µs | ×2 |
| 1 500 | 8,6 Mio | 1,08 ms | 3,43 ms | ×3,2 |
| 2 000 | 15,3 Mio | 1,91 ms | 15,1 ms | ×7,9 |
| 2 048 | 16 Mio | 3,25 ms | 61,9 ms | ×19 |
| 4 096 | 64 Mio | 12,2 ms | 272 ms | ×22 |

**Jusqu'à 1 024**, l'écart reste d'environ ×2. Même parcourue par colonnes, la matrice tient dans le L3
(6 Mio). Une ligne de cache chargée pour la colonne j sert encore pour les colonnes j+1 à j+15 : chacune
coûte un accès au L3, pas à la mémoire centrale. Le ×2 restant tient à ce que le parcours par lignes profite
du préchargeur et de la vectorisation.

**Au-delà de 6 Mio** (N = 1 500), la matrice ne tient plus dans le L3 et le parcours par colonnes va chercher
en mémoire centrale. L'écart grimpe à ×8, puis ×20.

**Le saut entre 2 000 et 2 048**, à taille presque égale, montre un second effet. Avec N = 2 048, deux lignes
successives de la matrice sont séparées d'exactement 8 Kio, une puissance de deux. Or un cache ne range une
adresse que dans un petit nombre d'emplacements, déterminés par certains bits de l'adresse. Des adresses
espacées d'une puissance de deux tombent toutes aux mêmes emplacements et s'évincent mutuellement, alors que
le cache est loin d'être plein.

Remarque honnête : une autre exécution de la même mesure avait donné 17,8 ms au lieu de 62 ms pour les colonnes
à N = 2 048 (83 ms au lieu de 272 ms à 4 096). Les lignes, elles, ne bougeaient pas. Le résultat dépend donc
aussi de la façon dont le système a placé la matrice en mémoire physique, par exemple en grandes pages ou non.
On ne l'a pas vérifié. Pour le parcours par colonnes, le rapport, lui, est toujours mauvais.

## E5 — Compteurs locaux

| 8 goroutines × 1 000 000 | Temps |
|---|---|
| `CountShared` (même ligne de cache) | 8,2 ms |
| `CountPadded` (une ligne par compteur) | 2,1 ms |
| `CountLocal` (variable locale, une écriture à la fin) | 0,77 ms |

×10 sur le partage, ×2,7 sur le rembourrage. La variable locale vit dans un **registre**, et la boucle ne
touche plus du tout la mémoire. Le tableau partagé n'est écrit que 8 fois au total, une fois par goroutine.
Quand c'est possible, c'est le meilleur remède : pas de rembourrage à choisir, pas de dépendance au matériel.

---

## Exercice intermédiaire — Un index en mémoire

Le paquet `index`, dans un sous-répertoire du même module. Les livres ont un auteur et un titre (`string`),
une année et un prix (`int32`), soit 40 octets par livre.

```go
// Fichier : index/index.go
// Package index : deux façons d'indexer 1 000 000 de livres par auteur.
package index

import (
	"cmp"
	"fmt"
	"slices"
)

type Book struct {
	Author string
	Title  string
	Year   int32
	Price  int32
}

// Generate produit n livres répartis sur authors auteurs, dans un ordre mélangé.
func Generate(n, authors int) []Book {
	names := make([]string, authors)
	for i := range names {
		names[i] = fmt.Sprintf("auteur-%05d", i)
	}
	books := make([]Book, n)
	for i := range books {
		a := (i * 7919) % authors // mélange déterministe
		books[i] = Book{
			Author: names[a],
			Title:  fmt.Sprintf("titre-%07d", i),
			Year:   int32(1900 + (i*31)%125),
			Price:  int32(i % 5000),
		}
	}
	return books
}

// --- Version A : auteur -> pointeurs ---

type IndexA map[string][]*Book

func BuildA(books []Book) IndexA {
	idx := IndexA{}
	for _, b := range books {
		b := b // une copie par livre, allouée séparément : c'est le modèle « un objet par livre »
		idx[b.Author] = append(idx[b.Author], &b)
	}
	for _, l := range idx {
		slices.SortFunc(l, func(x, y *Book) int { return cmp.Compare(x.Year, y.Year) })
	}
	return idx
}

func (idx IndexA) ByAuthor(a string) []*Book { return idx[a] }

func (idx IndexA) TotalPrice() (t int64) {
	for _, l := range idx {
		for _, b := range l {
			t += int64(b.Price)
		}
	}
	return
}

// --- Version B : une slice triée + des tranches ---

type span struct{ start, end int32 }

type IndexB struct {
	books []Book
	spans map[string]span
}

func BuildB(books []Book) IndexB {
	s := slices.Clone(books)
	slices.SortFunc(s, func(x, y Book) int {
		return cmp.Or(cmp.Compare(x.Author, y.Author), cmp.Compare(x.Year, y.Year))
	})
	spans := make(map[string]span)
	for i := 0; i < len(s); {
		j := i
		for j < len(s) && s[j].Author == s[i].Author {
			j++
		}
		spans[s[i].Author] = span{int32(i), int32(j)}
		i = j
	}
	return IndexB{books: s, spans: spans}
}

// ByAuthor renvoie une vue sur la slice partagée : l'appelant ne doit pas la modifier.
func (idx IndexB) ByAuthor(a string) []Book {
	sp, ok := idx.spans[a]
	if !ok {
		return nil
	}
	return idx.books[sp.start:sp.end:sp.end]
}

func (idx IndexB) TotalPrice() (t int64) {
	for i := range idx.books {
		t += int64(idx.books[i].Price)
	}
	return
}
```

```go
// Fichier : index/index_test.go
package index

import (
	"runtime"
	"runtime/metrics"
	"sync"
	"testing"
	"time"
)

const nBooks, nAuthors = 1_000_000, 10_000

var books = Generate(nBooks, nAuthors)

func TestSame(t *testing.T) {
	a, b := BuildA(books), BuildB(books)
	if a.TotalPrice() != b.TotalPrice() {
		t.Fatal("totaux différents")
	}
	for _, au := range []string{"auteur-00000", "auteur-04242", "auteur-09999", "inconnu"} {
		la, lb := a.ByAuthor(au), b.ByAuthor(au)
		if len(la) != len(lb) {
			t.Fatalf("%s : %d contre %d livres", au, len(la), len(lb))
		}
		for i := range la {
			if la[i].Year != lb[i].Year {
				t.Fatalf("%s : ordre différent en %d", au, i)
			}
		}
	}
}

func heapAfter(build func() any) (uint64, any) {
	var m0, m1 runtime.MemStats
	runtime.GC()
	runtime.ReadMemStats(&m0)
	x := build()
	runtime.GC()
	runtime.ReadMemStats(&m1)
	return m1.HeapAlloc - m0.HeapAlloc, x
}

func TestMemory(t *testing.T) {
	ma, a := heapAfter(func() any { return BuildA(books) })
	mb, b := heapAfter(func() any { return BuildB(books) })
	t.Logf("tas retenu : A = %d Mio, B = %d Mio", ma>>20, mb>>20)
	runtime.KeepAlive(a)
	runtime.KeepAlive(b)
}

// gcShare mesure la part du processeur consacrée au GC pendant une charge d'allocations
// parallèle, l'index étant vivant (leçon 3).
func gcShare(t *testing.T, name string, idx any) {
	read := func() (gc, total float64) {
		s := []metrics.Sample{{Name: "/cpu/classes/gc/total:cpu-seconds"}, {Name: "/cpu/classes/total:cpu-seconds"}}
		metrics.Read(s)
		return s[0].Value.Float64(), s[1].Value.Float64()
	}
	runtime.GC()
	gc0, tot0 := read()
	var wg sync.WaitGroup
	start := time.Now()
	for range 4 {
		wg.Go(func() {
			var keep [][]byte
			for i := range 2_000_000 {
				keep = append(keep, make([]byte, 64))
				if i%1000 == 0 {
					keep = keep[:0]
				}
			}
		})
	}
	wg.Wait()
	el := time.Since(start)
	gc1, tot1 := read()
	t.Logf("%s : charge en %v, part du GC %.0f %%", name, el.Round(time.Millisecond), 100*(gc1-gc0)/(tot1-tot0))
	runtime.KeepAlive(idx)
}

func TestGCShare(t *testing.T) {
	gcShare(t, "sans index", nil)
	gcShare(t, "A", BuildA(books))
	gcShare(t, "B", BuildB(books))
}

var sinkI int64
var sinkN int

func BenchmarkBuild(b *testing.B) {
	b.Run("A", func(b *testing.B) {
		for b.Loop() {
			_ = BuildA(books)
		}
	})
	b.Run("B", func(b *testing.B) {
		for b.Loop() {
			_ = BuildB(books)
		}
	})
}

func BenchmarkQuery(b *testing.B) {
	a, x := BuildA(books), BuildB(books)
	b.Run("A", func(b *testing.B) {
		for b.Loop() {
			for _, bk := range a.ByAuthor("auteur-04242") {
				sinkN += int(bk.Price)
			}
		}
	})
	b.Run("B", func(b *testing.B) {
		for b.Loop() {
			for _, bk := range x.ByAuthor("auteur-04242") {
				sinkN += int(bk.Price)
			}
		}
	})
}

func BenchmarkScan(b *testing.B) {
	a, x := BuildA(books), BuildB(books)
	b.Run("A", func(b *testing.B) {
		for b.Loop() {
			sinkI = a.TotalPrice()
		}
	})
	b.Run("B", func(b *testing.B) {
		for b.Loop() {
			sinkI = x.TotalPrice()
		}
	})
}

// Requêtes sur des auteurs différents à chaque tour : les données ne restent pas en cache.
func BenchmarkQueryMany(b *testing.B) {
	a, x := BuildA(books), BuildB(books)
	names := make([]string, nAuthors)
	for i := range names {
		names[i] = books[i].Author
	}
	b.Run("A", func(b *testing.B) {
		i := 0
		for b.Loop() {
			for _, bk := range a.ByAuthor(names[(i*7)%nAuthors]) {
				sinkN += int(bk.Price)
			}
			i++
		}
	})
	b.Run("B", func(b *testing.B) {
		i := 0
		for b.Loop() {
			for _, bk := range x.ByAuthor(names[(i*7)%nAuthors]) {
				sinkN += int(bk.Price)
			}
			i++
		}
	})
}
```

Mesures (`go test -v -run 'Same|Memory|GCShare'`, puis `go test -bench . -benchmem`) :

| | A : `map[string][]*Book` | B : `[]Book` triée + tranches |
|---|---|---|
| Tas retenu par l'index | 57 Mio | 38 Mio |
| Construction | 209 ms, 1 080 081 allocations | 380 ms, 82 allocations |
| Requête, même auteur répété | 167 ns | 173 ns |
| Requête, auteur différent à chaque fois | 2,2 µs | 0,88 µs |
| Parcours de tous les livres | 19,3 ms | 5,2 ms |
| Part du GC pendant la charge parallèle | 52 % (314 ms) | 43 % (256 ms) |

Référence pour la dernière ligne : la même charge sans index donne 25 % de GC et dure 196 ms.

**Lecture :**

- **Mémoire.** A coûte un objet de 48 octets par livre : les 40 octets du livre, arrondis à la classe de
  taille. S'ajoutent les pointeurs (8 octets par livre) et les capacités en trop des slices construites par
  `append`. B, c'est 40 octets par livre et rien d'autre. Les chaînes ne comptent dans aucune des deux, car
  elles sont partagées avec les données d'origine.
- **Construction.** B est **plus lente** : elle trie 1 000 000 de livres en comparant des `string`, alors
  qu'A ne trie que 10 000 petites listes. Mais B fait 82 allocations contre plus d'un million.
- **Requête.** Avec le même auteur répété, les deux sont égales : ses 100 livres restent dans le cache, et le
  benchmark mesure surtout la map. Avec des auteurs différents, ce qui ressemble à un vrai service, B est ×2,5
  plus rapide. Ses 100 livres sont contigus (4 Kio), alors qu'A suit 100 pointeurs vers des objets dispersés.
  Le premier benchmark était trompeur : **un benchmark qui répète la même entrée mesure le cas où tout est en
  cache**.
- **Parcours complet.** ×3,7 pour B : contigu contre dispersé, comme dans le cours.
- **GC.** Les deux index contiennent des pointeurs (les `string`), donc le GC doit les parcourir tous les
  deux. A a en plus un million d'objets à marquer un par un. B n'élimine pas le coût, il le réduit.

**Quel choix ?**

- **Index en lecture seule, reconstruit chaque nuit : B.** Moins de mémoire, des requêtes et des parcours
  plus rapides, moins de travail pour le GC. Une construction plus lente une fois par nuit ne coûte rien.
  On publie le nouvel index d'un bloc (un `atomic.Pointer[IndexB]`, N6 L7) : les lecteurs n'ont pas de verrou
  à prendre.
- **Livres modifiés souvent : A, ou un hybride.** Insérer dans B oblige à décaler la slice et à recalculer
  toutes les tranches qui suivent, en O(n). A modifie une seule liste.

  Entre les deux, on peut garder B pour le gros des données et une petite structure A pour les
  modifications récentes, les deux étant fusionnées périodiquement. C'est le principe de beaucoup de moteurs
  de stockage.

  Dans tous les cas, ce sont les **mesures** sur ta charge (rapport lectures/écritures) qui tranchent.

---

## Défi — Le faux partage dans un vrai programme

Le paquet `hits`, lui aussi dans un sous-répertoire du module :

```go
// Fichier : hits/hits.go
// Package hits : compter les requêtes par route sans se marcher dessus.
package hits

import "sync/atomic"

const Routes = 16

// Flat : 16 compteurs côte à côte, deux lignes de cache de 64 octets.
type Flat struct{ n [Routes]atomic.Int64 }

func (c *Flat) Inc(route int)       { c.n[route].Add(1) }
func (c *Flat) Get(route int) int64 { return c.n[route].Load() }

// Padded : un compteur par ligne de cache.
type paddedInt struct {
	atomic.Int64
	_ [56]byte
}

type Padded struct{ n [Routes]paddedInt }

func (c *Padded) Inc(route int)       { c.n[route].Add(1) }
func (c *Padded) Get(route int) int64 { return c.n[route].Load() }

// Sharded : chaque worker a sa propre rangée de compteurs, sommées à la lecture.
// Le worker passe son numéro : pas besoin de savoir sur quel processeur il tourne.
type shard struct {
	n [Routes]atomic.Int64 // 128 octets
	_ [64]byte             // sépare deux rangées voisines
}

type Sharded struct{ s []shard }

func NewSharded(workers int) *Sharded { return &Sharded{s: make([]shard, workers)} }

func (c *Sharded) Inc(worker, route int) { c.s[worker].n[route].Add(1) }

func (c *Sharded) Get(route int) (t int64) {
	for i := range c.s {
		t += c.s[i].n[route].Load()
	}
	return
}
```

```go
// Fichier : hits/hits_test.go
package hits

import (
	"expvar"
	"sync"
	"sync/atomic"
	"testing"
)

const workers, per = 8, 1_000_000

// run lance 8 workers ; route(w) dit quelle route le worker w incrémente.
func run(inc func(w, r int), route func(w int) int) {
	var wg sync.WaitGroup
	for w := range workers {
		wg.Go(func() {
			r := route(w)
			for range per {
				inc(w, r)
			}
		})
	}
	wg.Wait()
}

var distinct = func(w int) int { return w } // routes différentes : faux partage
var same = func(int) int { return 3 }       // même route : vrai partage

func TestCounts(t *testing.T) {
	var f Flat
	var p Padded
	s := NewSharded(workers)
	run(func(_, r int) { f.Inc(r) }, same)
	run(func(_, r int) { p.Inc(r) }, same)
	run(s.Inc, same)
	if f.Get(3) != workers*per || p.Get(3) != workers*per || s.Get(3) != workers*per {
		t.Fatal(f.Get(3), p.Get(3), s.Get(3))
	}
}

func BenchmarkHits(b *testing.B) {
	for _, tc := range []struct {
		name  string
		route func(int) int
	}{{"routes-differentes", distinct}, {"meme-route", same}} {
		b.Run("flat/"+tc.name, func(b *testing.B) {
			for b.Loop() {
				var c Flat
				run(func(_, r int) { c.Inc(r) }, tc.route)
			}
		})
		b.Run("padded/"+tc.name, func(b *testing.B) {
			for b.Loop() {
				var c Padded
				run(func(_, r int) { c.Inc(r) }, tc.route)
			}
		})
		b.Run("sharded/"+tc.name, func(b *testing.B) {
			for b.Loop() {
				c := NewSharded(workers)
				run(c.Inc, tc.route)
			}
		})
	}
}

// Question 3 : expvar.Int est un seul atomic.Int64 ; sync.Pool a un cache par processeur.
func BenchmarkExpvar(b *testing.B) {
	var v expvar.Int // non publié : on mesure seulement Add
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			v.Add(1)
		}
	})
}

func BenchmarkShardedParallel(b *testing.B) {
	c := NewSharded(64)
	var next sync.Mutex
	id := 0
	b.RunParallel(func(pb *testing.PB) {
		next.Lock()
		w := id
		id++
		next.Unlock()
		for pb.Next() {
			c.Inc(w, 3)
		}
	})
}

var pool = sync.Pool{New: func() any { return new([64]byte) }}

func BenchmarkPool(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			x := pool.Get().(*[64]byte)
			x[0]++
			pool.Put(x)
		}
	})
}

// La mesure qui a levé la surprise : le même compteur, rembourré à 128 octets.
type padded128 struct {
	atomic.Int64
	_ [120]byte
}

func BenchmarkPadded128(b *testing.B) {
	for b.Loop() {
		var c [Routes]padded128
		run(func(_, r int) { c[r].Add(1) }, distinct)
	}
}
```

**1 et 2.** `go test -bench Hits` (8 workers × 1 000 000 incréments atomiques, médianes) :

| | 8 routes différentes | même route pour tous |
|---|---|---|
| `Flat` (`[16]atomic.Int64`) | 192 ms | 196 ms |
| `Padded` (64 octets par compteur) | 54 ms | 132 ms |
| `Sharded` (une rangée par worker) | 14 ms | 15 ms |

- **`Flat`, routes différentes, est aussi lent que le vrai partage** : 192 ms contre 196 ms. Pour le
  processeur, c'est la même chose. Les 8 compteurs tiennent dans une ligne, et la ligne fait la navette
  entre les cœurs à chaque incrément.
- **`Padded` ne règle que le faux partage.** Si tout le monde incrémente vraiment la même route, le
  rembourrage n'y peut rien (132 ms).
- **`Sharded` règle les deux** : chaque worker a sa rangée, et la lecture additionne les rangées. On paie à
  la lecture, qui est rare, pour rendre l'écriture rapide, alors qu'elle est fréquente.

**Une surprise.** Avec des routes différentes, `Padded` à 64 octets (54 ms) reste ×4 plus lent que
`Sharded` (14 ms). Or dans les deux cas, chaque compteur a sa propre ligne de 64 octets. On a donc mesuré un
rembourrage de 128 octets :

```
BenchmarkHits/padded/routes-differentes-8   54,4 ms
BenchmarkHits/sharded/routes-differentes-8  19,1 ms   (même exécution)
BenchmarkPadded128-8                        19,8 ms   (_ [120]byte au lieu de _ [56]byte)
```

Avec 128 octets, `Padded` rejoint `Sharded`. L'explication la plus probable est que ce processeur charge les
lignes **par paires de 128 octets** (le préchargeur de la ligne voisine). Deux compteurs dans deux lignes
voisines se gênent encore. Les rangées de `Sharded` font 192 octets, ce qui les met à l'abri par hasard.

Sur l'exemple du cours, avec des incréments ordinaires et non atomiques, 128 octets n'apportaient rien de
plus que 64 (2,9 ms contre 2,1 ms). L'effet dépend donc du type d'accès autant que du processeur. C'est
pourquoi la bibliothèque standard rembourre à 128 octets.

**Sur quel processeur tourne-t-on ?** Le code ordinaire ne peut pas le savoir. Aucune API publique ne
le dit, et une goroutine peut de toute façon changer de processeur à tout moment, entre deux instructions.
La bonne conception est celle de `Sharded` : c'est le **worker** qui a une identité, fixée à son
démarrage (son numéro dans le pool), et il la passe à `Inc`. Pour un serveur HTTP, dont les goroutines ne
sont pas des workers numérotés, on peut répartir les requêtes sur les rangées par un hachage de quelque chose
qui varie : un compteur de requêtes, ou l'adresse du client.

**3. `sync.Pool` et `expvar`.**

- Le code source de `sync.Pool` (dans `$(go env GOROOT)/src/sync/pool.go`) garde un cache **par processeur**
  (le *runtime*, lui, sait où il tourne). Chaque cache est rembourré : `pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte`, avec
  le commentaire « Prevents false sharing on widespread platforms with 128 mod (cache line size) = 0 ».
- `expvar.Int`, au contraire, est un simple `atomic.Int64`. Il n'y a pas de faux partage, mais il y a du
  **vrai** partage : tous les cœurs écrivent la même ligne.

La mesure qui le vérifie consiste à faire varier le parallélisme avec `-cpu 1,2,4,8` et à regarder si le temps
par opération baisse :

| ns/op | `-cpu 1` | 2 | 4 | 8 |
|---|---|---|---|---|
| `expvar.Int.Add` | 7,0 | 17,8 | 18,9 | 18,1 |
| `Sharded.Inc` (une rangée par goroutine) | 7,2 | 3,6 | 1,8 | 1,6 |
| `sync.Pool` `Get` + `Put` | 15,9 | 7,1 | 5,7 | 5,1 |

Avec `RunParallel`, le temps par opération d'une structure qui **passe à l'échelle** baisse quand on ajoute
des cœurs, comme `Sharded` et `sync.Pool`. Le temps d'`expvar.Int` **monte** dès 2 cœurs, puis reste
constant : la ligne fait la navette. Pour un compteur incrémenté à chaque requête d'un serveur très chargé,
un compteur réparti comme `Sharded`, lu par `expvar.Func`, passe mieux à l'échelle.

Ce n'est un problème qu'à des dizaines de millions d'incréments par seconde. À quelques milliers de requêtes
par seconde, `expvar.Int` est parfait.

---

## Quiz

1. Le processeur lit la mémoire par **lignes** de 64 octets : un accès séquentiel utilise tous les octets de
   chaque ligne chargée, et le **préchargeur** anticipe les suivantes. Un accès aléatoire paie une ligne
   entière pour quelques octets, et attend la mémoire centrale (des centaines de cycles) à chaque fois.
2. Des réallocations successives : environ 36 pour un million d'`int`, chacune avec une **recopie** de tout le
   contenu, plus le travail du GC pour les anciens tableaux (×5 mesuré dans le cours). Les recopies cumulées
   restent proportionnelles à n, grâce à la croissance géométrique, mais avec une constante élevée.
3. Les valeurs sont **contiguës** : un parcours lit un flux séquentiel. Une slice de pointeurs oblige à
   suivre une adresse par élément, et si les objets sont dispersés, chaque accès est un défaut de cache.
   Le GC a aussi un objet de plus à marquer par élément.
4. La boucle **intérieure** doit parcourir l'indice **le plus à droite** : `m[i][j]` avec `j` à l'intérieur.
   C'est l'ordre des éléments en mémoire.
5. Non. Avec des clés `string`, la map gagnait dès 4 éléments. Avec des `int`, la slice gagne jusqu'à environ
   16 dans le pire cas. Cela dépend du type de clé, de la position de ce qu'on cherche et de la version de
   Go : on mesure.
6. Chaque champ doit être placé à une adresse multiple de son **alignement** (8 pour un `int64`). Le
   compilateur ne réordonne pas les champs ; il insère du **rembourrage** pour respecter l'alignement. Ranger
   les champs du plus grand au plus petit le minimise.
7. Deux cœurs écrivent des variables **différentes** situées dans la **même ligne de cache**. Chaque écriture
   invalide la ligne chez l'autre, qui doit la recharger. Le détecteur de *data races* ne le signale pas, car
   il n'y a pas de course : les variables sont distinctes. Le problème est matériel, invisible dans le code.
8. **Séparer** les données écrites par des cœurs différents en les rembourrant : 64 octets, ou 128 pour être
   sûr (mesuré dans le défi). Ou mieux, **ne pas écrire de mémoire partagée** dans la boucle : une variable
   locale, ou un compteur par worker, fusionnés à la fin ou à la lecture.
9. Depuis Go 1.24, les maps sont des **tables Swiss** : les entrées sont rangées par groupes de 8, et chaque
   groupe porte des octets de contrôle comparés d'un bloc. Une petite map tient dans un seul groupe. Les
   recherches et les insertions sont en général plus rapides qu'avec l'ancienne implémentation.
10. Hors des **boucles chaudes** et des **grands volumes**. Pour quelques dizaines d'éléments dans un handler
    qui attend la base de données ou le réseau, la lisibilité prime. Le profil (leçons 4 et 5) dit où ces
    optimisations comptent, et un benchmark dit si elles ont marché.
