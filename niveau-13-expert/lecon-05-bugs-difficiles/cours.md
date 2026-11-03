# Module 5 — Bugs mémoire et races difficiles

> Go est un langage sûr pour la mémoire : pas de pointeur libéré qu'on réutilise, pas de débordement de
> tableau silencieux. Il reste pourtant des défauts qui corrompent des données ou retiennent de la mémoire
> sans jamais faire planter le programme, et des courses qui ne se montrent qu'une fois sur dix mille. Ce
> module donne la méthode pour les traiter : **amplifier** un défaut rare jusqu'à le reproduire à volonté,
> **réduire** le cas jusqu'à l'essentiel, **corriger** la cause et verrouiller la correction par un test. Il
> démontre aussi, en recopiant ce que produit le compilateur, pourquoi une course « bénigne » n'existe pas.

## Objectifs

- Amplifier un défaut rare : répétitions, ordonnancement, points d'injection.
- Réduire un cas d'échec : dichotomie dans l'historique, fuzzing qui minimise, suppression de code.
- Reconnaître les défauts mémoire propres à Go : alias d'`append`, sous-slices qui retiennent, fuites.
- Savoir pourquoi une course de données n'est jamais inoffensive, preuves à l'appui.
- Corriger la cause, et non le symptôme, et empêcher le retour du défaut.

## Explication

### Reproduire l'irreproductible

Un test qui échoue une fois sur 10 000 en CI échouera des centaines de fois par jour en production. La
première tâche est d'**augmenter la fréquence** de l'échec, pour pouvoir l'observer et vérifier une
correction.

Les leviers, du plus simple au plus ciblé :

1. **Répéter** : `go test -run TestX -count=10000 -failfast`. L'outil `stress` du projet Go
   (`golang.org/x/tools/cmd/stress`) relance un binaire de test en parallèle jusqu'à l'échec.
2. **Varier l'ordonnancement** : `-cpu 1,2,4,8` (plusieurs valeurs de `GOMAXPROCS`), plus de goroutines que
   de processeurs, une charge processeur concurrente.
3. **Élargir la fenêtre** : injecter un `runtime.Gosched()` ou un court `time.Sleep` **à l'endroit
   soupçonné**, par un point d'injection prévu pour les tests (un champ de type `func()`, nul en production).
4. **Rendre déterministe** : `testing/synctest` pour le temps, une graine fixée pour le hasard, une
   simulation (module 3) pour les protocoles.

L'exemple le mesure sur un registre qui attribue un identifiant par nom. Il vérifie « le nom existe-t-il ? »
dans une section critique, puis l'insère dans une autre. Deux goroutines enregistrent le même nom, 20 000
fois :

| Deux exécutions | `GOMAXPROCS=1` | 2 | 8 |
|---|---|---|---|
| sans injection | 0 et 0 échec | 5 et 2 (0,01 à 0,03 %) | 2 et 2 (0,01 %) |
| `runtime.Gosched()` dans la fenêtre | 97,6 % et 97,6 % | 96,9 % et 96,9 % | 96,2 % et 95,9 % |

Sans aide, le défaut est presque invisible : **aucun** échec avec un seul processeur, et deux à cinq sur
20 000 sinon. Avec un point d'injection placé dans la fenêtre soupçonnée, il apparaît 97 fois sur 100.
L'hypothèse est confirmée **et** on dispose d'un test qui échouera à coup sûr tant que le défaut est là.

Et `-race` ne dit **rien** : 0 rapport. Tous les accès sont protégés par le verrou, et la course est
**logique** (module 1). Seul un invariant (« un nom, un identifiant ») la révèle.

### Réduire

Un cas qui échoue n'est utile que s'il est **petit** : peu de code, peu de données, aucun hasard. On réduit :

- **dans le temps**, avec `git bisect` (N12 L1) : quel commit a introduit le défaut ? Le diff est souvent
  l'explication ;
