# Leçon 7 — `atomic`, `Once`, `Pool`

> Trois outils pour trois besoins précis, tous plus spécialisés qu'un `Mutex` : un **compteur**
> ou un **pointeur** partagé sans verrou (`sync/atomic`), une initialisation **unique** et
> paresseuse (`sync.Once`, `OnceValue`), et la **réutilisation** d'objets temporaires pour
> soulager le ramasse-miettes (`sync.Pool`). Chacun est simple à utiliser, et chacun a une façon
> précise d'être mal utilisé.

## Objectifs

1. Utiliser les types atomiques (`atomic.Int64`, `atomic.Bool`, `atomic.Pointer[T]`) et savoir où
   s'arrête leur garantie.
2. Initialiser une ressource une seule fois avec `sync.OnceValue` / `OnceFunc` / `Once`.
3. Réutiliser des objets avec `sync.Pool`, et le mesurer.

## Explication

### `sync/atomic` : une opération indivisible

Une opération **atomique** s'exécute d'un seul bloc, sans qu'aucune autre goroutine puisse en voir
un état intermédiaire. `n++` n'est pas atomique (lire, ajouter, écrire : leçon 1) ; `n.Add(1)` l'est.

Depuis Go 1.19, on utilise les **types** atomiques plutôt que les anciennes fonctions sur pointeurs :

| Type | Méthodes principales |
|---|---|
| `atomic.Int32`, `Int64`, `Uint32`, `Uint64` | `Load`, `Store`, `Add`, `Swap`, `CompareAndSwap`, `And`, `Or` |
| `atomic.Bool` | `Load`, `Store`, `Swap`, `CompareAndSwap` |
| `atomic.Pointer[T]` | `Load`, `Store`, `Swap`, `CompareAndSwap` |
| `atomic.Value` | ancienne forme non typée de `Pointer`, à éviter dans le code neuf |

Leur zéro-valeur vaut 0 / false / nil, et ils ne se copient pas (`go vet` le signale). Le
modernisateur `atomictypes` de `go fix` convertit l'ancienne forme (`atomic.AddInt64(&n, 1)`).

### Ce qu'un atomique ne fait pas

Chaque opération atomique est sûre **isolément**. Deux opérations successives ne forment **pas** une
opération atomique :

```go
if n.Load() < max {   // 1
	n.Add(1)          // 2 — une autre goroutine a pu incrémenter entre 1 et 2
}
```

C'est encore « vérifier puis agir » (leçon 6). Pour ce cas précis, la boucle **CAS**
(*compare-and-swap*) :

```go
for {
	old := n.Load()
	if old >= max {
		return false
	}
	if n.CompareAndSwap(old, old+1) {   // réussit seulement si personne n'a changé n entre-temps
		return true
	}
	// sinon : quelqu'un est passé avant, on recommence
}
```

Dès qu'un invariant porte sur **plusieurs** variables, ou que la logique dépasse ce schéma, un
`Mutex` est plus clair et plus sûr. Les atomiques sont l'outil des **compteurs**, des **drapeaux** et
de la **publication de pointeurs** — pas des structures de données.

### Publier une valeur immuable : `atomic.Pointer`

Un cas d'usage remarquable : une configuration lue à chaque requête et rechargée rarement.

```go
var cfg atomic.Pointer[Config]
cfg.Store(&Config{…})          // publication
c := cfg.Load()                // lecture, sans verrou
```

La règle qui rend cela correct : **la valeur pointée n'est jamais modifiée** après publication. Pour
changer la configuration, on en construit une **nouvelle** et on publie le nouveau pointeur. Un
lecteur voit l'ancienne ou la nouvelle, jamais un mélange — et il doit faire **un seul** `Load` par
traitement, pour ne pas voir deux versions dans la même requête.

### `sync.Once` et ses descendants

Initialiser une ressource coûteuse **une seule fois**, au premier usage, même si cent goroutines la
demandent en même temps :

| Fonction | Rôle |
|---|---|
| `sync.OnceValue(f)` (Go 1.21) | rend une fonction qui appelle `f` la première fois et rend ensuite toujours **son résultat** |
| `sync.OnceValues(f)` | idem pour une fonction qui rend `(T, error)` |
| `sync.OnceFunc(f)` | idem pour une fonction sans résultat |
| `sync.Once` (`once.Do(f)`) | la forme historique, sur laquelle les trois précédentes sont construites |

```go
var loadConfig = sync.OnceValues(func() (*Config, error) {
	return parseFile("config.toml")
})

cfg, err := loadConfig()   // le fichier n'est lu qu'au premier appel
```

Garanties : `f` s'exécute une seule fois ; tous les appelants concurrents **attendent** qu'elle ait
fini ; ce qu'elle a écrit leur est visible. Attention : avec `OnceValues`, une **erreur** est aussi
mémorisée — un échec au premier appel est définitif. Si l'on veut pouvoir réessayer, `Once` n'est pas
le bon outil.

