# Module 3 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Le module déclare maintenant `pgx` pour l'exercice intermédiaire :

```go.mod
// Fichier : go.mod
module example.com/dist

go 1.27

require github.com/jackc/pgx/v5 v5.9.2
```

Les exercices faciles ajoutent des fichiers au paquet `dist` :

```go
// Fichier : dist/vv.go
package dist

import (
	"math/rand/v2"
	"slices"
	"time"
)

// --- E1 : vecteurs de versions ---

// VV : un compteur par réplique.
type VV map[string]uint64

type Order int

const (
	Before Order = iota
	After
	Equal
	Concurrent
)

// Compare dit si a précède b, le suit, lui est égal, ou s'ils sont concurrents (ni l'un ni l'autre).
func Compare(a, b VV) Order {
	aLess, bLess := false, false
	for _, k := range keys(a, b) {
		switch {
		case a[k] < b[k]:
			aLess = true
		case a[k] > b[k]:
			bLess = true
		}
	}
	switch {
	case aLess && bLess:
		return Concurrent
	case aLess:
		return Before
	case bLess:
		return After
	}
	return Equal
}

func keys(a, b VV) []string {
	var ks []string
	for k := range a {
		ks = append(ks, k)
	}
	for k := range b {
		if _, ok := a[k]; !ok {
			ks = append(ks, k)
		}
	}
	slices.Sort(ks)
	return ks
}

// Bump renvoie une copie de v avec le compteur de node incrémenté (une écriture sur node).
func Bump(v VV, node string) VV {
	w := VV{}
	for k, x := range v {
		w[k] = x
	}
	w[node]++
	return w
}

// DetectConcurrent rejoue le scénario de LostWrites avec des vecteurs : le premier écrivain (réplique
// « a ») écrit ; le second (réplique « b ») écrit soit APRÈS avoir lu la première valeur (si sa
// lecture arrive après la réplication, délai lag), soit sans l'avoir vue. Renvoie les fractions
// d'écritures ordonnées et de conflits détectés.
func DetectConcurrent(seed uint64, n int, gap, lag time.Duration) (ordered, conflicts float64) {
	rng := rand.New(rand.NewPCG(seed, 3))
	o, c := 0, 0
	for range n {
		first := Bump(VV{}, "a")
		second := Bump(VV{}, "b") // n'a rien vu
		if d := time.Duration(rng.Int64N(int64(gap))); d >= lag {
			second = Bump(first, "b") // la réplication de la première est arrivée avant la lecture du second
		}
		switch Compare(first, second) {
		case Before:
			o++
		case Concurrent:
			c++
		}
	}
	return float64(o) / float64(n), float64(c) / float64(n)
}
```

```go
// Fichier : dist/session.go
package dist

import (
	"math/rand/v2"
	"slices"
	"time"
)

// --- E2 : lire ses propres écritures avec W=1, R=1 ---

// ReadYourWrites : le client exige une réplique qui a sa version. Il essaie les répliques dans un ordre
// aléatoire ; si aucune ne l'a, il attend step et recommence. Renvoie la proportion servie au premier
// essai, le nombre moyen d'essais, et le p99 de l'attente.
func ReadYourWrites(seed uint64, n, N int, lag, step time.Duration) (first, meanTries float64, p99 time.Duration) {
	rng := rand.New(rand.NewPCG(seed, 4))
	var waits []time.Duration
	firstOK, tries := 0, 0
	for range n {
		arrive := make([]time.Duration, N)
		for i := range arrive {
			arrive[i] = time.Duration(rng.Int64N(int64(lag)))
		}
		ack := slices.Min(arrive) // W=1
		t := ack
		k := 0
		for done := false; !done; t += step {
			for _, i := range rng.Perm(N) {
				k++
				if arrive[i] <= t {
					done = true
					break
				}
			}
			if done {
				break
			}
		}
		if k == 1 {
			firstOK++
		}
		tries += k
		waits = append(waits, t-ack)
	}
	slices.Sort(waits)
	return float64(firstOK) / float64(n), float64(tries) / float64(n), waits[int(0.99*float64(n-1))]
}

// --- E3 : latence d'une écriture à quorum, avec une queue lourde ---

// QuorumLatency : chaque réplique répond en 2 ms, ou en 200 ms une fois sur cent. L'écriture est
// finie à la W-ième réponse. Renvoie le p50 et le p99.
func QuorumLatency(seed uint64, n, N, W int) (p50, p99 time.Duration) {
	rng := rand.New(rand.NewPCG(seed, 5))
	var lat []time.Duration
	for range n {
		r := make([]time.Duration, N)
		for i := range r {
			r[i] = 2 * time.Millisecond
			if rng.IntN(100) == 0 {
				r[i] = 200 * time.Millisecond
			}
		}
		slices.Sort(r)
		lat = append(lat, r[W-1])
	}
	slices.Sort(lat)
	return lat[n/2], lat[int(0.99*float64(n-1))]
}
```