- **dans les données**, avec le **fuzzing** (N5 L7), qui **minimise** de lui-même l'entrée qui échoue.
  Dans l'exemple, un analyseur de plages (`1-3,7,10-12`) panique sur une entrée que personne n'avait
  prévue. Le fuzzing la trouve en 0,03 s, et l'enregistre déjà réduite à **`"10-0"`** dans
  `testdata/fuzz/FuzzRanges/`. Ce fichier devient un cas de test permanent, rejoué par chaque `go test` ;
- **dans le code** : supprimer ce qui n'est pas nécessaire tant que le défaut reste (*delta debugging*), à la
  main ou par une recherche automatique sur les entrées. Le cas réduit est celui qu'on joint au ticket.

### Les défauts mémoire de Go

Go n'a pas les erreurs mémoire du C, mais il a les siennes.

**L'alias d'`append`.** `append(base, x)` écrit **dans le tableau de `base`** s'il a de la capacité libre.
Deux appels sur la même `base` partagent alors le même emplacement :

```
append : x = [a b y], y = [a b y] (x a été modifié par le second appel)
expression complète : x = [a b x], y = [a b y]
```

Le premier résultat a été **écrasé** par le second appel, sans erreur et sans course. La parade est
l'expression de slice complète, `base[:len(base):len(base)]` : sa capacité égale sa longueur, et `append` est
obligé d'allouer un nouveau tableau. On peut aussi utiliser `slices.Clip(base)`. Toute fonction qui fait
`append` sur une slice reçue en paramètre et la renvoie doit se poser la question.

**La sous-slice qui retient.** `msg[:16]` est une **vue** sur le tableau de `msg` : tant qu'elle vit, tout le
tableau vit. Mesuré : 100 en-têtes de 16 octets, pris sur des messages de 1 Mio, retiennent **environ
102 400 Kio**. Copiés (`bytes.Clone`), ils en retiennent 4 à 10, selon l'exécution. Même chose pour une sous-chaîne d'une grande `string`
(`strings.Clone`).

**Les autres** :
- **les fuites de goroutines** (N11 L5) ;
- **une map** qui a contenu un million de clés garde sa taille après les suppressions. On la recrée si
  elle a beaucoup rétréci ;
- **les caches sans borne** ;
- **une copie de `sync.Mutex`** (une struct passée par valeur) : `go vet` (`copylocks`) la signale ;
- **`unsafe` et cgo**, qui rouvrent la porte aux vraies erreurs mémoire.

### Une course « bénigne » n'existe pas

On entend souvent : « cette variable n'est qu'un indicateur ; si on lit une valeur un peu ancienne, ce n'est
pas grave ». Deux expériences montrent pourquoi c'est faux.

**Le compilateur peut supprimer l'écrivain.** Une goroutine alterne deux valeurs dans une variable globale,
sans synchronisation, et une autre la lit. Le lecteur ne voit **jamais** la seconde valeur : 72 millions de
lectures, toutes égales à la valeur initiale. Le code machine de l'écrivain explique pourquoi :

```
$ go tool objdump -s 'main\.main\.func1' torn
  main.go:25		CALL runtime.mallocgcTinySC2(SB)
  main.go:26		NOPL
  main.go:1		JMP 0x49bce5          ← une boucle vide : plus aucune écriture
```

Rien n'oblige une goroutine à rendre ses écritures visibles aux autres sans synchronisation (le modèle
mémoire, module 1). Le compilateur a donc remplacé la boucle par une boucle vide, ce qui est parfaitement
légal. Le programme « marchait » en ce sens qu'il ne plantait pas, mais il ne faisait plus ce qui était écrit.

**Une lecture peut être déchirée.** Une valeur d'interface tient en **deux** mots : le type et la donnée.
Quand l'écriture a lieu (dans une fonction non *inlinée*, pour que le compilateur la garde), une lecture
concurrente peut voir le **type** de l'une et la **donnée** de l'autre :

```
lecture déchirée après 2517 lectures : Area() = 0                               (une première exécution)
lectures : 18827652 fois 9, 17496477 fois 10, 4142634 valeurs impossibles
```

