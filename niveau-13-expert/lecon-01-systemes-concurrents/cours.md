# Module 1 — Systèmes concurrents

> Le niveau 6 a donné les briques : goroutines, canaux, verrous, `context`, et les pathologies à éviter.
> Ce module change d'échelle : comment **concevoir** un composant concurrent dont on peut affirmer qu'il est
> correct, au lieu d'espérer qu'il l'est parce que les tests passent. La méthode tient en trois gestes :
> - écrire l'**état** et ses **invariants** avant le code ;
> - **justifier** par une preuve informelle que chaque opération les préserve et que personne n'attend
>   indéfiniment ;
> - **tester** ces invariants, pas seulement des exemples.
>
> Le module le fait sur un pool de ressources borné. Il montre qu'un défaut de conception passe à travers
> `-race` **et** à travers les invariants internes, et qu'il faut un invariant qui relie le composant au
> monde extérieur pour le voir.

## Objectifs

- Décrire un composant concurrent par son état, ses invariants et ses propriétés de progrès.
- Raisonner avec le modèle mémoire de Go : sections critiques, relations « arrivé-avant », transfert de
  propriété.
- Écrire une preuve informelle : chaque opération préserve les invariants, chaque attente se termine.
- Repérer les fenêtres de course **logiques**, celles que le détecteur de races ne voit pas.
- Tester les invariants sous contrainte, y compris ceux qui portent sur l'extérieur du composant.

## Explication

### L'état d'abord, le code ensuite

Avant d'écrire une ligne, on écrit **ce que le composant mémorise** et **ce qui doit toujours être vrai**.
Pour un pool qui prête au plus `max` ressources coûteuses (des connexions, par exemple) :

```
État (protégé par mu) :
  open     ressources créées et non détruites
  idle     ressources libres
  lent     ressources prêtées
  creating créations en cours (faites hors du verrou)
  waiters  appelants qui attendent, dans l'ordre d'arrivée

Invariants (vrais chaque fois que mu est libre) :
  I1  open == len(idle) + lent + creating
  I2  open <= max
  I3  len(waiters) > 0  ⇒  len(idle) == 0
  I4  une ressource prêtée l'est à un seul appelant
  I5  lent == nombre d'appelants qui tiennent réellement une ressource
```

- **I1** dit qu'aucune ressource n'est perdue ni comptée deux fois.
- **I2** est la raison d'être du pool.
- **I3** est une propriété d'**efficacité** : personne n'attend pendant qu'une ressource dort.
- **I4** est la sûreté vue par les appelants.
- **I5** relie le pool à l'**extérieur** : c'est celui que l'exemple trouvera violé.

Écrire ces lignes oblige à des décisions qu'on aurait sinon prises sans les voir. Que vaut `open` pendant
une création ? D'où le compteur `creating`. Qui compte un prêt fait directement à un attendant ? `Put`, au nom
de l'attendant.

Il y a aussi des propriétés de **progrès**, qui disent que quelque chose finit par arriver :
- **P1** : un appelant qui attend finit par recevoir une ressource, ou par voir son `ctx` annulé ;
- **P2** : un appelant n'est jamais servi avant un appelant arrivé plus tôt, sauf annulation (équité FIFO).

### Les outils du raisonnement

**La section critique.** Entre `mu.Lock()` et `mu.Unlock()`, un invariant peut être **temporairement** faux
(on a décrémenté `lent`, pas encore ajouté à `idle`). L'important est qu'il soit vrai à chaque `Unlock`. Pour
prouver un invariant, on vérifie donc, pour chaque section critique : « s'il était vrai en entrant, il est
vrai en sortant ».

**Les relations « arrivé-avant »** du modèle mémoire de Go (go.dev/ref/mem, la référence officielle) :
- un `Unlock` arrive avant le `Lock` suivant ;
- un envoi sur un canal arrive avant la fin de la réception correspondante ;
- la fermeture d'un canal arrive avant une réception qui voit le canal fermé.

Ce qui a été écrit avant l'un est visible après l'autre. Sans une telle relation, deux accès à la même
variable, dont l'un en écriture, forment une **course de données** : le détecteur de races la signale, et le
programme n'a plus de sens défini.

**Le transfert de propriété.** Une ressource envoyée sur un canal **change de propriétaire**. L'expéditeur
ne doit plus y toucher, et le destinataire en est seul responsable. C'est ce qui rend sûr `w <- r` dans
`Put` : après l'envoi, le pool ne touche plus à `r`. C'est aussi ce qui crée le piège de l'annulation (plus
bas). Si le destinataire disparaît, la ressource reste dans le canal, sans propriétaire.

