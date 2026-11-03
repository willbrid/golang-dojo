# Module 8 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Le corrigé complète le module du cours (`example.com/visits`). Les textes (ADR, revues) sont **une** réponse
argumentée. D'autres décisions sont défendables, si elles s'appuient sur les mêmes faits.

## E1 — Remplacer une décision

```markdown
# 0002 — Ne confirmer une visite qu'une fois écrite en base

Statut : proposé, 2027-05-04. Remplacera 0001 s'il est accepté.

## Contexte (ce qui change depuis 0001)

- Les visites des pages partenaires sont facturées à la visite. Une visite pour laquelle la page a été servie
  doit être en base, même si l'instance est tuée juste après. La perte d'une seconde de visites acceptée
  par 0001 ne l'est plus.
- Le reste du contexte de 0001 tient : base partagée avec les commandes, trois instances, 3 000 visites/s
  visées.

## Critères, dans l'ordre

1. Aucune visite servie sans être en base (nouveau, et premier).
2. Ne pas charger la base partagée.
3. Ne pas ralentir l'affichage des pages.
4. Simplicité.

## Options

- A. Une écriture par visite, avant de servir la page. Satisfait le critère 1. Mesurée dans 0001 : tient
  1 000/s de façon irrégulière, une transaction par visite. Insuffisante pour 3 000/s.
- C. A avec `synchronous_commit = off`. **Écartée** : une panne de la base perd des transactions déjà
  confirmées, ce qui viole le critère 1.
- B. Les lots de 0001. **Écartée** : perte d'une seconde de visites par instance tuée.
- G. Une validation groupée : la visite attend la validation de son lot (quelques millisecondes). Satisfait
  le critère 1. **À mesurer** : latence ajoutée, transactions par seconde.
- B pour les pages ordinaires, A ou G pour les seules pages partenaires. **À évaluer** : la part des pages
  partenaires dans le trafic, et le coût de deux mécanismes.

## Modification de 0001

Seul son statut change : « remplacé par 0002 ». Son texte reste tel quel : il explique pourquoi B était le
bon choix dans l'ancien contexte.
```

- **Le contexte ne répète pas 0001** : il dit ce qui **change**, et renvoie au reste.
- **Le nouveau critère passe en premier.** C'est lui qui écarte C et B, quelles que soient leurs performances.
- **« À mesurer » est une réponse honnête.** Un ADR proposé peut contenir des inconnues, s'il dit comment les
  lever.
- **L'option mixte** (B pour les pages ordinaires, G pour les pages partenaires) est souvent oubliée : la
  contrainte nouvelle ne porte que sur une partie du trafic.

## E2 — Réversible ou non ?

| # | Décision | Qui est touché par un retour en arrière | Effort mérité |
|---|---|---|---|
| 1 | Intervalle d'écriture | personne : une constante | une heure, puis mesure en production |
| 2 | Nom d'une colonne lue par une autre équipe | l'autre équipe, ses requêtes et ses tableaux de bord | relecture par l'équipe concernée ; un renommage se fait comme pour le JSON (M6 E5) |
| 3 | `pgx` ou `database/sql` | le seul service, derrière son repository (N8 L7) | prototype d'une heure si besoin ; pas d'ADR |
| 4 | Format des événements publiés | trois équipes, et les messages déjà stockés dans la file | ADR, relecture large, plan d'évolution du format (M6) |
| 5 | `id` entier dans une API publique | tous les clients, qui le stockent et le comparent ; il révèle aussi le volume de l'activité | ADR : quasi irréversible une fois publié |
| 6 | Monolithe → deux services | l'équipe entière, le déploiement, l'exploitation, la cohérence des données | ADR, prototype, extraction progressive ; voir le défi |
| 7 | Bibliothèque de journalisation | tout le code, mais mécaniquement ; `slog.Handler` isole la sortie (N12 L6) | une heure ; `slog` par défaut |
| 8 | Chiffrement des sauvegardes | la sécurité de toutes les données passées ; les sauvegardes non chiffrées existent déjà | ADR et relecture sécurité : une sauvegarde volée ne se rattrape pas |

Le critère n'est pas la taille du changement, mais **qui d'autre** en dépend. Une décision coûte cher à défaire
dès qu'elle sort du service : une équipe, un client, une donnée stockée.

## E3 — Relire une PR

La revue :

> **Résumé.** Le cache est bienvenu : 40 ms par requête pour un classement qui change peu. L'interface est
> simple. Trois points bloquants, prouvés par les tests de `top/cache_test.go`, et deux à discuter.
>
> 1. **[bloquant] Ruée à l'expiration.** Entre le `RUnlock` et le `Lock`, chaque requête qui trouve le cache
>    expiré lance son propre calcul : 100 requêtes simultanées, 100 calculs de 40 ms en base, à chaque
>    expiration (test 1). Sous charge, la base reçoit une rafale à chaque seconde. Proposition : un seul
>    chargement à la fois, que les autres attendent ; et servir l'ancienne valeur pendant le rechargement.
> 2. **[bloquant] Le cache renvoie sa slice interne.** Un appelant qui trie ou modifie le résultat modifie le
>    cache pour tout le monde (test 2). Proposition : `slices.Clone`.
> 3. **[bloquant] Une base en panne est martelée.** Après une erreur, chaque requête relance le calcul : une
>    minute de panne, 5 900 calculs tentés et 5 900 erreurs (test 3). Le cache aggrave la panne qu'il devrait
>    amortir. Proposition : ne pas réessayer avant un délai, et servir la dernière valeur connue, tant qu'elle
>    n'est pas trop vieille.
> 4. **[à discuter] Le calcul utilise le contexte de l'appelant.** Un appelant pressé qui abandonne fait
>    échouer le calcul, que d'autres auraient pu utiliser. Dans la version corrigée, il devient visible avec un
>    chargement partagé : il faut un contexte propre au chargement, avec son délai (test 4).
> 5. **[à discuter] Servir une valeur périmée masque la panne.** Il faudrait une métrique de l'âge de la
>    valeur servie, et une alerte au-delà de quelques minutes.