`Area()` ne peut valoir que 9 (`Square` de côté 3) ou 10 (`Rect` 2 × 5). Pourtant **10 %** des lectures
donnent autre chose : 0, 4, selon la combinaison. On appelle la méthode d'un type sur la donnée d'un autre. Le
programme ne plante pas ici, mais avec des pointeurs dans la donnée, il pourrait lire n'importe où, ou planter
loin de la cause. Les slices (trois mots), les chaînes (deux mots) et les interfaces sont toutes exposées.

`-race` signale ces deux programmes dès la première exécution (3 rapports chacun, mesuré). La règle est sans exception : **toute course de
données est un bug**. Pour un indicateur partagé, on utilise `atomic.Bool` ou `atomic.Pointer`, qui coûtent
presque rien (N6 L7).

### Corriger la cause

Une fois le défaut reproduit et réduit :

- **corriger la cause, pas le symptôme**. Un `if id != 0` qui masque le doublon du registre ne corrige rien,
  il déplace le problème. La correction réunit la vérification et l'insertion dans **une** section critique ;
- **garder le test qui reproduisait le défaut**, avec son point d'injection, et le cas réduit du fuzzing.
  Il échouera si le défaut revient ;
- **chercher les autres occurrences** du même motif : un `append` sur un paramètre, un `Unlock` entre une
  vérification et une action. Un défaut a rarement un seul exemplaire.

## Exemple

Trois paquets et un programme :
- `mem` : l'alias, la rétention, le registre et son amplification ;
- `parse` : l'analyseur que le fuzzing prend en défaut ;
- `torn` et `torn2` : les deux expériences sur la course « bénigne ».

```go.mod
// Fichier : go.mod
module example.com/mem

go 1.27
```

```go
// Fichier : mem/mem.go
// Package mem : des défauts de mémoire qui ne font pas planter Go, mais corrompent ou retiennent.
package mem

import (
	"bytes"
	"runtime"
	"sync"
)

// --- 1. L'alias d'append ---

// WithPrefix renvoie base suivi de x. FAUX : si base a de la capacité libre, deux appels écrivent
// dans le MÊME tableau, et le second écrase le résultat du premier.
func WithPrefix(base []string, x string) []string { return append(base, x) }

// WithPrefixSafe : l'expression de slice complète base[:len:len] interdit d'écrire au-delà de len.
func WithPrefixSafe(base []string, x string) []string {
	return append(base[:len(base):len(base)], x)
}

// --- 2. La sous-slice qui retient ---

// Header garde les 16 premiers octets d'un gros message. FAUX : la sous-slice référence tout le
// tableau d'origine, qui ne peut plus être libéré.
func Header(msg []byte) []byte { return msg[:16] }

// HeaderSafe copie : seuls 16 octets restent vivants.
func HeaderSafe(msg []byte) []byte { return bytes.Clone(msg[:16]) }

// HeapInUse mesure le tas vivant après un GC.
func HeapInUse() uint64 {
	runtime.GC()
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	return m.HeapAlloc
}

// --- 3. Une course logique rare ---

// Registry attribue un identifiant unique par nom. FAUX : la vérification et l'insertion sont deux
// sections critiques séparées (une fenêtre de quelques instructions).
type Registry struct {
	mu    sync.Mutex
	ids   map[string]int
	next  int
	Yield func() // point d'injection pour les tests : appelé dans la fenêtre
}

func NewRegistry() *Registry { return &Registry{ids: map[string]int{}} }

func (r *Registry) Register(name string) int {
	r.mu.Lock()
	id, ok := r.ids[name]
	r.mu.Unlock()
	if ok {
		return id
	}
	if r.Yield != nil {
		r.Yield()
	}
	r.mu.Lock()
	defer r.mu.Unlock()
	r.next++
	r.ids[name] = r.next
	return r.next
}
```

