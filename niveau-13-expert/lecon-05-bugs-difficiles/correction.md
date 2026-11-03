# Module 5 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Corriger l'analyseur, et renforcer la propriété

```go
// Fichier : parse/parse.go
// Package parse : un analyseur de plages « 1-3,7,10-12 » (corrigé).
package parse

import (
	"errors"
	"fmt"
	"strconv"
	"strings"
)

var errDecreasing = errors.New("plage décroissante")

// Ranges renvoie les entiers d'une liste de plages. Les entrées invalides donnent une erreur, jamais
// une panique.
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
		if b < a {
			return nil, fmt.Errorf("%q : %w", part, errDecreasing) // le cas trouvé par le fuzzing
		}
		if b-a > 1000 {
			return nil, errors.New("plage trop grande")
		}
		for v := a; v <= b; v++ {
			out = append(out, v)
		}
	}
	return out, nil
}

// Format écrit une liste d'entiers comme une suite de valeurs séparées par des virgules.
func Format(xs []int) string {
	var b strings.Builder
	for i, x := range xs {
		if i > 0 {
			b.WriteByte(',')
		}
		b.WriteString(strconv.Itoa(x))
	}
	return b.String()
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
	if _, err := Ranges("10-0"); err == nil {
		t.Fatal("plage décroissante acceptée")
	}
}

func FuzzRanges(f *testing.F) {
	f.Add("1-3,7,10-12")
	f.Add("42")
	f.Fuzz(func(t *testing.T, s string) {
		got, err := Ranges(s)
		if err != nil {
			return
		}
		// Propriété : reformater puis relire redonne exactement le même résultat.
		again, err := Ranges(Format(got))
		if err != nil || !slices.Equal(got, again) {
			t.Fatalf("%q → %v → %q → %v, %v", s, got, Format(got), again, err)
		}
	})
}
```

Le cas trouvé par le fuzzing reste dans `parse/testdata/fuzz/FuzzRanges/f980c85e1c00b5dc` :

```
go test fuzz v1
string("10-0")
```

```
$ go test -v -run 'FuzzRanges/f980' ./parse/
=== RUN   FuzzRanges/f980c85e1c00b5dc
    --- PASS: FuzzRanges/f980c85e1c00b5dc (0.00s)
$ go test -run XXX -fuzz FuzzRanges -fuzztime 60s ./parse/
fuzz: elapsed: 1m0s, execs: 8078144 (113693/sec), new interesting: 23 (total: 26)
PASS
```

- **Le cas réduit est rejoué** à chaque `go test`, sans `-fuzz` : il fait partie des tests ordinaires, et le
  défaut ne peut pas revenir sans qu'on le voie.
- **La propriété d'aller-retour** (« reformater puis relire donne le même résultat ») ne trouve rien en
  8 millions d'exécutions. C'est un résultat, pas un échec : la propriété est vraie pour cette version.
- **« Le résultat est trié » est faux par conception** : `Ranges("3,1")` renvoie `[3 1]`. Le fuzzing le
  signalerait aussitôt. Ce n'est pas un défaut du code, c'est une question de **spécification** : doit-il
  trier ? Une propriété de fuzzing est une affirmation sur la spécification. Il faut l'écrire en sachant ce
  qu'on promet.

## E2 — La map qui ne rétrécit pas