Et si `f` panique, `OnceValue`/`OnceFunc` repaniquent à chaque appel suivant (le `Once` historique,
lui, considère `f` comme exécutée et ne la relance pas).

`OnceValue` remplace avantageusement `init()` (N2 L8) pour une ressource coûteuse : elle n'est payée
que si elle sert.

### `sync.Pool` : réutiliser des objets temporaires

Un serveur qui alloue un buffer de 4 Kio par requête produit beaucoup de travail pour le
ramasse-miettes (niveau 11). `sync.Pool` garde des objets **inutilisés** pour les resservir :

```go
var bufPool = sync.Pool{New: func() any { return new(bytes.Buffer) }}

b := bufPool.Get().(*bytes.Buffer)   // un objet réutilisé, ou neuf via New
b.Reset()                             // TOUJOURS réinitialiser : il contient l'ancien contenu
defer bufPool.Put(b)
```

Règles :

- un pool est un **cache**, pas un stock : le ramasse-miettes peut **vider** le pool à tout moment.
  Ne jamais y garder quelque chose dont on a besoin (connexions, état) ;
- **réinitialiser** à la prise (ou au dépôt) — sinon des données d'une requête fuient dans la
  suivante, ce qui est aussi une faille de sécurité ;
- ne **plus utiliser** un objet après l'avoir rendu, ni rendre une référence vers son contenu
  (`b.Bytes()` partage le tableau interne ; `b.String()` copie) ;
- éviter d'y remettre des objets **devenus énormes** (un buffer qui a grossi à 100 Mo pour une
  requête exceptionnelle resterait en circulation) : tester la capacité avant `Put` ;
- `Get` rend un `any` : une assertion de type est nécessaire.

Et surtout : **mesurer**. Le gain n'existe que si l'allocation évitée est significative et fréquente.

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : svc/svc.go
// Package svc illustre les primitives fines de sync et sync/atomic.
package svc

import (
	"bytes"
	"fmt"
	"sync"
	"sync/atomic"
)

// Config est IMMUABLE une fois publiée : on la remplace en entier, on ne la modifie jamais.
type Config struct {
	Greeting string
	MaxItems int
}

// Server compte ses requêtes avec un atomique et lit sa configuration sans verrou.
type Server struct {
	requests atomic.Int64
	cfg      atomic.Pointer[Config]
}

func NewServer(c *Config) *Server {
	s := &Server{}
	s.cfg.Store(c)
	return s
}

// Reload publie une nouvelle configuration ; les lecteurs voient l'ancienne OU la nouvelle, jamais un mélange.
func (s *Server) Reload(c *Config) { s.cfg.Store(c) }

func (s *Server) Handle(name string) string {
	s.requests.Add(1)
	c := s.cfg.Load() // un seul Load par requête : toute la requête voit la même config
	return render(c.Greeting, name)
}

func (s *Server) Requests() int64 { return s.requests.Load() }

// bufPool réutilise des buffers entre requêtes pour éviter une allocation à chaque fois.
var bufPool = sync.Pool{
	New: func() any { return new(bytes.Buffer) },
}

func render(greeting, name string) string {
	b := bufPool.Get().(*bytes.Buffer)
	b.Reset() // un objet du pool garde son ancien contenu : toujours le réinitialiser
	defer bufPool.Put(b)
	fmt.Fprintf(b, "%s, %s !", greeting, name)
	return b.String() // String COPIE le contenu : on peut rendre b au pool sans risque
}

// renderNoPool : la même chose sans pool, pour comparer.
func renderNoPool(greeting, name string) string {
	var b bytes.Buffer
	fmt.Fprintf(&b, "%s, %s !", greeting, name)
	return b.String()
}

// loadDictionary est coûteux ; sync.OnceValue garantit UN seul calcul, même sous concurrence,
// et le fait paresseusement, au premier usage.
var loadDictionary = sync.OnceValue(func() map[string]bool {
	loads.Add(1)
	d := map[string]bool{}
	for _, w := range []string{"go", "rust", "zig"} {
		d[w] = true
	}
	return d
})

var loads atomic.Int32

func Known(word string) bool { return loadDictionary()[word] }
```

```go
// Fichier : svc/svc_test.go
package svc

import (
	"sync"
	"testing"
)

func TestServerConcurrent(t *testing.T) {
	s := NewServer(&Config{Greeting: "Bonjour"})
	var wg sync.WaitGroup
	for i := range 100 {
		wg.Go(func() {
			for range 100 {
				got := s.Handle("Awa")
				if got != "Bonjour, Awa !" && got != "Salut, Awa !" {
					t.Errorf("Handle = %q : configuration incohérente", got)
				}
			}
			if i == 50 {
				s.Reload(&Config{Greeting: "Salut"})
			}
		})
	}
	wg.Wait()
	if n := s.Requests(); n != 10_000 {
		t.Errorf("Requests = %d, attendu 10000", n)
	}
}