```go
// Fichier : top/pr.go
// Paquet top : un cache devant le calcul du classement.
package top

import (
	"context"
	"sync"
	"time"
)

// PRCache est la version proposée dans la PR, avant la revue.
type PRCache struct {
	mu   sync.RWMutex
	val  []string
	at   time.Time
	ttl  time.Duration
	load func(context.Context) ([]string, error)
}

func NewPRCache(ttl time.Duration, load func(context.Context) ([]string, error)) *PRCache {
	return &PRCache{ttl: ttl, load: load}
}

// Get renvoie le classement, recalculé au plus une fois par ttl.
func (c *PRCache) Get(ctx context.Context) ([]string, error) {
	c.mu.RLock()
	if time.Since(c.at) < c.ttl {
		v := c.val
		c.mu.RUnlock()
		return v, nil
	}
	c.mu.RUnlock()
	v, err := c.load(ctx)
	if err != nil {
		return nil, err
	}
	c.mu.Lock()
	c.val, c.at = v, time.Now()
	c.mu.Unlock()
	return v, nil
}
```

```go
// Fichier : top/cache.go
package top

import (
	"context"
	"slices"
	"sync"
	"time"
)

// Cache : la version corrigée après la revue.
//   - un seul chargement à la fois, partagé par tous les appelants qui attendent ;
//   - une valeur périmée depuis moins de MaxStale est servie tout de suite, pendant le rechargement ;
//   - après un échec, pas de nouvel essai avant Retry : une base en panne n'est pas martelée ;
//   - chaque appelant reçoit sa propre copie.
type Cache struct {
	TTL         time.Duration // durée de fraîcheur
	MaxStale    time.Duration // au-delà, une valeur périmée n'est plus servie
	Retry       time.Duration // délai minimal entre deux chargements après un échec
	LoadTimeout time.Duration // délai d'un chargement

	load func(context.Context) ([]string, error)

	mu      sync.Mutex
	val     []string
	at      time.Time // date du dernier chargement réussi ; zéro : jamais
	lastErr error
	retryAt time.Time
	call    *call // chargement en cours, ou nil
}

type call struct {
	done chan struct{}
	val  []string
	err  error
}

func NewCache(ttl time.Duration, load func(context.Context) ([]string, error)) *Cache {
	return &Cache{TTL: ttl, MaxStale: 10 * time.Minute, Retry: time.Second, LoadTimeout: 5 * time.Second, load: load}
}

func (c *Cache) Get(ctx context.Context) ([]string, error) {
	c.mu.Lock()
	age := time.Since(c.at)
	fresh := !c.at.IsZero() && age < c.TTL
	usable := !c.at.IsZero() && age < c.MaxStale
	if fresh {
		v := c.val
		c.mu.Unlock()
		return slices.Clone(v), nil
	}
	if time.Now().Before(c.retryAt) { // un échec récent : ne pas recharger tout de suite
		v, err := c.val, c.lastErr
		c.mu.Unlock()
		if usable {
			return slices.Clone(v), nil
		}
		return nil, err
	}
	cl := c.call
	if cl == nil {
		cl = &call{done: make(chan struct{})}
		c.call = cl
		go c.refresh(cl)
	}
	stale := c.val
	c.mu.Unlock()
	if usable {
		return slices.Clone(stale), nil // servir l'ancienne valeur pendant le rechargement
	}
	select {
	case <-cl.done:
		if cl.err != nil {
			return nil, cl.err
		}
		return slices.Clone(cl.val), nil
	case <-ctx.Done():
		return nil, ctx.Err()
	}
}

// refresh charge avec son propre délai : l'annulation d'un appelant n'annule pas le chargement des autres.
func (c *Cache) refresh(cl *call) {
	ctx, cancel := context.WithTimeout(context.Background(), c.LoadTimeout)
	v, err := c.load(ctx)
	cancel()
	cl.val, cl.err = v, err
	c.mu.Lock()
	if err == nil {
		c.val, c.at, c.lastErr = v, time.Now(), nil
	} else {
		c.lastErr, c.retryAt = err, time.Now().Add(c.Retry)
	}
	c.call = nil
	c.mu.Unlock()
	close(cl.done)
}
```