```go
// Fichier : mem/fix_test.go
package mem

import (
	"fmt"
	"runtime"
	"slices"
	"testing"
)

var keepMap map[int][128]byte

// E2 : une map qui a contenu un million d'entrées garde sa taille.
func TestMapShrink(t *testing.T) {
	h0 := HeapInUse()
	m := map[int][128]byte{}
	for i := range 1_000_000 {
		m[i] = [128]byte{}
	}
	h1 := HeapInUse()
	for i := range 1_000_000 {
		if i >= 10 {
			delete(m, i)
		}
	}
	keepMap = m
	h2 := HeapInUse()
	fresh := make(map[int][128]byte, len(m))
	for k, v := range m {
		fresh[k] = v
	}
	keepMap = fresh
	h3 := HeapInUse()
	fmt.Printf("E2 : 1 M entrées %d Mio ; 10 entrées après suppressions %d Mio ; recopiées dans une map neuve %d Kio\n",
		(h1-h0)>>20, (h2-h0)>>20, (int64(h3)-int64(h0))>>10)
	runtime.KeepAlive(keepMap)
}

// E3 : l'alias dans une boucle.
func tagged(items []string, common []string) [][]string {
	var out [][]string
	for _, it := range items {
		out = append(out, append(common, it))
	}
	return out
}

func taggedSafe(items []string, common []string) [][]string {
	var out [][]string
	for _, it := range items {
		out = append(out, append(slices.Clip(common), it)) // capacité = longueur : chaque append alloue
	}
	return out
}

func TestTaggedAlias(t *testing.T) {
	items := []string{"x", "y", "z"}
	lit := []string{"a", "b"}
	withCap := make([]string, 2, 8)
	copy(withCap, lit)
	fmt.Println("E3 : littéral    :", tagged(items, lit))
	fmt.Println("E3 : capacité 8  :", tagged(items, withCap))
	fmt.Println("E3 : corrigé     :", taggedSafe(items, withCap))
	if got := taggedSafe(items, withCap); got[0][2] != "x" {
		t.Fatal("taggedSafe")
	}
}
```

```
E2 : 1 M entrées 287 Mio ; 10 entrées après suppressions 287 Mio ; recopiées dans une map neuve -21 Kio
```

Après avoir supprimé 999 990 entrées sur un million, la map occupe **toujours 287 Mio**. Les suppressions
libèrent les emplacements, pas les tableaux sous-jacents. Recopier les dix entrées dans une map neuve rend
toute la mémoire (le tas finit même 21 Kio sous son niveau de départ).

Un cache qui a connu un pic de trafic garde la mémoire du pic **pour toujours**. On le recrée périodiquement
(quand sa taille est tombée sous une fraction de son maximum), ou on utilise une structure bornée dès le
départ.

## E3 — L'alias dans une boucle

Dans `fix_test.go` ci-dessus :

```
E3 : littéral    : [[a b x] [a b y] [a b z]]
E3 : capacité 8  : [[a b z] [a b z] [a b z]]
E3 : corrigé     : [[a b x] [a b y] [a b z]]
```

Le défaut ne se déclenche que si `common` a **de la capacité libre**. Un littéral `[]string{"a", "b"}` a une
capacité égale à sa longueur (2). Chaque `append` doit alors allouer un nouveau tableau, et les résultats sont
indépendants.

