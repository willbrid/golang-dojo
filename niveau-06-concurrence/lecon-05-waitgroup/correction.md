# Leçon 5 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Les quatre pièges

| Piège | `go vet` | `go run -race` | À l'exécution |
|---|---|---|---|
| `Add` dans la goroutine | `WaitGroup.Add called from inside new goroutine` | race sur ce que la goroutine écrit, si `main` le lit | `Wait` rend trop tôt (mesuré : « fini » avec 93 puis 58 tâches comptées sur 100) |
| compteur négatif | rien | rien | `panic: sync: negative WaitGroup counter` |
| copie par valeur | `passes lock by value: sync.WaitGroup contains sync.noCopy` | rien | `Wait` bloque pour toujours (ou interblocage détecté) |
| réutilisation pendant un `Wait` | rien | parfois | comportement indéfini, panique possible (`WaitGroup is reused before previous Wait has returned`) |

Le compteur négatif et la réutilisation ne sont détectés **qu'à l'exécution** — d'où l'intérêt de
`wg.Go`, qui les rend impossibles pour le cas courant.

## E2 — Moderniser

Mesuré : `go fix` convertit la forme `wg.Add(1)` + `go func() { defer wg.Done() … }()` **dans la
boucle** en `wg.Go(func() { … })`. Il **ne convertit pas** la forme `wg.Add(len(items))` placée avant
la boucle : la transformation n'est sûre que si chaque `Add(1)` est visiblement apparié à **une**
goroutine et à son `defer Done`. Un modernisateur ne propose que des changements dont il peut
prouver qu'ils préservent le comportement.

## E3 — `append` concurrent

Résultats typiques : 912, 967, 1 000, 884… et `-race` signale la race sur `results`. `append` lit
la longueur, écrit au-delà, puis réaffecte l'en-tête du slice : deux goroutines peuvent écrire dans
la même case, ou l'une écraser l'en-tête de l'autre. Corrections : `results[i] = r` dans un slice
pré-alloué, ou `ch := make(chan result, 1000)` lu après `Wait` (ou un `Mutex`, leçon 6).

## E4 — Attendre avec un délai

```go
func WaitTimeout(wg *sync.WaitGroup, d time.Duration) bool {
	done := make(chan struct{})
	go func() {
		wg.Wait()
		close(done)
	}()
	select {
	case <-done:
		return true
	case <-time.After(d):
		return false
	}
}
```

Coût caché : quand le délai expire, la goroutine qui appelle `wg.Wait()` **reste bloquée** jusqu'à
ce que le groupe finisse — et pour toujours s'il ne finit jamais. De plus, les goroutines du groupe
continuent de travailler. Un délai sur une attente ne résout rien s'il n'y a pas d'**annulation**
derrière : `context` (leçon 8).

## E5 — Groupes imbriqués

Un seul `WaitGroup` suffit, **à condition** que chaque goroutine de répertoire fasse ses `Add` (ou
`wg.Go`) pour ses fichiers **avant** de se terminer : le compteur ne peut alors jamais tomber à 0
prématurément, puisque le `Done` du répertoire arrive après les `Add` de ses fichiers.

```go
var wg sync.WaitGroup
for _, dir := range dirs {
	wg.Go(func() {
		for _, f := range listFiles(dir) {
			wg.Go(func() { countBytes(f) })   // ajouté avant la fin de la goroutine parente
		}
	})
}
wg.Wait()
```

La version à deux niveaux (un groupe par répertoire, attendu par sa goroutine) est souvent plus
**lisible** : chaque niveau attend ce qu'il a lancé, comme une fonction synchrone. Elle permet aussi
d'agréger par répertoire sans structure partagée. Le choix dépend de ce qu'on doit faire des
résultats.

## Exercice intermédiaire — Vérificateur de liens en parallèle