```go
// Fichier : top/cache_test.go
package top

import (
	"context"
	"errors"
	"fmt"
	"sync"
	"sync/atomic"
	"testing"
	"testing/synctest"
	"time"
)

type getter interface {
	Get(context.Context) ([]string, error)
}

// db : le calcul du classement, 40 ms, qui peut être en panne.
type db struct {
	loads atomic.Int64
	down  atomic.Bool
}

func (d *db) load(ctx context.Context) ([]string, error) {
	d.loads.Add(1)
	select {
	case <-time.After(40 * time.Millisecond):
	case <-ctx.Done():
		return nil, ctx.Err()
	}
	if d.down.Load() {
		return nil, errors.New("base indisponible")
	}
	return []string{"/accueil", "/produits", "/contact"}, nil
}

var versions = []struct {
	name string
	mk   func(*db) getter
}{
	{"PR", func(d *db) getter { return NewPRCache(time.Second, d.load) }},
	{"corrigée", func(d *db) getter { return NewCache(time.Second, d.load) }},
}

// burst lance n Get simultanés et renvoie le nombre d'erreurs.
// settle laisse finir les chargements lancés en arrière-plan, avant la fin de la bulle.
func settle() { time.Sleep(time.Second); synctest.Wait() }

func burst(c getter, n int) (errs int64) {
	var wg sync.WaitGroup
	var e atomic.Int64
	for range n {
		wg.Go(func() {
			if _, err := c.Get(context.Background()); err != nil {
				e.Add(1)
			}
		})
	}
	wg.Wait()
	return e.Load()
}

func TestStampede(t *testing.T) {
	for _, v := range versions {
		synctest.Test(t, func(t *testing.T) {
			d := &db{}
			c := v.mk(d)
			burst(c, 100) // cache vide
			cold := d.loads.Load()
			time.Sleep(2 * time.Second) // expiration
			start := time.Now()
			burst(c, 100)
			fmt.Printf("1. %-9s 100 Get, cache vide : %3d calculs ; à l'expiration : %3d calculs, servis en %v\n",
				v.name, cold, d.loads.Load()-cold, time.Since(start))
			settle()
		})
	}
}

func TestAliasing(t *testing.T) {
	for _, v := range versions {
		synctest.Test(t, func(t *testing.T) {
			c := v.mk(&db{})
			r, _ := c.Get(context.Background())
			r[0] = "/piratée" // un appelant qui trie, filtre ou modifie « son » résultat
			r2, _ := c.Get(context.Background())
			fmt.Printf("2. %-9s après modification par un appelant, le cache contient : %s\n", v.name, r2[0])
			settle()
		})
	}
}

func TestOutage(t *testing.T) {
	for _, v := range versions {
		synctest.Test(t, func(t *testing.T) {
			d := &db{}
			c := v.mk(d)
			c.Get(context.Background())
			d.down.Store(true)
			before := d.loads.Load()
			var errs int
			for range 6000 { // une requête toutes les 10 ms pendant une minute de panne
				if _, err := c.Get(context.Background()); err != nil {
					errs++
				}
				time.Sleep(10 * time.Millisecond)
			}
			fmt.Printf("3. %-9s une minute de panne, 6000 Get : %4d calculs tentés, %4d erreurs\n",
				v.name, d.loads.Load()-before, errs)
			settle()
		})
	}
}

func TestCancelledCaller(t *testing.T) {
	for _, v := range versions {
		synctest.Test(t, func(t *testing.T) {
			d := &db{}
			c := v.mk(d)
			ctx, cancel := context.WithTimeout(context.Background(), 10*time.Millisecond)
			defer cancel()
			_, err1 := c.Get(ctx) // un appelant pressé abandonne
			time.Sleep(time.Second / 10)
			_, err2 := c.Get(context.Background())
			fmt.Printf("4. %-9s premier appelant : %v ; calculs pour deux Get : %d ; second appelant : erreur %v\n",
				v.name, err1, d.loads.Load(), err2)
			settle()
		})
	}
}
```

```
$ go test -v ./top/
1. PR        100 Get, cache vide : 100 calculs ; à l'expiration : 100 calculs, servis en 40ms
1. corrigée  100 Get, cache vide :   1 calculs ; à l'expiration :   1 calculs, servis en 0s
2. PR        après modification par un appelant, le cache contient : /piratée
2. corrigée  après modification par un appelant, le cache contient : /accueil
3. PR        une minute de panne, 6000 Get : 5900 calculs tentés, 5900 erreurs
3. corrigée  une minute de panne, 6000 Get :   57 calculs tentés,    0 erreurs
4. PR        premier appelant : context deadline exceeded ; calculs pour deux Get : 2 ; second appelant : erreur <nil>
4. corrigée  premier appelant : context deadline exceeded ; calculs pour deux Get : 1 ; second appelant : erreur <nil>
ok  	example.com/visits/top	0.025s
```

- **Les tests tournent en temps virtuel** (`synctest`) : une minute de panne dure quelques millisecondes. Pour
  cela, la version corrigée attend le chargement en cours par un **canal** (`cl.done`), et pas par un verrou.
  Dans une bulle, un `select` sur un canal bloque durablement, un `sync.Mutex` non. Le temps virtuel
  n'avancerait pas.
- **À l'expiration, la version corrigée sert tout de suite** (« en 0s ») l'ancienne valeur, et recharge en
  arrière-plan : aucun appelant n'attend les 40 ms.
- **Pendant la panne**, 57 calculs en une minute : un par seconde (`Retry`), et aucune erreur, puisque la
  valeur d'avant la panne a moins de 10 minutes (`MaxStale`).
- Le cache vide, lui, ne peut servir aucune valeur : là, les appelants attendent le chargement unique. S'il
  échoue, ils reçoivent l'erreur.

## E4 — La matrice qui décide trop vite