Avec `make([]string, 2, 8)`, les trois `append` écrivent au même emplacement (l'indice 2) du **même** tableau.
Les trois résultats sont trois vues sur ce tableau, et tous valent `[a b z]`, la dernière écriture.

C'est pourquoi ce défaut passe les tests écrits avec des littéraux. Il n'apparaît qu'en production, quand
`common` vient d'une construction par `append` qui a laissé de la capacité. Le test qui le montre doit donc
**construire `common` avec de la capacité libre**. La correction, `slices.Clip(common)`, est identique à
`common[:len(common):len(common)]`.

## E4 — L'outil `stress`

```go
// Fichier : mem/flaky_test.go
package mem

import "testing"

// E4 : un seul essai, sans point d'injection ; il échoue si les deux goroutines ont reçu deux identifiants.
func TestFlaky(t *testing.T) {
	if trial(nil) {
		t.Fatal("deux identifiants pour le même nom")
	}
}
```

```
$ go test -c -o reg.test ./mem/
$ stress -timeout 5s -p 8 ./reg.test -test.run '^TestFlaky$'
5s: 6877 runs so far, 6 failures (0.09%), 8 active
10s: 13929 runs so far, 8 failures (0.06%), 8 active
…
1m0s: 83789 runs so far, 58 failures (0.07%), 8 active
$ go test -count=20000 -run '^TestFlaky$' ./mem/
(aucun échec)
```

| Technique | Résultat |
|---|---|
| `-count=20000`, un seul processus | **0** échec |
| `stress`, 8 processus en parallèle, une minute | 58 échecs sur 83 789 (0,07 %), le premier dans les 5 premières secondes |
| point d'injection (`runtime.Gosched()` dans la fenêtre) | environ 97 % |

`stress` relance un **processus neuf** à chaque essai, huit à la fois. Chaque démarrage met l'ordonnanceur
dans un état différent, et les huit processus se disputent les processeurs. Dans un seul processus, les
20 000 essais s'enchaînent dans un état stable qui ne tombe jamais sur la fenêtre.

`stress` est l'outil des défauts dont on ne connaît **pas encore** l'emplacement. Le point d'injection est
celui des défauts dont on soupçonne la fenêtre. `stress` écrit la sortie de chaque échec dans un fichier
`/tmp/go-stress-…`, qu'on garde pour l'enquête.

## E5 — L'indicateur d'arrêt

```go
// Fichier : cmd/stop/main.go
// Commande stop : un indicateur d'arrêt non synchronisé (E5).
package main

import (
	"fmt"
	"os"
	"sync/atomic"
	"time"
)

var stop bool
var stopAtomic atomic.Bool
var n int

//go:noinline
func work() { n++ }

func main() {
	mode := os.Args[1]
	done := make(chan struct{})
	go func() {
		switch mode {
		case "vide":
			for !stop {
			}
		case "appel":
			for !stop {
				work()
			}
		case "atomic":
			for !stopAtomic.Load() {
			}
		}
		close(done)
	}()
	time.Sleep(100 * time.Millisecond)
	stop = true
	stopAtomic.Store(true)
	select {
	case <-done:
		fmt.Printf("%-6s : le travailleur s'est arrêté\n", mode)
	case <-time.After(2 * time.Second):
		fmt.Printf("%-6s : le travailleur tourne encore 2 s après stop = true\n", mode)
	}
}
```

```
$ ./stop vide; ./stop appel; ./stop atomic
vide   : le travailleur s'est arrêté
appel  : le travailleur s'est arrêté
atomic : le travailleur s'est arrêté
$ go tool objdump -s 'main\.main\.func1' stop        # la boucle vide
  main.go:24		CMPB main.stop(SB), $0x0
  main.go:24		JE 0x4a81f0
$ go build -race -o stop-race ./cmd/stop && ./stop-race vide 2>&1 | grep -c "DATA RACE"
1
```

**Le résultat contredit l'énoncé** : avec go1.27.3, le travailleur s'arrête dans les trois cas. Le code machine
montre pourquoi : la boucle vide **relit** `main.stop` à chaque tour (`CMPB main.stop(SB)`). Le compilateur
actuel ne sort pas cette lecture de la boucle.

**Ce n'est pas une garantie.** Le compilateur **a le droit** de le faire, puisqu'aucune synchronisation n'oblige
la goroutine à voir l'écriture d'une autre. Le cours montre qu'il supprime bien les **écritures** d'une boucle
semblable. Une autre version de Go, une autre architecture, ou une boucle un peu différente peut donner une
boucle infinie. `-race` signale la course dès la première exécution.

`atomic.Bool` (`stopAtomic.Load()`) garantit la visibilité, pour un coût négligeable : une lecture ordinaire sur
x86-64 (`MOVL main.stopAtomic(SB)`), que le compilateur ne peut plus sortir de la boucle.

Un programme qui marche aujourd'hui grâce à un comportement non garanti du compilateur est un défaut qui
attend la prochaine version de Go.

---

## Exercice intermédiaire — Le test qui échoue parfois

**1. La fréquence** (`go test -count=2000 -cpu N`, deux abonnés vérifiés par exécution) :

| | Messages faux, sur 4 000 vérifiés |
|---|---|
| `-cpu 1` | **4 000**, soit chaque exécution, chaque abonné |
| `-cpu 4` | 2 321 |
| `-cpu 8` | 2 319 |

Avec un seul processeur logique, l'échec est **systématique**. `Publish` crée deux goroutines, l'une pour « un »
puis l'autre pour « deux ». L'ordonnanceur exécute en premier la goroutine **la plus récemment créée** sur ce
processeur (son emplacement « suivant »), et « deux » arrive donc toujours avant « un ». Avec plusieurs
processeurs, l'ordre dépend de la course, d'où environ 58 % d'échecs.

L'auteur qui a « lancé deux fois » son test sur un poste à plusieurs cœurs a pu tomber deux fois du bon côté.

**2. Les causes.**

- **L'ordre** : une goroutine par message ne garantit **rien** sur l'ordre de livraison. C'est le défaut du
  code.
- **Le `time.Sleep` ne sert à rien** : `<-ch` attend de toute façon la livraison. Il donne seulement
  l'illusion d'avoir traité le problème.
- **Une fuite** : le test ne lit qu'un message par abonné, et le canal n'a qu'une place. La goroutine qui
  envoie le second message reste **bloquée pour toujours**. Mesuré sur l'original : 2 goroutines restantes
  après le test, une par abonné.

**3. La spécification retenue** : l'ordre de publication est garanti pour chaque abonné ; `Publish` ne bloque
jamais ; un abonné dont la file est pleine **perd** le message, et la perte est comptée. D'autres choix sont
défendables : bloquer l'éditeur (contre-pression, N6 L11), ou déconnecter l'abonné lent. Ce qui ne l'est pas,
c'est de ne rien décider.

**4. La correction** :

```go
// Fichier : notify/notify.go
// Package notify : diffuser des messages à des abonnés (corrigé).
//
// Spécification : chaque abonné reçoit les messages DANS L'ORDRE de publication. Publish ne bloque
// jamais : un abonné dont la file (capacité Buffer) est pleine perd le message, et la perte est comptée.
// Aucune goroutine n'est créée par message.
package notify

import (
	"sync"
	"sync/atomic"
)

const Buffer = 16

type Notifier struct {
	mu      sync.Mutex
	subs    []chan string
	Dropped atomic.Int64
}

func (n *Notifier) Subscribe() <-chan string {
	ch := make(chan string, Buffer)
	n.mu.Lock()
	n.subs = append(n.subs, ch)
	n.mu.Unlock()
	return ch
}

// Publish dépose le message dans la file de chaque abonné, sous le verrou : deux Publish successifs
// déposent dans le même ordre pour tous. L'envoi est non bloquant.
func (n *Notifier) Publish(msg string) {
	n.mu.Lock()
	defer n.mu.Unlock()
	for _, ch := range n.subs {
		select {
		case ch <- msg:
		default:
			n.Dropped.Add(1) // abonné trop lent : on ne bloque pas les autres
		}
	}
}
```

```go
// Fichier : notify/notify_test.go
package notify

import (
	"fmt"
	"testing"
)

// Plus de time.Sleep : l'envoi a lieu PENDANT Publish, les messages sont déjà dans les files.
func TestNotify(t *testing.T) {
	n := &Notifier{}
	a, b := n.Subscribe(), n.Subscribe()
	n.Publish("un")
	n.Publish("deux")
	for _, ch := range []<-chan string{a, b} {
		for _, want := range []string{"un", "deux"} {
			if got := <-ch; got != want {
				t.Fatalf("reçu %q, attendu %q", got, want)
			}
		}
	}
}

func TestSlowSubscriberDrops(t *testing.T) {
	n := &Notifier{}
	slow := n.Subscribe()
	for i := range Buffer + 5 {
		n.Publish(fmt.Sprint(i))
	}
	if n.Dropped.Load() != 5 || len(slow) != Buffer {
		t.Fatalf("perdus %d, en file %d", n.Dropped.Load(), len(slow))
	}
	if got := <-slow; got != "0" { // l'ordre reste celui de publication
		t.Fatalf("premier : %q", got)
	}
}
```

```
$ go test -race ./notify/
ok
$ go test -count=100000 -cpu 1,4,8 ./notify/
ok  	example.com/notify/notify	7.533s
```

Il n'y a plus de goroutine par message, donc plus de fuite ni de désordre. L'envoi a lieu **pendant**
`Publish`, sous le verrou : deux `Publish` successifs déposent dans le même ordre pour tous les abonnés. Le test
n'a plus besoin d'attendre, puisque les messages sont déjà dans les files quand `Publish` rend la main.
300 000 exécutions passent, sur trois réglages de `-cpu`.

---

## Défi — La mémoire qui ne correspond à rien

```go
// Fichier : sess/sess.go
// Package sess : les sessions du défi, version d'origine et version corrigée.
package sess

import (
	"bytes"
	"crypto/rand"
	"fmt"
	"io"
	"net/http"
	"sync"
	"time"
)

type Session struct {
	ID    string
	Token []byte
}

type Store struct {
	TTL     time.Duration
	Fixed   bool
	m       sync.Map
	mu      sync.Mutex
	expires map[string]time.Time // version corrigée : une échéance par session, balayée par UNE goroutine
}

func New(ttl time.Duration, fixed bool) *Store {
	s := &Store{TTL: ttl, Fixed: fixed, expires: map[string]time.Time{}}
	if fixed {
		go s.sweep()
	}
	return s
}

func (s *Store) Login(w http.ResponseWriter, r *http.Request) {
	body, _ := io.ReadAll(r.Body)
	sess := &Session{ID: rand.Text()}
	if s.Fixed {
		sess.Token = bytes.Clone(body[:32]) // 32 octets à eux, le corps de 64 Kio peut être libéré
		s.mu.Lock()
		s.expires[sess.ID] = time.Now().Add(s.TTL)
		s.mu.Unlock()
	} else {
		sess.Token = body[:32] // retient tout le corps
		go s.expire(sess.ID)   // une goroutine (et une minuterie) par session, pendant toute sa vie
	}
	s.m.Store(sess.ID, sess)
	fmt.Fprintln(w, sess.ID)
}

func (s *Store) expire(id string) {
	select {
	case <-time.After(s.TTL):
		s.m.Delete(id)
	}
}

func (s *Store) sweep() {
	for range time.Tick(s.TTL / 10) {
		now := time.Now()
		s.mu.Lock()
		for id, at := range s.expires {
			if now.After(at) {
				delete(s.expires, id)
				s.m.Delete(id)
			}
		}
		s.mu.Unlock()
	}
}

func (s *Store) Len() int {
	n := 0
	s.m.Range(func(_, _ any) bool { n++; return true })
	return n
}
```

```go
// Fichier : sess/sess_test.go
package sess

import (
	"bytes"
	"fmt"
	"net/http/httptest"
	"os"
	"path/filepath"
	"runtime"
	"runtime/metrics"
	"runtime/pprof"
	"testing"
	"time"
)

func stacks() uint64 {
	m := []metrics.Sample{{Name: "/memory/classes/heap/stacks:bytes"}}
	metrics.Read(m)
	return m[0].Value.Uint64()
}

func heap() uint64 {
	runtime.GC()
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	return m.HeapAlloc
}

// Une « journée » de 2 s : la durée de vie est de 2 s, 10 000 connexions sont réparties sur 3 s.
func TestDay(t *testing.T) {
	for _, fixed := range []bool{false, true} {
		s := New(2*time.Second, fixed)
		body := make([]byte, 64<<10)
		h0, g0, st0 := heap(), runtime.NumGoroutine(), stacks()
		name := map[bool]string{false: "origine", true: "corrigé"}[fixed]
		f, _ := os.Create(filepath.Join(os.TempDir(), "heap-"+name+"-avant.pprof"))
		pprof.Lookup("heap").WriteTo(f, 0)
		f.Close()
		for i := range 10_000 {
			s.Login(httptest.NewRecorder(), httptest.NewRequest("POST", "/login", bytes.NewReader(body)))
			if i%1000 == 999 {
				time.Sleep(300 * time.Millisecond)
			}
		}
		h1, g1, n, st1 := heap(), runtime.NumGoroutine(), s.Len(), stacks()
		f, _ = os.Create(filepath.Join(os.TempDir(), "heap-"+name+"-apres.pprof"))
		pprof.Lookup("heap").WriteTo(f, 0)
		f.Close()
		fmt.Printf("%-8s sessions vivantes %5d : tas +%4d Mio (%.1f Kio par session), goroutines +%d, piles +%d Kio\n",
			name, n, (h1-h0)>>20, float64(h1-h0)/float64(n)/1024, g1-g0, (int64(st1)-int64(st0))>>10)
		time.Sleep(3 * time.Second) // laisser expirer avant la version suivante
	}
}
```

```
$ go test -v ./sess/
origine  sessions vivantes  5000 : tas + 318 Mio (65.2 Kio par session), goroutines +5000, piles +10880 Kio
corrigé  sessions vivantes  5000 : tas +   1 Mio (0.3 Kio par session), goroutines +0, piles +32 Kio
$ go tool pprof -top -sample_index=inuse_space -base heap-origine-avant.pprof heap-origine-apres.pprof
Showing nodes accounting for 322.80MB, 98.46% of 327.87MB total
      flat  flat%   sum%        cum   cum%
  322.33MB 98.31% 98.31%   322.33MB 98.31%  io.ReadAll
       1MB  0.31% 98.62%   324.33MB 98.92%  example.com/mem/sess.(*Store).Login
```

**1. La journée simulée** : une durée de vie de 2 s et 10 000 connexions sur 3 s. En régime établi, environ
5 000 sessions sont vivantes, celles des 2 dernières secondes.

**2. Les deux causes**, par session :

| Cause | Ce qui la montre | Coût par session |
|---|---|---|
| `Token: body[:32]` retient tout le corps | la différence des profils de tas : 98 % dans `io.ReadAll`, l'allocation du corps | **65 Kio** (le corps de 64 Kio) |
| `go expire(id)` : une goroutine et une minuterie par session, pendant 24 h | `NumGoroutine` (+5 000), et la mémoire des piles | **2,2 Kio** de pile (10 880 Kio pour 5 000), plus la minuterie |

La différence de profils (`-base`) isole ce qui a été **ajouté** entre deux instants. Elle désigne la ligne qui
**alloue** (`io.ReadAll`), pas celle qui **retient** (`body[:32]`). C'est la question 10 du quiz de N11 L5 :
pour savoir qui retient, on cherche qui garde une référence sur cette allocation. Ici, c'est une sous-slice
stockée dans une structure qui vit 24 h.

Les piles des goroutines n'apparaissent pas dans le profil de tas : elles se lisent dans `runtime/metrics`
(`/memory/classes/heap/stacks:bytes`) ou dans le profil des goroutines.

**3. La correction** :
- `bytes.Clone(body[:32])`, pour ne garder que 32 octets ;
- **une seule** goroutine de balayage pour toutes les échéances, au lieu d'une par session.

Résultat : 0,3 Kio par session (l'identifiant, le jeton, l'entrée de la map) et aucune goroutine
supplémentaire.

**4. En régime établi, avec 10 000 connexions par jour** (10 000 sessions vivantes, puisqu'elles durent 24 h) :

| | Mémoire | Goroutines |
|---|---|---|
| avant | environ 10 000 × 67 Kio, soit **650 Mio** | 10 000 |
| après | environ 10 000 × 0,3 Kio, soit **3 Mio** | 1 |

Un facteur 200 sur la mémoire, pour deux lignes changées. La mémoire « monte pendant un jour puis se stabilise »
parce que les sessions de 24 h finissent par expirer : ce n'est pas une fuite au sens strict, mais une
**rétention** à un niveau absurde. Le premier pic de connexions la fera dépasser la limite du conteneur
(`GOMEMLIMIT`, N11 L3).

---

## Quiz

1. Pour l'observer, vérifier une hypothèse, et prouver qu'une correction marche. Un défaut qui apparaît une fois
   sur 10 000 ne se corrige pas « à l'aveugle ».
2. Répéter (`-count`, `stress`), varier l'ordonnancement (`-cpu`, la charge), élargir la fenêtre par un point
   d'injection (`runtime.Gosched()`), ou rendre l'exécution déterministe (`synctest`, graine).
3. Une fonction, nulle en production, appelée à l'endroit soupçonné pour y provoquer un changement de
   goroutine. On la garde pour que le test de régression reste capable de reproduire le défaut.
4. Il minimise de lui-même l'entrée qui échoue (`"10-0"`), et l'enregistre comme cas de test permanent.
5. Si la slice a de la capacité libre, `append` écrit dans son tableau sans allouer. Deux appels sur la même
   base écrivent au même emplacement, et le second écrase le premier.
6. Elle référence tout le tableau d'origine, qui ne peut pas être libéré. On copie le morceau utile
   (`bytes.Clone`, `strings.Clone`).
7. Sans synchronisation, rien n'oblige une goroutine à rendre ses écritures visibles aux autres. Le compilateur
   peut donc les supprimer s'il juge qu'elles ne changent rien pour la goroutine elle-même, comme la boucle de
   l'écrivain du cours.
8. Une lecture qui voit une partie d'une valeur avant écriture et une partie après. Les interfaces (type et
   donnée), les chaînes (pointeur et longueur) et les slices (pointeur, longueur, capacité) y sont exposées.
9. Tous les accès au registre sont protégés par le verrou : il n'y a pas de course de **données**. Le défaut est
   la fenêtre entre deux sections critiques, une course **logique**.
10. Garder le test qui reproduisait le défaut, chercher les autres occurrences du même motif, et, si c'était un
    défaut de spécification, écrire la spécification.
