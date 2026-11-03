# Module 1 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé. Pour ce niveau, le corrigé donne **une** conception argumentée ;
> d'autres sont défendables.

## E1 et E2 — Fermer le pool, et une durée de vie maximale

Les deux exercices modifient l'état du pool. Voici la version complète, qui remplace celle du cours (dont les
tests passent toujours) :

```go
// Fichier : pool/pool.go
// Package pool : un pool borné de ressources, conçu à partir de ses invariants (version du corrigé :
// fermeture et durée de vie maximale).
//
// État, protégé par mu :
//
//	open, creating, lent, idle, waiters : comme dans le cours
//	closed   : Close a été appelé
//	born     : date de création de chaque ressource ouverte (pour MaxLifetime)
//
// Invariants, vrais chaque fois que mu est libre :
//
//	I1  open == len(idle) + lent + creating
//	I2  open <= max
//	I3  len(waiters) > 0  ⇒  len(idle) == 0
//	I4  une ressource prêtée l'est à un seul appelant à la fois
//	I5  (extérieur) lent == nombre d'appelants qui tiennent une ressource
//	C1  closed  ⇒  len(idle) == 0 && len(waiters) == 0
//	L1  aucune ressource d'idle n'a plus de MaxLifetime (vérifié à la sortie d'idle, voir Get)
//	B1  len(born) == open - creating            (une date par ressource ouverte, ni plus ni moins)
package pool

import (
	"context"
	"errors"
	"sync"
	"time"
)

var ErrClosed = errors.New("pool fermé")

type entry[T comparable] struct {
	r    T
	born time.Time
}

type Pool[T comparable] struct {
	mu       sync.Mutex
	max      int
	open     int
	creating int
	lent     int
	idle     []entry[T]
	waiters  []chan entry[T]
	closed   bool
	drained  chan struct{}   // fermé quand open atteint 0 après Close
	born     map[T]time.Time // T comparable : la ressource sert de clé
	create   func(context.Context) (T, error)
	destroy  func(T)

	MaxLifetime time.Duration    // 0 : illimitée
	Now         func() time.Time // injectable pour les tests
	BuggyCancel bool
}

func New[T comparable](max int, create func(context.Context) (T, error)) *Pool[T] {
	return &Pool[T]{max: max, create: create, destroy: func(T) {}, Now: time.Now,
		drained: make(chan struct{}), born: map[T]time.Time{}}
}

// WithDestroy règle la fonction qui libère une ressource (fermer une connexion…).
func (p *Pool[T]) WithDestroy(f func(T)) *Pool[T] { p.destroy = f; return p }

func (p *Pool[T]) expired(e entry[T]) bool {
	return p.MaxLifetime > 0 && p.Now().Sub(e.born) >= p.MaxLifetime
}

// destroyLocked retire une ressource du compte ; la destruction elle-même se fait hors du verrou.
func (p *Pool[T]) destroyLocked(e entry[T], later *[]T) {
	p.open--
	delete(p.born, e.r) // sinon la map garderait une clé par ressource détruite
	*later = append(*later, e.r)
	if p.closed && p.open == 0 {
		close(p.drained)
	}
}

func (p *Pool[T]) Get(ctx context.Context) (T, error) {
	var zero T
	var trash []T
	defer func() { // destruction hors du verrou (elle peut être lente)
		for _, r := range trash {
			p.destroy(r)
		}
	}()
	p.mu.Lock()
	if p.closed {
		p.mu.Unlock()
		return zero, ErrClosed
	}
	for len(p.idle) > 0 {
		e := p.idle[len(p.idle)-1]
		p.idle = p.idle[:len(p.idle)-1]
		if p.expired(e) { // L1 : trop vieille, on la détruit et on regarde la suivante
			p.destroyLocked(e, &trash)
			continue
		}
		p.lent++
		p.mu.Unlock()
		return e.r, nil
	}
	if p.open < p.max {
		p.open++
		p.creating++
		p.mu.Unlock()
		r, err := p.create(ctx)
		p.mu.Lock()
		p.creating--
		if err != nil {
			p.open--
			if p.closed && p.open == 0 {
				close(p.drained)
			}
			p.mu.Unlock()
			return zero, err
		}
		if p.closed { // fermé pendant la création : on ne la prête pas
			p.destroyLocked(entry[T]{r: r}, &trash)
			p.mu.Unlock()
			return zero, ErrClosed
		}
		p.born[r] = p.Now()
		p.lent++
		p.mu.Unlock()
		return r, nil
	}
	w := make(chan entry[T], 1)
	p.waiters = append(p.waiters, w)
	p.mu.Unlock()
	select {
	case e, ok := <-w:
		if !ok {
			return zero, ErrClosed // Close a fermé notre canal
		}
		return e.r, nil
	case <-ctx.Done():
	}
	if p.BuggyCancel {
		return zero, ctx.Err() // l'erreur du cours, conservée pour ses tests
	}
	p.mu.Lock()
	defer p.mu.Unlock()
	for i, x := range p.waiters {
		if x == w {
			p.waiters = append(p.waiters[:i], p.waiters[i+1:]...)
			return zero, ctx.Err()
		}
	}
	if e, ok := <-w; ok { // servis pendant l'annulation : on rend la ressource
		p.putLocked(e, &trash)
	}
	return zero, ctx.Err()
}

func (p *Pool[T]) Put(r T) {
	var trash []T
	p.mu.Lock()
	p.putLocked(entry[T]{r: r, born: p.born[r]}, &trash)
	p.mu.Unlock()
	for _, r := range trash {
		p.destroy(r)
	}
}

func (p *Pool[T]) putLocked(e entry[T], trash *[]T) {
	p.lent--
	if p.closed || p.expired(e) { // après Close, ou trop vieille : détruite, jamais reprêtée
		p.destroyLocked(e, trash)
		return
	}
	if len(p.waiters) > 0 {
		w := p.waiters[0]
		p.waiters = p.waiters[1:]
		p.lent++
		w <- e
		return
	}
	p.idle = append(p.idle, e)
}

// Close refuse tout nouveau Get, réveille les attendants (ErrClosed), détruit les ressources libres,
// puis attend que les ressources prêtées soient rendues (et détruites), ou l'annulation de ctx.
func (p *Pool[T]) Close(ctx context.Context) error {
	var trash []T
	p.mu.Lock()
	if !p.closed {
		p.closed = true
		for _, w := range p.waiters {
			close(w)
		}
		p.waiters = nil
		for _, e := range p.idle {
			p.destroyLocked(e, &trash)
		}
		p.idle = nil
		if p.open == 0 {
			close(p.drained)
		}
	}
	p.mu.Unlock()
	for _, r := range trash {
		p.destroy(r)
	}
	select {
	case <-p.drained: // open == 0 : tout a été détruit
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

var ErrInvariant = errors.New("invariant violé")

func (p *Pool[T]) Check() error {
	p.mu.Lock()
	defer p.mu.Unlock()
	switch {
	case p.open != len(p.idle)+p.lent+p.creating:
		return errors.Join(ErrInvariant, errors.New("I1"))
	case p.open > p.max:
		return errors.Join(ErrInvariant, errors.New("I2"))
	case len(p.waiters) > 0 && len(p.idle) > 0:
		return errors.Join(ErrInvariant, errors.New("I3"))
	case p.closed && (len(p.idle) > 0 || len(p.waiters) > 0):
		return errors.Join(ErrInvariant, errors.New("C1"))
	case len(p.born) != p.open-p.creating:
		return errors.Join(ErrInvariant, errors.New("B1"))
	}
	return nil
}

func (p *Pool[T]) Stats() (open, idle, lent, waiting int) {
	p.mu.Lock()
	defer p.mu.Unlock()
	return p.open, len(p.idle), p.lent, len(p.waiters)
}
```