```go
// Fichier : cmd/matrix/main.go
// matrix : la sensibilité d'une matrice de décision pondérée (E4).
package main

import (
	"fmt"
)

var (
	criteria = []string{"base partagée", "latence", "capacité", "simplicité"}
	weights  = []float64{0.4, 0.3, 0.2, 0.1}
	options  = []string{"A", "C", "B"}
	scores   = [][]float64{ // une ligne par option, une colonne par critère
		{1, 2, 1, 5},
		{2, 4, 4, 5},
		{5, 5, 5, 2},
	}
)

func totals(w []float64) []float64 {
	t := make([]float64, len(options))
	for o := range options {
		for c := range criteria {
			t[o] += w[c] * scores[o][c]
		}
	}
	return t
}

func winner(w []float64) int {
	t := totals(w)
	best := 0
	for o := range t {
		if t[o] > t[best] {
			best = o
		}
	}
	return best
}

// shifted renvoie les poids où le critère i vaut wi ; les autres gardent leurs proportions, et la somme reste 1.
func shifted(i int, wi float64) []float64 {
	w := make([]float64, len(weights))
	scale := (1 - wi) / (1 - weights[i])
	for c := range weights {
		w[c] = weights[c] * scale
	}
	w[i] = wi
	return w
}

func main() {
	t := totals(weights)
	for o := range options {
		fmt.Printf("total %s = %.2f\n", options[o], t[o])
	}
	base := winner(weights)
	fmt.Printf("gagnante : %s\n\n", options[base])
	for i, name := range criteria {
		found := false
		// Chercher le plus petit écart, vers le haut ou vers le bas, qui change la gagnante.
		for step := 1; step <= 1000; step++ {
			d := float64(step) / 1000
			for _, wi := range []float64{weights[i] + d, weights[i] - d} {
				if wi < 0 || wi > 1 {
					continue
				}
				if w := winner(shifted(i, wi)); w != base {
					fmt.Printf("%-14s %.2f → %.3f (écart %+.3f) : %s gagne\n", name, weights[i], wi, wi-weights[i], options[w])
					found = true
					break
				}
			}
			if found {
				break
			}
		}
		if !found {
			fmt.Printf("%-14s %.2f : aucun poids entre 0 et 1 ne change la gagnante\n", name, weights[i])
		}
	}

	// Les notes, elles aussi, sont des jugements : quelle note, changée seule, renverse le résultat ?
	fmt.Println()
	for o := range options {
		for c, name := range criteria {
			orig := scores[o][c]
			for _, v := range []float64{1, 2, 3, 4, 5} {
				if v == orig {
					continue
				}
				scores[o][c] = v
				if w := winner(weights); w != base {
					fmt.Printf("note de %s en %-13s %.0f → %.0f : %s gagne\n", options[o], name, orig, v, options[w])
				}
			}
			scores[o][c] = orig
		}
	}
}
```

```
$ go run ./cmd/matrix
total A = 1.70
total C = 3.30
total B = 4.70
gagnante : B

base partagée  0.40 : aucun poids entre 0 et 1 ne change la gagnante
latence        0.30 : aucun poids entre 0 et 1 ne change la gagnante
capacité       0.20 : aucun poids entre 0 et 1 ne change la gagnante
simplicité     0.10 → 0.387 (écart +0.287) : C gagne

note de B en base partagée 5 → 1 : C gagne
```

- **Le poids le plus fragile est celui de la simplicité**, mais il faut presque le quadrupler (0,10 → 0,39)
  pour que C gagne. Les trois autres peuvent prendre n'importe quelle valeur : B domine sur les trois.
- **Une seule note, changée seule, renverse le résultat** : celle de B sur la base partagée, de 5 à 1. Toute
  la décision repose sur ce critère. Or cette note n'est pas un jugement : c'est la mesure du cours, 13
  transactions contre plus de 5 000. **La confiance est donc justifiée ici**, parce que le point fragile est
  mesuré.
- **La conclusion générale** : avant de croire un total, chercher de quoi il dépend. Si le résultat tient à
  une note donnée à l'estime, c'est cette note qu'il faut mesurer, pas le total qu'il faut défendre. Si deux
  options sont à quelques dixièmes l'une de l'autre, la matrice ne départage rien : la décision se prend sur
  le critère principal, ou sur la réversibilité.

## E5 — Recevoir une revue

1. **« Pourquoi pas Redis ? »** Une question, pas un défaut. *Réponse* : « Trois instances, un calcul par
   seconde chacune au plus (E3) : trois calculs de 40 ms par seconde, la base le supporte sans peine. Redis
   ajouterait une dépendance à exploiter et un aller-retour réseau, pour un gain que je ne vois pas. Je le
   reconsidérerais si les instances devaient servir exactement la même valeur au même instant. Tu as un cas en
   tête ? »
2. **`?limit=100000000`** : juste, et bloquant. *Réponse* : « Corrigé : `limit` est borné à 100, et une valeur
   invalide donne un 400. Test ajouté pour les deux cas. »
3. **`fetchTop` ou `loadTop`** : une préférence, non classée. *Réponse* : « Le paquet utilise déjà `load` pour
   les autres fonctions de chargement, je garde `loadTop` pour la cohérence. » Ou on renomme, si c'est la
   convention : ça ne vaut pas une discussion.
4. **Le test de panne** : juste. *Réponse* : « Ajouté : `TestOutage`, base en panne pendant une minute
   virtuelle. Il a d'ailleurs trouvé un défaut : les appels échouaient tous au lieu de servir la dernière
   valeur. »
5. **« Trop compliqué »** : inexploitable tel quel. *Réponse* : « Quelle partie ? Si c'est le chargement
   partagé, il répond à la ruée du test 1 ; je peux ajouter un commentaire, ou le simplifier si tu vois une
   façon plus simple d'éviter les 100 calculs. »
6. **« `sync.Map`, c'est plus rapide »** : refusé, avec un argument. *Réponse* : « La documentation de
   `sync.Map` la réserve à deux cas : des clés écrites une fois et lues souvent, ou des goroutines qui
   travaillent sur des clés disjointes. Ici, il n'y a pas de map : une slice remplacée en entier sous un
   verrou tenu quelques nanosecondes. Je ne vois pas de contention. Si tu en soupçonnes une, je fais volontiers
   un benchmark avec `-cpu 1,8`, et je change si le profil le montre. »

