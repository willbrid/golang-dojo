# Leçon 8 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — `lostcancel`

```
the cancel function returned by context.WithTimeout should be called, not discarded, to avoid a context leak
```

Si la fonction rend le contexte sans rendre `cancel`, l'analyseur le signale aussi : la fonction
`cancel` n'est appelée sur aucun chemin. Quand un contexte doit vivre plus longtemps que la fonction
qui le crée, elle rend **le contexte et `cancel`** (comme les fonctions `With…` elles-mêmes), et
c'est l'**appelant** qui devient responsable du `defer cancel()`.

## E2 — Réparer `fetch`

```go
func fetch(ctx context.Context, d time.Duration, v string) <-chan string {
	ch := make(chan string, 1)
	go func() {
		select {
		case <-time.After(d):
			ch <- v
		case <-ctx.Done():          // annulé : on s'arrête sans rien envoyer
		}
	}()
	return ch
}

func withTimeout(ctx context.Context, res <-chan string, limit time.Duration) (string, error) {
	ctx, cancel := context.WithTimeout(ctx, limit)
	defer cancel()                   // ← annule fetch dès qu'on cesse d'attendre
	select {
	case v := <-res:
		return v, nil
	case <-ctx.Done():
		return "", ctx.Err()
	}
}
```

Pour que l'annulation atteigne `fetch`, les deux doivent partager le contexte : l'appelant crée
`ctx`, le passe à `fetch` **et** à `withTimeout`… ou, plus simplement, `withTimeout` ne reçoit pas un
channel mais la fonction à exécuter, et lui passe son propre contexte dérivé. Une fois `fetch`
annulée au `return` de `withTimeout`, le `time.Sleep(time.Hour)` du test `synctest` devient
**inutile** : plus aucune goroutine ne reste bloquée. Le test qui exigeait une rustine passe
désormais sans elle — c'était le signe que le code avait un défaut.

## E3 — Une boucle de calcul annulable

```go
func countPrimes(ctx context.Context, lo, hi int) (int, error) {
	n := 0
	for x := max(lo, 2); x < hi; x++ {
		if x%10_000 == 0 {
			if err := ctx.Err(); err != nil {
				return n, err
			}
		}
		if isPrime(x) {
			n++
		}
	}
	return n, nil
}
```

Le délai de réaction est le temps de traiter 10 000 nombres — de l'ordre de la milliseconde. Tester
à **chaque** nombre fonctionne aussi, mais `ctx.Err()` prend un verrou interne : sur une boucle très
courte, le surcoût devient mesurable (à vérifier par benchmark). Le bon intervalle équilibre réactivité
et coût — quelques millisecondes de réaction suffisent presque toujours.

## E4 — La chaîne coupée

`context.Background()` **coupe** la chaîne : `loadUser` ne sera jamais annulé, même si le client
s'est déconnecté ou si le délai global est dépassé. Correction : `loadUser(ctx, id)`.

```go
func TestHandleRespectsCancel(t *testing.T) {
	ctx, cancel := context.WithCancel(context.Background())
	cancel()                                   // déjà annulé
	err := Handle(ctx, "42")
	if !errors.Is(err, context.Canceled) {
		t.Fatalf("Handle(ctx annulé) = %v, attendu context.Canceled", err)
	}
}
```

Ce test exige que `loadUser` (ou son faux) vérifie son contexte : c'est un bon test pour **toute**
fonction qui accepte un `ctx`.

## E5 — Clé de valeur

Avec des clés `"id"` de type `string`, `trace` écrase la valeur d'`auth` (ou l'inverse) : `ctx.Value`
rend la valeur la plus proche dans la chaîne. Correction :

```go
package auth

type userKey struct{}

func WithUser(ctx context.Context, u User) context.Context { return context.WithValue(ctx, userKey{}, u) }

func UserFrom(ctx context.Context) (User, bool) {
	u, ok := ctx.Value(userKey{}).(User)
	return u, ok
}
```

`go vet` ne signale **pas** la clé `string`. `staticcheck` la signale avec la règle `SA1029`
(« should not use built-in type string as key for value; define your own type to avoid collisions »).
Un exemple de plus où les outils se complètent (N5 L8).

## Exercice intermédiaire — Recherche avec budget

```go
func Search(ctx context.Context, query string) ([]Result, error) {
	ctx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
	defer cancel()
	ctx, cancelCause := context.WithCancelCause(ctx)
	defer cancelCause(nil)

	sources := []source{web, images, news}
	results := make([][]Result, len(sources))
	var wg sync.WaitGroup
	for i, src := range sources {
		wg.Go(func() {
			r, err := src(ctx, query)
			if err != nil {
				cancelCause(err)        // annule les autres, la première erreur devient la cause
				return
			}
			results[i] = r
		})
	}
	wg.Wait()                           // AUCUNE goroutine ne survit au retour

	if err := context.Cause(ctx); err != nil {
		return nil, fmt.Errorf("recherche %q : %w", query, err)
	}
	return slices.Concat(results...), nil
}
```