**Ne jamais bloquer en tenant le verrou.** Un blocage sous verrou bloque tout le composant, et c'est la
première cause d'interblocage (N6 L10). `Put` envoie sur `w` en tenant `mu`. C'est sûr parce que `w` a une
capacité de 1, et que **seul** le `Put` qui a retiré `w` de la file peut y envoyer : `w` ne peut être plein.
Cette justification doit figurer en commentaire. C'est elle qu'un relecteur doit pouvoir vérifier.

### La preuve informelle

Pour chaque opération et chaque chemin, on montre que I1 à I3 sont préservés :

| Chemin | Effet | I1 | I2 | I3 |
|---|---|---|---|---|
| `Get`, une ressource libre | `idle` −1, `lent` +1 | conservé | inchangé | `idle` diminue : ne peut pas devenir faux |
| `Get`, de la place | `open` +1, `creating` +1, puis `creating` −1 et `lent` +1, ou `open` −1 si la création échoue | conservé à chaque `Unlock` | `open < max` vérifié sous le verrou | `idle` inchangé et vide |
| `Get`, attente | `waiters` +1 | inchangé | inchangé | on n'attend que si `idle` est vide |
| `Put`, un attendant | `lent` −1 puis +1, `waiters` −1 | conservé | inchangé | `idle` reste vide |
| `Put`, personne | `lent` −1, `idle` +1 | conservé | inchangé | `waiters` est vide, sinon on serait dans le cas précédent |

Puis les **attentes**. Un appelant qui attend sort de son `select` par l'une des deux branches. Le cas délicat
est l'**annulation**, parce qu'elle peut **croiser** une remise :

1. le `ctx` est annulé, l'appelant sort du `select` par `ctx.Done()` ;
2. **entre-temps**, ou juste avant, un `Put` a retiré `w` de la file et y a déposé une ressource, en la
   comptant comme prêtée (`lent` +1).

Les deux cas sont distingués **sous le verrou**. Si `w` est encore dans `waiters`, personne ne l'a servi : on
le retire et on part. Si `w` n'y est plus, c'est qu'un `Put` l'a choisi : la ressource est **dans** `w`, et on
la rend aussitôt (`putLocked(<-w)`). La réception ne bloque pas, puisque la ressource a été déposée avant que
`w` ne quitte la file, sous le même verrou.

La version erronée (`BuggyCancel`) oublie le second cas. Elle rend `ctx.Err()` et abandonne `w`. La
ressource reste dans un canal que plus personne ne lira, et `lent` la compte comme prêtée **pour toujours**.

### Ce que les tests voient, et ce qu'ils ne voient pas

L'exemple soumet les deux versions à 32 emprunteurs impatients (délais de 0 à 300 µs) pendant une seconde,
avec un vérificateur qui appelle `Check` (I1 à I3) en continu, sous `-race`.

| | Version correcte | Version erronée |
|---|---|---|
| détecteur de races | rien | **rien** |
| violations de I1 à I4 pendant la charge | 0 | **0** |
| état final, plus personne ne tenant de ressource | `open=4 idle=4 prêtées=0` | `open=4 idle=0 prêtées=4` |
| un `Get` patient (200 ms) ensuite | réussit | `context deadline exceeded` : **le pool est mort** |

- **Le détecteur de races ne voit rien** : il n'y a **pas** de course de données. Tous les accès sont
  protégés, et la ressource passe par un canal. Le défaut est une course **logique** : un ordre
  d'événements que la conception n'a pas prévu.
- **Les invariants internes restent vrais** : le pool compte de façon parfaitement cohérente 4 ressources
  prêtées. Il a seulement tort sur le monde.
- **Seul I5 le révèle**, l'invariant qui compare le pool aux appelants. On ne peut le vérifier qu'à un
  moment où l'on connaît la vérité extérieure. Ici, c'est la fin du test, quand plus personne ne tient rien :
  `lent` doit valoir 0.

D'où la règle : **chercher les invariants qui relient le composant à l'extérieur**. Exemples : des octets lus
égaux aux octets envoyés, des messages acquittés égaux aux messages traités, des ressources prêtées égales
aux détenteurs. On les vérifie au **repos** (fin de test, arrêt), et en production par une métrique : un pool
dont `lent` ne redescend jamais la nuit fuit.

### Choisir entre canaux et verrous