```go
// Fichier : mem/mem_test.go
package mem

import (
	"fmt"
	"runtime"
	"sync"
	"testing"
)

func TestAppendAlias(t *testing.T) {
	base := make([]string, 2, 10) // de la capacité libre : le piège
	base[0], base[1] = "a", "b"
	x := WithPrefix(base, "x")
	y := WithPrefix(base, "y")
	fmt.Printf("append : x = %v, y = %v (x a été modifié par le second appel)\n", x, y)
	xs := WithPrefixSafe(base, "x")
	ys := WithPrefixSafe(base, "y")
	fmt.Printf("expression complète : x = %v, y = %v\n", xs, ys)
	if xs[2] != "x" {
		t.Fatal("WithPrefixSafe")
	}
}

var kept [][]byte

func TestRetention(t *testing.T) {
	for _, safe := range []bool{false, true} {
		kept = nil
		h0 := HeapInUse()
		for range 100 {
			msg := make([]byte, 1<<20) // un message de 1 Mio
			if safe {
				kept = append(kept, HeaderSafe(msg))
			} else {
				kept = append(kept, Header(msg))
			}
		}
		h1 := HeapInUse()
		fmt.Printf("copie=%-5v 100 en-têtes de 16 octets retiennent %d Kio\n", safe, (h1-h0)>>10)
	}
	runtime.KeepAlive(kept)
}

// trial : deux goroutines enregistrent le même nom ; renvoie vrai si elles ont reçu deux identifiants.
func trial(yield func()) bool {
	r := NewRegistry()
	r.Yield = yield
	var a, b int
	var wg sync.WaitGroup
	start := make(chan struct{})
	wg.Go(func() { <-start; a = r.Register("x") })
	wg.Go(func() { <-start; b = r.Register("x") })
	close(start)
	wg.Wait()
	return a != b
}

func TestAmplify(t *testing.T) {
	const n = 20_000
	for _, tc := range []struct {
		name  string
		yield func()
	}{{"sans injection", nil}, {"runtime.Gosched() dans la fenêtre", runtime.Gosched}} {
		for _, procs := range []int{1, 2, 8} {
			prev := runtime.GOMAXPROCS(procs)
			bad := 0
			for range n {
				if trial(tc.yield) {
					bad++
				}
			}
			runtime.GOMAXPROCS(prev)
			fmt.Printf("%-34s GOMAXPROCS=%d : %5d échecs sur %d (%.2f %%)\n", tc.name, procs, bad, n, 100*float64(bad)/n)
		}
	}
}
```

```go
// Fichier : parse/parse.go
// Package parse : un analyseur de plages « 1-3,7,10-12 », avec un défaut caché.
package parse

import (
	"errors"
	"strconv"
	"strings"
)

// Ranges renvoie les entiers d'une liste de plages. Les entrées invalides doivent donner une erreur,
// jamais une panique.
func Ranges(s string) ([]int, error) {
	var out []int
	for part := range strings.SplitSeq(s, ",") {
		lo, hi, isRange := strings.Cut(part, "-")
		a, err := strconv.Atoi(lo)
		if err != nil {
			return nil, err
		}
		b := a
		if isRange {
			if b, err = strconv.Atoi(hi); err != nil {
				return nil, err
			}
		}
		if b-a > 1000 {
			return nil, errors.New("plage trop grande")
		}
		seg := make([]int, b-a+1) // le défaut : b < a donne une longueur négative
		for i := range seg {
			seg[i] = a + i
		}
		out = append(out, seg...)
	}
	return out, nil
}
```

```go
// Fichier : parse/parse_test.go
package parse

import (
	"slices"
	"testing"
)

func TestRanges(t *testing.T) {
	got, err := Ranges("1-3,7,10-12")
	if err != nil || !slices.Equal(got, []int{1, 2, 3, 7, 10, 11, 12}) {
		t.Fatalf("%v %v", got, err)
	}
}

func FuzzRanges(f *testing.F) {
	f.Add("1-3,7,10-12")
	f.Add("42")
	f.Fuzz(func(t *testing.T, s string) {
		Ranges(s) // la propriété : ne jamais paniquer
	})
}
```