Ce qui fait une bonne réponse : chaque commentaire reçoit une réponse. Un désaccord s'appuie sur un fait (une
mesure, la documentation) et propose une façon de trancher. Une question vague appelle une question en retour.

---

## Exercice intermédiaire — Mesurer, puis décider

```go
// Fichier : visits/group.go
package visits

import (
	"context"
	"sync"
	"time"
)

// GroupCounter : une visite n'est confirmée qu'une fois écrite en base (ADR 0002).
// Les visites qui arrivent pendant une écriture forment le lot suivant : une transaction en valide
// des centaines (une validation groupée, faite par l'application).
type GroupCounter struct {
	store    Store
	interval time.Duration
	maxBatch int
	timeout  time.Duration

	mu     sync.Mutex
	cur    *group
	closed bool

	kick chan struct{} // le lot courant est plein
	stop chan struct{}
	done chan struct{}
}

type group struct {
	counts map[string]int64
	n      int
	done   chan struct{} // fermé quand le lot est écrit (ou a échoué)
	err    error
}

func newGroup() *group { return &group{counts: map[string]int64{}, done: make(chan struct{})} }

func NewGroupCounter(s Store, interval time.Duration, maxBatch int) *GroupCounter {
	c := &GroupCounter{
		store: s, interval: interval, maxBatch: maxBatch, timeout: 5 * time.Second,
		cur: newGroup(), kick: make(chan struct{}, 1), stop: make(chan struct{}), done: make(chan struct{}),
	}
	go c.loop()
	return c
}

// Visit compte une visite, et attend qu'elle soit écrite en base.
// Une erreur ctx.Err() ne dit PAS que la visite est perdue : elle peut être écrite quand même
// (la troisième issue d'un appel, M3). Seul nil garantit qu'elle est en base.
func (c *GroupCounter) Visit(ctx context.Context, page string) error {
	c.mu.Lock()
	if c.closed {
		c.mu.Unlock()
		return ErrClosed
	}
	g := c.cur
	g.counts[page]++
	g.n++
	if g.n >= c.maxBatch {
		select {
		case c.kick <- struct{}{}:
		default: // déjà signalé
		}
	}
	c.mu.Unlock()
	select {
	case <-g.done:
		return g.err
	case <-ctx.Done():
		return ctx.Err()
	}
}

func (c *GroupCounter) loop() {
	defer close(c.done)
	t := time.NewTicker(c.interval)
	defer t.Stop()
	for {
		select {
		case <-t.C:
		case <-c.kick:
		case <-c.stop:
			c.flush()
			return
		}
		c.flush()
	}
}

// flush écrit le lot courant. Une seule écriture à la fois : pendant qu'elle dure, le lot suivant grossit.
func (c *GroupCounter) flush() {
	c.mu.Lock()
	g := c.cur
	if g.n == 0 {
		c.mu.Unlock()
		return
	}
	c.cur = newGroup()
	c.mu.Unlock()
	ctx, cancel := context.WithTimeout(context.Background(), c.timeout)
	g.err = c.store.Add(ctx, g.counts)
	cancel()
	close(g.done) // publie g.err : la fermeture du canal ordonne l'écriture avant les lectures
}

// Close refuse les nouvelles visites, écrit les visites en attente, et rend la main quand c'est fait.
func (c *GroupCounter) Close() {
	c.mu.Lock()
	if c.closed {
		c.mu.Unlock()
		return
	}
	c.closed = true
	c.mu.Unlock()
	close(c.stop)
	<-c.done
}
```

```go
// Fichier : visits/group_test.go
package visits

import (
	"context"
	"errors"
	"sync"
	"sync/atomic"
	"testing"
	"testing/synctest"
	"time"
)

type memStore struct {
	mu    sync.Mutex
	total int64
	down  atomic.Bool
	calls atomic.Int64
}

func (s *memStore) Add(ctx context.Context, counts map[string]int64) error {
	s.calls.Add(1)
	time.Sleep(3 * time.Millisecond) // une validation en base
	if s.down.Load() {
		return errors.New("base indisponible")
	}
	s.mu.Lock()
	defer s.mu.Unlock()
	for _, n := range counts {
		s.total += n
	}
	return nil
}

func (s *memStore) sum() int64 {
	s.mu.Lock()
	defer s.mu.Unlock()
	return s.total
}

// Aucune visite n'est confirmée sans être en base.
func TestConfirmedMeansStored(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		s := &memStore{}
		c := NewGroupCounter(s, 5*time.Millisecond, 500)
		defer c.Close()
		var confirmed atomic.Int64
		var wg sync.WaitGroup
		for i := range 5000 {
			wg.Go(func() {
				time.Sleep(time.Duration(i%50) * time.Millisecond)
				if err := c.Visit(context.Background(), "/a"); err != nil {
					t.Error(err)
					return
				}
				if n := confirmed.Add(1); s.sum() < n {
					t.Errorf("%d visites confirmées, %d en base", n, s.sum())
				}
			})
		}
		wg.Wait()
		t.Logf("5000 visites, %d écritures en base", s.calls.Load())
	})
}

// Une panne de la base renvoie une erreur à chaque visite du lot.
func TestOutageFailsWholeBatch(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		s := &memStore{}
		s.down.Store(true)
		c := NewGroupCounter(s, 5*time.Millisecond, 500)
		defer c.Close()
		var failed atomic.Int64
		var wg sync.WaitGroup
		for range 100 {
			wg.Go(func() {
				if err := c.Visit(context.Background(), "/a"); err != nil {
					failed.Add(1)
				}
			})
		}
		wg.Wait()
		if failed.Load() != 100 || s.sum() != 0 {
			t.Errorf("%d erreurs sur 100, %d en base", failed.Load(), s.sum())
		}
	})
}

// Close attend les visites en cours ; celles qui ont réussi sont en base.
func TestCloseWaits(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		s := &memStore{}
		c := NewGroupCounter(s, time.Hour, 1_000_000) // aucune écriture avant Close
		var ok, closed atomic.Int64
		var wg sync.WaitGroup
		for range 100 {
			wg.Go(func() {
				switch err := c.Visit(context.Background(), "/a"); {
				case err == nil:
					ok.Add(1)
				case errors.Is(err, ErrClosed):
					closed.Add(1)
				default:
					t.Error(err)
				}
			})
		}
		synctest.Wait() // les 100 visites attendent leur lot
		c.Close()
		wg.Wait()
		if ok.Load() != 100 || s.sum() != 100 {
			t.Errorf("confirmées %d, refusées %d, en base %d", ok.Load(), closed.Load(), s.sum())
		}
		if err := c.Visit(context.Background(), "/a"); !errors.Is(err, ErrClosed) {
			t.Errorf("Visit après Close : %v", err)
		}
	})
}
```

