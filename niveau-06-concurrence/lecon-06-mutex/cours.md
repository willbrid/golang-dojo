# Leçon 6 — `Mutex` et `RWMutex`

> Les channels font circuler la propriété des données. Mais quand plusieurs goroutines doivent
> lire et modifier **le même état** — un cache, un compteur, une table de sessions —, la solution
> la plus simple est un **verrou** : une seule goroutine à la fois dans la section critique. Le
> proverbe de la leçon 2 n'interdit pas les verrous ; il dit seulement de ne pas en mettre
> partout. Cette leçon apprend à les placer **juste**.

## Objectifs

1. Protéger un état partagé avec `sync.Mutex`, et encapsuler verrou et données ensemble.
2. Choisir entre `Mutex` et `RWMutex`, en le mesurant.
3. Éviter les pièges : verrou oublié, copie, réentrance, « vérifier puis agir », données qui s'échappent.
4. Savoir quand préférer un channel, et ce qu'est `sync.Map`.

## Explication

### Le verrou

```go
var mu sync.Mutex
mu.Lock()
// section critique : une seule goroutine à la fois ici
mu.Unlock()
```

`Lock` bloque tant qu'une autre goroutine détient le verrou. Comme `Wait` et les channels, le
déverrouillage établit une relation du modèle mémoire : ce qui a été écrit avant `Unlock` est visible
par la goroutine qui obtient ensuite le verrou. La zéro-valeur d'un `Mutex` est un verrou ouvert, prêt
à l'emploi.

La forme idiomatique :

```go
mu.Lock()
defer mu.Unlock()
```

`defer` garantit le déverrouillage sur **tous** les chemins, y compris un retour anticipé ou une
panique. Un `Unlock` oublié bloque définitivement toutes les goroutines suivantes.

### Encapsuler

Le verrou n'a de sens qu'avec les données qu'il protège. La convention :

```go
type Counter struct {
	mu     sync.Mutex
	counts map[string]int // protégé par mu
}
```

- verrou et données dans la **même** struct, champs **non exportés** ;
- le verrou **juste au-dessus** des champs qu'il protège, avec un commentaire ;
- **toutes** les méthodes qui touchent les données prennent le verrou ;
- récepteurs **pointeurs** : une copie de `Counter` copierait le verrou (`go vet` : `copylocks`).

Ne **jamais** exporter le verrou ni rendre une référence vers les données protégées : une méthode
qui rend `c.counts` laisse l'appelant lire la map sans verrou pendant que d'autres l'écrivent. On rend
une **copie** (`maps.Clone`, `slices.Clone`).

### Les erreurs fatales

| Erreur | Effet |
|---|---|
| `Unlock` d'un verrou non verrouillé | `fatal error: sync: unlock of unlocked mutex` — pas une panique : **non récupérable** |
| `Lock` deux fois dans la même goroutine | interblocage : un `Mutex` Go n'est **pas réentrant** |
| copier un `Mutex` verrouillé | la copie est verrouillée pour toujours |

La non-réentrance surprend ceux qui viennent de Java : une méthode verrouillée qui appelle une autre
méthode verrouillée du même objet s'interbloque. La solution idiomatique : des méthodes **internes**
non verrouillées (`incLocked`) appelées par les méthodes publiques, qui seules prennent le verrou.

### Granularité

Une section critique doit être **courte** : chaque nanoseconde passée sous verrou est une
nanoseconde où les autres attendent. Règles :

- jamais d'E/S (fichier, réseau, `fmt.Println`) sous verrou ;
- jamais d'envoi sur un channel ni d'appel à une fonction inconnue (un *callback*) sous verrou —
  c'est la recette de l'interblocage ;
- calculer **avant**, verrouiller pour l'échange, déverrouiller.

À l'inverse, des verrous trop fins (un par champ) créent des incohérences : deux champs qui doivent
changer ensemble doivent être protégés par le **même** verrou. Un verrou protège un **invariant**,
pas une variable.

### « Vérifier puis agir »

```go
if _, ok := cache.Get(k); !ok {   // 1. verrou pris et rendu
	cache.Set(k, compute())         // 2. verrou pris et rendu
}
```

Chaque opération est sûre, la **séquence** ne l'est pas : deux goroutines peuvent toutes deux voir la
clé absente et calculer. Une séquence qui doit être atomique doit se faire sous **un seul** verrou —
ou, si le calcul est long, se faire hors verrou et être **revérifiée** sous verrou avant l'écriture
(c'est `GetOrCompute` dans l'exemple). Ce problème n'est **pas** une *data race* : `-race` ne le
voit pas. C'est une erreur de logique, à trouver en relisant.

### `RWMutex`

`sync.RWMutex` distingue lecteurs et écrivains :