| Situation | Outil naturel |
|---|---|
| un **état partagé** avec des invariants sur plusieurs variables | un `sync.Mutex`, une section critique par opération |
| **transférer** une valeur et sa responsabilité | un canal |
| **attendre** un événement, avec annulation | un canal dans un `select`, avec `ctx.Done()` |
| un compteur ou un drapeau isolé | `sync/atomic` (N6 L7) |

Le pool combine les deux : un verrou pour l'état et ses invariants, un canal par attendant pour la remise et
l'annulation. Un pool écrit **seulement** avec des canaux (un canal de ressources bufferisé de `max`) est plus
court. Mais il ne sait pas créer à la demande, ni être équitable, ni compter ce qui est prêté. Le choix
dépend des invariants qu'on veut tenir.

### Les limites

Une preuve informelle peut être fausse, et un test de stress peut ne pas tomber sur le mauvais
entrelacement. Ils se complètent, sans garantie absolue. Pour un composant **critique** (un verrou distribué,
un protocole), il existe des outils de vérification de modèles. Ils explorent **tous** les entrelacements
d'une description abstraite du protocole, et c'est un investissement qui se justifie rarement pour du code
d'application. Le minimum professionnel est celui de ce module : invariants écrits, preuve en commentaire,
tests de stress sous `-race` avec vérification des invariants, y compris extérieurs.

## Exemple

```go.mod
// Fichier : go.mod
module example.com/pool

go 1.27
```

```go
// Fichier : pool/pool.go
// Package pool : un pool borné de ressources (connexions, clients…), conçu à partir de ses invariants.
//
// État, protégé par mu :
//
//	open     : ressources créées et non détruites
//	idle     : ressources libres, prêtes à être prêtées
//	lent     : ressources prêtées (rendues à un appelant de Get, pas encore rendues par Put)
//	waiters  : appelants de Get qui attendent, dans l'ordre d'arrivée
//
// Invariants, vrais chaque fois que mu est libre :
//
//	I1  open == len(idle) + lent + creating      (aucune ressource n'est perdue ni comptée deux fois)
//	I2  open <= max                              (la borne est respectée)
//	I3  len(waiters) > 0  ⇒  len(idle) == 0      (personne n'attend s'il y a une ressource libre)
//	I4  une ressource prêtée l'est à un seul appelant à la fois
package pool

import (
	"context"
	"errors"
	"sync"
)

type Pool[T any] struct {
	mu       sync.Mutex
	max      int
	open     int
	creating int // créations en cours, hors verrou
	lent     int
	idle     []T
	waiters  []chan T // chacun bufferisé de 1 : Put n'y bloque jamais
	create   func(context.Context) (T, error)

	// BuggyCancel reproduit une erreur de conception (voir le cours) : à l'annulation, l'appelant
	// abandonne son canal sans vérifier qu'une ressource ne lui a pas déjà été remise.
	BuggyCancel bool
}

func New[T any](max int, create func(context.Context) (T, error)) *Pool[T] {
	return &Pool[T]{max: max, create: create}
}

// Get prête une ressource, ou attend qu'il y en ait une, ou échoue à l'annulation de ctx.
func (p *Pool[T]) Get(ctx context.Context) (T, error) {
	var zero T
	p.mu.Lock()
	if n := len(p.idle); n > 0 { // une ressource libre : on la prend
		r := p.idle[n-1]
		p.idle = p.idle[:n-1]
		p.lent++
		p.mu.Unlock()
		return r, nil
	}
	if p.open < p.max { // de la place : on crée, HORS du verrou (la création peut être lente)
		p.open++
		p.creating++
		p.mu.Unlock()
		r, err := p.create(ctx)
		p.mu.Lock()
		p.creating--
		if err != nil {
			p.open--
			p.mu.Unlock()
			return zero, err
		}
		p.lent++
		p.mu.Unlock()
		return r, nil
	}
	w := make(chan T, 1) // sinon : on attend
	p.waiters = append(p.waiters, w)
	p.mu.Unlock()

	select {
	case r := <-w:
		return r, nil // Put a déjà compté le prêt (lent++) pour nous
	case <-ctx.Done():
	}
	if p.BuggyCancel {
		return zero, ctx.Err() // ERREUR : une ressource a peut-être déjà été déposée dans w
	}
	p.mu.Lock()
	defer p.mu.Unlock()
	for i, x := range p.waiters {
		if x == w { // toujours en attente : personne ne nous a rien donné, on se retire
			p.waiters = append(p.waiters[:i], p.waiters[i+1:]...)
			return zero, ctx.Err()
		}
	}
	// Plus dans la file : Put nous a choisis entre-temps, la ressource est dans w. On la rend.
	p.putLocked(<-w)
	return zero, ctx.Err()
}

// Put rend une ressource prêtée.
func (p *Pool[T]) Put(r T) {
	p.mu.Lock()
	defer p.mu.Unlock()
	p.putLocked(r)
}

func (p *Pool[T]) putLocked(r T) {
	p.lent--
	if len(p.waiters) > 0 { // remise directe au plus ancien en attente (équité FIFO)
		w := p.waiters[0]
		p.waiters = p.waiters[1:]
		p.lent++ // prêtée à nouveau, au nom de l'attendant
		w <- r   // ne bloque pas : capacité 1, et w n'a qu'un seul expéditeur possible
		return
	}
	p.idle = append(p.idle, r)
}

// ErrInvariant signale un état incohérent ; Check n'est appelé que par les tests.
var ErrInvariant = errors.New("invariant violé")

// Check vérifie I1 à I3 sous le verrou. (I4 se vérifie côté appelants, dans le test.)
func (p *Pool[T]) Check() error {
	p.mu.Lock()
	defer p.mu.Unlock()
	switch {
	case p.open != len(p.idle)+p.lent+p.creating:
		return errors.Join(ErrInvariant, errors.New("I1 : open != idle + prêtées + en création"))
	case p.open > p.max:
		return errors.Join(ErrInvariant, errors.New("I2 : open > max"))
	case len(p.waiters) > 0 && len(p.idle) > 0:
		return errors.Join(ErrInvariant, errors.New("I3 : des appelants attendent alors qu'une ressource est libre"))
	}
	return nil
}

// Stats renvoie l'état, pour les métriques et les tests.
func (p *Pool[T]) Stats() (open, idle, lent, waiting int) {
	p.mu.Lock()
	defer p.mu.Unlock()
	return p.open, len(p.idle), p.lent, len(p.waiters)
}
```