```
$ go test -race -v -run 'Confirmed|Outage|CloseWaits' ./visits/
    group_test.go:61: 5000 visites, 13 écritures en base
--- PASS: TestConfirmedMeansStored (0.07s)
--- PASS: TestOutageFailsWholeBatch (0.00s)
--- PASS: TestCloseWaits (0.00s)
PASS
ok  	example.com/visits/visits	1.117s
```

Le banc du cours reçoit l'option `-mode group` :

```go
// Fichier : cmd/bench/main.go
// bench : mesurer les deux options sur PostgreSQL, en boucle ouverte (N11 L7).
package main

import (
	"context"
	"flag"
	"fmt"
	"log"
	"math/rand/v2"
	"os"
	"slices"
	"sync"
	"time"

	"example.com/visits/visits"
	"github.com/jackc/pgx/v5/pgxpool"
)

func main() {
	mode := flag.String("mode", "direct", "direct (option A), batch (option B) ou group (option G)")
	rate := flag.Int("rate", 1000, "visites par seconde")
	dur := flag.Duration("d", 5*time.Second, "durée")
	flag.Parse()
	ctx := context.Background()
	pool, err := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatal(err)
	}
	defer pool.Close()
	if _, err := pool.Exec(ctx, visits.Schema+"; TRUNCATE visits"); err != nil {
		log.Fatal(err)
	}
	store := visits.PGStore{Pool: pool}
	var visit func(string) error
	closeFn := func() {}
	switch *mode {
	case "direct":
		visit = func(p string) error { return store.Increment(ctx, p) }
	case "batch":
		bc := visits.NewBatchCounter(store, time.Second, 100_000)
		visit, closeFn = bc.Visit, bc.Close
	case "group":
		gc := visits.NewGroupCounter(store, 5*time.Millisecond, 500)
		visit = func(p string) error { return gc.Visit(ctx, p) }
		closeFn = gc.Close
	default:
		log.Fatalf("mode inconnu : %s", *mode)
	}

	// 1 000 pages, très inégalement visitées : la page k reçoit une part proportionnelle à 1/k.
	zipf := rand.NewZipf(rand.New(rand.NewPCG(1, 2)), 1.1, 1, 999)
	var mu sync.Mutex
	var lat []float64
	var errs int
	var wg sync.WaitGroup
	interval := time.Second / time.Duration(*rate)
	start := time.Now()
	for i := 0; ; i++ {
		planned := start.Add(time.Duration(i) * interval)
		if planned.Sub(start) >= *dur {
			break
		}
		time.Sleep(time.Until(planned))
		page := fmt.Sprintf("/page/%d", zipf.Uint64())
		wg.Go(func() {
			err := visit(page)
			d := float64(time.Since(planned).Microseconds()) / 1000
			mu.Lock()
			defer mu.Unlock()
			if err != nil {
				errs++
				return
			}
			lat = append(lat, d)
		})
	}
	wg.Wait()
	closeFn() // options B et G : écrire le dernier lot
	var inDB int64
	if err := pool.QueryRow(ctx, "SELECT coalesce(sum(n), 0) FROM visits").Scan(&inDB); err != nil {
		log.Fatal(err)
	}
	slices.Sort(lat)
	q := func(p float64) float64 { return lat[int(p*float64(len(lat)-1))] }
	fmt.Printf("%-6s %5d/s  conn. %2d : p50 %8.2f ms  p99 %8.2f ms  erreurs %d  visites %6d  en base %6d\n",
		*mode, *rate, pool.Stat().MaxConns(), q(0.5), q(0.99), errs, len(lat), inDB)
}
```

```
$ ./bench -mode group -rate 1000     (transactions : delta de xact_commit, comme dans le cours)
group   1000/s  conn.  6 : p50     5.90 ms  p99     9.65 ms  erreurs 0  visites   5000  en base   5000
   transactions : 1007
group   5000/s  conn.  6 : p50     6.37 ms  p99    10.56 ms  erreurs 0  visites  25000  en base  25000
   transactions : 1007
group  20000/s  conn.  6 : p50     7.41 ms  p99    12.32 ms  erreurs 0  visites 100000  en base 100000
   transactions : 1004
group   1000/s  conn.  6 : p50     6.09 ms  p99     9.58 ms  erreurs 0  visites   5000  en base   5000
   transactions : 1007
group   5000/s  conn.  6 : p50     6.63 ms  p99    10.65 ms  erreurs 0  visites  25000  en base  25000
   transactions : 1007
group  20000/s  conn.  6 : p50     7.88 ms  p99    16.66 ms  erreurs 0  visites 100000  en base 100000
   transactions : 992
```