```go
type finding struct {
	file string
	link Link
}

func checkTree(root string, workers int) ([]finding, error) {
	files, err := markdownFiles(root)          // filepath.WalkDir, N4 L7
	if err != nil {
		return nil, err
	}
	perFile := make([][]finding, len(files))
	errs := make([]error, len(files))
	sem := make(chan struct{}, workers)         // sémaphore : au plus `workers` fichiers à la fois
	var wg sync.WaitGroup
	for i, path := range files {
		wg.Go(func() {
			sem <- struct{}{}                    // prendre un jeton (bloque s'il n'y en a plus)
			defer func() { <-sem }()             // le rendre
			perFile[i], errs[i] = checkFile(path)
		})
	}
	wg.Wait()

	var all []finding
	for _, fs := range perFile {
		all = append(all, fs...)
	}
	slices.SortFunc(all, func(a, b finding) int {
		return cmp.Or(cmp.Compare(a.file, b.file), cmp.Compare(a.link.Line, b.link.Line))
	})
	return all, errors.Join(errs...)
}
```

- Résultats par case, erreurs par case, fusion et tri après `Wait` : aucun verrou.
- Le **sémaphore** : un channel bufferisé de capacité N, dont on remplit une case pour « entrer » et
  qu'on vide pour « sortir ». Au plus N goroutines sont dans la section en même temps. Noter qu'on
  lance quand même **toutes** les goroutines (elles attendent leur jeton) : pour des millions de
  fichiers, un *worker pool* qui ne lance que N goroutines est préférable (leçon 9).
- Le gain mesuré dépend du rapport entre lecture disque et calcul : sur un petit dépôt dont les
  fichiers sont dans le cache système, le travail par fichier est si court que le gain est modeste
  (×2 à ×3 sur 8 cœurs logiques) ; le coût de lancement et de tri devient visible.

## Défi — Implémenter un `WaitGroup`

```go
type Group struct {
	ops  chan int           // +n / -1
	wait chan chan struct{} // demandes d'attente
}

func NewGroup() *Group {
	g := &Group{ops: make(chan int), wait: make(chan chan struct{})}
	go g.loop()
	return g
}

func (g *Group) loop() {
	n := 0
	var waiters []chan struct{}
	for {
		select {
		case d := <-g.ops:
			n += d
			if n < 0 {
				panic("Group : compteur négatif")
			}
		case w := <-g.wait:
			waiters = append(waiters, w)
		}
		if n == 0 {
			for _, w := range waiters {
				close(w)          // réveille TOUS les Wait en cours
			}
			waiters = nil
		}
	}
}

func (g *Group) Add(n int) { g.ops <- n }
func (g *Group) Done()     { g.ops <- -1 }
func (g *Group) Wait() {
	w := make(chan struct{})
	g.wait <- w
	<-w
}
```

- `close(w)` réveille chaque attente ; plusieurs `Wait` simultanés fonctionnent.
- La panique se produit dans la goroutine propriétaire, ce qui **tue le programme** (leçon 12) —
  c'est voulu, comme pour `sync.WaitGroup`.
- La goroutine `loop` ne s'arrête jamais : un vrai type aurait besoin d'un `Close`. `sync.WaitGroup`
  n'a pas ce problème : il n'a aucune goroutine.
- Performance : chaque `Add`/`Done` coûte ici un rendez-vous sur un channel (quelques centaines de
  ns, leçon 2) ; `sync.WaitGroup` fait une **addition atomique** (quelques ns) et ne réveille les
  goroutines en attente que lorsque le compteur atteint zéro. Facteur typique : 50 à 100.

## Quiz

1. `Add(n)` ajoute n au compteur, `Done` retire 1, `Wait` attend zéro, `Go(f)` fait `Add(1)`, lance
   `f` et appelle `Done` à sa fin.
2. Tout ce qui a été écrit avant chaque `Done` est visible après le retour de `Wait`.
3. Sinon `Wait` peut voir un compteur à zéro avant que la goroutine ait compté, et rendre trop tôt.
4. Pour qu'il soit appelé quel que soit le chemin de sortie.
5. `panic: sync: negative WaitGroup counter`.
6. La copie a son propre compteur : les `Done` ne décrémentent pas l'original ; `go vet` (copylocks).
7. Non, il compte ; pour limiter : sémaphore ou *worker pool*.
8. Une case pré-allouée par goroutine, lue après `Wait`.
9. `Wait` dans une goroutine qui ferme un channel, et `select` avec `time.After` — en sachant que
   cela n'annule rien.
10. Le modernisateur `waitgroupgo` les transforme en `wg.Go` quand l'appariement `Add(1)`/`go`/`Done`
    est évident.