- `RLock`/`RUnlock` : **plusieurs** lecteurs en même temps ;
- `Lock`/`Unlock` : un écrivain, **seul**, sans aucun lecteur.

Il ne paie que si les lectures sont **très** majoritaires **et** concurrentes. Mesuré sur la machine
de rédaction (lecture d'une map, 8 goroutines en parallèle avec `b.RunParallel`) :

| Verrou | sans concurrence | 8 lecteurs en parallèle |
|---|---|---|
| `Mutex` | ~28 ns | ~71 ns |
| `RWMutex` (`RLock`) | ~26 ns | ~45 ns |

Un gain réel sous contention de lecture, **aucun** sans concurrence — et `RWMutex` est plus complexe
à raisonner (un lecteur qui prend `RLock` deux fois peut s'interbloquer si un écrivain attend entre
les deux). Commencer par `Mutex`, passer à `RWMutex` sur **mesure**.

### `sync.Map`

`sync.Map` est une map sûre pour la concurrence, **non typée** (`any`), optimisée pour deux cas
précis : des clés écrites une fois puis lues souvent (cache qui ne fait que grandir), ou des
goroutines qui travaillent sur des ensembles de clés **disjoints**. Pour tout le reste, une `map`
ordinaire protégée par un `Mutex` est plus simple, typée, et souvent plus rapide. La documentation le
dit elle-même.

### Verrou ou channel ?

| Situation | Outil |
|---|---|
| protéger un état partagé (cache, compteur, table) | **Mutex** |
| transférer la propriété d'une donnée | **channel** |
| coordonner des étapes, signaler un événement | **channel** |
| distribuer du travail | **channel** |
| un compteur ou un drapeau isolé | `atomic` (leçon 7) |

Les deux ne s'opposent pas. Le critère de *Go Code Review Comments* : utiliser ce qui rend le code
**le plus simple**.

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : stats/stats.go
// Package stats compte des événements par clé, depuis plusieurs goroutines.
package stats

import (
	"maps"
	"sync"
)

// Counter est sûr pour un usage concurrent. Sa zéro-valeur est prête à l'emploi.
// Le verrou et les données qu'il protège sont dans la même struct, non exportés :
// aucun appelant ne peut toucher aux données sans passer par les méthodes.
type Counter struct {
	mu     sync.Mutex
	counts map[string]int // protégé par mu
}

func (c *Counter) Inc(key string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.counts == nil {
		c.counts = make(map[string]int)
	}
	c.counts[key]++
}

func (c *Counter) Get(key string) int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.counts[key]
}

// Snapshot rend une COPIE : rendre c.counts laisserait l'appelant lire la map
// sans verrou pendant que d'autres l'écrivent.
func (c *Counter) Snapshot() map[string]int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return maps.Clone(c.counts)
}

// Cache : beaucoup de lectures, peu d'écritures → RWMutex.
type Cache struct {
	mu   sync.RWMutex
	data map[string]string
}

func NewCache() *Cache { return &Cache{data: make(map[string]string)} }

func (c *Cache) Get(k string) (string, bool) {
	c.mu.RLock() // plusieurs lecteurs simultanés autorisés
	defer c.mu.RUnlock()
	v, ok := c.data[k]
	return v, ok
}

func (c *Cache) Set(k, v string) {
	c.mu.Lock() // exclusif : attend que tous les lecteurs aient fini
	defer c.mu.Unlock()
	c.data[k] = v
}

// GetOrCompute : le piège du « vérifier puis agir ». Le calcul se fait HORS du verrou
// (il peut être long), puis on revérifie sous verrou d'écriture.
func (c *Cache) GetOrCompute(k string, compute func() string) string {
	if v, ok := c.Get(k); ok {
		return v
	}
	v := compute() // sans verrou : deux goroutines peuvent calculer la même clé
	c.mu.Lock()
	defer c.mu.Unlock()
	if old, ok := c.data[k]; ok { // une autre goroutine a gagné entre-temps : garder sa valeur
		return old
	}
	c.data[k] = v
	return v
}
```

```go
// Fichier : stats/stats_test.go
package stats

import (
	"fmt"
	"sync"
	"testing"
)

func TestCounterConcurrent(t *testing.T) {
	var c Counter
	var wg sync.WaitGroup
	for g := range 50 {
		wg.Go(func() {
			for range 1000 {
				c.Inc("total")
				c.Inc(fmt.Sprint("g", g%5))
			}
		})
	}
	wg.Wait()
	if got := c.Get("total"); got != 50_000 {
		t.Errorf("total = %d, attendu 50000", got)
	}
	snap := c.Snapshot()
	snap["total"] = 0 // modifier la copie ne touche pas le compteur
	if c.Get("total") != 50_000 {
		t.Error("Snapshot a rendu la map interne au lieu d'une copie")
	}
}