- **G tient 20 000 visites/s** avec un p99 de 12 à 17 ms, et **aucune** visite confirmée hors de la base. A ne
  tenait pas 2 000/s.
- **Environ 200 transactions par seconde, quel que soit le débit** : une par période de 5 ms. Le lot grossit
  avec le trafic, le nombre de transactions ne bouge pas.
- **Le coût par rapport à B** :
  - 6 à 8 ms de latence médiane sur le chemin de chaque page, contre moins d'une milliseconde ;
  - 200 transactions par seconde et par instance, contre une ;
  - un comportement en panne plus dur : quand la base ne répond pas, les **pages** attendent ou échouent,
    alors qu'avec B elles s'affichaient et seul le comptage prenait du retard.
- **Le test `TestConfirmedMeansStored`** vérifie la propriété, pas un exemple : à chaque confirmation, le
  nombre de visites en base est au moins le nombre de visites confirmées.
- **La troisième issue** (M3) : une visite dont le contexte expire peut être écrite quand même. `Visit` le dit
  dans sa documentation ; pour la facturation, c'est une sur-facturation possible, à mentionner dans l'ADR.

La fin de l'ADR 0002 :

```markdown
## Mesures (cmd/bench, PostgreSQL 17, machine de développement)

| | A | G (lots de 5 ms) |
|---|---|---|
| Tient (p99 < 50 ms) | 1 000/s, irrégulier | 20 000/s (p99 12-17 ms) |
| Transactions par seconde | une par visite | ≈ 200 par instance, quel que soit le débit |
| Latence médiane ajoutée à la page | 3 ms | 6 à 8 ms |

## Décision

G pour toutes les pages, période de 5 ms. Statut : accepté, 2027-05-11.

- Le critère 1 écarte B et C.
- A ne tient pas 3 000 visites/s.
- L'option mixte (B + G) garderait deux mécanismes pour une économie de transactions modeste : avec trois
  instances, G impose 600 transactions par seconde à la base partagée, contre 3 000 pour A.

## Conséquences

- 6 à 8 ms de plus sur l'affichage de chaque page.
- 600 transactions par seconde sur la base partagée (trois instances). À surveiller dans le tableau de bord
  de la base.
- Si la base est indisponible, les pages ne s'affichent plus (elles attendent la validation de leur visite,
  au plus 5 s). C'est le prix de la facturation exacte. Les pages non facturées pourraient être exemptées : voir
  Réexamen.
- Une visite abandonnée par son contexte peut quand même être écrite : sur-facturation possible, rare
  (délai de 5 s). Les partenaires en sont informés.

## Réexamen

- Si la charge de la base devient un problème : allonger la période (10 ms : 100 transactions par seconde et
  par instance), ou revenir à B pour les pages non facturées.
- Si la latence ajoutée dépasse le budget des pages.
```

---

## Défi — Relire une conception

**La revue de conception.**

> **Résumé.** Le document décrit deux vrais problèmes : la latence aux heures de pointe, et des déploiements
> qui cassent des fonctions sans rapport. La solution proposée ne traite ni l'un ni l'autre, et ajoute des
> risques graves. Je recommande de ne pas l'adopter en l'état, et propose une autre voie (ADR 0003 ci-dessous).
>
> **Questions préalables.**
> 1. Qu'est-ce qui ralentit entre 12 h et 14 h ? Profil CPU du monolithe, temps des requêtes SQL
>    (`pg_stat_statements`), attente de connexions au pool, saturation de la base ? Le document ne le dit pas.
> 2. Quel est l'objectif chiffré ? Un p99 de 200 ms ? À quel trafic, avec quelle croissance ?
> 3. Comment la modification du catalogue a-t-elle cassé le paiement : un test manquant, un couplage par les
>    tables, une dépendance de paquet ?
>
> **Risques, du plus grave au moins grave.**
> 1. **[bloquant] La commande devient une transaction répartie, sans atomicité.** Elle enchaîne trois appels
>    HTTP : le prix, le paiement, puis le stock. Si le paiement réussit et que la décrémentation échoue (délai
>    dépassé, panne, déploiement), le client a payé un article qui n'est pas réservé. Si l'appel est réessayé,
>    le stock peut être décrémenté deux fois (M3 : la troisième issue). Le monolithe le fait aujourd'hui dans
>    une transaction SQL. Il faudrait des clés d'idempotence, une compensation (rembourser) et un outbox
>    (N10 L8, L9). Le document n'en dit rien.
> 2. **[bloquant] Le problème de latence n'est pas traité.** Si la base sature, quatre services devant la même
>    base la saturent autant, avec quatre pools de connexions au lieu d'un (M7 : ajouter des instances devant
>    une base saturée). Et une page qui appelle plusieurs services attend le plus lent (N11 L7) : le p99 se
>    dégrade.
> 3. **[bloquant] La base partagée garde le couplage.** Quatre services qui partagent les mêmes tables forment
>    un monolithe distribué. Une migration du schéma du catalogue peut toujours casser le paiement, avec en plus
>    des déploiements à coordonner. « On séparera les bases plus tard » est la partie difficile du projet.
> 4. **[bloquant] La bascule d'un seul coup, un dimanche soir, sans retour arrière.** Rien ne dit comment
>    revenir au monolithe si la commande échoue lundi matin.
> 5. **[à discuter] Quatre services pour quatre développeurs.** Chacun devient seul responsable d'un service,
>    de son astreinte, de son pipeline, de ses tableaux de bord. Les congés deviennent un risque.
> 6. **[à discuter] L'exploitation n'est pas prévue** : traces réparties (N12 L7), délais et réessais entre
>    services (N10 L9), contrats d'API entre équipes (M6).
> 7. **[détail]** « La norme dans l'industrie » n'est pas un argument : il ne dit rien de ce système.
>
> **Informations manquantes.** La mesure de la cause du ralentissement. L'objectif chiffré. Le coût (les
> trois mois de quatre développeurs, et leur exploitation). Le traitement des pannes partielles de la commande.
> La migration des données, et le plan de retour arrière.