```go
// Fichier : pool/pool_test.go
package pool

import (
	"context"
	"fmt"
	"math/rand/v2"
	"sync"
	"sync/atomic"
	"testing"
	"time"
)

type res struct {
	id      int
	holders atomic.Int32 // I4 : jamais plus d'un
}

// stress lance des emprunteurs impatients (délais aléatoires très courts) pendant d, et vérifie les
// invariants en continu. Il renvoie le nombre de violations et l'état final.
func stress(t *testing.T, p *Pool[*res], d time.Duration) (violations int64, open, idle, lent int) {
	var viol atomic.Int64
	stop := make(chan struct{})
	var wg sync.WaitGroup
	for g := range 32 {
		wg.Go(func() {
			rng := rand.New(rand.NewPCG(uint64(g), 1))
			for {
				select {
				case <-stop:
					return
				default:
				}
				ctx, cancel := context.WithTimeout(context.Background(), time.Duration(rng.IntN(300))*time.Microsecond)
				r, err := p.Get(ctx)
				cancel()
				if err != nil {
					continue
				}
				if r.holders.Add(1) != 1 {
					viol.Add(1) // I4 violé : deux appelants tiennent la même ressource
				}
				time.Sleep(time.Duration(rng.IntN(100)) * time.Microsecond)
				r.holders.Add(-1)
				p.Put(r)
			}
		})
	}
	wg.Go(func() { // le vérificateur
		for {
			select {
			case <-stop:
				return
			default:
			}
			if err := p.Check(); err != nil {
				viol.Add(1)
			}
		}
	})
	time.Sleep(d)
	close(stop)
	wg.Wait()
	open, idle, lent, _ = p.Stats()
	return viol.Load(), open, idle, lent
}

func newPool(buggy bool) *Pool[*res] {
	var n atomic.Int64
	p := New(4, func(context.Context) (*res, error) { return &res{id: int(n.Add(1))}, nil })
	p.BuggyCancel = buggy
	return p
}

func TestStress(t *testing.T) {
	p := newPool(false)
	viol, open, idle, lent := stress(t, p, time.Second)
	t.Logf("correct : violations %d ; fin : open=%d idle=%d prêtées=%d", viol, open, idle, lent)
	if viol != 0 || lent != 0 || open != idle {
		t.Fatal("état incohérent")
	}
}

func TestBuggyCancelLeaks(t *testing.T) {
	p := newPool(true)
	viol, open, idle, lent := stress(t, p, time.Second)
	t.Logf("erroné : violations %d ; fin : open=%d idle=%d prêtées=%d (aucun appelant ne tient plus rien)", viol, open, idle, lent)
	if lent == 0 {
		t.Skip("fuite non reproduite cette fois-ci")
	}
	// La conséquence : un Get patient finit par attendre indéfiniment, une fois toutes les ressources perdues.
	ctx, cancel := context.WithTimeout(context.Background(), 200*time.Millisecond)
	defer cancel()
	_, err := p.Get(ctx)
	fmt.Println("Get avec 200 ms de patience :", err)
}
```