```go
// Fichier : cmd/torn/main.go
// Commande torn : une course « bénigne » sur une variable d'interface. Une interface tient DEUX mots
// (le type et la donnée) ; une lecture concurrente peut voir le type de l'une et la donnée de l'autre.
package main

import (
	"fmt"
	"os"
	"time"
)

type Shape interface{ Area() int }

type Square struct{ side int }

func (s *Square) Area() int { return s.side * s.side }

type Rect struct{ w, h int } // une valeur, pas un pointeur : sa donnée est ailleurs en mémoire

func (r Rect) Area() int { return r.w * r.h }

var current Shape = &Square{side: 3}

func main() {
	go func() { // l'écrivain : alterne deux types, sans synchronisation
		sq, rc := Shape(&Square{side: 3}), Shape(Rect{w: 2, h: 5})
		for i := 0; ; i++ {
			if i%2 == 0 { // une alternance que le compilateur ne peut pas réduire à une seule écriture
				current = sq
			} else {
				current = rc
			}
		}
	}()
	deadline := time.Now().Add(3 * time.Second)
	bad, nine, ten := 0, 0, 0
	for n := 0; time.Now().Before(deadline); n++ {
		a := current.Area() // le lecteur : attend 9 ou 10
		switch a {
		case 9:
			nine++
		case 10:
			ten++
		}
		if a != 9 && a != 10 {
			bad++
			if bad == 1 {
				fmt.Printf("lecture déchirée après %d lectures : Area() = %d\n", n, a)
			}
		}
	}
	fmt.Printf("lectures : %d fois 9, %d fois 10, %d valeurs impossibles\n", nine, ten, bad)
	os.Exit(0)
}
```

```go
// Fichier : cmd/torn2/main.go
// Commande torn : une course « bénigne » sur une variable d'interface. Une interface tient DEUX mots
// (le type et la donnée) ; une lecture concurrente peut voir le type de l'une et la donnée de l'autre.
package main

import (
	"fmt"
	"os"
	"time"
)

type Shape interface{ Area() int }

type Square struct{ side int }

func (s *Square) Area() int { return s.side * s.side }

type Rect struct{ w, h int } // une valeur, pas un pointeur : sa donnée est ailleurs en mémoire

func (r Rect) Area() int { return r.w * r.h }

var current Shape = &Square{side: 3}

// set fait l'écriture dans une fonction que le compilateur ne voit pas de l'appelant : elle a lieu.
//
//go:noinline
func set(s Shape) { current = s }

func main() {
	go func() { // l'écrivain : alterne deux types, sans synchronisation
		sq, rc := Shape(&Square{side: 3}), Shape(Rect{w: 2, h: 5})
		for i := 0; ; i++ {
			if i%2 == 0 {
				set(sq)
			} else {
				set(rc)
			}
		}
	}()
	deadline := time.Now().Add(3 * time.Second)
	bad, nine, ten := 0, 0, 0
	for n := 0; time.Now().Before(deadline); n++ {
		a := current.Area() // le lecteur : attend 9 ou 10
		switch a {
		case 9:
			nine++
		case 10:
			ten++
		}
		if a != 9 && a != 10 {
			bad++
			if bad == 1 {
				fmt.Printf("lecture déchirée après %d lectures : Area() = %d\n", n, a)
			}
		}
	}
	fmt.Printf("lectures : %d fois 9, %d fois 10, %d valeurs impossibles\n", nine, ten, bad)
	os.Exit(0)
}
```

```
$ go test -v ./mem/
append : x = [a b y], y = [a b y] (x a été modifié par le second appel)
expression complète : x = [a b x], y = [a b y]
copie=false 100 en-têtes de 16 octets retiennent 102418 Kio
copie=true  100 en-têtes de 16 octets retiennent 4 Kio
sans injection                     GOMAXPROCS=1 :     0 échecs sur 20000 (0.00 %)
sans injection                     GOMAXPROCS=2 :     2 échecs sur 20000 (0.01 %)
sans injection                     GOMAXPROCS=8 :     2 échecs sur 20000 (0.01 %)
runtime.Gosched() dans la fenêtre  GOMAXPROCS=1 : 19520 échecs sur 20000 (97.60 %)
runtime.Gosched() dans la fenêtre  GOMAXPROCS=2 : 19382 échecs sur 20000 (96.91 %)
runtime.Gosched() dans la fenêtre  GOMAXPROCS=8 : 19171 échecs sur 20000 (95.86 %)
$ go test -race -run Amplify ./mem/ 2>&1 | grep -c "DATA RACE"
0
$ go test -run XXX -fuzz FuzzRanges -fuzztime 60s ./parse/
--- FAIL: FuzzRanges (0.03s)
    --- FAIL: FuzzRanges (0.00s)
        testing.go:2078: panic: runtime error: makeslice: len out of range
            …
            example.com/mem/parse.Ranges-range1(…)
            	…/parse/parse.go:29 +0x391
$ cat parse/testdata/fuzz/FuzzRanges/*
go test fuzz v1
string("10-0")
$ ./torn          # l'écrivain écrit directement dans la globale
lectures : 72536140 fois 9, 0 fois 10, 0 valeurs impossibles
$ ./torn2         # l'écrivain passe par set, non inlinée
lecture déchirée après 2628 lectures : Area() = 4
lectures : 15993385 fois 9, 22483108 fois 10, 3966300 valeurs impossibles
```