```markdown
# 0003 — Rester en monolithe modulaire ; traiter la latence et les déploiements à leur source

Statut : proposé, en réponse à la proposition « microservices ».

## Contexte

- p99 des pages : 80 ms hors pointe, 900 ms entre 12 h et 14 h. Cause non mesurée.
- Une modification du catalogue a cassé le paiement le mois dernier.
- Quatre développeurs, une base PostgreSQL, un monolithe Go.

## Critères, dans l'ordre

1. Ne pas mettre en danger la cohérence des commandes et des paiements.
2. Ramener le p99 de pointe sous un objectif à fixer (proposé : 200 ms).
3. Empêcher qu'un changement d'un domaine casse un autre domaine.
4. Coût et risque de la transition, pour quatre personnes.

## Options

- **Microservices** (la proposition). Ne traite pas la cause de la latence, tant qu'elle n'est pas mesurée.
  Rend la commande répartie (critère 1). Coût : un trimestre, puis l'exploitation de quatre services.
- **Monolithe modulaire** (N10 L7). Des frontières imposées dans le code entre catalogue, commandes, paiement et
  comptes : paquets `internal`, interfaces entre domaines, tables propres à chaque module. Avec des tests de
  contrat entre modules et un déploiement progressif (N12 L9). La commande reste une transaction SQL.
- **Ne rien changer à l'architecture**, et seulement corriger la latence. Ne traite pas le critère 3.

## Décision proposée

Monolithe modulaire, en trois étapes :
1. **Mesurer la pointe** : profils, requêtes lentes, pool de connexions. Corriger la cause trouvée.
2. **Imposer les frontières** entre domaines, une par une, en commençant par le paiement, et ajouter les tests
   de contrat qui auraient attrapé l'incident du mois dernier.
3. **Déployer progressivement** : une instance d'abord, puis toutes.

## Conséquences

- Un seul déploiement pour tous les domaines : un défaut du catalogue peut encore bloquer une mise en
  production. Mais les tests de contrat le trouvent avant, et le déploiement progressif en limite l'effet.
- La mise à l'échelle reste globale : on ajoute des instances du monolithe, pas d'un seul domaine.
- La cohérence de la commande reste garantie par la base.

## Réexamen

- Si un domaine a un besoin de mise à l'échelle ou de rythme de déploiement **mesuré** et différent des
  autres : extraire ce seul module en service. Ses frontières, déjà en place, rendent l'extraction progressive
  possible.
```

- **La revue commence par des questions** : le document ne donne pas la cause du problème qu'il veut résoudre.
  Une conception qui ne mesure pas sa cause ne peut pas montrer qu'elle la traite.
- **Les risques sont classés par gravité**, et le premier est celui qu'une revue de code ne verrait jamais : la
  perte de l'atomicité de la commande. C'est le travail d'une revue de **conception**.
- **La recommandation garde la porte ouverte** : le monolithe modulaire rend l'extraction d'un service plus
  facile plus tard. La décision irréversible (le découpage) est repoussée jusqu'à ce qu'une mesure la justifie.

---

## Quiz

1. Le contexte, les critères classés, les options, la décision, les conséquences et les conditions de
   réexamen. Le contexte est le plus important : c'est lui qui change, et qui dit quand la décision n'est plus
   valable.
2. Pour garder l'historique des raisons : un nouvel ADR remplace l'ancien, qui reste lisible avec le statut
   « remplacé par ».
3. Par le coût du retour en arrière, c'est-à-dire par qui d'autre dépend de la décision : une équipe, un
   client, une donnée stockée.
4. Parce qu'un désaccord sur les options est souvent un désaccord sur l'ordre des critères. Le régler d'abord
   évite un débat sans fin.
5. « Ne rien faire » oblige à justifier le changement. L'option ennuyeuse, celle que l'équipe connaît, est la
   référence qu'une option nouvelle doit battre sur un critère important.
6. Sa précision fausse : le total dépend de notes et de poids donnés à l'estime. On cherche de quoi dépend le
   résultat (E4), plutôt que de croire le total.
7. Le bon changement (la conception), puis la correction (concurrence, erreurs, cycle de vie, ressources), les
   contrats, l'exploitation, les tests, la lisibilité. Le style revient aux outils.
8. Pour que l'auteur sache ce qui doit changer avant la fusion, ce qui mérite une discussion, et ce qu'il peut
   ignorer.
9. Répondre avec un argument ou une mesure, et proposer un moyen de trancher (un benchmark, un test) ; ne pas
   céder par politesse, ni refuser par autorité.
10. Un verrou tenu pendant une entrée-sortie, une goroutine sans arrêt, une erreur qui perd des données, un
    contexte sans délai, une mémoire non bornée, un ordre de parcours de map qui compte.