```go
// Fichier : dist/idem.go
package dist

import "math/rand/v2"

// --- E4 : réessayer sans dupliquer ---

// Transfers : n virements ; le serveur exécute chaque requête reçue, mais la RÉPONSE se perd une fois
// sur dix, et le client réessaie jusqu'à recevoir une réponse. Avec une clé d'idempotence, le serveur
// reconnaît une requête déjà exécutée et renvoie le résultat enregistré. Renvoie le nombre de virements
// exécutés plus d'une fois.
func Transfers(seed uint64, n int, idempotency bool) (duplicated, executions int) {
	rng := rand.New(rand.NewPCG(seed, 6))
	done := map[int]bool{} // côté serveur : clés déjà traitées (et leur résultat)
	for id := range n {
		count := 0
		for {
			if !idempotency || !done[id] {
				count++ // le virement est exécuté
				done[id] = true
			}
			if rng.IntN(10) != 0 { // la réponse arrive
				break
			}
		}
		executions += count
		if count > 1 {
			duplicated++
		}
	}
	return
}
```

```go
// Fichier : dist/skew.go
package dist

import "time"

// --- E5 : le bail calculé avec l'horloge du client ---

// Overlap : le service accorde un bail de ttl (horloge juste). Le client mesure le temps avec une
// horloge qui avance à la vitesse rate (0,9 = 10 % trop lente) et s'arrête d'écrire quand SON horloge
// atteint ttl - margin. Renvoie le temps réel pendant lequel il écrit encore après l'expiration.
func Overlap(ttl, margin time.Duration, rate float64) time.Duration {
	stopReal := time.Duration(float64(ttl-margin) / rate) // le client croit tenir le bail jusque-là
	return max(0, stopReal-ttl)
}

// SafeMargin : la marge minimale pour que le client s'arrête avant l'expiration, avec une dérive
// maximale de drift (0,1 = 10 %) et un délai réseau maximal latency pour la réponse d'Acquire (le bail a
// commencé côté service AVANT que le client ne reçoive la réponse).
func SafeMargin(ttl time.Duration, drift float64, latency time.Duration) time.Duration {
	// Il faut (ttl - margin) / (1 - drift) + latency <= ttl.
	return time.Duration(float64(ttl) - (1-drift)*float64(ttl-latency))
}
```

```go
// Fichier : dist/fix_test.go
package dist

import (
	"fmt"
	"testing"
	"time"
)

func TestE1(t *testing.T) {
	a := Bump(VV{}, "a")
	if Compare(a, Bump(a, "b")) != Before || Compare(Bump(VV{}, "b"), a) != Concurrent || Compare(a, a) != Equal {
		t.Fatal("Compare")
	}
	o, c := DetectConcurrent(1, 100_000, 10*time.Millisecond, 5*time.Millisecond)
	fmt.Printf("E1 : écritures ordonnées %.1f %%, conflits DÉTECTÉS %.1f %%, perdues en silence 0 %%\n", 100*o, 100*c)
}

func TestE2(t *testing.T) {
	f, m, p := ReadYourWrites(1, 100_000, 3, 20*time.Millisecond, time.Millisecond)
	fmt.Printf("E2 : servies au premier essai %.1f %%, essais en moyenne %.2f, attente p99 %v\n", 100*f, m, p)
}

func TestE3(t *testing.T) {
	for _, w := range []int{1, 2, 3} {
		p50, p99 := QuorumLatency(1, 100_000, 3, w)
		fmt.Printf("E3 : N=3 W=%d  p50 %v  p99 %v\n", w, p50, p99)
	}
}

func TestE4(t *testing.T) {
	for _, idem := range []bool{false, true} {
		d, e := Transfers(1, 10_000, idem)
		fmt.Printf("E4 : idempotence=%-5v virements exécutés deux fois ou plus : %4d ; exécutions %d pour 10000\n", idem, d, e)
	}
}

func TestE5(t *testing.T) {
	ttl := 10 * time.Second
	fmt.Printf("E5 : bail de 10 s, horloge du client 10 %% trop lente, sans marge : il écrit encore %v après l'expiration\n", Overlap(ttl, 0, 0.9))
	m := SafeMargin(ttl, 0.1, 200*time.Millisecond)
	fmt.Printf("E5 : marge sûre (dérive 10 %%, réseau 200 ms) : %v ; dépassement avec cette marge : %v\n", m, Overlap(ttl, m, 0.9))
}
```