(Les nombres de `torn2` varient d'une exécution à l'autre : trois exécutions ont donné 3,5 à 4,2 millions de
valeurs impossibles en 3 s. Le corrigé de l'analyseur est l'objet de l'exercice E1.)

## Explication du code

| Élément | Rôle |
|---|---|
| `base[:len(base):len(base)]` | L'expression de slice complète : capacité égale à la longueur, `append` doit allouer. |
| `bytes.Clone(msg[:16])` | Copie les 16 octets dans un tableau à eux : le message d'origine peut être libéré. |
| `Registry.Yield` | Un point d'injection, nul en production, appelé **dans** la fenêtre entre la vérification et l'insertion. |
| `start` fermé une fois | Les deux goroutines partent en même temps : on maximise le chevauchement. |
| `runtime.GOMAXPROCS(procs)` | Le même test sous plusieurs ordonnancements. |
| `f.Fuzz(… Ranges(s) …)` | La propriété la plus simple, « ne jamais paniquer », trouve déjà le défaut. |
| `set` en `//go:noinline` | Garde l'écriture visible du compilateur ; sans cela, la boucle de l'écrivain disparaît. |
| `Rect` en valeur, `Square` en pointeur | Deux représentations différentes de la donnée : une lecture déchirée appelle la méthode de l'un sur la donnée de l'autre. |

## Erreurs fréquentes

1. **Relancer la CI** sur un test instable au lieu de l'amplifier pour comprendre.
2. **Ajouter un `time.Sleep`** dans le code de production pour « faire passer » une course.
3. **Croire que `-race` silencieux prouve l'absence de course** : les courses logiques lui échappent.
4. **Garder un cas d'échec de 500 lignes** dans un ticket.
5. **`append` sur un paramètre renvoyé** sans se demander qui partage le tableau.
6. **Garder une petite sous-slice** d'un gros tampon dans une structure qui vit longtemps.
7. **Tolérer une course « inoffensive »** sur un indicateur ou un compteur de statistiques.
8. **Corriger le symptôme** (filtrer les doublons) au lieu de la fenêtre qui les crée.

## Bonnes pratiques Go

- Amplifier : `-count`, `-cpu`, `stress`, points d'injection dans les fenêtres soupçonnées, `synctest`.
- Réduire : `git bisect`, le fuzzing et son corpus minimisé, puis un test court et déterministe.
- `slices.Clip` ou l'expression complète avant un `append` sur une slice reçue ; `bytes.Clone` et
  `strings.Clone` pour garder un petit morceau d'un grand tampon.
- `atomic.*` pour tout indicateur partagé ; aucune course tolérée, même « bénigne ».
- Chaque correction s'accompagne du test qui reproduisait le défaut, et d'une recherche des autres
  occurrences du motif.

## Ce que je dois retenir

- Un défaut rare se rend fréquent : 0,03 % sans aide, 97 % avec un point d'injection.
- Le fuzzing trouve et réduit : `"10-0"` en 0,03 s, gardé comme cas de test.
- `append` peut écraser le résultat d'un appel précédent ; une sous-slice de 16 octets peut retenir 1 Mio.
- Une course « bénigne » peut faire disparaître l'écrivain, ou faire lire une valeur déchirée (10 % ici).
- On corrige la cause, on garde le test, on cherche les jumeaux.

---

➡️ [Exercices](exercices.md)