```go
// Fichier : pool/close_test.go
package pool

import (
	"context"
	"errors"
	"sync"
	"sync/atomic"
	"testing"
	"time"
)

// E1 : Close réveille les attendants, détruit les libres, attend les prêtées.
func TestClose(t *testing.T) {
	var made, destroyed atomic.Int64
	p := New(2, func(context.Context) (*res, error) { made.Add(1); return &res{}, nil }).
		WithDestroy(func(*res) { destroyed.Add(1) })
	ctx := context.Background()
	a, _ := p.Get(ctx)
	b, _ := p.Get(ctx)
	p.Put(b)           // b est libre, a est prêtée
	c, _ := p.Get(ctx) // reprend b
	var wg sync.WaitGroup
	var waitErr error
	wg.Go(func() { _, waitErr = p.Get(ctx) }) // attend : le pool est plein
	time.Sleep(20 * time.Millisecond)

	closed := make(chan error)
	go func() { closed <- p.Close(ctx) }()
	wg.Wait()
	if !errors.Is(waitErr, ErrClosed) {
		t.Fatalf("attendant : %v", waitErr)
	}
	if _, err := p.Get(ctx); !errors.Is(err, ErrClosed) {
		t.Fatalf("Get après Close : %v", err)
	}
	if err := p.Check(); err != nil {
		t.Fatal(err)
	}
	select {
	case <-closed:
		t.Fatal("Close a rendu la main avant le retour des ressources prêtées")
	case <-time.After(20 * time.Millisecond):
	}
	p.Put(a)
	p.Put(c)
	if err := <-closed; err != nil {
		t.Fatal(err)
	}
	if made.Load() != destroyed.Load() {
		t.Fatalf("créées %d, détruites %d", made.Load(), destroyed.Load())
	}
}

// E2 : une ressource trop vieille n'est plus prêtée.
func TestMaxLifetime(t *testing.T) {
	now := time.Date(2026, 3, 1, 10, 0, 0, 0, time.UTC)
	var destroyed atomic.Int64
	var n atomic.Int64
	p := New(1, func(context.Context) (*res, error) { return &res{id: int(n.Add(1))}, nil }).
		WithDestroy(func(*res) { destroyed.Add(1) })
	p.MaxLifetime = time.Minute
	p.Now = func() time.Time { return now }
	ctx := context.Background()
	r1, _ := p.Get(ctx)
	p.Put(r1)
	now = now.Add(30 * time.Second)
	r2, _ := p.Get(ctx)
	if r2 != r1 {
		t.Fatal("ressource encore jeune non réutilisée")
	}
	now = now.Add(time.Minute) // r1 a maintenant 90 s
	p.Put(r2)                  // détruite au retour
	r3, _ := p.Get(ctx)
	if r3 == r1 || destroyed.Load() != 1 || r3.id != 2 {
		t.Fatalf("r3=%d détruites=%d", r3.id, destroyed.Load())
	}
	if err := p.Check(); err != nil {
		t.Fatal(err)
	}
}

// La map des dates ne grossit pas : une entrée par ressource OUVERTE.
func TestBornMapBounded(t *testing.T) {
	now := time.Now()
	var n atomic.Int64
	p := New(2, func(context.Context) (*res, error) { return &res{id: int(n.Add(1))}, nil })
	p.MaxLifetime = time.Second
	p.Now = func() time.Time { return now }
	for range 100 {
		r, _ := p.Get(context.Background())
		now = now.Add(2 * time.Second) // chaque ressource expire avant d'être rendue
		p.Put(r)
	}
	if open, _, _, _ := p.Stats(); len(p.born) != open {
		t.Fatalf("born : %d entrées pour %d ressources ouvertes (%d créées)", len(p.born), open, n.Load())
	}
}

// Une ressource qui expire PENDANT qu'elle est libre est détruite par Get (autre chemin que Put).
func TestExpiredWhileIdle(t *testing.T) {
	now := time.Now()
	var n atomic.Int64
	p := New(2, func(context.Context) (*res, error) { return &res{id: int(n.Add(1))}, nil })
	p.MaxLifetime = time.Second
	p.Now = func() time.Time { return now }
	r, _ := p.Get(context.Background())
	p.Put(r)                   // libre, encore jeune
	now = now.Add(time.Minute) // elle expire dans idle
	r2, _ := p.Get(context.Background())
	if r2 == r {
		t.Fatal("ressource expirée reprêtée")
	}
	if err := p.Check(); err != nil {
		t.Fatal(err) // B1 détecte une date oubliée pour la ressource détruite
	}
}
```

```
$ go test -race -v -run 'Close|Lifetime|Born|Expired' ./pool/
--- PASS: TestClose (0.04s)
--- PASS: TestMaxLifetime (0.00s)
--- PASS: TestBornMapBounded (0.00s)
--- PASS: TestExpiredWhileIdle (0.00s)
```