```
$ go test -v -run 'TestE' ./dist/
E1 : écritures ordonnées 49.7 %, conflits DÉTECTÉS 50.3 %, perdues en silence 0 %
E2 : servies au premier essai 33.6 %, essais en moyenne 2.00, attente p99 0s
E3 : N=3 W=1  p50 2ms  p99 2ms
E3 : N=3 W=2  p50 2ms  p99 2ms
E3 : N=3 W=3  p50 2ms  p99 200ms
E4 : idempotence=false virements exécutés deux fois ou plus : 1062 ; exécutions 11183 pour 10000
E4 : idempotence=true  virements exécutés deux fois ou plus :    0 ; exécutions 10000 pour 10000
E5 : bail de 10 s, horloge du client 10 % trop lente, sans marge : il écrit encore 1.111111111s après l'expiration
E5 : marge sûre (dérive 10 %, réseau 200 ms) : 1.18s ; dépassement avec cette marge : 0s
```

## E1 — Détecter les écritures concurrentes

Sur le scénario de `LostWrites`, deux écritures séparées de 0 à 10 ms, avec une réplication qui prend 5 ms :
- **49,7 %** sont ordonnées (le second écrivain avait vu la première valeur) ;
- **50,3 %** sont détectées **concurrentes** ;
- **aucune n'est perdue en silence**.

Avec « le dernier écrit gagne », ces 50 % de cas concurrents auraient été tranchés par l'heure, au hasard
des horloges.

Un conflit détecté n'est pas résolu pour autant. Le système peut :
- **garder les deux valeurs** (des « sœurs ») et laisser le prochain lecteur, ou l'application, les
  fusionner ;
- **appliquer une règle métier** : additionner deux paniers, garder le maximum d'un compteur ;
- **refuser la seconde écriture** (verrouillage optimiste, N8 L4) : le client relit, puis réécrit.

Le bon choix dépend de la donnée. Ce qui est sûr, c'est qu'un système qui ne **détecte** pas les conflits ne
peut pas choisir.

## E2 — Lire ses propres écritures

Servies au premier essai : 33,6 % (une chance sur trois de tomber sur la bonne réplique). Il faut 2,00 essais
en moyenne, et **aucune attente** (p99 de 0).

Ce dernier résultat s'explique : avec W = 1, la confirmation vient de la réplique qui **a** l'écriture. En
essayant les trois répliques, le client la trouve toujours au premier tour, sans attendre la réplication.
D'où la solution la plus simple : **lire sur la réplique qui a confirmé**, ou sur le leader, pendant quelques
secondes après une écriture (lecture « collante »). Cela ne coûte qu'un aller-retour, alors que W = R = 2
coûte une réplique de plus à chaque opération.

## E3 — Le quorum et la latence de queue

| W sur N=3 | p50 | p99 |
|---|---|---|
| 1 | 2 ms | 2 ms |
| 2 | 2 ms | **2 ms** |
| 3 | 2 ms | **200 ms** |

Avec W = 3, l'écriture attend la **plus lente** des trois répliques. Une réplique est lente une fois sur cent,
donc l'une des trois l'est environ 3 % du temps, et le p99 vaut 200 ms. C'est la page qui attend le plus lent
de ses services (N11 L7).

Avec W = 2, il faudrait **deux** répliques lentes en même temps : environ 0,03 % des cas, sous le p99. Un
quorum majoritaire protège donc à la fois de la **panne** d'une réplique et de sa **lenteur**. C'est une
raison souvent sous-estimée de préférer 2 sur 3 à « toutes ».

## E4 — Réessayer sans dupliquer

| | Virements exécutés deux fois ou plus | Exécutions pour 10 000 virements |
|---|---|---|
| sans clé d'idempotence | **1 062** | 11 183 |
| avec clé d'idempotence | 0 | 10 000 |

Une réponse perdue une fois sur dix suffit à exécuter deux fois plus de 10 % des virements. Le client n'a
rien fait de mal : il ne peut pas distinguer « pas exécuté » de « exécuté, réponse perdue ».

Le serveur doit enregistrer la clé **et le résultat**, dans la **même transaction** que l'effet. Sinon, une
panne entre les deux recrée le problème. Il garde la clé au moins aussi longtemps que le client peut
réessayer, plus la marge des horloges : des heures pour un client interactif, des jours pour un traitement par
lots qui peut reprendre après un incident (N10 L9).

## E5 — Le bail et l'horloge du client

Avec un bail de 10 s et une horloge cliente 10 % trop lente, le client croit tenir le bail jusqu'à 10 s à **son**
horloge, soit 11,1 s en temps réel. Il continue d'écrire **1,11 s** après l'expiration, alors que le bail a
peut-être été réattribué.

Il faut une marge qui couvre :
- la **dérive** de l'horloge ;
- le **délai réseau** de la réponse : le bail a commencé côté service avant que le client ne reçoive la
  réponse.

Pour 10 % de dérive et 200 ms de réseau, `SafeMargin` donne **1,18 s**. Le client doit s'arrêter, ou renouveler,
quand son horloge indique 8,82 s.

