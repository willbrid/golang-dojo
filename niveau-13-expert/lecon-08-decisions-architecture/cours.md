# Module 8 — Décisions d'architecture

> Le code d'un système change chaque semaine. Ses grandes décisions restent pendant des années : le stockage,
> la forme des données, le découpage en services, ce qu'on accepte de perdre en cas de panne. Six mois plus
> tard, personne ne se souvient **pourquoi** elles ont été prises, ni dans quel contexte. On les défend alors
> par habitude, ou on les défait sans savoir ce qu'elles protégeaient.
>
> Ce module traite du travail d'un développeur senior sur ces décisions :
> - les **prendre** : critères explicites, options mesurées, conséquences écrites, y compris les mauvaises ;
> - les **consigner**, dans un ADR (*Architecture Decision Record*) ;
> - les **faire respecter** dans le code, par la revue.
>
> L'exemple suit une décision du début à la fin : trois options mesurées contre PostgreSQL, l'ADR qui en
> sort, puis la revue de la PR qui l'implémente, où chaque défaut signalé est prouvé par un test.

## Objectifs

- Écrire un ADR : contexte, critères, options, décision, conséquences, conditions de réexamen.
- Doser l'effort selon qu'une décision est réversible ou non.
- Arbitrer avec des critères explicites et des mesures, pas des préférences.
- Faire une revue de code senior : dans quel ordre regarder, comment écrire un commentaire, comment prouver un
  défaut.
- Recevoir une revue, et y répondre.

## Explication

### Pourquoi écrire une décision

Une décision d'architecture répond à un **contexte** : un trafic, une équipe, une contrainte métier, l'état
d'une dépendance. Le code garde la décision, mais pas son contexte. Sans trace écrite :
- **on la remet en cause sans fin** : chaque nouvel arrivant repose la question, et l'équipe refait le même
  débat ;