- **`wg.Wait()` avant de rendre** : même en cas d'erreur, on attend que les autres sources aient
  **constaté** l'annulation et soient sorties. C'est ce qui garantit la contrainte 4, et c'est
  pourquoi les sources **doivent** respecter leur contexte (sinon `Search` attendrait leur fin
  normale).
- **`cancelCause(err)`** : seule la **première** cause est retenue ; les appels suivants n'ont pas
  d'effet. Le `defer cancelCause(nil)` libère le contexte dans le cas nominal.
- **`context.Cause(ctx)`** à la fin : `nil` si tout a réussi ; l'erreur de la source fautive ; ou
  `DeadlineExceeded` si le budget a expiré (le délai du parent est respecté lui aussi, puisque le
  contexte en dérive). `errors.Is(err, context.DeadlineExceeded)` fonctionne grâce au `%w`.
- Avec `synctest`, le cas « parent avec un délai de 50 ms » vérifie que `Search` rend à **50 ms**
  exactement (et non 200) : un enfant ne peut pas allonger le délai de son parent.
- La leçon 9 montrera `errgroup.WithContext`, qui fait exactement ce `WaitGroup` + `WithCancelCause`.

## Défi — Implémenter `WithTimeout`

```go
type timeoutCtx struct {
	parent context.Context
	done   chan struct{}
	once   sync.Once
	mu     sync.Mutex
	err    error           // protégé par mu
	dl     time.Time
	timer  *time.Timer
	stop   func() bool     // désinscription d'AfterFunc
}

func MyWithTimeout(parent context.Context, d time.Duration) (context.Context, context.CancelFunc) {
	c := &timeoutCtx{parent: parent, done: make(chan struct{}), dl: time.Now().Add(d)}
	c.timer = time.AfterFunc(d, func() { c.finish(context.DeadlineExceeded) })
	c.stop = context.AfterFunc(parent, func() { c.finish(parent.Err()) })
	return c, func() { c.finish(context.Canceled) }
}

func (c *timeoutCtx) finish(err error) {
	c.once.Do(func() {
		c.mu.Lock()
		c.err = err
		c.mu.Unlock()
		close(c.done)
		c.timer.Stop()
		c.stop()
	})
}

func (c *timeoutCtx) Deadline() (time.Time, bool) { return c.dl, true }
func (c *timeoutCtx) Done() <-chan struct{}       { return c.done }
func (c *timeoutCtx) Value(k any) any             { return c.parent.Value(k) }
func (c *timeoutCtx) Err() error {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.err
}
```

- `sync.Once` rend `finish` idempotente : le premier événement (délai, `cancel`, parent) gagne, les
  autres ne font rien — pas de « close of closed channel ».
- `context.AfterFunc` évite une goroutine qui attendrait `parent.Done()` pour toujours.
- Ce que la vraie implémentation fait de plus : si le parent a un délai **plus court**, `WithDeadline`
  se contente de `WithCancel` (inutile d'armer une minuterie qui ne servira jamais) ; `Deadline()` rend
  le plus proche des deux délais ; elle gère les causes ; et elle s'intègre aux contextes de la
  bibliothèque standard sans goroutine supplémentaire. Ici, `Deadline()` est faux si le parent expire
  avant nous — un bug que le test comparatif doit faire apparaître.

## Quiz

1. Un channel fermé quand le contexte est annulé ou expire.
2. `nil`, `context.Canceled` ou `context.DeadlineExceeded`.
3. Pour libérer la minuterie et le lien avec le parent ; sinon, fuite.
4. Non : il est annulé au plus tard en même temps que son parent.
5. Le propager aux fonctions appelées, écouter `Done()` dans ses attentes, et rendre une erreur
   enveloppant `ctx.Err()` (ou la cause) quand il est annulé.
6. En premier paramètre ; il appartient à un appel, pas à un objet de longue durée.
7. Quand on n'a pas encore de contexte à transmettre (code en cours de migration) — jamais `nil`.
8. À connaître la raison d'une annulation, au-delà de `Canceled`/`DeadlineExceeded`.
9. Des données de requête qui traversent les couches (identifiant, identité, traçage) ; jamais des
   dépendances, des paramètres ou des options.
10. Pour qu'aucun autre paquet ne puisse utiliser la même clé et écraser la valeur.