Le jeton de clôture reste nécessaire, parce que la marge suppose une dérive et un délai **bornés**. Une pause
du GC de 3 s, une machine virtuelle suspendue ou une horloge qui saute sortent de toute borne raisonnable. La
marge réduit la fréquence du problème, le jeton le rend inoffensif.

---

## Exercice intermédiaire — Un verrou à bail sur PostgreSQL

```go
// Fichier : pglease/lease.go
// Package pglease : un bail exclusif arbitré par PostgreSQL, et une ressource protégée par un jeton
// de clôture. Toutes les heures sont celles de la base (now()), jamais celles des processus.
package pglease

import (
	"context"
	"database/sql"
	"errors"
	"time"
)

const Schema = `
CREATE TABLE IF NOT EXISTS leases (
    name       text PRIMARY KEY,
    holder     text NOT NULL,
    expires_at timestamptz NOT NULL,
    token      bigint NOT NULL
);
CREATE TABLE IF NOT EXISTS fence (name text PRIMARY KEY, token bigint NOT NULL);
CREATE TABLE IF NOT EXISTS writes (
    id     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    holder text NOT NULL,
    token  bigint NOT NULL
);`

var ErrNotHeld = errors.New("bail non obtenu")

// Acquire prend le bail s'il n'existe pas ou s'il a expiré, en incrémentant le jeton : UNE instruction.
func Acquire(ctx context.Context, db *sql.DB, name, holder string, ttl time.Duration) (int64, error) {
	var token int64
	err := db.QueryRowContext(ctx, `
		INSERT INTO leases (name, holder, expires_at, token) VALUES ($1, $2, now() + $3 * interval '1 millisecond', 1)
		ON CONFLICT (name) DO UPDATE
		   SET holder = EXCLUDED.holder, expires_at = EXCLUDED.expires_at, token = leases.token + 1
		 WHERE leases.expires_at < now()
		RETURNING token`, name, holder, ttl.Milliseconds()).Scan(&token)
	if errors.Is(err, sql.ErrNoRows) {
		return 0, ErrNotHeld // le bail est tenu par quelqu'un d'autre, et n'a pas expiré
	}
	return token, err
}

// Renew prolonge le bail seulement si on le tient TOUJOURS (même titulaire, même jeton, pas expiré).
func Renew(ctx context.Context, db *sql.DB, name, holder string, token int64, ttl time.Duration) error {
	res, err := db.ExecContext(ctx, `
		UPDATE leases SET expires_at = now() + $4 * interval '1 millisecond'
		 WHERE name = $1 AND holder = $2 AND token = $3 AND expires_at >= now()`, name, holder, token, ttl.Milliseconds())
	if err != nil {
		return err
	}
	if n, _ := res.RowsAffected(); n == 0 {
		return ErrNotHeld
	}
	return nil
}

var ErrFenced = errors.New("jeton périmé : écriture refusée")

// Write écrit sur la ressource protégée. Le verrou de la ligne fence sérialise les écrivains, et
// l'écriture n'a lieu que si le jeton n'est pas plus petit que le plus grand déjà vu.
func Write(ctx context.Context, db *sql.DB, name, holder string, token int64) error {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	defer tx.Rollback()
	if _, err := tx.ExecContext(ctx, `INSERT INTO fence (name, token) VALUES ($1, 0) ON CONFLICT DO NOTHING`, name); err != nil {
		return err
	}
	res, err := tx.ExecContext(ctx, `UPDATE fence SET token = $2 WHERE name = $1 AND token <= $2`, name, token)
	if err != nil {
		return err
	}
	if n, _ := res.RowsAffected(); n == 0 {
		return ErrFenced
	}
	if _, err := tx.ExecContext(ctx, `INSERT INTO writes (holder, token) VALUES ($1, $2)`, holder, token); err != nil {
		return err
	}
	return tx.Commit()
}

// WriteUnfenced : la même écriture, sans vérifier le jeton (pour comparer).
func WriteUnfenced(ctx context.Context, db *sql.DB, holder string, token int64) error {
	_, err := db.ExecContext(ctx, `INSERT INTO writes (holder, token) VALUES ($1, $2)`, holder, token)
	return err
}
```