**E1, les invariants après `Close`.** I1 reste vrai, et c'est lui qui permet de savoir quand tout est
détruit. Après `Close`, `idle` est vide (les ressources libres sont détruites, `open` diminue d'autant), et
chaque `Put` détruit la ressource rendue. Le nouvel invariant C1 (« fermé ⇒ `idle` et `waiters` vides »)
remplace I3. **`Close` peut rendre la main quand `open == 0`** : toutes les ressources créées ont été
détruites.

Il ne peut pas le savoir plus tôt. Une ressource prêtée est entre les mains d'un appelant, et le pool ne peut
pas la lui reprendre. D'où le canal `drained`, fermé par la dernière destruction, et le `ctx` de `Close`
pour ne pas attendre indéfiniment un appelant qui ne rend jamais sa ressource.

Trois points de conception :

- **Les attendants sont réveillés** par la **fermeture** de leur canal (`close(w)`). La réception rend alors
  `ok == false`, et `Get` renvoie `ErrClosed`. C'est un signal à tous, sans envoyer de valeur.
- **Une création en cours** pendant `Close` se termine hors du verrou. Au retour, on constate `closed`, et on
  détruit la ressource au lieu de la prêter. Sans ce cas, `Close` attendrait une ressource prêtée après lui.
- **La destruction se fait hors du verrou** (la liste `trash`) : fermer une connexion peut prendre du temps,
  et le pool ne doit pas se figer pendant ce temps.

**E2, la durée de vie.** La date de création vit dans `born`, une map indexée par la ressource : c'est
pourquoi `T` est maintenant `comparable`. La ressource trop vieille est détruite à **deux** endroits :
- par `Get`, qui la trouve dans `idle` (elle a vieilli en attendant) ;
- par `Put`, qui la reçoit (elle a vieilli pendant le prêt).

Une goroutine de nettoyage n'est pas nécessaire. Une ressource trop vieille et libre ne gêne personne tant
qu'on ne la prête pas. Elle occupe seulement une connexion, ce qui peut justifier un nettoyage périodique si
le pool est souvent au repos.

L'invariant L1 (« jamais une ressource trop vieille **prêtée** ») se vérifie **à la sortie d'`idle`**, pas en
continu. Une ressource libre devient trop vieille par le seul passage du temps, sans qu'aucune opération ne
s'exécute.

**Un défaut trouvé en écrivant ce corrigé.** La première version retirait la date de `born` seulement dans
`Put`. Une ressource détruite par `Get` (expirée en attendant) ou par `Close` gardait sa date : la map
grossissait sans fin. Les tests de E1 et E2 passaient quand même. L'écriture d'un invariant de plus, B1
(« une date par ressource ouverte »), suivie d'un test qui passe par le chemin « expirée dans `idle` »
(`TestExpiredWhileIdle`), l'a fait apparaître : `invariant violé`. La correction consiste à placer le
`delete` dans `destroyLocked`, le seul point de destruction. Chaque structure annexe ajoutée à l'état appelle
son invariant.

## E3 — Le pool en canaux seulement

```go
// Fichier : chanpool/chanpool.go
// Package chanpool : le pool le plus court, un canal bufferisé de max ressources (E3).
package chanpool

import "context"

type Pool[T any] struct{ ch chan T }

// New crée les max ressources d'avance : un canal ne sait pas « créer à la demande » sans état en plus.
func New[T any](max int, create func() T) *Pool[T] {
	p := &Pool[T]{ch: make(chan T, max)}
	for range max {
		p.ch <- create()
	}
	return p
}

func (p *Pool[T]) Get(ctx context.Context) (T, error) {
	select {
	case r := <-p.ch:
		return r, nil
	case <-ctx.Done():
		var zero T
		return zero, ctx.Err() // rien n'a pu être reçu : pas de remise croisée possible
	}
}

func (p *Pool[T]) Put(r T) { p.ch <- r }
```

```go
// Fichier : chanpool/fair_test.go
package chanpool

import (
	"context"
	"fmt"
	"slices"
	"sync"
	"testing"
	"time"

	"example.com/pool/pool"
)

type getter interface {
	get(ctx context.Context) (int, error)
	put(int)
}

type chanG struct{ p *Pool[int] }

func (g chanG) get(ctx context.Context) (int, error) { return g.p.Get(ctx) }
func (g chanG) put(r int)                            { g.p.Put(r) }

type mutexG struct{ p *pool.Pool[*int] }

func (g mutexG) get(ctx context.Context) (int, error) { r, err := g.p.Get(ctx); return *r, err }
func (g mutexG) put(r int)                            { x := r; g.p.Put(&x) }

// waits : 64 emprunteurs, 4 ressources, 2 s ; renvoie les temps d'attente triés.
func waits(g getter) []time.Duration {
	var mu sync.Mutex
	var all []time.Duration
	stop := time.Now().Add(2 * time.Second)
	var wg sync.WaitGroup
	for range 64 {
		wg.Go(func() {
			var local []time.Duration
			for time.Now().Before(stop) {
				t0 := time.Now()
				r, err := g.get(context.Background())
				if err != nil {
					continue
				}
				local = append(local, time.Since(t0))
				time.Sleep(200 * time.Microsecond) // le travail
				g.put(r)
			}
			mu.Lock()
			all = append(all, local...)
			mu.Unlock()
		})
	}
	wg.Wait()
	slices.Sort(all)
	return all
}

func pct(d []time.Duration, p float64) time.Duration { return d[int(p*float64(len(d)-1))] }

func TestFairness(t *testing.T) {
	cp := New(4, func() int { return 0 })
	n := 0
	mp := pool.New(4, func(context.Context) (*int, error) { n++; x := n; return &x, nil })
	for _, tc := range []struct {
		name string
		g    getter
	}{{"canal", chanG{cp}}, {"verrou + file FIFO", mutexG{mp}}} {
		d := waits(tc.g)
		fmt.Printf("%-20s %6d prêts  p50 %-9v p99 %-9v max %v\n", tc.name, len(d),
			pct(d, 0.5).Round(10*time.Microsecond), pct(d, 0.99).Round(10*time.Microsecond), d[len(d)-1].Round(10*time.Microsecond))
	}
}
```

```
$ go test -v -run Fairness ./chanpool/      # 64 emprunteurs, 4 ressources, 2 s, deux exécutions
canal                  7108 prêts  p50 16.98ms   p99 20.87ms   max 23.08ms
verrou + file FIFO     7031 prêts  p50 17.26ms   p99 18.39ms   max 20.86ms
canal                  7138 prêts  p50 17.11ms   p99 17.88ms   max 18.31ms
verrou + file FIFO     7035 prêts  p50 17.31ms   p99 18.46ms   max 19.3ms
```

**Ce que le pool en canaux garantit :**

- **I2 et I4** : le canal contient au plus `max` ressources, et une réception la donne à un seul receveur.
- **I1 se lit autrement** : `prêtées = max − len(ch)`.
- **I5** aussi, à condition que chaque `Put` corresponde à un `Get`. Mais il n'a pas de compteur pour le
  vérifier.
- **L'annulation est simple** : un `select` qui sort par `ctx.Done()` n'a **rien** reçu. La remise croisée du
  cours n'existe pas. C'est un vrai avantage de ce modèle.

**Ce qu'il ne garantit pas :** il ne crée pas à la demande (les `max` ressources existent dès `New`), il n'a
ni fermeture ni durée de vie sans état supplémentaire, et **P2 (l'équité) n'est pas promis** par la
spécification du langage.

**La mesure contredit l'intuition** : les deux pools sont aussi équitables l'un que l'autre. p99 de 18 à 21 ms
contre 18 ms, maximums voisins, pour une attente médiane de 17 ms. Les receveurs bloqués sur un canal sont,
dans l'implémentation actuelle du *runtime*, servis dans l'ordre d'arrivée. C'est un détail
d'implémentation, et non une garantie : un composant qui **doit** être équitable l'écrit lui-même, comme le
pool du cours, et le teste (E5).

## E4 — Trouver le défaut

```go
// Fichier : latch/latch.go
// Package latch : le loquet de E4, sa version fautive et sa correction.
//
// Spécification : Wait ne rend la main que pendant que le loquet est OUVERT (sémantique « de niveau »).
// Invariant au retour de Wait : open == true au moment où Wait libère le verrou.
package latch

import (
	"sync"
	"sync/atomic"
)

type Latch struct {
	mu   sync.Mutex
	cond *sync.Cond
	open bool

	Fixed          bool         // false : la version de l'énoncé (if) ; true : la correction (for)
	ReturnedClosed atomic.Int64 // instrumentation de test : retours de Wait avec le loquet fermé
}

func New() *Latch { l := &Latch{}; l.cond = sync.NewCond(&l.mu); return l }

func (l *Latch) Wait() {
	l.mu.Lock()
	if l.Fixed {
		for !l.open { // on revérifie la condition à CHAQUE réveil
			l.cond.Wait()
		}
	} else if !l.open {
		l.cond.Wait() // réveillé… mais la condition peut être redevenue fausse
	}
	if !l.open {
		l.ReturnedClosed.Add(1) // l'invariant est violé
	}
	l.mu.Unlock()
}

func (l *Latch) Open()  { l.mu.Lock(); l.open = true; l.cond.Broadcast(); l.mu.Unlock() }
func (l *Latch) Reset() { l.mu.Lock(); l.open = false; l.mu.Unlock() }
```

```go
// Fichier : latch/latch_test.go
package latch

import (
	"fmt"
	"testing"
	"time"
)

// Open puis Reset aussitôt : l'attendant réveillé par Broadcast reprend le verrou APRÈS Reset.
func TestLatch(t *testing.T) {
	for _, fixed := range []bool{false, true} {
		var violations, blocked int
		for range 1000 {
			l := New()
			l.Fixed = fixed
			done := make(chan struct{})
			go func() { l.Wait(); close(done) }()
			time.Sleep(50 * time.Microsecond) // l'attendant est (probablement) bloqué dans cond.Wait
			l.Open()
			l.Reset()
			select {
			case <-done:
			case <-time.After(5 * time.Millisecond):
				blocked++ // attend toujours (correct pour la version corrigée : le loquet est fermé)
				l.Open()  // pour libérer la goroutine
				<-done
			}
			violations += int(l.ReturnedClosed.Load())
		}
		fmt.Printf("corrigé=%-5v  retours avec le loquet fermé : %4d / 1000   toujours en attente après Reset : %4d\n",
			fixed, violations, blocked)
	}
}
```

```
$ go test -race -v ./latch/      # deux exécutions
corrigé=false  retours avec le loquet fermé :  996 / 1000   toujours en attente après Reset :    3
corrigé=true   retours avec le loquet fermé :    0 / 1000   toujours en attente après Reset :  996
corrigé=false  retours avec le loquet fermé : 1000 / 1000   toujours en attente après Reset :    0
corrigé=true   retours avec le loquet fermé :    0 / 1000   toujours en attente après Reset : 1000
```

**L'invariant au retour de `Wait`** : le loquet est ouvert au moment où `Wait` libère le verrou.

**L'entrelacement** :
1. `W` appelle `Wait`, trouve le loquet fermé, et s'endort dans `cond.Wait` (le verrou est relâché) ;
2. `Open` passe `open` à vrai et diffuse (`Broadcast`) : `W` est réveillé, mais doit **reprendre le verrou**
   avant de revenir de `cond.Wait` ;
3. `Reset` prend le verrou **avant** `W`, et repasse `open` à faux ;
4. `W` reprend le verrou, sort du `if` sans revérifier, et rend la main **avec le loquet fermé**.

La mesure montre que ce n'est pas rare, mais presque systématique : 996 à 1 000 fois sur 1 000. La goroutine
qui a appelé `Open` enchaîne avec `Reset` bien avant que `W` ne soit ordonnancée.

**La correction** : `for !l.open { l.cond.Wait() }`. Un réveil ne garantit pas que la condition est vraie, il
dit seulement qu'elle a **pu** changer. C'est la règle d'usage de `sync.Cond`, écrite dans sa documentation.
`-race` ne dit rien : tous les accès à `open` sont sous le verrou.

Avec la correction, `W` reste en attente (996 et 1 000 fois) : le loquet est fermé, c'est le comportement
attendu par la spécification « de niveau ».

Si l'on voulait au contraire que `W` soit libéré par toute ouverture survenue pendant son attente, même brève
(une sémantique « d'événement »), il faudrait une **génération** : un compteur incrémenté par `Open`, et
`Wait` qui attend tant que la génération n'a pas changé. Le choix entre les deux est une décision de
**spécification**, à écrire avant le code.

## E5 — Vérifier l'équité

```go
// Fichier : pool/fifo_test.go
package pool

import (
	"context"
	"fmt"
	"slices"
	"testing"
	"time"
)

// E5 : les attendants sont servis dans leur ordre d'arrivée (P2), et un annulé est sauté sans
// perturber les autres. On sert UN attendant à la fois, pour que l'ordre observé soit celui des remises.
func TestFIFO(t *testing.T) {
	p := newPool(false)
	ctx := context.Background()
	var held []*res
	for range 4 { // le pool est plein
		r, _ := p.Get(ctx)
		held = append(held, r)
	}
	type got struct {
		i int
		r *res
	}
	servedCh := make(chan got)
	const n = 8
	cancels := make([]context.CancelFunc, n)
	for i := range n {
		cctx, cancel := context.WithCancel(ctx)
		cancels[i] = cancel
		go func() {
			if r, err := p.Get(cctx); err == nil {
				servedCh <- got{i, r} // on rend la ressource au test, qui décide du Put suivant
			}
		}()
		for { // attendre que l'attendant i soit dans la file : l'ordre d'arrivée est fixé
			if _, _, _, w := p.Stats(); w == i+1 {
				break
			}
			time.Sleep(time.Millisecond)
		}
	}
	cancels[3]() // l'attendant 3 renonce
	for {
		if _, _, _, w := p.Stats(); w == n-1 {
			break
		}
		time.Sleep(time.Millisecond)
	}
	var served []int
	next := held[0]
	pending := held[1:]
	for range n - 1 {
		p.Put(next) // sert le plus ancien attendant
		g := <-servedCh
		served = append(served, g.i)
		pending = append(pending, g.r)
		next, pending = pending[0], pending[1:]
	}
	p.Put(next)
	for _, r := range pending {
		p.Put(r)
	}
	fmt.Println("ordre de service :", served)
	if want := []int{0, 1, 2, 4, 5, 6, 7}; !slices.Equal(served, want) {
		t.Fatalf("servis %v, attendu %v", served, want)
	}
	if err := p.Check(); err != nil {
		t.Fatal(err)
	}
	if open, idle, lent, _ := p.Stats(); lent != 0 || open != idle {
		t.Fatalf("au repos : open=%d idle=%d lent=%d", open, idle, lent)
	}
}
```

```
$ go test -race -v -run FIFO ./pool/        # trois exécutions
ordre de service : [0 1 2 4 5 6 7]
ordre de service : [0 1 2 4 5 6 7]
ordre de service : [0 1 2 4 5 6 7]
```

L'attendant 3, annulé, est **retiré** de la file sous le verrou, et les autres sont servis dans leur ordre
d'arrivée. P2 tient, avec l'exception prévue pour l'annulation.

**Le premier essai de ce test était faux.** Il rendait les quatre ressources d'un coup, et notait l'ordre dans
lequel les goroutines servies ajoutaient leur numéro à la liste. Il obtenait `[4 2 1 5 0 7 6]`, puis
`[2 4 6 5 7 1 0]` : quatre attendants servis **en même temps** s'ordonnancent dans n'importe quel ordre. Le
pool n'était pas en cause, c'est l'observation qui ne mesurait pas ce qu'elle croyait mesurer. La version
correcte sert **un** attendant à la fois. Chaque ressource revient au test, qui décide du `Put` suivant, et
l'ordre observé est alors celui des remises.

Pour tester un ordre, il faut que l'ordre observé soit causé par celui qu'on veut vérifier, et par rien
d'autre.

---

## Exercice intermédiaire — Un regroupeur borné

```go
// Fichier : batch/batch.go
// Package batch : un regroupeur borné (exercice intermédiaire).
//
// État :
//
//	in      : file bornée des éléments acceptés et pas encore pris par la boucle (capacité Capacity)
//	pending : le lot en construction, dans la boucle (une seule goroutine : pas de verrou)
//	closed  : Close a commencé ; protégé par mu
//
// Invariants et propriétés :
//
//	E1 (extérieur)  chaque élément pour lequel Add a rendu nil est passé à flush avec succès
//	                exactement une fois, sauf si Close échoue (son ctx expire), qui le signale
//	E2  len(pending) <= BatchSize ; aucun appel à flush n'en chevauche un autre
//	P1  un élément accepté part au plus Interval après le début de son lot, si flush réussit
//	P2  Add bloque au plus jusqu'à ce que la file ait de la place, ctx soit annulé, ou Close
package batch

import (
	"context"
	"errors"
	"sync"
	"sync/atomic"
	"time"
)

var ErrClosed = errors.New("regroupeur fermé")

type Batcher[T any] struct {
	in        chan T
	flush     func(context.Context, []T) error
	size      int
	interval  time.Duration
	retryWait time.Duration

	mu       sync.RWMutex // Add : RLock pendant l'envoi ; Close : Lock pour basculer closed
	closed   bool
	closing  atomic.Bool     // un seul Close
	stopAdds chan struct{}   // fermé en premier par Close : réveille les Add bloqués
	quit     chan struct{}   // fermé ensuite, quand plus aucun Add n'est en cours : la boucle vide
	closeCtx context.Context // écrit par Close AVANT close(quit) : la fermeture le publie à la boucle
	done     chan error      // erreur finale de la boucle

	// BuggyClose : Close n'attend pas les Add en cours (voir le corrigé).
	BuggyClose bool
}

func New[T any](capacity, batchSize int, interval time.Duration, flush func(context.Context, []T) error) *Batcher[T] {
	b := &Batcher[T]{in: make(chan T, capacity), flush: flush, size: batchSize, interval: interval,
		retryWait: time.Millisecond, stopAdds: make(chan struct{}), quit: make(chan struct{}), done: make(chan error, 1)}
	go b.loop()
	return b
}

// Add accepte un élément, ou bloque tant que la file est pleine.
func (b *Batcher[T]) Add(ctx context.Context, x T) error {
	if !b.BuggyClose {
		// Tenir RLock pendant l'envoi : Close (Lock) attend que les Add en cours aient fini, et
		// aucun Add ne peut plus réussir une fois closed vrai. La boucle continue de vider in
		// pendant ce temps, donc un Add bloqué finit toujours par passer.
		b.mu.RLock()
		defer b.mu.RUnlock()
		if b.closed {
			return ErrClosed
		}
	}
	select {
	case b.in <- x:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	case <-b.stopAdds: // Close a commencé : on renonce (la boucle n'a pas encore vidé in)
		return ErrClosed
	}
}

func (b *Batcher[T]) loop() {
	var pending []T
	var timer <-chan time.Time
	send := func(ctx context.Context) error {
		for len(pending) > 0 {
			n := min(len(pending), b.size)
			for { // réessayer jusqu'au succès : E1 « exactement une fois » côté regroupeur
				err := b.flush(ctx, pending[:n:n])
				if err == nil {
					break
				}
				select {
				case <-time.After(b.retryWait):
				case <-ctx.Done():
					return errors.Join(ctx.Err(), err)
				}
			}
			pending = pending[n:]
		}
		pending = nil
		timer = nil
		return nil
	}
	for {
		select {
		case x := <-b.in:
			if len(pending) == 0 {
				timer = time.After(b.interval)
			}
			pending = append(pending, x)
			if len(pending) == b.size {
				send(context.Background())
			}
		case <-timer:
			send(context.Background())
		case <-b.quit:
			for { // vider ce qui reste dans in : plus aucun Add ne peut y ajouter (voir Close)
				select {
				case x := <-b.in:
					pending = append(pending, x)
					continue
				default:
				}
				break
			}
			b.done <- send(b.closeCtx)
			return
		}
	}
}

// Close cesse d'accepter, écrit tout ce qui a été accepté, et rend la main.
//
// L'ordre compte : (1) réveiller les Add bloqués ; (2) attendre qu'aucun Add ne soit en cours
// (Lock) et interdire les suivants ; (3) seulement alors, demander à la boucle de vider in. Un Add
// ne peut donc plus déposer un élément après le vidage : E1 tient.
func (b *Batcher[T]) Close(ctx context.Context) error {
	if !b.closing.CompareAndSwap(false, true) {
		return ErrClosed
	}
	if b.BuggyClose {
		b.closeCtx = ctx
		close(b.stopAdds)
		close(b.quit) // ERREUR : un Add peut encore réussir son envoi après le vidage de la boucle
		return <-b.done
	}
	close(b.stopAdds) // (1)
	b.mu.Lock()       // (2) les Add réveillés relâchent RLock
	b.closed = true
	b.mu.Unlock()
	b.closeCtx = ctx
	close(b.quit) // (3)
	return <-b.done
}
```

```go
// Fichier : batch/batch_test.go
package batch

import (
	"context"
	"errors"
	"fmt"
	"math/rand/v2"
	"sync"
	"sync/atomic"
	"testing"
	"testing/synctest"
	"time"
)

// sink enregistre ce que flush a écrit avec succès ; il échoue au hasard (une fois sur cinq).
type sink struct {
	mu      sync.Mutex
	got     map[int]int // élément → nombre d'écritures réussies
	calls   atomic.Int32
	overlap atomic.Int32 // E2 : appels simultanés
	maxLen  int
}

func (s *sink) flush(_ context.Context, xs []int) error {
	if s.calls.Add(1) > 1 {
		s.overlap.Add(1)
	}
	defer s.calls.Add(-1)
	if rand.IntN(5) == 0 {
		return errors.New("échec passager")
	}
	s.mu.Lock()
	defer s.mu.Unlock()
	s.maxLen = max(s.maxLen, len(xs))
	for _, x := range xs {
		s.got[x]++
	}
	return nil
}

// stress : 16 producteurs ajoutent des éléments distincts, puis Close pendant qu'ils tournent encore.
func stress(t *testing.T, buggy bool) (accepted, written, dup, lost int) {
	s := &sink{got: map[int]int{}}
	b := New(64, 32, 2*time.Millisecond, s.flush)
	b.BuggyClose = buggy
	var acc sync.Map
	var wg sync.WaitGroup
	var next atomic.Int64
	for range 16 {
		wg.Go(func() {
			for {
				x := int(next.Add(1))
				if err := b.Add(context.Background(), x); err != nil {
					return // fermé
				}
				acc.Store(x, true)
			}
		})
	}
	time.Sleep(200 * time.Millisecond)
	if err := b.Close(context.Background()); err != nil {
		t.Fatal(err)
	}
	wg.Wait()
	acc.Range(func(k, _ any) bool {
		accepted++
		switch s.got[k.(int)] {
		case 0:
			lost++
		case 1:
		default:
			dup++
		}
		return true
	})
	written = len(s.got)
	if s.overlap.Load() != 0 || s.maxLen > 32 {
		t.Errorf("E2 violé : chevauchements %d, lot max %d", s.overlap.Load(), s.maxLen)
	}
	return
}

func TestExactlyOnce(t *testing.T) {
	for _, buggy := range []bool{false, true} {
		var totLost int
		for range 20 {
			acc, wr, dup, lost := stress(t, buggy)
			totLost += lost
			if !buggy && (lost != 0 || dup != 0 || wr != acc) {
				t.Fatalf("E1 violé : acceptés %d écrits %d doublons %d perdus %d", acc, wr, dup, lost)
			}
		}
		fmt.Printf("BuggyClose=%-5v éléments acceptés mais jamais écrits, sur 20 essais : %d\n", buggy, totLost)
	}
}

// P1 : déclenchement par Interval, en temps virtuel.
func TestInterval(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		var mu sync.Mutex
		var at []time.Duration
		start := time.Now()
		b := New(10, 100, time.Second, func(_ context.Context, xs []int) error {
			mu.Lock()
			at = append(at, time.Since(start))
			mu.Unlock()
			return nil
		})
		b.Add(context.Background(), 1)
		time.Sleep(1500 * time.Millisecond)
		synctest.Wait()
		mu.Lock()
		if len(at) != 1 || at[0] != time.Second {
			t.Errorf("lots : %v", at)
		}
		mu.Unlock()
		b.Close(context.Background())
	})
}

// Close pendant que des Add sont bloqués (file pleine, flush lent) : ils rendent la main aussitôt.
func TestCloseUnblocksAdd(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		var written atomic.Int32
		b := New(1, 1, time.Hour, func(_ context.Context, xs []int) error {
			time.Sleep(100 * time.Millisecond) // lent : la file se remplit, les Add suivants bloquent
			written.Add(int32(len(xs)))
			return nil
		})
		var wg sync.WaitGroup
		errs := make([]error, 10)
		for i := range 10 {
			wg.Go(func() { errs[i] = b.Add(context.Background(), i) })
		}
		synctest.Wait() // tous bloqués ou acceptés
		start := time.Now()
		if err := b.Close(context.Background()); err != nil {
			t.Fatal(err)
		}
		wg.Wait()
		accepted := 0
		for _, e := range errs {
			if e == nil {
				accepted++
			} else if !errors.Is(e, ErrClosed) {
				t.Fatal(e)
			}
		}
		t.Logf("acceptés %d, refusés %d, écrits %d, Close en %v (temps virtuel)", accepted, 10-accepted, written.Load(), time.Since(start))
		if int(written.Load()) != accepted {
			t.Error("E1 violé")
		}
	})
}
```

```
$ go test -race -v ./batch/
BuggyClose=false éléments acceptés mais jamais écrits, sur 20 essais : 0
BuggyClose=true  éléments acceptés mais jamais écrits, sur 20 essais : 32
--- PASS: TestExactlyOnce (10.35s)
--- PASS: TestInterval (0.00s)
    batch_test.go:149: acceptés 2, refusés 8, écrits 2, Close en 200ms (temps virtuel)
--- PASS: TestCloseUnblocksAdd (0.00s)
```

**L'invariant extérieur E1** est vérifié au repos, après `Close`. Chaque élément pour lequel `Add` a rendu
`nil` est dans le puits, exactement une fois, malgré un `flush` qui échoue une fois sur cinq. Les réessais se
font dans la boucle, sur le **même** lot, et sans jamais lancer deux `flush` à la fois (E2, vérifié par un
compteur d'appels simultanés).

**La course de `Close`**, la difficulté de l'exercice. Une version naïve (`BuggyClose`) ferme `quit`, et la
boucle vide `in` puis écrit. Mais un `Add` déjà dans son `select` peut encore réussir `b.in <- x` **après** le
vidage : l'élément est accepté (`Add` rend `nil`) et ne sera jamais écrit. Mesuré : **32** éléments perdus en
20 essais, entre 15 et 55 selon les exécutions. `-race` ne voit rien, et seule la vérification au repos de E1
le révèle, comme pour le pool du cours.

La correction ordonne trois étapes :
1. fermer `stopAdds`, qui réveille les `Add` bloqués : ils rendent `ErrClosed` ;
2. prendre le verrou en écriture, ce qui attend la fin des `Add` en cours (ils tiennent un verrou en lecture
   pendant leur envoi) et interdit les suivants ;
3. seulement alors, fermer `quit` : la boucle vide `in`, que plus personne ne peut remplir.

**Deux défauts de ma propre première version**, trouvés par les tests :

- **Un test qui ne testait rien.** `TestCloseUnblocksAdd` « réussissait » parce qu'aucun `Add` n'était
  bloqué : la boucle vidait la file aussitôt. Refait avec un `flush` lent, il a montré le second défaut.
- **Un `Close` qui ne débloquait pas les `Add`.** La première version prenait le verrou **avant** de les
  réveiller : elle attendait des `Add` qui attendaient la boucle, laquelle attendait un `flush` lent. Pire,
  sous `synctest` le test restait bloqué indéfiniment. Une goroutine bloquée sur un `sync.Mutex` n'est pas
  « durablement bloquée » pour `synctest`, qui n'avance donc plus le temps virtuel. La correction réveille
  d'abord, et vérifie la double fermeture par un `atomic.Bool`, sans verrou.

**P1, le déclenchement par `Interval`**, est vérifié en temps virtuel (`synctest`). Un seul élément, et le lot
part à exactement 1 s.

---

## Défi — Vérifier la linéarisabilité

```go
// Fichier : linz/linz.go
// Package linz : enregistrer un historique d'opérations sur un registre clé-valeur, et vérifier
// qu'il est linéarisable (défi).
package linz

import (
	"sync"
	"time"
)

type Kind int

const (
	Put Kind = iota
	Get
)

// Op : une opération observée de l'extérieur, entre son appel et son retour.
type Op struct {
	Kind         Kind
	Key, Value   int // Value : écrite (Put) ou lue (Get) ; 0 = absente
	Call, Return time.Duration
}

// Recorder note les opérations avec une horloge monotone commune.
type Recorder struct {
	mu    sync.Mutex
	start time.Time
	ops   []Op
}

func NewRecorder() *Recorder { return &Recorder{start: time.Now()} }

func (r *Recorder) Now() time.Duration { return time.Since(r.start) } // monotone (time.Since)

func (r *Recorder) Add(op Op) {
	r.mu.Lock()
	r.ops = append(r.ops, op)
	r.mu.Unlock()
}

func (r *Recorder) Ops() []Op {
	r.mu.Lock()
	defer r.mu.Unlock()
	return append([]Op(nil), r.ops...)
}

// Linearizable vérifie un historique. La linéarisabilité est compositionnelle : un historique est
// linéarisable si et seulement si sa restriction à chaque clé l'est. On vérifie donc clé par clé.
func Linearizable(ops []Op) bool {
	byKey := map[int][]Op{}
	for _, o := range ops {
		byKey[o.Key] = append(byKey[o.Key], o)
	}
	for _, h := range byKey {
		if !checkKey(h) {
			return false
		}
	}
	return true
}

// checkKey : recherche en profondeur d'un ordre séquentiel. À chaque étape, on peut linéariser une
// opération si aucune opération restante n'a RETOURNÉ avant son APPEL (sinon l'autre doit passer
// d'abord). On applique l'opération au modèle (la valeur courante) ; un Get doit lire la valeur
// courante. Les états déjà vus (ensemble linéarisé, valeur) sont mémorisés pour couper la recherche.
func checkKey(h []Op) bool {
	n := len(h)
	if n > 63 {
		panic("trop d'opérations pour une clé")
	}
	type state struct {
		done  uint64
		value int
	}
	seen := map[state]bool{}
	var dfs func(done uint64, value int) bool
	dfs = func(done uint64, value int) bool {
		if done == 1<<n-1 {
			return true
		}
		if seen[state{done, value}] {
			return false
		}
		seen[state{done, value}] = true
		for i := range n {
			if done&(1<<i) != 0 {
				continue
			}
			minimal := true
			for j := range n {
				if j != i && done&(1<<j) == 0 && h[j].Return < h[i].Call {
					minimal = false
					break
				}
			}
			if !minimal {
				continue
			}
			o := h[i]
			switch o.Kind {
			case Put:
				if dfs(done|1<<i, o.Value) {
					return true
				}
			case Get:
				if o.Value == value && dfs(done|1<<i, value) {
					return true
				}
			}
		}
		return false
	}
	return dfs(0, 0)
}
```

```go
// Fichier : linz/linz_test.go
package linz

import (
	"fmt"
	"math/rand/v2"
	"sync"
	"sync/atomic"
	"testing"
	"time"
)

type KV interface {
	Put(client, k, v int)
	Get(client, k int) int
}

// Correct : une map sous verrou.
type locked struct {
	mu sync.Mutex
	m  map[int]int
}

func (s *locked) Put(_, k, v int)  { s.mu.Lock(); s.m[k] = v; s.mu.Unlock() }
func (s *locked) Get(_, k int) int { s.mu.Lock(); defer s.mu.Unlock(); return s.m[k] }

// Fautif : chaque client garde un cache de lecture, jamais invalidé par les écritures des autres.
type cached struct {
	store locked
	mu    sync.Mutex
	cache map[int]map[int]int // client → clé → valeur
}

func (c *cached) Put(cl, k, v int) {
	c.store.Put(cl, k, v)
	c.mu.Lock()
	c.cache[cl][k] = v
	c.mu.Unlock()
}

func (c *cached) Get(cl, k int) int {
	c.mu.Lock()
	v, ok := c.cache[cl][k]
	c.mu.Unlock()
	if ok {
		return v // peut-être périmée
	}
	v = c.store.Get(cl, k)
	c.mu.Lock()
	c.cache[cl][k] = v
	c.mu.Unlock()
	return v
}

var nextVal atomic.Int64

// history : 4 clients, opsPerClient opérations chacun sur nkeys clés ; valeurs écrites toutes distinctes.
func history(kv KV, opsPerClient, nkeys int) []Op {
	rec := NewRecorder()
	var wg sync.WaitGroup
	for cl := range 4 {
		wg.Go(func() {
			rng := rand.New(rand.NewPCG(rand.Uint64(), uint64(cl)))
			for range opsPerClient {
				k := rng.IntN(nkeys)
				if rng.IntN(2) == 0 {
					v := int(nextVal.Add(1))
					call := rec.Now()
					kv.Put(cl, k, v)
					rec.Add(Op{Put, k, v, call, rec.Now()})
				} else {
					call := rec.Now()
					v := kv.Get(cl, k)
					rec.Add(Op{Get, k, v, call, rec.Now()})
				}
				if rng.IntN(4) == 0 {
					time.Sleep(time.Duration(rng.IntN(50)) * time.Microsecond)
				}
			}
		})
	}
	wg.Wait()
	return rec.Ops()
}

func TestCheckerOnKnownHistories(t *testing.T) {
	// A écrit 1 et a fini avant que B ne lise : B doit lire 1.
	ok := []Op{{Put, 0, 1, 0, 10}, {Get, 0, 1, 20, 30}}
	bad := []Op{{Put, 0, 1, 0, 10}, {Get, 0, 0, 20, 30}}
	// Les deux se chevauchent : B peut lire 0 ou 1.
	overlap := []Op{{Put, 0, 1, 0, 30}, {Get, 0, 0, 10, 20}}
	if !Linearizable(ok) || Linearizable(bad) || !Linearizable(overlap) {
		t.Fatal("vérificateur faux sur les cas connus")
	}
}

func TestImplementations(t *testing.T) {
	for _, tc := range []struct {
		name string
		kv   func() KV
	}{
		{"map sous verrou", func() KV { return &locked{m: map[int]int{}} }},
		{"cache par client", func() KV {
			c := &cached{store: locked{m: map[int]int{}}, cache: map[int]map[int]int{}}
			for cl := range 4 {
				c.cache[cl] = map[int]int{}
			}
			return c
		}},
	} {
		bad := 0
		for range 200 {
			if !Linearizable(history(tc.kv(), 8, 2)) {
				bad++
			}
		}
		fmt.Printf("%-16s historiques non linéarisables : %3d / 200\n", tc.name, bad)
		if tc.name == "map sous verrou" && bad != 0 {
			t.Error("faux positif sur l'implémentation correcte")
		}
	}
}

// Le coût : historiques concurrents sur UNE clé, de taille croissante.
func TestCost(t *testing.T) {
	for _, per := range []int{2, 3, 4, 5, 6, 8} {
		var worst time.Duration
		for range 20 {
			h := history(&locked{m: map[int]int{}}, per, 1)
			start := time.Now()
			Linearizable(h)
			worst = max(worst, time.Since(start))
		}
		fmt.Printf("%2d opérations sur une clé : vérification, pire cas sur 20 : %v\n", 4*per, worst)
	}
}

// Le pire cas : n opérations qui se chevauchent TOUTES, et un Get impossible. Il faut explorer tous
// les ordres avant de conclure « non linéarisable ».
func TestWorstCase(t *testing.T) {
	for _, n := range []int{8, 12, 16} { // 18 et 20 : voir le corrigé (3 s et 18 s)
		h := []Op{{Get, 0, -1, 0, 100}} // lit une valeur jamais écrite
		for i := range n - 1 {
			h = append(h, Op{Put, 0, i + 1, 0, 100})
		}
		start := time.Now()
		r := Linearizable(h)
		fmt.Printf("%2d opérations simultanées : %-5v en %v\n", n, r, time.Since(start).Round(time.Microsecond))
	}
}
```

```
$ go test -race -v ./linz/
--- PASS: TestCheckerOnKnownHistories (0.00s)
map sous verrou  historiques non linéarisables :   0 / 200
cache par client historiques non linéarisables : 198 / 200
--- PASS: TestImplementations (0.54s)
 8 opérations sur une clé : vérification, pire cas sur 20 : 181.594µs
[…]
32 opérations sur une clé : vérification, pire cas sur 20 : 173.364µs
--- PASS: TestCost (0.10s)
$ go test -v -run Worst ./linz/      # liste étendue à 18 et 20 pour la mesure
 8 opérations simultanées : false en 251µs
12 opérations simultanées : false en 7.224ms
16 opérations simultanées : false en 448.972ms
18 opérations simultanées : false en 3.118485s
20 opérations simultanées : false en 17.879361s
```

- **Les cas connus d'abord** : un vérificateur se teste lui-même, sur des historiques dont on connaît la
  réponse, avant de servir à juger une implémentation.
- **La map sous verrou** n'a aucun historique non linéarisable sur 200. Le **cache par client** en a 198 : dès
  qu'un client relit une clé qu'un autre a changée, il rend l'ancienne valeur, alors que l'écriture est
  terminée avant sa lecture.
- **La compositionnalité** : un historique est linéarisable si et seulement si sa restriction à chaque objet
  (ici, chaque clé) l'est. On vérifie donc clé par clé, ce qui réduit énormément la recherche.
- **Le coût**. Sur les historiques réels d'une map sous verrou, la vérification prend moins de 0,2 ms jusqu'à
  32 opérations par clé : les opérations se chevauchent peu, et la mémorisation des états déjà vus coupe la
  recherche.

  Le **pire cas** est exponentiel : toutes les opérations simultanées, et une réponse négative qui oblige à
  tout explorer. Il faut 251 µs pour 8 opérations, 449 ms pour 16 et 17,9 s pour 20, soit environ ×6 par paire
  d'opérations ajoutée. La vérification de linéarisabilité est un problème difficile en général. En pratique,
  on garde des historiques courts et nombreux, plutôt que longs et rares.

---

## Quiz

1. L'état (ce que le composant mémorise), ses invariants (ce qui est toujours vrai), et ses propriétés de
   progrès (ce qui finit par arriver).
2. Une propriété de l'état qui doit être vraie chaque fois que le verrou est libre. Elle peut être
   temporairement fausse **dans** une section critique.
3. La sûreté dit que rien de mauvais n'arrive (I2 : jamais plus de `max` ressources). Le progrès dit que
   quelque chose de bon finit par arriver (P1 : un attendant est servi ou annulé).
4. Un `Unlock` arrive avant le `Lock` suivant. Un envoi sur un canal arrive avant la fin de la réception
   correspondante. La fermeture d'un canal arrive avant une réception qui la constate.
5. La valeur envoyée change de responsable : l'expéditeur ne doit plus y toucher, le destinataire en
   répond.
6. `w` a une capacité de 1, et seul le `Put` qui l'a retiré de la file sous le verrou peut y envoyer : `w` ne
   peut pas être plein.
7. Une annulation qui croise une remise : `Put` a déposé une ressource dans `w` pendant que l'attendant
   sortait par `ctx.Done()`.
8. Il n'y a pas de course de données : tous les accès sont protégés. Le défaut est un ordre d'événements non
   prévu par la conception, une course **logique**.
9. Un invariant qui relie l'état du composant au monde extérieur (prêtées = détenteurs réels, acceptés =
   écrits). On le vérifie au repos : fin de test, arrêt, ou par une métrique en production.
10. Quand il faut protéger un état fait de plusieurs variables liées par des invariants. Les canaux servent à
    transférer et à attendre.