```
$ go test -race -v ./pool/
    pool_test.go:77: correct : violations 0 ; fin : open=4 idle=4 prêtées=0
--- PASS: TestStress (1.00s)
    pool_test.go:86: erroné : violations 0 ; fin : open=4 idle=0 prêtées=4 (aucun appelant ne tient plus rien)
Get avec 200 ms de patience : context deadline exceeded
--- PASS: TestBuggyCancelLeaks (1.20s)
PASS
ok  	example.com/pool/pool	3.282s
```

(Trois exécutions ont donné les mêmes états finaux. La fuite de la version erronée est reproductible : il
suffit de quatre annulations qui croisent une remise pour vider le pool, et la charge en produit en moins
d'une seconde.)

## Explication du code

| Élément | Rôle |
|---|---|
| Le commentaire de paquet | État et invariants **écrits**, là où le relecteur les trouve. |
| `creating` | Sans lui, I1 serait faux pendant une création, faite hors du verrou parce qu'elle peut être lente. |
| `make(chan T, 1)` par attendant | Un canal **privé** : `Put` y dépose sans bloquer, et l'attendant le lit dans un `select` avec `ctx.Done()`. |
| `p.lent++` dans `putLocked` avant `w <- r` | Le prêt est compté **au moment de la remise**, sous le verrou, au nom de l'attendant. C'est ce qui rend l'annulation délicate. |
| La boucle sur `p.waiters` après l'annulation | Distingue « personne ne m'a servi » (on se retire) de « on m'a servi pendant que je partais » (on rend la ressource). |
| `BuggyCancel` | La même conception, sans le second cas. |
| `Check` | I1 à I3 vérifiés sous le verrou. Exporté pour les tests ; en production, les mêmes valeurs sortent en métriques (`Stats`). |
| `holders atomic.Int32` dans le test | Vérifie I4 côté appelants : jamais deux détenteurs à la fois. |
| L'état final de `stress` | I5 : au repos, `lent` doit valoir 0 et `open == idle`. |

## Erreurs fréquentes

1. **Écrire le code avant l'état et les invariants** : les cas limites sont découverts en production.
2. **Considérer que `-race` silencieux prouve la correction** : il ne voit que les courses de **données**.
3. **Oublier qu'une annulation peut croiser une remise** : `select` sur `ctx.Done()` sans traiter le cas « la
   valeur était déjà en route ».
4. **Bloquer en tenant un verrou**, sur un canal non bufferisé ou un appel réseau.
5. **Des invariants seulement internes** : le composant peut être cohérent avec lui-même et faux sur le
   monde.
6. **Des tests de stress sans vérification** : ils passent tant que rien ne panique, et ne prouvent rien.
7. **Créer ou détruire une ressource sous le verrou**, et sérialiser tout le pool derrière une connexion
   lente.

## Bonnes pratiques Go

- État et invariants en commentaire du type ; justification en commentaire de chaque opération non
  évidente (« ne bloque pas parce que… »).
- Un verrou pour un état à plusieurs variables ; des canaux pour transférer et attendre.
- Chaque `select` sur `ctx.Done()` se demande : « et si l'autre branche venait d'arriver ? ».
- Une méthode de vérification des invariants, appelée en continu par les tests de stress.
- Des invariants extérieurs vérifiés au repos dans les tests, et suivis en production par une métrique.
- `-race` toujours, en sachant ce qu'il ne voit pas.

## Ce que je dois retenir

- Concevoir un composant concurrent, c'est d'abord écrire son état, ses invariants et ses propriétés de
  progrès.
- Un invariant est vrai à chaque libération du verrou ; on le prouve section critique par section critique.
- Une annulation peut croiser une remise : c'est la course logique la plus courante, et `-race` ne la voit
  pas.
- Mesuré : une version erronée sans course de données, aux invariants internes intacts, perd ses quatre
  ressources en une seconde.
- Les invariants qui relient le composant à l'extérieur sont ceux qui trouvent ces bugs.

---

➡️ [Exercices](exercices.md)