```go
// Fichier : pglease/lease_test.go
//go:build integration

package pglease

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"math/rand/v2"
	"os"
	"sync"
	"sync/atomic"
	"testing"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
)

// db ouvre une connexion dans un schéma jetable (N8 L7), supprimé à la fin du test.
func db(t *testing.T) *sql.DB {
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		t.Fatal("DATABASE_URL manquante")
	}
	admin, err := sql.Open("pgx", dsn)
	if err != nil {
		t.Fatal(err)
	}
	schema := fmt.Sprintf("lease_%d", time.Now().UnixNano())
	if _, err := admin.Exec("CREATE SCHEMA " + schema); err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() {
		admin.Exec("DROP SCHEMA " + schema + " CASCADE")
		admin.Close()
	})
	d, err := sql.Open("pgx", dsn+"&search_path="+schema)
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { d.Close() })
	if _, err := d.Exec(Schema); err != nil {
		t.Fatal(err)
	}
	return d
}

func TestLeaseUnderPauses(t *testing.T) {
	for _, fencing := range []bool{true, false} {
		t.Run(fmt.Sprintf("jeton=%v", fencing), func(t *testing.T) { run(t, fencing) })
	}
}

func run(t *testing.T, fencing bool) {
	d := db(t)
	ctx := context.Background()
	const ttl = 300 * time.Millisecond
	var accepted, fenced, pauses atomic.Int64
	deadline := time.Now().Add(5 * time.Second)
	var wg sync.WaitGroup
	for w := range 3 {
		name := fmt.Sprintf("worker-%d", w)
		wg.Go(func() {
			rng := rand.New(rand.NewPCG(uint64(w), 7))
			for time.Now().Before(deadline) {
				token, err := Acquire(ctx, d, "planificateur", name, ttl)
				if errors.Is(err, ErrNotHeld) {
					time.Sleep(100 * time.Millisecond)
					continue
				}
				if err != nil {
					t.Error(err)
					return
				}
				lastRenew := time.Now()
				for time.Now().Before(deadline) { // titulaire : écrire, renouveler, parfois se figer
					if rng.IntN(10) == 0 {
						pauses.Add(1)
						time.Sleep(time.Duration(rng.IntN(500)) * time.Millisecond) // pause du GC, VM suspendue…
					}
					write := func() error { return Write(ctx, d, "planificateur", name, token) }
					if !fencing {
						write = func() error { return WriteUnfenced(ctx, d, name, token) }
					}
					switch err := write(); {
					case err == nil:
						accepted.Add(1)
					case errors.Is(err, ErrFenced):
						fenced.Add(1)
					default:
						t.Error(err)
						return
					}
					if time.Since(lastRenew) > ttl/3 {
						if Renew(ctx, d, "planificateur", name, token, ttl) != nil {
							break // perdu : on retente l'acquisition
						}
						lastRenew = time.Now()
					}
					time.Sleep(20 * time.Millisecond)
				}
			}
		})
	}
	wg.Wait()

	// La propriété : dans l'ordre des écritures ACCEPTÉES, les jetons ne décroissent jamais.
	rows, err := d.Query("SELECT token, holder FROM writes ORDER BY id")
	if err != nil {
		t.Fatal(err)
	}
	defer rows.Close()
	var prev, maxTok int64
	violations := 0
	for rows.Next() {
		var tok int64
		var h string
		rows.Scan(&tok, &h)
		if tok < maxTok {
			violations++ // un ancien titulaire a écrit APRÈS le nouveau
		}
		maxTok = max(maxTok, tok)
		prev = tok
	}
	_ = prev
	fmt.Printf("jeton=%-5v acceptées %3d, refusées %d, pauses %2d, baux %d, écritures d'un ancien titulaire acceptées : %d\n",
		fencing, accepted.Load(), fenced.Load(), pauses.Load(), maxTok, violations)
	if fencing && violations != 0 {
		t.Fatal("la clôture n'a pas protégé la ressource")
	}
}
```

```
$ DATABASE_URL=… go test -tags integration -race -v ./pglease/       # deux exécutions
jeton=true  acceptées 112, refusées 3, pauses 11, baux 4, écritures d'un ancien titulaire acceptées : 0
jeton=false acceptées 134, refusées 0, pauses 12, baux 3, écritures d'un ancien titulaire acceptées : 2
jeton=true  acceptées 122, refusées 3, pauses 11, baux 4, écritures d'un ancien titulaire acceptées : 0
jeton=false acceptées 125, refusées 0, pauses 12, baux 3, écritures d'un ancien titulaire acceptées : 2
```

- **`Acquire` est une seule instruction**, `INSERT … ON CONFLICT DO UPDATE … WHERE expires_at < now()`. Deux
  processus qui tentent en même temps ne peuvent pas obtenir tous deux le bail : la ligne est verrouillée par
  la mise à jour. Le jeton est incrémenté dans la même instruction.
- **`now()` est l'heure de la base** : tous les processus jugent l'expiration avec la même horloge. Un
  processus dont l'horloge dérive ne peut pas prolonger son bail en croyant qu'il est encore valable.
- **`Write` sérialise les écrivains** par le verrou de la ligne `fence` (`UPDATE … WHERE token <= $2`).
  L'écriture n'a lieu que si le jeton n'est pas plus petit que le dernier vu. Un simple « lire le maximum,
  puis insérer » laisserait passer deux écritures concurrentes.
