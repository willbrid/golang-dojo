# Leçon 6 — Slices, maps et cache

> Deux programmes qui font exactement les mêmes opérations peuvent différer d'un facteur 15, simplement
> parce que l'un lit la mémoire dans l'ordre et l'autre non. Un processeur moderne passe l'essentiel de son
> temps à **attendre la mémoire** : il la lit par lignes de 64 octets, garde les lignes récentes dans ses
> caches, et anticipe les lectures séquentielles. Cette leçon mesure ce que cela implique pour les slices
> et les maps de Go — préallocation, valeurs contre pointeurs, ordre de parcours, choix de la structure,
> disposition des champs, partage entre cœurs.

## Objectifs

1. Préallouer slices et maps quand la taille est connue, et mesurer le gain.
2. Préférer les données **contiguës** (slices de valeurs) aux données dispersées (pointeurs).
3. Parcourir la mémoire dans l'ordre où elle est rangée.
4. Choisir entre slice et map à partir d'une mesure, pas d'une règle apprise.
5. Comprendre l'alignement des champs et le faux partage.

## Explication

### Ce que fait le matériel

La mémoire principale est lente (de l'ordre de 100 ns par accès isolé). Le processeur la lit par **lignes de
cache** de 64 octets, conservées dans des caches de plus en plus grands et lents (L1 : quelques dizaines de
Kio, environ 1 ns ; L2 ; L3 : quelques Mio partagés). Deux conséquences :

- lire **un** octet amène ses 63 voisins dans le cache : lire ensuite les voisins est presque gratuit ;
- un parcours **séquentiel** est détecté et anticipé (*prefetching*) : les lignes suivantes arrivent avant
  qu'on les demande. Un parcours **aléatoire** (sauts de pointeur en pointeur) ne l'est pas.

Tout ce qui suit découle de ces deux faits. Mesuré sur la machine du cours (Intel i5-8265U, 8 fils), médiane
de trois exécutions.

### Préallouer

`append` sur une slice pleine en alloue une **nouvelle**, plus grande (environ deux fois pour les petites,
moins pour les grandes), et y **recopie** tout. Une map qui grossit se réorganise de même.

| Opération (n éléments) | Sans préallocation | Avec | Gain |
|---|---|---|---|
| slice, `append` × 1 000 000 | 16,3 ms, 41,7 Mo, 38 allocations | 3,0 ms, 8,0 Mo, 1 allocation | ×5 |
| map, 100 000 insertions | 9,4 ms, 4,7 Mo, 532 allocations | 5,0 ms, 2,4 Mo, 258 allocations | ×1,9 |

Quand la taille finale est connue (ou bornée), `make([]T, 0, n)` et `make(map[K]V, n)`. Quand elle ne l'est
pas, une estimation suffit souvent. Et pour une slice destinée à être agrandie dans une boucle chaude, une
slice **réutilisée** (`s = s[:0]`) évite toute allocation (leçon 8).

### Contigu ou dispersé

Une `[]Book` range les livres **côte à côte** : la boucle lit la mémoire dans l'ordre. Une `[]*Book` range
des **pointeurs** côte à côte, mais chaque livre est ailleurs, alloué séparément ; si l'ordre des pointeurs
ne suit pas l'ordre en mémoire (ce qui arrive dès que les objets ont été créés, triés ou supprimés au fil de
l'eau), chaque accès est un saut.

| Somme d'un champ sur 1 000 000 livres de 32 octets | Temps |
|---|---|
| `[]Book` (valeurs) | 3,8 ms |
| `[]*Book` (pointeurs, ordre mélangé) | 15,6 ms (×4) |

S'y ajoute le coût pour le ramasse-miettes (leçon 3, E2) : un million de pointeurs à suivre contre un seul
objet sans pointeur. Slices de **valeurs** pour les grands ensembles ; pointeurs quand il faut partager ou
modifier un élément en place depuis plusieurs endroits, ou pour de très grosses structs qu'on ne veut pas
copier.

### L'ordre de parcours

Un tableau à deux dimensions `[N][N]int32` est rangé **ligne par ligne**. Le parcourir ligne par ligne lit la
mémoire dans l'ordre ; colonne par colonne, chaque accès saute de N × 4 octets, donc d'une ligne de cache à une
autre.

| Somme de 2048 × 2048 `int32` | Temps |
|---|---|
| par lignes (`m[i][j]`, j varie le plus vite) | 3,7 ms |
| par colonnes (`m[i][j]`, i varie le plus vite) | 55,4 ms (×15) |

Même nombre d'opérations, même résultat. C'est la mesure la plus spectaculaire de cette leçon, et la règle la
plus simple : **la boucle intérieure parcourt l'indice qui varie le plus vite en mémoire**.

### Slice ou map : mesurer

L'idée reçue veut qu'une petite slice parcourue linéairement batte une map. Mesuré, avec des clés `string` et
dans le pire cas pour la slice (l'élément cherché est le dernier) :

| Éléments | Recherche linéaire | Map |
|---|---|---|
| 4 | 17,4 ns | 12,9 ns |
| 8 | 35,4 ns | 14,6 ns |
| 16 | 34,0 ns | 11,7 ns |
| 64 | 264 ns | 18,6 ns |

Ici la map gagne **dès 4 éléments** : chaque comparaison de chaîne coûte, et les maps de Go (réécrites en
Go 1.24 sur le modèle des *Swiss tables*, une organisation par groupes qui compare plusieurs clés à la fois)
sont très rapides pour les petites tailles. Avec des clés entières et un élément trouvé tôt, la slice
gagnerait sans doute. Pas de règle : on mesure **avec ses clés et sa distribution réelle de recherches**. La
slice garde d'autres avantages : ordre déterministe, moins de mémoire, parcours rapide.

### L'alignement des champs

Le compilateur aligne chaque champ sur sa taille (un `int64` commence à une adresse multiple de 8). Entre un
`bool` et un `int64`, il insère donc 7 octets de **rembourrage**.

```go
type Padded struct { A bool; B int64; C bool; D int64; E bool } // 40 octets (mesuré)
type Packed struct { B int64; D int64; A bool; C bool; E bool } // 24 octets
```

Même contenu, 40 % de mémoire en moins, donc plus d'éléments par ligne de cache. Ranger les champs du plus
grand au plus petit suffit. Ne vaut la peine que pour des structs présentes en très grand nombre ; l'analyseur
`fieldalignment` (de `golang.org/x/tools`) les signale.

### Le faux partage

Huit goroutines incrémentent chacune **son** compteur ; aucune donnée n'est partagée. Mais si les huit
compteurs (8 octets chacun) tiennent dans **une** ligne de cache de 64 octets, les cœurs se disputent cette
ligne : chaque écriture l'invalide dans les caches des autres.

| 8 goroutines × 1 000 000 incréments | Temps |
|---|---|
| `[8]int64` : une ligne de cache pour tous | 8,2 ms |
| chaque compteur rembourré à 64 octets | 2,1 ms (×4) |

C'est le **faux partage** (*false sharing*) : un partage matériel sans partage logique. Il n'apparaît pas dans
le code, ni dans le détecteur de *data races* (il n'y en a pas), seulement dans les mesures. Remède : séparer
les données écrites par des cœurs différents (rembourrage, ou un compteur local par goroutine fusionné à la
fin, ce qui est encore mieux).

Combien rembourrer ? 64 octets suffisent ici. Mais certains processeurs chargent les lignes **par paires**
(le préchargeur anticipe la ligne voisine), et le défi de cette leçon mesure, sur la même machine, des compteurs
atomiques encore ralentis ×2,7 avec 64 octets et libérés avec 128. C'est la valeur choisie par la bibliothèque
standard pour les caches par processeur de `sync.Pool` (« 128 mod (cache line size) = 0 », dit son code
source). Dans le doute : 128 octets, et une mesure.

### Quand ne pas s'en soucier

Ces effets comptent dans les **boucles chaudes** sur de **grands** volumes. Pour une liste de 20 éléments
dans un handler HTTP, ni l'ordre des champs ni la préallocation ne se verront jamais face au réseau. Le profil
(leçons 4 et 5) dit où cela compte.

## Exemple

Les six mesures, en une série de benchmarks. Les tableaux ci-dessus en sont les résultats.

```go
// Fichier : mem.go
// Package mem : ce que coûtent vraiment les structures de données.
package mem

import (
	"math/rand/v2"
	"sync"
)

// --- 1. Préallouer ---

func AppendGrow(n int) []int {
	var s []int
	for i := range n {
		s = append(s, i)
	}
	return s
}

func AppendPrealloc(n int) []int {
	s := make([]int, 0, n)
	for i := range n {
		s = append(s, i)
	}
	return s
}

func MapGrow(n int) map[int]int {
	m := map[int]int{}
	for i := range n {
		m[i] = i
	}
	return m
}

func MapPrealloc(n int) map[int]int {
	m := make(map[int]int, n)
	for i := range n {
		m[i] = i
	}
	return m
}

// --- 2. Valeurs contiguës ou pointeurs dispersés ---

type Book struct {
	ID    int64
	Price int64
	Pages int64
	Year  int64
}

func Values(n int) []Book {
	s := make([]Book, n)
	for i := range s {
		s[i] = Book{ID: int64(i), Price: int64(i % 100)}
	}
	return s
}

// Pointers alloue chaque livre séparément, puis mélange l'ordre : les objets sont dispersés.
func Pointers(n int) []*Book {
	s := make([]*Book, n)
	for i := range s {
		s[i] = &Book{ID: int64(i), Price: int64(i % 100)}
	}
	rng := rand.New(rand.NewPCG(1, 2))
	rng.Shuffle(n, func(i, j int) { s[i], s[j] = s[j], s[i] })
	return s
}

func SumValues(s []Book) (t int64) {
	for i := range s {
		t += s[i].Price
	}
	return
}

func SumPointers(s []*Book) (t int64) {
	for _, b := range s {
		t += b.Price
	}
	return
}

// --- 3. Parcours par lignes ou par colonnes ---

const N = 2048

type Matrix [N][N]int32

func SumRows(m *Matrix) (t int64) {
	for i := range N {
		for j := range N {
			t += int64(m[i][j])
		}
	}
	return
}

func SumCols(m *Matrix) (t int64) {
	for j := range N {
		for i := range N {
			t += int64(m[i][j])
		}
	}
	return
}

// --- 4. Petite slice ou map ---

type kv struct {
	k string
	v int
}

func FindSlice(s []kv, k string) int {
	for _, e := range s {
		if e.k == k {
			return e.v
		}
	}
	return -1
}

// --- 5. Alignement des champs ---

type Padded struct {
	A bool
	B int64
	C bool
	D int64
	E bool
}

type Packed struct {
	B int64
	D int64
	A bool
	C bool
	E bool
}

// --- 6. Faux partage ---

type counters struct{ n [8]int64 } // 8 compteurs dans la même ligne de cache de 64 octets

type paddedCounter struct {
	n int64
	_ [56]byte // chaque compteur occupe sa propre ligne de cache
}

func CountShared(workers, per int) int64 {
	var c counters
	var wg sync.WaitGroup
	for w := range workers {
		wg.Go(func() {
			for range per {
				c.n[w]++
			}
		})
	}
	wg.Wait()
	var t int64
	for _, x := range c.n {
		t += x
	}
	return t
}

func CountPadded(workers, per int) int64 {
	var c [8]paddedCounter
	var wg sync.WaitGroup
	for w := range workers {
		wg.Go(func() {
			for range per {
				c[w].n++
			}
		})
	}
	wg.Wait()
	var t int64
	for _, x := range c {
		t += x.n
	}
	return t
}
```

```go
// Fichier : mem_test.go
package mem

import (
	"fmt"
	"testing"
	"unsafe"
)

var (
	sinkS []int
	sinkM map[int]int
	sinkI int64
	sinkN int
)

func BenchmarkAppend(b *testing.B) {
	b.Run("grow", func(b *testing.B) {
		for b.Loop() {
			sinkS = AppendGrow(1_000_000)
		}
	})
	b.Run("prealloc", func(b *testing.B) {
		for b.Loop() {
			sinkS = AppendPrealloc(1_000_000)
		}
	})
}

func BenchmarkMap(b *testing.B) {
	b.Run("grow", func(b *testing.B) {
		for b.Loop() {
			sinkM = MapGrow(100_000)
		}
	})
	b.Run("prealloc", func(b *testing.B) {
		for b.Loop() {
			sinkM = MapPrealloc(100_000)
		}
	})
}

func BenchmarkSum(b *testing.B) {
	vals, ptrs := Values(1_000_000), Pointers(1_000_000)
	b.Run("valeurs", func(b *testing.B) {
		for b.Loop() {
			sinkI = SumValues(vals)
		}
	})
	b.Run("pointeurs", func(b *testing.B) {
		for b.Loop() {
			sinkI = SumPointers(ptrs)
		}
	})
}

func BenchmarkMatrix(b *testing.B) {
	m := new(Matrix)
	b.Run("lignes", func(b *testing.B) {
		for b.Loop() {
			sinkI = SumRows(m)
		}
	})
	b.Run("colonnes", func(b *testing.B) {
		for b.Loop() {
			sinkI = SumCols(m)
		}
	})
}

func BenchmarkLookup(b *testing.B) {
	for _, n := range []int{4, 8, 16, 64} {
		s := make([]kv, n)
		m := make(map[string]int, n)
		for i := range n {
			k := fmt.Sprintf("clé-%d", i)
			s[i], m[k] = kv{k, i}, i
		}
		target := s[n-1].k // le pire cas pour la slice : le dernier élément
		b.Run(fmt.Sprintf("slice-%d", n), func(b *testing.B) {
			for b.Loop() {
				sinkN = FindSlice(s, target)
			}
		})
		b.Run(fmt.Sprintf("map-%d", n), func(b *testing.B) {
			for b.Loop() {
				sinkN = m[target]
			}
		})
	}
}

func BenchmarkFalseSharing(b *testing.B) {
	b.Run("partage", func(b *testing.B) {
		for b.Loop() {
			sinkI = CountShared(8, 1_000_000)
		}
	})
	b.Run("rembourre", func(b *testing.B) {
		for b.Loop() {
			sinkI = CountPadded(8, 1_000_000)
		}
	})
}

func TestSizes(t *testing.T) {
	t.Logf("Padded=%d octets, Packed=%d octets", unsafe.Sizeof(Padded{}), unsafe.Sizeof(Packed{}))
}
```

```sh
go test -run Sizes -v          # Padded=40 octets, Packed=24 octets
go test -bench . -benchmem -count 3
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `AppendPrealloc` : `make([]int, 0, n)` | Une allocation au lieu de 38, et aucune recopie. |
| `MapPrealloc` : `make(map[int]int, n)` | La map ne se réorganise pas en grandissant. |
| `Pointers` mélange l'ordre | Reproduit la situation réelle d'objets créés et réordonnés au fil de l'eau. |
| `SumRows` / `SumCols` | Même calcul, ordre d'accès différent : ×15. |
| `FindSlice` cherche le **dernier** élément | Le pire cas pour la slice ; à adapter à sa distribution réelle. |
| `unsafe.Sizeof` | Mesure la taille réelle, rembourrage compris. |
| `paddedCounter` avec `_ [56]byte` | 8 + 56 = 64 octets : un compteur par ligne de cache. |
| `wg.Go` (N6 L5) | Les huit goroutines tournent vraiment en parallèle (8 fils). |

## Erreurs fréquentes

1. **Ne pas préallouer** une slice dont la taille est connue.
2. **Des slices de pointeurs** pour de grands ensembles de petites structs.
3. **Parcourir un tableau à deux dimensions par colonnes.**
4. **Appliquer une règle** (« petite slice > map ») sans mesurer avec ses données.
5. **Des compteurs par goroutine côte à côte** dans un tableau partagé.
6. **Réordonner les champs de toutes les structs** « pour la performance », au détriment de la lisibilité, là
   où cela ne compte pas.

## Bonnes pratiques Go

- `make([]T, 0, n)`, `make(map[K]V, n)` quand n est connu ou estimable.
- Slices de valeurs pour les grands ensembles ; indices plutôt que pointeurs entre éléments.
- La boucle intérieure suit l'ordre de la mémoire.
- Choisir slice ou map par un benchmark avec les vraies clés.
- Champs du plus grand au plus petit pour les structs nombreuses.
- Données écrites par des cœurs différents : séparées (ou locales, puis fusionnées).

## Ce que je dois retenir

- Le processeur lit par lignes de 64 octets et anticipe les accès séquentiels : la localité est reine.
- Préallouer évite allocations et recopies (×5 mesuré pour une slice).
- Contigu bat dispersé (×4) ; l'ordre de parcours peut coûter ×15.
- Slice ou map : on mesure ; ici la map gagne dès 4 clés `string`.
- Le faux partage ralentit des goroutines qui ne partagent rien logiquement (×4).

---

➡️ [Exercices](exercices.md)