func TestGetOrComputeKeepsFirst(t *testing.T) {
	c := NewCache()
	var mu sync.Mutex // protège calls : les appels à compute sont concurrents
	calls := 0
	var wg sync.WaitGroup
	results := make([]string, 20)
	for i := range 20 {
		wg.Go(func() {
			results[i] = c.GetOrCompute("k", func() string {
				mu.Lock()
				calls++
				n := calls
				mu.Unlock()
				return fmt.Sprint("valeur-", n)
			})
		})
	}
	wg.Wait()
	for _, r := range results[1:] {
		if r != results[0] {
			t.Fatalf("résultats différents : %q et %q", results[0], r)
		}
	}
	t.Logf("compute appelé %d fois pour 20 appels concurrents", calls)
}

// mutexCache : même cache avec un Mutex simple, pour comparer.
type mutexCache struct {
	mu   sync.Mutex
	data map[string]string
}

func (c *mutexCache) Get(k string) (string, bool) {
	c.mu.Lock()
	defer c.mu.Unlock()
	v, ok := c.data[k]
	return v, ok
}

func BenchmarkGetMutex(b *testing.B) {
	c := &mutexCache{data: map[string]string{"k": "v"}}
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			c.Get("k")
		}
	})
}

func BenchmarkGetRWMutex(b *testing.B) {
	c := NewCache()
	c.Set("k", "v")
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			c.Get("k")
		}
	})
}
```

Sortie de `go test -race -v ./stats` (extrait) :

```
--- PASS: TestCounterConcurrent (0.21s)
    stats_test.go:54: compute appelé 3 fois pour 20 appels concurrents
--- PASS: TestGetOrComputeKeepsFirst (0.00s)
PASS
```

Le nombre d'appels à `compute` varie d'une exécution à l'autre (de 1 à 20) : c'est le prix d'un
calcul hors verrou. Ce qui ne varie **jamais**, et que le test vérifie, c'est que les 20 appelants
obtiennent **la même** valeur.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `var c Counter` sans constructeur | Zéro-valeur utile : la map est créée paresseusement, **sous verrou**. |
| `defer c.mu.Unlock()` partout | Aucun chemin ne peut oublier de déverrouiller. |
| `Snapshot` rend `maps.Clone` | La map interne ne sort jamais du verrou ; le test le vérifie en modifiant la copie. |
| 50 goroutines × 1 000 `Inc` | `-race` silencieux et total exact : 50 000. Sans le verrou, le test échoue **et** `-race` le signale. |
| `RLock` dans `Get`, `Lock` dans `Set` | Lectures parallèles, écriture exclusive. |
| `GetOrCompute` | `compute` est appelé **hors** verrou (il peut être long), puis la clé est **revérifiée** sous verrou d'écriture : la première valeur écrite gagne, les autres calculs sont jetés. |
| Le compteur `calls` dans le test | Les appels à `compute` sont concurrents : leur compteur a besoin de **son** verrou. |
| `b.RunParallel` | Lance le corps du benchmark sur `GOMAXPROCS` goroutines : c'est ainsi qu'on mesure la contention. |

## Erreurs fréquentes

1. **Oublier de verrouiller dans une méthode** (« elle ne fait que lire »). Une lecture concurrente
   d'une écriture est une race.
2. **Rendre une référence** aux données protégées (map, slice, pointeur).
3. **Copier une struct qui contient un verrou** (récepteur valeur, passage par valeur).
4. **Verrouiller deux fois** dans la même goroutine.
5. **E/S, channel ou callback sous verrou.**
6. **« Vérifier puis agir »** en deux sections critiques séparées.
7. **`RWMutex` par défaut**, sans mesure.
8. **`sync.Map` partout** à la place d'une map typée.
9. **Deux verrous pris dans des ordres différents** par deux goroutines : interblocage (leçon 10).

## Bonnes pratiques Go

- Verrou et données dans la même struct, non exportés, commentaire « protégé par mu ».
- `Lock` + `defer Unlock` ; sections critiques courtes.
- Méthodes publiques qui verrouillent, méthodes internes `…Locked` qui supposent le verrou pris.
- Rendre des copies.
- `Mutex` par défaut ; `RWMutex` et `sync.Map` sur mesure.
- Toujours tester sous `-race`, avec beaucoup de goroutines et d'itérations.

## Ce que je dois retenir

- `Lock`/`defer Unlock` ; zéro-valeur utilisable ; jamais copié.
- Un verrou protège un **invariant** ; tout accès aux données passe par lui.
- Pas réentrant ; `Unlock` en trop = erreur fatale.
- Rien de lent ni d'inconnu sous verrou.
- « Vérifier puis agir » exige une seule section critique (ou une revérification).
- `RWMutex` : gain seulement sous forte contention de lecture.
- Verrou pour un état partagé, channel pour transférer ou coordonner.

---

➡️ [Exercices](exercices.md)