- **Le résultat.** Avec le jeton, l'ancien titulaire réveillé voit **3** écritures refusées, et aucune
  n'est acceptée après celles du nouveau titulaire. Sans le jeton, **2** écritures d'un ancien titulaire
  sont acceptées **après** celles du suivant : deux « planificateurs » auraient lancé les mêmes tâches.

**5. Les verrous consultatifs** (`pg_try_advisory_lock`) sont plus simples : pas de table, et le verrou est
libéré automatiquement quand la **session** se termine. Mais ce verrou est attaché à la **connexion**, pas au
processus vivant.

Un processus figé (pause du GC, machine suspendue) dont la connexion TCP reste ouverte **garde** le verrou.
Personne d'autre ne peut prendre le rôle, et la panne n'est jamais détectée. Il faudrait un délai
d'inactivité côté serveur (`idle_session_timeout`, et les sondes TCP) pour qu'elle le soit. À l'inverse, si la
connexion est coupée mais que le processus continue, il **perd** le verrou sans le savoir, et continue
d'écrire.

Le bail à échéance explicite plus le jeton de clôture se comporte mieux dans les deux cas. C'est l'arbitrage :
la simplicité contre la sûreté face aux pauses.

---

## Défi — Chercher le scénario qui casse

```go
// Fichier : election/sim.go
// Package election : une élection par battements de cœur, simulée en temps virtuel (défi).
package election

import (
	"container/heap"
	"math/rand/v2"
	"time"
)

// Faults : les pannes d'une exécution.
type Faults struct {
	Loss          float64       // probabilité de perdre un message
	PartitionFrom time.Duration // la partition isole le nœud Isolated de [From, To)
	PartitionTo   time.Duration
	Isolated      int
}

// Write : une écriture ACCEPTÉE par la ressource.
type Write struct {
	At   time.Duration
	Node int
	Term int64
}

type event struct {
	at   time.Duration
	kind int // 0 : battement à envoyer, 1 : réception d'un battement, 2 : vérification du délai, 3 : écriture
	node int
	term int64
	seq  int
}

type queue []event

func (q queue) Len() int { return len(q) }
func (q queue) Less(i, j int) bool {
	if q[i].at != q[j].at {
		return q[i].at < q[j].at
	}
	return q[i].seq < q[j].seq // ordre total : la simulation est déterministe
}
func (q queue) Swap(i, j int) { q[i], q[j] = q[j], q[i] }
func (q *queue) Push(x any)   { *q = append(*q, x.(event)) }
func (q *queue) Pop() any {
	old := *q
	e := old[len(old)-1]
	*q = old[:len(old)-1]
	return e
}

// Run simule 3 nœuds pendant d. Un nœud qui n'a reçu aucun battement depuis 300 ms se proclame leader ;
// un leader envoie un battement toutes les 100 ms et écrit toutes les 50 ms ; un leader qui reçoit
// le battement d'un mandat supérieur (ou, sans mandats, de n'importe qui) abdique.
// fenced : mandats numérotés, et la ressource refuse un mandat inférieur au plus grand vu.
func Run(seed uint64, d time.Duration, f Faults, fenced bool) []Write {
	rng := rand.New(rand.NewPCG(seed, 9))
	const n = 3
	leader := make([]bool, n)
	term := make([]int64, n)
	lastSeen := make([]time.Duration, n)
	var q queue
	seq := 0
	push := func(e event) { seq++; e.seq = seq; heap.Push(&q, e) }
	cut := func(a, b int, at time.Duration) bool {
		return at >= f.PartitionFrom && at < f.PartitionTo && (a == f.Isolated) != (b == f.Isolated)
	}
	for i := range n {
		push(event{at: time.Duration(rng.IntN(300)) * time.Millisecond, kind: 2, node: i})
	}
	var writes []Write
	var maxTerm int64
	for q.Len() > 0 {
		e := heap.Pop(&q).(event)
		if e.at > d {
			break
		}
		i := e.node
		switch e.kind {
		case 2: // délai sans battement ?
			if !leader[i] && e.at-lastSeen[i] >= 300*time.Millisecond {
				leader[i] = true
				term[i] = (maxSeen(term)/n+1)*n + int64(i) // un mandat plus grand que tous ceux connus, propre au nœud
				push(event{at: e.at, kind: 0, node: i})
				push(event{at: e.at, kind: 3, node: i})
			}
			push(event{at: e.at + 50*time.Millisecond, kind: 2, node: i})
		case 0: // le leader envoie un battement à chacun
			if !leader[i] {
				continue
			}
			for j := range n {
				if j == i || rng.Float64() < f.Loss || cut(i, j, e.at) {
					continue
				}
				push(event{at: e.at + time.Duration(rng.IntN(50))*time.Millisecond, kind: 1, node: j, term: term[i]})
			}
			push(event{at: e.at + 100*time.Millisecond, kind: 0, node: i})
		case 1: // réception d'un battement
			if !fenced || e.term >= term[i] {
				lastSeen[i] = e.at
				if leader[i] && (!fenced || e.term > term[i]) {
					leader[i] = false // abdique
				}
				term[i] = max(term[i], e.term)
			}
		case 3: // le leader écrit sur la ressource partagée
			if !leader[i] {
				continue
			}
			if !fenced || term[i] >= maxTerm {
				maxTerm = max(maxTerm, term[i])
				writes = append(writes, Write{e.at, i, term[i]})
			}
			push(event{at: e.at + 50*time.Millisecond, kind: 3, node: i})
		}
	}
	return writes
}

func maxSeen(t []int64) int64 {
	m := t[0]
	for _, x := range t[1:] {
		m = max(m, x)
	}
	return m
}

// Violates : la propriété est violée si un nœud écrit de nouveau APRÈS qu'un autre a écrit à sa suite
// (A, B, A) à moins de 300 ms d'écart : deux leaders se sont partagé la ressource en même temps.
func Violates(ws []Write) bool {
	for k := 2; k < len(ws); k++ {
		a, b, c := ws[k-2], ws[k-1], ws[k]
		if a.Node == c.Node && b.Node != a.Node && c.At-a.At < 300*time.Millisecond {
			return true
		}
	}
	return false
}
```

