# Leçon 9 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — La fuite du pipeline

Sans le `case <-ctx.Done()`, `generate` reste bloquée sur `out <- job{…}` après que le pool a cessé
de lire : `NumGoroutine()` reste supérieur d'au moins 1 (plus les workers bloqués sur leur propre
envoi s'ils n'écoutent pas non plus `ctx`). Avec le `case`, tout revient à la seule goroutine `main`.

En test :

```go
func TestPipelineStopsOnCancel(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		ctx, cancel := context.WithCancel(t.Context())
		res := workerPool(ctx, generate(ctx, manyTexts(1000)), 3)
		<-res
		<-res
		cancel()
		synctest.Wait()      // attend que toutes les goroutines de la bulle soient bloquées ou finies
	})                       // si une goroutine reste bloquée : « blocked goroutines remain »
}
```

Il faut aussi **vider** `res` ou s'assurer que les workers ne bloquent pas : avec le `select` sur
`ctx.Done()`, ils sortent. Sans lui, `synctest.Test` échoue en signalant les goroutines restantes —
le détecteur de fuites est gratuit.

## E2 — Réécrire la leçon 8

```go
func Search(ctx context.Context, query string) ([]Result, error) {
	ctx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
	defer cancel()
	g, ctx := errgroup.WithContext(ctx)
	results := make([][]Result, len(sources))
	for i, src := range sources {
		g.Go(func() error {
			r, err := src(ctx, query)
			results[i] = r
			return err
		})
	}
	if err := g.Wait(); err != nil {
		return nil, fmt.Errorf("recherche %q : %w", query, err)
	}
	return slices.Concat(results...), nil
}
```

Disparaissent : le `WaitGroup`, `WithCancelCause`, l'appel à `cancelCause`, la lecture de
`context.Cause`. Les tests passent sans modification **sauf** une nuance : sur un délai, `g.Wait()`
rend l'erreur rendue par la source annulée — qui doit donc envelopper `ctx.Err()` pour que
`errors.Is(err, context.DeadlineExceeded)` reste vrai. C'est le cas des sources de la leçon 8.

## E3 — `SetLimit`

100 tâches de 10 ms, 10 à la fois : **100 ms** (exactement, sous `synctest`). Compteur des actives :

```go
var active, peak atomic.Int32
g.Go(func() error {
	n := active.Add(1)
	for {                                       // maximum par CAS (leçon 7)
		p := peak.Load()
		if n <= p || peak.CompareAndSwap(p, n) {
			break
		}
	}
	time.Sleep(10 * time.Millisecond)
	active.Add(-1)
	return nil
})
```

`peak` ne dépasse jamais 10 : `g.Go` **bloque** l'appelant tant que 10 tâches sont actives.

## E4 — L'ordre

Ordre d'arrivée : quelconque. Tri final et slice indexé donnent l'ordre d'entrée, mais seulement
**à la fin**. Pour afficher **au fur et à mesure** dans l'ordre d'entrée, il faut un **tampon de
réordonnancement** : le consommateur garde les résultats arrivés en avance dans une
`map[int]result`, et affiche tant que le prochain indice attendu est disponible.

```go
next := 0
pending := map[int]result{}
for r := range results {
	pending[r.id] = r
	for {
		p, ok := pending[next]
		if !ok {
			break
		}
		fmt.Println(p)
		delete(pending, next)
		next++
	}
}
```

La mémoire du tampon dépend de l'écart maximal entre tâches lentes et rapides.

## E5 — Les erreurs perdues

`Wait` rend **une seule** erreur : la première arrivée. Variante :

```go
g, ctx := errgroup.WithContext(ctx)
errs := make([]error, len(tasks))
for i, task := range tasks {
	g.Go(func() error {
		errs[i] = task(ctx)
		return errs[i]          // la première erreur annule quand même ctx
	})
}
g.Wait()
return errors.Join(errs...)
```

Attention : une partie des erreurs collectées sera `context.Canceled`, conséquence de l'annulation
et non cause première. Les filtrer (`errors.Is(err, context.Canceled)`) si on veut seulement les
vraies causes.

## Exercice intermédiaire — Indexeur de fichiers concurrent

Les points qui comptent :

```go
func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
	defer stop()
	…
}

func walk(ctx context.Context, fsys fs.FS) (<-chan string, <-chan error) {
	out := make(chan string)
	errc := make(chan error, 1)
	go func() {
		defer close(out)
		errc <- fs.WalkDir(fsys, ".", func(p string, d fs.DirEntry, err error) error {
			if err != nil {
				return nil                          // illisible : on continue
			}
			if !d.Type().IsRegular() {
				return nil
			}
			select {
			case out <- p:
				return nil
			case <-ctx.Done():
				return ctx.Err()                    // interrompt le parcours
			}
		})
	}()
	return out, errc
}
```

- `signal.NotifyContext` transforme Ctrl-C en annulation de contexte : tout le pipeline s'arrête par
  le mécanisme déjà en place, sans code spécifique au signal.
- Rendre une erreur depuis la fonction de `WalkDir` **interrompt** le parcours : c'est ainsi qu'on
  annule un `WalkDir`.
- Mesures typiques sur `$GOROOT/src` (fichiers dans le cache système) : gain net jusqu'à 4 workers,
  plafond vers 8 (cœurs logiques), rien au-delà. Fichiers **hors** cache (premier passage, disque
  lent) : le plafond arrive plus tôt — le disque, pas le processeur, limite.
- Mémoire bornée : lecture en flux, et channels non bufferisés — au plus N fichiers en cours.

## Défi — Pipeline par lots

Ordres de grandeur attendus (à mesurer chez soi) :

1. Ligne par ligne, le pipeline à 3 étages est **plus lent** que la boucle : des millions d'échanges
   à ~100–200 ns chacun, pour un travail par ligne du même ordre.
2. Par lots : le coût d'échange est divisé par la taille du lot. Gros gain entre 1 et 100, gain
   faible au-delà de 1 000 (le coût d'échange est devenu négligeable).
3. Avec 4 workers sur `words` et des maps partielles fusionnées à la fin : le travail coûteux (découpe
   et comptage) est enfin parallèle, sans contention sur une map partagée.
4. Le pipeline bat alors la boucle séquentielle, typiquement d'un facteur 2 à 3 sur 4 cœurs physiques.
   Les lots **amortissent** le coût de coordination ; les workers **parallélisent** le travail. Il faut
   les deux : des workers sans lots passent leur temps à se synchroniser, des lots sans workers ne
   gagnent rien.

## Quiz

1. Fermer sa sortie ; s'arrêter quand l'entrée est fermée ; écouter l'annulation (et rendre `<-chan`).
2. Il reste bloqué pour toujours sur un envoi : fuite de goroutine.
3. Fan-out : plusieurs goroutines lisent le même channel ; fan-in : plusieurs channels (ou émetteurs)
   fusionnés en un seul.
4. Pour borner le nombre de goroutines et donc la consommation de ressources.
5. Non : ordre de fin, pas d'entrée.
6. Il rend la première erreur et annule le contexte partagé dès cette erreur.
7. `SetLimit` borne le nombre de goroutines actives (`Go` bloque au-delà) ; `TryGo` lance seulement
   si la limite n'est pas atteinte, et rend `false` sinon.
8. La première erreur ; les autres sont perdues.
9. Parce que `Wait` attend toutes les goroutines : sans respect de `ctx`, l'annulation ne raccourcit rien.
10. Quand le travail est rapide, peu nombreux, ou sans structure concurrente naturelle : une boucle suffit.