- **on la défait par erreur** : « pourquoi ne pas écrire directement en base ? » La réponse (la base est
  partagée, et la mesure l'a montré) n'est plus là pour arrêter le changement ;
- **on la garde trop longtemps** : le contexte a changé, mais personne ne sait que la décision en dépendait.

### L'ADR

Un ADR est un court document, **un par décision**, rangé avec le code (`docs/adr/0001-compter-par-lots.md`).
Il tient en une ou deux pages :

| Section | Contenu |
|---|---|
| **Titre et numéro** | La décision elle-même, en une phrase : « Compter les visites par lots en mémoire ». |
| **Statut** | Proposé, accepté, ou **remplacé par** un ADR plus récent. Avec une date. |
| **Contexte** | Les faits qui rendent la décision nécessaire : chiffres, contraintes, ce qui est acceptable et ce qui ne l'est pas. |
| **Critères** | Ce qui compte, **dans l'ordre**. Classer les critères, c'est déjà décider. |
| **Options** | Au moins deux, souvent trois, dont « ne rien changer ». Avec leurs mesures quand c'est possible. |
| **Décision** | L'option retenue, et pourquoi, en reprenant les critères. |
| **Conséquences** | Ce qu'on gagne, et surtout ce qu'on **accepte de perdre**. Une décision sans inconvénient écrit est une décision mal analysée. |
| **Réexamen** | Les conditions qui doivent faire rouvrir la décision. |

Les règles :
- **Un ADR ne se modifie pas** après son acceptation. Quand la décision change, un nouvel ADR la **remplace**,
  et l'ancien passe au statut « remplacé par 0002 ». L'historique des raisons reste lisible.
- **Les faits sont sourcés** : une mesure, avec la façon de la refaire. Une estimation est signalée comme
  telle. La règle de ce niveau s'applique : une affirmation sans preuve est une opinion.
- **Court** : un ADR de dix pages n'est pas lu. Les détails vont en annexe, ou dans le code des mesures.
- **Pas pour tout** : un ADR par décision **coûteuse à défaire**. Le nom d'une variable n'en mérite pas.

### Doser l'effort : réversible ou non

La question qui décide de l'effort : **combien coûte de revenir en arrière ?**

- **Réversible** : un détail d'implémentation derrière une interface, une bibliothèque interne, un paramètre de
  configuration, l'intervalle d'écriture du compteur. On décide vite, on mesure en production, on corrige.
  Une longue analyse coûte ici plus cher qu'une erreur.
- **Difficile à défaire** : ce qui est **partagé** avec d'autres. Un format sur le fil ou une API publique
  (M6), un schéma de base lu par d'autres équipes, le découpage en services (N10 L7), la garantie donnée aux
  utilisateurs sur leurs données. Revenir en arrière coûte une migration, ou une promesse rompue. On prend le
  temps : prototype, mesures, ADR, relecture.

Une bonne architecture **rend réversibles** les décisions qui peuvent l'être. Dans l'exemple, le compteur
dépend d'une interface `Store` à une méthode (N10 L4). Changer d'option de stockage ne touche pas au code qui
compte les visites.

### Arbitrer

Un arbitrage n'est pas un vote de préférences. Ses étapes :
1. **Énoncer les critères, et les classer.** « Ne pas surcharger la base partagée » avant « la latence de la
   visite » avant « la simplicité ». Deux personnes en désaccord sur une option le sont souvent sur l'ordre
   des critères : en discuter en premier fait gagner du temps.
2. **Lister les options**, dont « ne rien faire » et l'option la plus **ennuyeuse** : la technologie que
   l'équipe connaît déjà. Une option nouvelle doit battre l'ennuyeuse sur un critère important, pas seulement
   être intéressante.
3. **Mesurer ce qui peut l'être.** Une après-midi de prototype vaut mieux qu'une semaine de débat. L'exemple le
   montre : la limite de l'option A, qu'on aurait attribuée aux lignes chaudes, vient en fait de l'écriture
   sur disque de chaque transaction. Une ligne de configuration la quadruple.
4. **Écrire les conséquences négatives** de l'option retenue : ce qu'on perd, qui paie, ce qui pourrait mal
   tourner.
5. **Fixer les conditions de réexamen** : un seuil de trafic, un changement métier.

**La matrice pondérée** (une note par option et par critère, des poids, un total) aide à structurer la
discussion. Elle a un piège : ses chiffres donnent une **précision fausse**. Un poids changé de 0,1 peut
renverser le résultat (exercice E4). On s'en sert pour voir **quels** critères départagent les options, pas
pour laisser le total décider.

### La revue de code senior

Une revue a deux buts : **trouver les défauts avant la production**, et **partager la connaissance**. Celui qui
relit apprend le code ; celui qui est relu apprend ce que l'autre voit. Ce que regarde un relecteur senior,
**dans cet ordre** :

1. **Est-ce le bon changement ?** Répond-il au besoin, est-il conforme aux décisions (l'ADR), au bon endroit ?
   Un défaut de conception trouvé en revue coûte une réécriture : mieux vaut relire la conception **avant** le
   code, sur une page.
2. **La correction** : concurrence (qui tient quel verrou, pendant quoi ?), erreurs (que devient la donnée
   quand l'appel échoue ?), cycle de vie (qui arrête la goroutine ? que devient l'état à l'arrêt ?), ressources
   (délais, mémoire bornée).
3. **Les contrats** : l'API publique, la compatibilité (M6), les formats.
4. **L'exploitation** : comment saura-t-on que ça va mal ? Journaux, métriques, délais, arrêt gracieux (N12).
5. **Les tests** : le comportement important est-il testé, y compris les cas d'échec ?
6. **La lisibilité** : noms, découpage, commentaires.

Le **style** n'est pas le travail du relecteur : `gofmt`, `go vet` et `staticcheck` le font, dans la CI
(N12 L4).

**Écrire un commentaire de revue** :
- **Le classer** : *bloquant* (un défaut, la PR ne passe pas en l'état), *à discuter* (un vrai doute, pas
  forcément un défaut), *détail* (une préférence, que l'auteur peut ignorer). Sans classement, l'auteur ne
  sait pas ce qui compte, et traite tout comme bloquant.
- **Dire pourquoi**, avec le scénario qui casse : « si la base met 200 ms, chaque visite attend 200 ms ».
- **Proposer**, sans réécrire la PR à la place de l'auteur.
- **Prouver quand c'est possible** : un test qui échoue vaut mieux qu'un long argument. Dans l'exemple, chaque
  défaut bloquant est accompagné du test qui le montre.
- **Viser le code, pas la personne** : « ce verrou est tenu pendant l'écriture », pas « tu as oublié ».

**Recevoir une revue** :
- Un commentaire n'est pas un ordre. Un désaccord se règle par un argument ou une mesure, pas par l'autorité.
- Répondre à **chaque** commentaire : corrigé, ou pas, et pourquoi.
- Un commentaire incompris appelle une question, pas une correction au hasard.
- Une PR petite (quelques centaines de lignes au plus) est mieux relue. Au-delà, les relecteurs survolent.

### Quand ne pas le faire

- **Un ADR pour une décision réversible et locale** : on décide, on code, le message de commit suffit (N12 L1).
- **Un comité pour chaque choix** : une décision a un responsable. On consulte, puis le responsable tranche,
  par écrit.
- **Bloquer une PR sur des détails** : les préférences se signalent comme détails, et la PR passe.
- **Relire 2 000 lignes en une fois** : demander le découpage plutôt que de faire semblant d'avoir relu.

## Exemple

Un site compte les visites de ses pages, pour un classement des pages populaires. Il en reçoit 300 par seconde
en pointe, et en attend dix fois plus dans un an. La base PostgreSQL est **partagée** avec le service des
commandes, qui passe avant tout. Trois instances du service tournent en parallèle.

**1. Mesurer les options.** Les deux implémentations, et un banc d'essai en boucle ouverte (N11 L7). Le
`BatchCounter` est montré dans sa version finale, après la revue de la partie 3.

```go.mod
// Fichier : go.mod
module example.com/visits

go 1.27

require github.com/jackc/pgx/v5 v5.9.2

require (
	github.com/jackc/pgpassfile v1.0.0 // indirect
	github.com/jackc/pgservicefile v0.0.0-20240606120523-5a60cdf6a761 // indirect
	github.com/jackc/puddle/v2 v2.2.2 // indirect
	golang.org/x/sync v0.17.0 // indirect
	golang.org/x/text v0.29.0 // indirect
)
```

```go
// Fichier : visits/store.go
// Paquet visits : compter les visites de pages dans PostgreSQL.
package visits

import (
	"context"
	"slices"

	"github.com/jackc/pgx/v5/pgxpool"
)

const Schema = `CREATE TABLE IF NOT EXISTS visits (page text PRIMARY KEY, n bigint NOT NULL)`

// Store ajoute des comptes de visites. Le BatchCounter n'en connaît que cette interface (N10 L4).
type Store interface {
	Add(ctx context.Context, counts map[string]int64) error
}

type PGStore struct{ Pool *pgxpool.Pool }

// Increment : option A, une écriture par visite.
func (s PGStore) Increment(ctx context.Context, page string) error {
	_, err := s.Pool.Exec(ctx,
		`INSERT INTO visits (page, n) VALUES ($1, 1)
		 ON CONFLICT (page) DO UPDATE SET n = visits.n + 1`, page)
	return err
}

// Add : option B, un lot de comptes en une instruction. Les pages sont triées : deux instances qui
// écrivent les mêmes pages verrouillent les lignes dans le même ordre, sans interblocage.
func (s PGStore) Add(ctx context.Context, counts map[string]int64) error {
	pages := make([]string, 0, len(counts))
	for p := range counts {
		pages = append(pages, p)
	}
	slices.Sort(pages)
	ns := make([]int64, len(pages))
	for i, p := range pages {
		ns[i] = counts[p]
	}
	_, err := s.Pool.Exec(ctx,
		`INSERT INTO visits (page, n) SELECT * FROM unnest($1::text[], $2::bigint[])
		 ON CONFLICT (page) DO UPDATE SET n = visits.n + excluded.n`, pages, ns)
	return err
}
```

```go
// Fichier : visits/batch.go
package visits

import (
	"context"
	"errors"
	"maps"
	"sync"
	"time"
)

// BatchCounter agrège les visites en mémoire et les écrit toutes les Interval.
// En cas d'arrêt brutal, on perd au plus une période de visites : c'est la décision de l'ADR 0001.
type BatchCounter struct {
	store    Store
	interval time.Duration
	timeout  time.Duration
	maxPages int

	mu      sync.Mutex
	pending map[string]int64
	closed  bool

	stop chan struct{}
	done chan struct{}
}

var ErrClosed = errors.New("visits : compteur fermé")

func NewBatchCounter(s Store, interval time.Duration, maxPages int) *BatchCounter {
	c := &BatchCounter{
		store: s, interval: interval, timeout: 5 * time.Second, maxPages: maxPages,
		pending: map[string]int64{}, stop: make(chan struct{}), done: make(chan struct{}),
	}
	go c.loop()
	return c
}

// Visit compte une visite. Elle ne touche jamais la base : son coût est celui d'un verrou.
// Au-delà de maxPages pages distinctes en attente, une nouvelle page est refusée (mémoire bornée).
func (c *BatchCounter) Visit(page string) error {
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.closed {
		return ErrClosed
	}
	if _, ok := c.pending[page]; !ok && len(c.pending) >= c.maxPages {
		return errors.New("visits : trop de pages distinctes en attente")
	}
	c.pending[page]++
	return nil
}

func (c *BatchCounter) loop() {
	defer close(c.done)
	t := time.NewTicker(c.interval)
	defer t.Stop()
	for {
		select {
		case <-t.C:
			c.flush()
		case <-c.stop:
			c.flush() // le dernier lot
			return
		}
	}
}

// flush échange la map sous le verrou, et écrit HORS du verrou : Visit n'attend jamais la base.
// En cas d'échec, les comptes sont remis dans la map, pour la prochaine tentative.
func (c *BatchCounter) flush() error {
	c.mu.Lock()
	batch := c.pending
	c.pending = make(map[string]int64, len(batch))
	c.mu.Unlock()
	if len(batch) == 0 {
		return nil
	}
	ctx, cancel := context.WithTimeout(context.Background(), c.timeout)
	defer cancel()
	err := c.store.Add(ctx, batch)
	if err != nil {
		c.mu.Lock()
		for p, n := range batch {
			c.pending[p] += n
		}
		c.mu.Unlock()
	}
	return err
}

// Close écrit le dernier lot et arrête la goroutine d'écriture.
func (c *BatchCounter) Close() {
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

// Pending renvoie une copie des comptes pas encore écrits (pour les tests et les métriques).
func (c *BatchCounter) Pending() map[string]int64 {
	c.mu.Lock()
	defer c.mu.Unlock()
	return maps.Clone(c.pending)
}
```

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
	mode := flag.String("mode", "direct", "direct (option A) ou batch (option B)")
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
	if *mode == "direct" {
		visit = func(p string) error { return store.Increment(ctx, p) }
	} else {
		bc := visits.NewBatchCounter(store, time.Second, 100_000)
		visit, closeFn = bc.Visit, bc.Close
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
	closeFn() // option B : écrire le dernier lot
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

Contre PostgreSQL 17 dans un conteneur, sur la même machine (le banc sur six cœurs). Deux exécutions de
l'option A sont montrées, parce qu'elles diffèrent beaucoup :

```
$ ./bench -mode direct -rate 1000        # option A
direct  1000/s  conn.  6 : p50     3.11 ms  p99  1451.25 ms  erreurs 0  visites   5000  en base   5000
direct  1000/s  conn.  6 : p50     2.95 ms  p99     7.01 ms  erreurs 0  visites   5000  en base   5000
$ ./bench -mode direct -rate 2000
direct  2000/s  conn.  6 : p50   109.42 ms  p99   226.65 ms  erreurs 0  visites  10000  en base  10000
direct  2000/s  conn.  6 : p50    66.11 ms  p99   140.49 ms  erreurs 0  visites  10000  en base  10000
$ DATABASE_URL="$DATABASE_URL&pool_max_conns=32" ./bench -mode direct -rate 2000
direct  2000/s  conn. 32 : p50     4.79 ms  p99  3218.97 ms  erreurs 0  visites  10000  en base  10000
direct  2000/s  conn. 32 : p50     3.02 ms  p99    54.60 ms  erreurs 0  visites  10000  en base  10000
$ DATABASE_URL="$DATABASE_URL&options=-c%20synchronous_commit%3Doff" ./bench -mode direct -rate 5000   # option C
direct  5000/s  conn.  6 : p50     0.50 ms  p99    21.71 ms  erreurs 0  visites  25000  en base  25000
direct  5000/s  conn.  6 : p50     0.50 ms  p99    14.52 ms  erreurs 0  visites  25000  en base  25000
$ ./bench -mode batch -rate 1000         # option B
batch   1000/s  conn.  6 : p50     0.56 ms  p99     1.19 ms  erreurs 0  visites   5000  en base   5000
$ ./bench -mode batch -rate 20000
batch  20000/s  conn.  6 : p50     0.49 ms  p99     1.30 ms  erreurs 0  visites 100000  en base 100000
batch  20000/s  conn.  6 : p50     0.49 ms  p99     1.55 ms  erreurs 0  visites 100000  en base 100000
```

Le nombre de transactions validées, lu dans `pg_stat_database` avant et après chaque banc à 1 000 visites/s :

```
option A : 5017 transactions validées pour 5000 visites
option C : 5018 transactions validées pour 5000 visites
option B :   13 transactions validées pour 5000 visites
```

- **A, une écriture par visite**, tient 1 000 visites/s, mais d'une exécution à l'autre son p99 passe de 7 ms
  à 1,4 s. À 2 000/s, sa latence explose. Plus de connexions n'aident pas de façon fiable.
- **La cause n'est pas celle qu'on croyait.** On pensait aux pages chaudes, qui sérialisent les mises à jour
  de leur ligne. Mais avec `synchronous_commit = off` (N8 L8), la même requête tient 5 000 visites/s avec un p99
  de 15 à 22 ms. La limite est l'**écriture sur disque** de chaque transaction validée. Cette option,
  appelons-la **C**, coûte une ligne de configuration. En échange, une panne de **la base** peut perdre les
  dernières transactions déjà confirmées au client, sur une fenêtre de quelques centaines de millisecondes.
- **B, des lots en mémoire**, tient 20 000 visites/s avec un p99 de 1,3 à 1,6 ms. Ce p99 est surtout celui du
  générateur : `Visit` ne prend qu'un verrou. En base, B fait **13 transactions** pour 5 000 visites, contre
  plus de 5 000 pour A et C. En échange, un arrêt brutal **d'une instance** perd ses visites de la dernière
  seconde.

**2. L'ADR.**

```markdown
# 0001 — Compter les visites par lots en mémoire

Statut : accepté, 2026-11-04.

## Contexte

- Le classement des pages populaires repose sur un compteur de visites par page.
- Pointe actuelle : 300 visites/s. Prévision : × 10 dans un an, soit 3 000 visites/s (estimation de
  l'équipe produit, non mesurée).
- La base PostgreSQL est partagée avec le service des commandes, prioritaire. Sa charge est le risque
  principal de la plateforme.
- Les comptes servent aux statistiques, pas à la facturation : une erreur de l'ordre de 0,1 % est acceptable.
- Trois instances du service tournent en parallèle, et sont redéployées plusieurs fois par semaine.

## Critères, dans l'ordre

1. Ne pas charger la base partagée.
2. Ne pas ralentir l'affichage des pages (le comptage est sur leur chemin).
3. Tenir 3 000 visites/s avec de la marge.
4. Simplicité.

## Options (mesures : `cmd/bench`, machine de développement, PostgreSQL 17)

| | A. Une écriture par visite | C. A, avec `synchronous_commit = off` | B. Lots en mémoire, écrits chaque seconde |
|---|---|---|---|
| Transactions pour 5 000 visites | 5 017 | 5 018 | 13 |
| Tient (p99 < 50 ms) | 1 000/s, de façon irrégulière | 5 000/s (p99 15-22 ms) | 20 000/s (p99 < 2 ms) |
| Perte possible | aucune | les dernières transactions, si **la base** tombe | jusqu'à 1 s de visites par instance, si **l'instance** est tuée |
| Code | une requête | une requête et un paramètre | ≈ 100 lignes et leurs tests |

## Décision

**B**, avec un intervalle d'une seconde.
- A ne tient pas l'objectif.
- C le tient, mais impose une transaction par visite à la base partagée, soit 3 000 transactions/s à terme
  (critère 1).
- B réduit cette charge à une transaction par seconde et par instance, et ne touche pas la base sur le chemin
  de la page (critère 2).

## Conséquences

- **Accepté** : un arrêt brutal d'une instance (OOM, `kill -9`, panne de machine) perd au plus une seconde de
  ses visites. Un arrêt normal (déploiement) n'en perd aucune : le compteur écrit son dernier lot (arrêt
  gracieux, N7 L7).
- Les comptes en base ont jusqu'à une seconde de retard.
- La mémoire en attente est bornée : au-delà de 100 000 pages distinctes par seconde, les nouvelles pages ne
  sont pas comptées.
- Les instances écrivent les mêmes lignes : les lots sont triés par page, pour éviter les interblocages.
- Plus de code qu'avec A ou C, et une goroutine à arrêter proprement.
- Repli rapide : si B pose problème, C est une modification d'une ligne, derrière la même interface `Store`.

## Réexamen

- Si les comptes deviennent une donnée facturée ou contractuelle : la perte n'est plus acceptable.
- Si la pointe dépasse 20 000 visites/s par instance, ou 100 000 pages distinctes par seconde.
```

**3. La revue de la PR.** Un collègue propose l'implémentation de l'option B :

```go
// Fichier : review/pr.go
// Paquet review : la PR telle qu'elle a été proposée, avant la revue. NE PAS UTILISER.
package review

import (
	"context"
	"log"
	"sync"
	"time"

	"example.com/visits/visits"
	"github.com/jackc/pgx/v5/pgxpool"
)

type Counter struct {
	store   visits.Store
	mu      sync.Mutex
	pending map[string]int64
	ticker  *time.Ticker
}

func New(s visits.Store, interval time.Duration) *Counter {
	c := &Counter{store: s, pending: map[string]int64{}, ticker: time.NewTicker(interval)}
	go func() {
		for range c.ticker.C {
			c.flush()
		}
	}()
	return c
}

func (c *Counter) Visit(page string) {
	c.mu.Lock()
	c.pending[page]++
	c.mu.Unlock()
}

func (c *Counter) flush() {
	c.mu.Lock()
	defer c.mu.Unlock()
	if len(c.pending) == 0 {
		return
	}
	if err := c.store.Add(context.Background(), c.pending); err != nil {
		log.Printf("flush: %v", err)
	}
	c.pending = map[string]int64{}
}

func (c *Counter) Close() { c.ticker.Stop() }

// PGStore : la version de la PR, qui parcourt la map dans l'ordre (aléatoire) de range.
type PGStore struct{ Pool *pgxpool.Pool }

func (s PGStore) Add(ctx context.Context, counts map[string]int64) error {
	var pages []string
	var ns []int64
	for p, n := range counts {
		pages = append(pages, p)
		ns = append(ns, n)
	}
	_, err := s.Pool.Exec(ctx,
		`INSERT INTO visits (page, n) SELECT * FROM unnest($1::text[], $2::bigint[])
		 ON CONFLICT (page) DO UPDATE SET n = visits.n + excluded.n`, pages, ns)
	return err
}
```

La revue, telle qu'elle est postée :

> **Résumé.** L'approche suit l'ADR 0001, et l'interface `Store` rend la suite facile à tester. Merci. Quatre
> points bloquants, tous montrés par un test (`review/review_test.go`, ci-dessous), et trois à discuter.
>
> 1. **[bloquant] `flush` tient le verrou pendant l'écriture en base.** Chaque `Visit` attend la base : si
>    elle met 200 ms, une visite peut attendre 200 ms (test 1). L'ADR repose justement sur une visite qui ne
>    touche pas la base. Proposition : échanger la map sous le verrou, écrire hors du verrou.
> 2. **[bloquant] Une écriture ratée perd ses comptes.** `c.pending` est remise à zéro même si `Add` échoue :
>    100 visites, une panne de la base, 0 en base (test 2). Proposition : remettre le lot dans la map en cas
>    d'échec.
> 3. **[bloquant] `Close` n'écrit pas le dernier lot.** À chaque déploiement, chaque instance perd jusqu'à une
>    seconde de visites (test 3). L'ADR n'accepte cette perte que pour un arrêt **brutal**. Et
>    `ticker.Stop()` ne ferme pas `ticker.C` : la goroutine reste bloquée pour toujours (test 4 : 100
>    goroutines perdues pour 100 compteurs). Proposition : un canal d'arrêt, un dernier `flush`, et `Close`
>    qui attend la fin de la goroutine.
> 4. **[bloquant] Interblocages entre instances.** `PGStore.Add` envoie les pages dans l'ordre de `range`, qui
>    est aléatoire. Deux instances qui écrivent les mêmes pages verrouillent leurs lignes dans des ordres
>    différents : 32 interblocages sur 40 lots, avec quatre instances (test 5). Proposition : trier les pages.
> 5. **[à discuter] `context.Background()` sans délai.** Si la base ne répond plus, `flush` attend pour
>    toujours. Avec le point 1, toutes les visites aussi. Proposition : un délai de quelques secondes.
> 6. **[à discuter] La mémoire n'est pas bornée.** `page` vient de l'URL : un robot qui visite un million
>    d'adresses inventées crée un million d'entrées par seconde. Proposition : un nombre maximal de pages en
>    attente, ou ne compter que les pages connues.
> 7. **[à discuter] Rien ne se voit en exploitation.** Une écriture ratée n'est qu'une ligne de journal. Je
>    proposerais une métrique pour le nombre d'échecs et la taille du lot en attente (N12 L7).
> 8. **[détail]** `Visit` après `Close` compte en silence ; une erreur serait plus claire.

Les tests qui accompagnent la revue mesurent chaque défaut, sur la PR et sur la version corrigée
(`visits/batch.go`, partie 1) :

```go
// Fichier : review/review_test.go
package review

import (
	"context"
	"errors"
	"fmt"
	"log"
	"os"
	"runtime"
	"sync"
	"testing"
	"time"

	"example.com/visits/visits"
	"github.com/jackc/pgx/v5/pgconn"
	"github.com/jackc/pgx/v5/pgxpool"
)

func init() { log.SetFlags(0) } // le journal de la PR, sans horodatage

// memStore : un Store en mémoire, qui peut être lent ou échouer.
type memStore struct {
	mu    sync.Mutex
	total map[string]int64
	delay time.Duration
	fails int // nombre d'appels à faire échouer
}

func (s *memStore) Add(ctx context.Context, counts map[string]int64) error {
	time.Sleep(s.delay)
	s.mu.Lock()
	defer s.mu.Unlock()
	if s.fails > 0 {
		s.fails--
		return errors.New("base indisponible")
	}
	if s.total == nil {
		s.total = map[string]int64{}
	}
	for p, n := range counts {
		s.total[p] += n
	}
	return nil
}

func (s *memStore) sum() (n int64) {
	s.mu.Lock()
	defer s.mu.Unlock()
	for _, v := range s.total {
		n += v
	}
	return n
}

// counter : ce que les deux versions ont en commun.
type counter interface {
	Visit(string)
	Close()
}

type fixed struct{ *visits.BatchCounter }

func (f fixed) Visit(p string) { f.BatchCounter.Visit(p) }

var versions = []struct {
	name string
	mk   func(visits.Store, time.Duration) counter
}{
	{"PR", func(s visits.Store, d time.Duration) counter { return New(s, d) }},
	{"corrigée", func(s visits.Store, d time.Duration) counter {
		return fixed{visits.NewBatchCounter(s, d, 100_000)}
	}},
}

// 1. Visit attend-elle la base ?
func TestVisitWaitsForDB(t *testing.T) {
	for _, v := range versions {
		s := &memStore{delay: 200 * time.Millisecond}
		c := v.mk(s, 50*time.Millisecond)
		var worst time.Duration
		deadline := time.Now().Add(time.Second)
		for time.Now().Before(deadline) {
			start := time.Now()
			c.Visit("/a")
			worst = max(worst, time.Since(start))
			time.Sleep(time.Millisecond)
		}
		c.Close()
		fmt.Printf("1. %-9s pire latence de Visit, base à 200 ms : %v\n", v.name, worst.Round(10*time.Microsecond))
	}
}

// 2. Que deviennent les comptes quand l'écriture échoue ?
func TestFailedFlush(t *testing.T) {
	for _, v := range versions {
		s := &memStore{fails: 1}
		c := v.mk(s, 20*time.Millisecond)
		for range 100 {
			c.Visit("/a")
		}
		time.Sleep(100 * time.Millisecond) // une écriture ratée, puis des réussies
		c.Close()
		time.Sleep(50 * time.Millisecond)
		fmt.Printf("2. %-9s 100 visites, une écriture ratée : %d en base\n", v.name, s.sum())
	}
}

// 3. Que devient le dernier lot à l'arrêt ?
func TestCloseFlushes(t *testing.T) {
	for _, v := range versions {
		s := &memStore{}
		c := v.mk(s, time.Hour)
		for range 100 {
			c.Visit("/a")
		}
		c.Close()
		time.Sleep(50 * time.Millisecond)
		fmt.Printf("3. %-9s 100 visites puis Close : %d en base\n", v.name, s.sum())
	}
}

// 4. Close arrête-t-il la goroutine ?
func TestGoroutineLeak(t *testing.T) {
	for _, v := range versions {
		before := runtime.NumGoroutine()
		for range 100 {
			v.mk(&memStore{}, time.Millisecond).Close()
		}
		time.Sleep(50 * time.Millisecond)
		fmt.Printf("4. %-9s 100 New + Close : %d goroutines de plus\n", v.name, runtime.NumGoroutine()-before)
	}
}

// 5. Deux instances écrivent les mêmes pages en même temps, contre PostgreSQL.
func TestDeadlocks(t *testing.T) {
	url := os.Getenv("DATABASE_URL")
	if url == "" {
		t.Skip("DATABASE_URL non défini")
	}
	ctx := context.Background()
	pool, err := pgxpool.New(ctx, url)
	if err != nil {
		t.Fatal(err)
	}
	defer pool.Close()
	if _, err := pool.Exec(ctx, visits.Schema+"; TRUNCATE visits"); err != nil {
		t.Fatal(err)
	}
	counts := map[string]int64{}
	for i := range 200 {
		counts[fmt.Sprintf("/page/%d", i)] = 1
	}
	for _, st := range []struct {
		name  string
		store visits.Store
	}{{"PR", PGStore{pool}}, {"corrigée", visits.PGStore{Pool: pool}}} {
		var mu sync.Mutex
		deadlocks := 0
		var wg sync.WaitGroup
		for range 4 { // quatre instances
			wg.Go(func() {
				for range 10 {
					err := st.store.Add(ctx, counts)
					var pgErr *pgconn.PgError
					if errors.As(err, &pgErr) && pgErr.Code == "40P01" {
						mu.Lock()
						deadlocks++
						mu.Unlock()
					} else if err != nil {
						t.Error(err)
					}
				}
			})
		}
		wg.Wait()
		fmt.Printf("5. %-9s 4 instances × 10 lots de 200 pages : %d interblocages\n", st.name, deadlocks)
	}
}
```

```
$ go test -v ./review/
1. PR        pire latence de Visit, base à 200 ms : 200.48ms
1. corrigée  pire latence de Visit, base à 200 ms : 120µs
flush: base indisponible
2. PR        100 visites, une écriture ratée : 0 en base
2. corrigée  100 visites, une écriture ratée : 100 en base
3. PR        100 visites puis Close : 0 en base
3. corrigée  100 visites puis Close : 100 en base
4. PR        100 New + Close : 100 goroutines de plus
4. corrigée  100 New + Close : 0 goroutines de plus
5. PR        4 instances × 10 lots de 200 pages : 32 interblocages
5. corrigée  4 instances × 10 lots de 200 pages : 0 interblocages
ok  	example.com/visits/review	28.252s
```

**4. La version corrigée** est le `BatchCounter` de la partie 1. Elle répond aux huit points :
- échange de la map sous le verrou, et écriture hors du verrou ;
- le lot remis en cas d'échec ;
- un canal d'arrêt et un dernier lot ;
- les pages triées ;
- un délai de 5 s ;
- `maxPages` ;
- `Pending()` pour une métrique ;
- `ErrClosed`.

## Explication du code

| Élément | Rôle |
|---|---|
| `type Store interface { Add(…) }` | La seule dépendance du compteur. Elle permet les tests sans base (`memStore`), et rend l'option de stockage réversible. |
| `ON CONFLICT (page) DO UPDATE SET n = visits.n + excluded.n` | Une insertion ou une mise à jour, en une instruction (N10 L8). `excluded` désigne la ligne qu'on tentait d'insérer. |
| `unnest($1::text[], $2::bigint[])` | Tout le lot en **une** instruction : deux tableaux deviennent des lignes. |
| `slices.Sort(pages)` | Toutes les instances verrouillent les lignes dans le même ordre : pas de cycle d'attente, donc pas d'interblocage (N6 L10). |
| `batch := c.pending; c.pending = make(…)` | L'échange sous le verrou : `Visit` reprend aussitôt, sur une nouvelle map, pendant que l'ancienne s'écrit. |
| `case <-c.stop: c.flush(); return` | L'arrêt : un dernier lot, puis la fin de la goroutine ; `Close` attend `c.done`. |
| `rand.NewZipf(…, 1.1, 1, 999)` | Des pages très inégalement visitées, comme sur un vrai site : quelques pages chaudes, une longue traîne. |
| `pgErr.Code == "40P01"` | Le code SQLSTATE d'un interblocage détecté par PostgreSQL, qui annule l'une des transactions. |
| `xact_commit` de `pg_stat_database` | Le nombre de transactions validées : la charge réellement imposée à la base partagée. |

## Erreurs fréquentes

1. **Une décision sans contexte écrit** : dans six mois, personne ne saura ce qu'elle protégeait.
2. **Un ADR sans inconvénient** : c'est un plaidoyer, pas une analyse.
3. **Modifier un ADR accepté** au lieu de le remplacer : l'historique des raisons disparaît.
4. **Débattre sans mesurer** : ici, la cause supposée (les lignes chaudes) n'était pas la bonne.
5. **Le même effort pour toutes les décisions** : des semaines sur un détail réversible, une heure sur un
   format public.
6. **Une revue qui commente le style** et laisse passer un verrou tenu pendant une écriture en base.
7. **Des commentaires de revue sans classement**, ou sans le scénario qui casse.
8. **Accepter ou refuser un commentaire par autorité**, sans argument ni mesure.

## Bonnes pratiques Go

- Les ADR dans le dépôt (`docs/adr/`), relus en PR comme le code.
- Une interface étroite devant chaque décision qu'on veut garder réversible.
- Dans une revue de code Go, regarder en priorité :
  - les verrous tenus pendant une entrée-sortie ;
  - les goroutines sans arrêt ;
  - les erreurs qui perdent des données ;
  - les contextes sans délai ;
  - les maps parcourues dans un ordre qui compte.
- Un test par défaut bloquant signalé, joint au commentaire quand c'est possible.
- `gofmt`, `go vet` et `staticcheck` dans la CI, pour que la revue humaine porte sur le reste.

## Ce que je dois retenir

- Un ADR garde le **pourquoi** d'une décision : contexte, critères classés, options mesurées, conséquences,
  réexamen.
- On ne modifie pas un ADR, on le remplace.
- L'effort se dose par le coût du retour en arrière.
- Mesurer avant de trancher : la limite de l'écriture directe venait du disque, pas des lignes chaudes.
- Une revue senior commence par la conception et la correction. Le style est laissé aux outils.
- Chaque commentaire de revue est classé, motivé, et prouvé quand c'est possible : ici, 4 défauts bloquants,
  4 tests.

---

➡️ [Exercices](exercices.md)