```go
// Fichier : election/sim_test.go
package election

import (
	"fmt"
	"math/rand/v2"
	"testing"
	"time"
)

func randomFaults(r *rand.Rand) Faults {
	from := time.Duration(r.IntN(5000)) * time.Millisecond
	return Faults{
		Loss:          r.Float64() * 0.5,
		PartitionFrom: from,
		PartitionTo:   from + time.Duration(r.IntN(2000))*time.Millisecond,
		Isolated:      r.IntN(3),
	}
}

// search : 10 000 exécutions aléatoires ; renvoie la première graine qui viole la propriété.
func search(fenced bool) (seed uint64, f Faults, found bool, count int) {
	r := rand.New(rand.NewPCG(42, 42))
	for s := range uint64(10_000) {
		ff := randomFaults(r)
		if Violates(Run(s, 10*time.Second, ff, fenced)) {
			if !found {
				seed, f, found = s, ff, true
			}
			count++
		}
	}
	return
}

func TestSearch(t *testing.T) {
	seed, f, found, count := search(false)
	fmt.Printf("sans mandats : %d exécutions sur 10 000 violent la propriété ; première : graine %d, %+v\n", count, seed, f)
	if !found {
		t.Fatal("aucune violation trouvée")
	}
	// Réduire : sans pertes ? avec une partition plus courte ?
	small := f
	for _, loss := range []float64{0, 0.01, 0.05, 0.1, f.Loss} {
		small.Loss = loss
		if Violates(Run(seed, 10*time.Second, small, false)) {
			break
		}
	}
	for d := 100 * time.Millisecond; d < f.PartitionTo-f.PartitionFrom; d += 100 * time.Millisecond {
		try := small
		try.PartitionTo = try.PartitionFrom + d
		if Violates(Run(seed, 10*time.Second, try, false)) {
			small = try
			break
		}
	}
	fmt.Printf("scénario réduit : pertes %.2f, partition de %v à %v, nœud isolé %d\n",
		small.Loss, small.PartitionFrom, small.PartitionTo, small.Isolated)

	_, _, foundFenced, countFenced := search(true)
	fmt.Printf("avec mandats et clôture : %d violations sur 10 000\n", countFenced)
	if foundFenced {
		t.Error("le correctif ne suffit pas")
	}
}
```

```go
// Fichier : election/nofault_test.go
package election

import (
	"fmt"
	"testing"
	"time"
)

func TestNoFaults(t *testing.T) {
	bad := 0
	var first []Write
	for s := range uint64(1000) {
		ws := Run(s, 10*time.Second, Faults{}, false)
		if Violates(ws) {
			if first == nil {
				first = ws
			}
			bad++
		}
	}
	fmt.Printf("sans aucune panne : %d exécutions sur 1 000 violent la propriété\n", bad)
	for i, w := range first {
		if i >= 8 {
			break
		}
		fmt.Printf("  %v : nœud %d écrit\n", w.At, w.Node)
	}
}
```

