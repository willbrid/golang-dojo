# Leçon 12 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Le `recover` inutile

Le programme affiche `panic: …` avec la pile de la goroutine fautive, puis se termine avec le code
de sortie **2**. « récupéré » n'apparaît **jamais** : le `defer` de `main` ne s'exécute même pas, car
le processus est arrêté pendant que `main` dort. Le `recover` de `main` ne voit que les paniques de la
pile de `main`.

## E2 — La frontière

```go
func SafeGo(f func()) <-chan error {
	errc := make(chan error, 1)
	go func() {
		errc <- safe(func() error { f(); return nil })
	}()
	return errc
}
```

| Cas | Erreur rendue |
|---|---|
| `f` réussit | `nil` |
| `panic("boum")` | `*PanicError{Value: "boum"}` |
| `panic(err)` | `*PanicError{Value: err}` — on peut ajouter une méthode `Unwrap()` qui rend `Value` si c'est une `error`, pour que `errors.Is` traverse |
| `panic(nil)` | depuis Go 1.21, `recover()` rend un `*runtime.PanicNilError` (« panic called with nil argument ») au lieu de `nil` — mesuré. Auparavant, `recover()` rendait `nil` et la panique passait **inaperçue** pour un `if r := recover(); r != nil` |

Le dernier cas montre pourquoi le test `r != nil` est correct en Go moderne, et faux avant 1.21.

## E3 — `errgroup` et les paniques

```go
type SafeGroup struct{ g *errgroup.Group }

func WithContext(ctx context.Context) (*SafeGroup, context.Context) {
	g, ctx := errgroup.WithContext(ctx)
	return &SafeGroup{g: g}, ctx
}

func (s *SafeGroup) Go(f func() error) {
	s.g.Go(func() error { return safe(f) })   // la panique devient l'erreur de la tâche
}

func (s *SafeGroup) Wait() error { return s.g.Wait() }
```

Comme la panique devient une erreur **retournée**, `errgroup` fait le reste : il annule le contexte et
`Wait` la rend. `errors.As(err, &pe)` permet de distinguer une panique d'une erreur ordinaire.

## E4 — Ce qui ne se récupère pas

(a) `fatal error: concurrent map writes` et (b) `fatal error: stack overflow` (`goroutine stack
exceeds 1000000000-byte limit`) **ne passent pas** par `recover` : ce sont des erreurs fatales du
runtime, pas des paniques. Protections : (a) synchroniser l'accès à la map (`Mutex`, leçon 6) et
lancer les tests avec `-race` ; (b) borner la récursion (profondeur maximale, ou version itérative),
surtout quand la profondeur dépend d'une entrée externe (un JSON imbriqué malveillant).

## E5 — Le superviseur emballé

Sans plafond ni attente : des **centaines de milliers** de redémarrages par seconde, un cœur à 100 %,
et si le worker journalise son erreur, des gigaoctets de journaux en quelques minutes. Avec l'attente
doublée mais sans plafond : 1 ms, 2 ms, 4 ms… au bout de 30 essais, l'attente dépasse **12 jours** —
le worker est en pratique abandonné sans que personne le sache. Il faut donc **plafonner l'attente**
(par exemple 30 s maximum) **et** le nombre d'essais, ou remonter l'échec à un niveau qui alerte.

## Exercice intermédiaire — Un pool de workers robuste

```go
type Pool struct {
	panics  atomic.Int64
	window  []bool        // les 100 dernières tâches : a-t-elle paniqué ? (accès sous mu)
	mu      sync.Mutex
	cancel  context.CancelCauseFunc
}

var ErrTooManyPanics = errors.New("trop de paniques : bug probable")

func (p *Pool) runTask(t Task) Result {
	err := safe(func() error { return t.Do() })
	var pe *PanicError
	panicked := errors.As(err, &pe)
	if panicked {
		p.panics.Add(1)
	}
	p.mu.Lock()
	p.window = append(p.window, panicked)
	if len(p.window) > 100 {
		p.window = p.window[1:]
	}
	n := 0
	for _, b := range p.window {
		if b {
			n++
		}
	}
	full := len(p.window) == 100
	p.mu.Unlock()
	if full && n > 5 {
		p.cancel(ErrTooManyPanics)    // les workers voient ctx.Done() et s'arrêtent
	}
	return Result{ID: t.ID, Err: err}
}
```

Pourquoi s'arrêter : une panique isolée est un cas rare (une entrée bizarre) ; 5 % de paniques
signalent un **bug systématique** (une dépendance qui a changé de format, une version défectueuse).
Continuer, c'est traiter mal des milliers de tâches et noyer les journaux ; s'arrêter fort et tôt rend
le problème visible et évite d'aggraver l'état. C'est un **disjoncteur** (*circuit breaker*, niveau 10).

## Défi — Panique et état partagé

1. Le verrou **est** libéré (le `defer c.mu.Unlock()` s'exécute pendant la remontée de la panique).
   Mais le cache est **incohérent** : `from` a été supprimé, `to` n'a jamais été écrit — la valeur est
   perdue.
2. Un plantage arrête tout et laisse un état reconstruit au redémarrage (le cache est vide, puis se
   remplit). Une panique récupérée laisse le programme tourner sur un état **faux**, qui produira des
   résultats faux pendant des heures, sans aucun signal.
3. ```go
   func (c *Cache) Move(from, to string) {
   	c.mu.Lock()
   	v, ok := c.data[from]
   	c.mu.Unlock()
   	if !ok {
   		return
   	}
   	nv := transform(v)            // peut paniquer : rien n'a encore été modifié
   	c.mu.Lock()
   	defer c.mu.Unlock()
   	if cur, ok := c.data[from]; ok && cur == v {   // revérifier (leçon 6)
   		delete(c.data, from)
   		c.data[to] = nv
   	}
   }
   ```
   Calculer **avant** de modifier : toutes les écritures se font après la dernière opération qui peut
   échouer. (La comparaison `cur == v` suppose `V` comparable.)
4. **Un superviseur peut relancer sans risque un worker dont l'état est reconstruit à partir de zéro
   au démarrage, ou dont les modifications d'état partagé sont atomiques** (tout ou rien). Si une
   panique peut laisser un état partagé à moitié modifié, il faut arrêter le processus, pas relancer.

## Quiz

1. Le programme entier s'arrête (code de sortie 2), avec la pile de la goroutine fautive.
2. Une panique ne remonte que la pile de sa goroutine ; il n'y a pas de lien de pile entre goroutines.
3. Dans un `defer` placé en tête de la fonction exécutée par la goroutine.
4. Parce qu'elle n'existe qu'au moment de la panique ; avec `runtime/debug.Stack()`.
5. Ce serait cacher des bugs ; les frontières doivent être rares et explicites (requêtes, tâches,
   goroutines de longue durée).
6. `net/http` oui (par requête, avec journalisation) ; `errgroup` non (v0.22) : une panique tue le programme.
7. Écritures concurrentes sur une map, interblocage total, déverrouillage d'un verrou libre,
   débordement de pile, épuisement de la mémoire.
8. Un plafond de redémarrages, une attente croissante (elle-même plafonnée), l'arrêt sur annulation,
   la remontée de l'erreur finale.
9. Quand la panique a pu laisser un état partagé incohérent.
10. Oui : c'est un bug en production, qui doit être corrigé.