func TestKnownLoadsOnce(t *testing.T) {
	var wg sync.WaitGroup
	for range 50 {
		wg.Go(func() { Known("go") })
	}
	wg.Wait()
	if !Known("go") || Known("java") {
		t.Error("dictionnaire incorrect")
	}
	if n := loads.Load(); n != 1 {
		t.Errorf("dictionnaire chargé %d fois, attendu 1", n)
	}
}

func BenchmarkRenderPool(b *testing.B) {
	for b.Loop() {
		render("Bonjour", "Awa")
	}
}

func BenchmarkRenderNoPool(b *testing.B) {
	for b.Loop() {
		renderNoPool("Bonjour", "Awa")
	}
}

func BenchmarkCounterAtomic(b *testing.B) {
	var n atomicCounter
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			n.v.Add(1)
		}
	})
}

func BenchmarkCounterMutex(b *testing.B) {
	var n mutexCounter
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			n.mu.Lock()
			n.v++
			n.mu.Unlock()
		}
	})
}
```

```go
// Fichier : svc/bench_types_test.go
package svc

import (
	"sync"
	"sync/atomic"
)

type atomicCounter struct{ v atomic.Int64 }

type mutexCounter struct {
	mu sync.Mutex
	v  int64
}
```

`go test -race ./svc` passe. Benchmarks sur la machine de rédaction :

```
BenchmarkRenderPool-8      	 6714320	       183.6 ns/op	      48 B/op	       3 allocs/op
BenchmarkRenderNoPool-8    	 4800622	       251.2 ns/op	     160 B/op	       5 allocs/op
BenchmarkCounterAtomic-8   	62976997	        17.67 ns/op	       0 B/op	       0 allocs/op
BenchmarkCounterMutex-8    	20988498	        52.85 ns/op	       0 B/op	       0 allocs/op
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `requests atomic.Int64` | Un compteur incrémenté par toutes les requêtes : le cas d'école de l'atomique. Le test vérifie exactement 10 000. |
| `cfg atomic.Pointer[Config]` | Lecture sans verrou sur le chemin chaud ; `Reload` remplace la config **entière**. |
| `c := s.cfg.Load()` une fois | Toute la requête utilise la même version. |
| Le test vérifie deux salutations possibles | Avant ou après `Reload` ; jamais une valeur incohérente. |
| `bufPool` + `Reset` + `defer Put` | Le patron complet. `b.String()` **copie** : on peut rendre `b` en toute sécurité. |
| Pool : 3 allocations au lieu de 5, 48 o au lieu de 160 o | Le buffer n'est plus alloué ; restent les allocations de `fmt` et de la chaîne résultat. Gain ≈ 27 % sur ce cas minuscule. |
| `sync.OnceValue` | Le dictionnaire est chargé **une fois** pour 50 appelants concurrents — le test le vérifie avec un compteur… atomique. |
| Atomique ≈ 3× plus rapide que `Mutex` | Pour un compteur sous contention à 8 cœurs. Pour une struct de plusieurs champs, l'atomique n'est simplement pas applicable. |

## Erreurs fréquentes

1. **Enchaîner deux opérations atomiques** en croyant obtenir une opération atomique.
2. **Mélanger accès atomiques et non atomiques** à la même variable : c'est une race.
3. **Modifier la valeur pointée** par un `atomic.Pointer` après publication.
4. **Plusieurs `Load` dans un même traitement** : on peut voir deux versions.
5. **`OnceValues` avec une erreur transitoire** : l'échec est mémorisé pour toujours.
6. **Oublier `Reset`** sur un objet du pool : fuite de données entre requêtes.
7. **Utiliser un objet après `Put`**, ou rendre `b.Bytes()` d'un buffer remis au pool.
8. **`sync.Pool` comme pool de connexions** : il peut être vidé à tout moment.
9. **Un pool sans mesure** : complexité sans gain.

## Bonnes pratiques Go

- Types atomiques (`atomic.Int64`…) plutôt que fonctions ; jamais copiés.
- Atomiques pour compteurs, drapeaux et pointeurs vers des valeurs immuables ; `Mutex` pour le reste.
- `sync.OnceValue(s)` pour l'initialisation paresseuse d'une ressource coûteuse.
- `sync.Pool` seulement sur un chemin chaud, mesuré, avec `Reset` systématique.

## Ce que je dois retenir

- Une opération atomique est indivisible ; deux opérations atomiques ne le sont pas ensemble.
- `CompareAndSwap` en boucle pour une mise à jour conditionnelle simple.
- `atomic.Pointer[T]` : publier des valeurs **immuables**, remplacées en entier.
- `OnceValue` / `OnceValues` / `OnceFunc` : une exécution, paresseuse, visible de tous ; l'erreur est mémorisée.
- `sync.Pool` : un cache d'objets temporaires, vidable à tout moment ; `Reset` et mesure.

---

➡️ [Exercices](exercices.md)