```
$ go test -v ./election/
sans aucune panne : 479 exécutions sur 1 000 violent la propriété
  311ms : nœud 1 écrit
  313ms : nœud 2 écrit
  661ms : nœud 1 écrit
  663ms : nœud 2 écrit
  711ms : nœud 1 écrit
  1.013s : nœud 2 écrit
  1.063s : nœud 2 écrit
  1.113s : nœud 2 écrit
sans mandats : 9694 exécutions sur 10 000 violent la propriété ; première : graine 0, {Loss:0.1930657854068158 PartitionFrom:3.096s PartitionTo:4.372s Isolated:1}
scénario réduit : pertes 0.00, partition de 3.096s à 3.196s, nœud isolé 1
avec mandats et clôture : 0 violations sur 10 000
```

**Le premier résultat est la leçon du défi.** La recherche aléatoire trouve des violations dans 9 694
exécutions sur 10 000. Puis la réduction ramène le premier scénario à « aucune perte, une partition de
100 ms ». La question devient : faut-il seulement une panne ? `TestNoFaults` répond : **sans aucune
panne**, 479 exécutions sur 1 000 violent la propriété.

Le scénario lu dans l'historique :
- les délais des nœuds 1 et 2 expirent à 2 ms d'écart ;
- les deux se proclament leaders et écrivent (à 311 et 313 ms) ;
- leurs battements sont **en vol** pendant 0 à 50 ms ;
- ils écrivent encore (661 et 663 ms) ;
- l'un finit par abdiquer.

Le protocole est faux dès son démarrage, sans qu'aucune panne ne soit nécessaire. Un test « avec quelques
pannes » sur une seule exécution aurait pu passer, ou échouer sans qu'on comprenne pourquoi.

**3. Le scénario réduit** est la configuration vide (`Faults{}`) et une graine quelconque parmi les 479. C'est
le meilleur test de régression possible : court, sans hasard caché, et il échoue à chaque fois.

**4. La correction : des mandats numérotés et une clôture.**
- Chaque nœud qui se proclame leader prend un mandat **plus grand** que tous ceux qu'il connaît, et propre à
  lui (`(max/3 + 1) × 3 + id`), pour que deux nœuds n'aient jamais le même.
- Un leader abdique en recevant un mandat **supérieur**.
- La ressource refuse tout mandat inférieur au plus grand vu.

Résultat : **0 violation sur 10 000** exécutions aléatoires avec pannes.

**5. Pourquoi deux nœuds peuvent encore se croire leaders**, et pourquoi ce n'est pas grave. Pendant les
quelques millisecondes où les battements sont en vol, deux nœuds ont encore `leader = true`. Aucun protocole
ne peut l'empêcher sans communication instantanée. Mais la **ressource** n'accepte que le mandat le plus
récent : l'ancien leader peut **croire** qu'il l'est, il ne peut plus **agir**.

La sûreté est garantie là où les effets ont lieu, pas dans la tête des nœuds. C'est exactement le rôle du
jeton de clôture. Un vrai système utilise un service de consensus pour attribuer les mandats (etcd, ou la
base), pas ce protocole maison.

---

## Quiz

1. Réussi, échoué, ou inconnu (délai dépassé : la requête a peut-être été traitée). Un réessai n'est sûr que
   si l'opération est idempotente, ou protégée par une clé d'idempotence.
2. Les horloges des machines diffèrent : une écriture plus récente, faite sur une machine en retard, porte un
   horodatage plus petit et perd. Mesuré : 100 % de pertes pour les écritures rapprochées avec 20 ms de
   décalage.
3. Si A a pu causer B (B a vu A), l'horodatage de B dépasse celui de A. Elle ne dit rien de deux événements
   concurrents, et ne détecte pas les conflits : il faut pour cela des vecteurs de versions (E1).
4. Un ensemble d'écriture de W répliques et un ensemble de lecture de R répliques ont au moins une réplique en
   commun dès que W + R > N : la lecture rencontre toujours une réplique à jour.
5. W = N : la panne ou la lenteur d'une seule réplique bloque ou ralentit toutes les écritures (p99 de 200 ms
   dans E3). W = 1 : une lecture juste après est souvent périmée (66,6 % avec R = 1), et la perte d'une
   réplique peut perdre l'écriture.
6. En l'absence de nouvelles écritures, toutes les répliques finissent par avoir la même valeur, sans délai
   garanti.
7. Refuser les écritures du côté minoritaire (cohérence), ou les accepter des deux côtés et réconcilier
   ensuite (disponibilité, avec des conflits à résoudre).
8. Deux majorités d'un même ensemble ont toujours au moins un membre en commun : deux décisions
   contradictoires ne peuvent pas être prises chacune par une majorité.
9. Le titulaire peut être figé au-delà de son bail sans le savoir, puis écrire alors qu'un autre tient le
   verrou. Le jeton croît à chaque attribution, et la ressource refuse tout jeton plus ancien que le dernier
   vu.
10. Par simulation déterministe (temps virtuel, graine fixée, pannes injectées, propriété vérifiée sur des
    milliers de graines), par injection de pannes dans un vrai déploiement, et par vérification des
    historiques observés par les clients.
