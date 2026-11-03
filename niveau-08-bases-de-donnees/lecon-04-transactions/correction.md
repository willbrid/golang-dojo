# Leçon 4 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — La mise à jour perdue, chez toi

L'`UPDATE` atomique donne **900**, sans aucun échec : la base lit et écrit dans la même instruction,
sous le verrou de ligne qu'elle prend pour l'`UPDATE`. C'est la meilleure des cinq variantes : une
instruction, aucun aller-retour supplémentaire, aucun réessai.

Version naïve, cinq exécutions (pool de 20 connexions) : 993, 992, 993, 993, 993. Le nombre de pertes
est **stable ici**, mais il dépend entièrement de la concurrence réelle : avec un pool d'une seule
connexion, ou dans un test qui fait les retraits l'un après l'autre, il tombe à **zéro**, et le test
passe. Un test unitaire séquentiel ne peut pas attraper ce bug ; il faut un test **concurrent** contre
une **vraie** base (leçon 7), avec assez de connexions pour que les transactions se chevauchent.

## E2 — `db` au lieu de `tx`

Pour un virement vers 99, rien n'est écrit dans le code du cours (l'existence de `to` est vérifiée
avant les écritures). Mais le problème est ailleurs : le crédit passé par `db` utilise une **autre**
connexion, qui veut modifier une ligne **verrouillée** par la transaction (`FOR UPDATE`). Elle attend
la fin de la transaction… qui attend la fin de l'`ExecContext`. Mesuré :

```
E2 crédit via db: timeout: context deadline exceeded 2s
```

Et **ce n'est pas lié à `SetMaxOpenConns(1)`** : même avec un pool sans limite, le crédit attend jusqu'au
délai. PostgreSQL ne détecte **pas** cet interblocage : de son point de vue, la connexion 2 attend un
verrou de la connexion 1, qui est simplement inactive (`idle in transaction`) ; le cycle passe par le
programme Go, que la base ne voit pas. Sans délai sur le contexte, le virement bloque pour toujours.

Avec `SetMaxOpenConns(1)`, c'est pire encore : `db.ExecContext` attend une connexion libre, que la
transaction ne rendra jamais. Même si la ligne n'était pas verrouillée, le programme serait bloqué.

## E3 — La transaction oubliée

Sans `Rollback`, la connexion reste réservée par la transaction, les verrous restent posés, et
`pg_stat_activity` montre `idle in transaction`. Un `UPDATE` de la même ligne depuis `psql` attend…
indéfiniment : `database/sql` n'annule pas une transaction abandonnée (il n'y a pas de finaliseur), et
la connexion n'est jamais rendue au pool.

Avec `SET idle_in_transaction_session_timeout = '2s'` dans la session, mesuré : l'`UPDATE` concurrent
attend **2 s**, puis passe ; le serveur a coupé la session fautive, et le `Commit` tardif de la
transaction oubliée échoue :

```
FATAL: terminating connection due to idle-in-transaction timeout (SQLSTATE 25P03)
```

C'est une ceinture de sécurité utile en production (réglée au niveau du rôle ou de la base), pas une
excuse pour oublier le `defer`.

## E4 — L'interblocage

Sans ordre : 15 interblocages `40P01` sur 20 virements (mesuré). Avec `ORDER BY id FOR UPDATE` : **0**.
Le `pg_sleep(0.01)` élargit la fenêtre entre le premier verrou et le second : sans lui, chaque
transaction prend ses deux verrous en quelques microsecondes, les chevauchements sont rares, et le bug
n'apparaît qu'en production, sous charge. Rendre une fenêtre de course **large** est la technique
standard pour reproduire un bug de concurrence.

## E5 — Ce que voit `REPEATABLE READ`

Mesuré (la table contient 2 lignes, T2 en insère une entre les deux lectures de T1) :

```
REPEATABLE READ : 2 puis 2
READ COMMITTED  : 3 puis 4
```

(La seconde ligne part de 3 parce que la première expérience a déjà inséré une ligne.) En
`REPEATABLE READ`, T1 travaille sur un **instantané** pris à sa première instruction : l'insertion de T2
lui est invisible. En `READ COMMITTED`, chaque instruction voit ce qui est validé à son début. Pour un
rapport qui fait plusieurs lectures et doit être **cohérent** (un total et son détail), `REPEATABLE READ`
en lecture seule est le bon outil.

## Exercice intermédiaire — Réessayer correctement

```go
// Fichier : txx/retry.go
package txx

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"math/rand/v2"
	"time"

	"github.com/jackc/pgx/v5/pgconn"
)

// WithTx : Commit si fn réussit, Rollback sinon.
func WithTx(ctx context.Context, db *sql.DB, opts *sql.TxOptions, fn func(*sql.Tx) error) error {
	tx, err := db.BeginTx(ctx, opts)
	if err != nil {
		return err
	}
	defer tx.Rollback()
	if err := fn(tx); err != nil {
		return err
	}
	return tx.Commit()
}

func retryable(err error) bool {
	var pgErr *pgconn.PgError
	return errors.As(err, &pgErr) && (pgErr.Code == "40001" || pgErr.Code == "40P01")
}

const maxAttempts = 5

// WithRetryTx rejoue TOUTE la transaction sur 40001 et 40P01. fn peut donc s'exécuter
// plusieurs fois : elle ne doit avoir aucun effet hors de la transaction.
func WithRetryTx(ctx context.Context, db *sql.DB, opts *sql.TxOptions, fn func(*sql.Tx) error) error {
	for attempt := 1; ; attempt++ {
		err := WithTx(ctx, db, opts, fn)
		if err == nil || !retryable(err) {
			return err
		}
		if attempt == maxAttempts {
			return fmt.Errorf("abandon après %d tentatives : %w", attempt, err)
		}
		// Attente aléatoire entre 1 et 2^attempt ms : les transactions en conflit ne se
		// représentent pas toutes au même instant.
		wait := time.Duration(1+rand.IntN(1<<attempt)) * time.Millisecond
		t := time.NewTimer(wait)
		select {
		case <-ctx.Done():
			t.Stop()
			return errors.Join(ctx.Err(), err)
		case <-t.C:
		}
	}
}
```

Mesuré, 100 retraits naïfs concurrents en `SERIALIZABLE` (pool de 20) :

| Tentatives max | Retraits réussis | Abandons | Exécutions de `fn` |
|---|---|---|---|
| 5 | 41 | 59 | 414 |
| 10 | 97 | 3 | 647 |
| 20 | 100 | 0 | 668 |

L'invariant tient dans tous les cas (solde = 1 000 − retraits réussis) : **rien n'est perdu**, mais
beaucoup est **refusé** ou **refait**. Pour 100 retraits réussis, il a fallu 668 transactions. Sur une
ligne très disputée, `SERIALIZABLE` + réessai est la plus coûteuse des corrections ; `FOR UPDATE`
(les transactions attendent au lieu d'échouer) ou mieux, l'`UPDATE` atomique, font le même travail en
100 transactions. `SERIALIZABLE` brille quand les conflits sont **rares** et les règles trop complexes
pour des verrous explicites.

Après épuisement des tentatives, l'appelant reçoit
`abandon après 5 tentatives : ERROR: could not serialize access … (SQLSTATE 40001)` ; le `%w` garde
l'erreur inspectable (`errors.As(err, &pgErr)` réussit, vérifié par le test). Le handler HTTP en fait un
503 avec `Retry-After`, ou un 409 (N7 L5).

Point 3 : `fn` est appelée 414 fois pour 100 retraits. Si elle envoyait un e-mail, le client en
recevrait plusieurs pour **un** retrait, voire pour un retrait **abandonné**. Tout effet externe se fait
**après** le `Commit` réussi, ou via une table *outbox* écrite dans la transaction (niveau 10).

## Défi — Réserver une place

```go
// Fichier : txx/booking.go
package txx

import (
	"context"
	"database/sql"
	"errors"
	"fmt"

	"github.com/jackc/pgx/v5/pgconn"
)

var (
	ErrFull          = errors.New("événement complet")
	ErrAlreadyBooked = errors.New("déjà réservé")
)

const Schema = `
	DROP TABLE IF EXISTS bookings, events;
	CREATE TABLE events (id int PRIMARY KEY, capacity int NOT NULL);
	CREATE TABLE bookings (
		event_id int NOT NULL REFERENCES events(id),
		user_id  int NOT NULL,
		UNIQUE (event_id, user_id)
	)`

func translate(err error) error {
	var pgErr *pgconn.PgError
	if errors.As(err, &pgErr) && pgErr.Code == "23505" { // unique_violation
		return ErrAlreadyBooked
	}
	return err
}

func countThenInsert(ctx context.Context, tx *sql.Tx, eventID, userID int, lock bool) error {
	var capacity int
	q := `SELECT capacity FROM events WHERE id = $1`
	if lock {
		q += ` FOR UPDATE` // sérialise toutes les réservations d'un même événement
	}
	if err := tx.QueryRowContext(ctx, q, eventID).Scan(&capacity); err != nil {
		return err
	}
	var n int
	if err := tx.QueryRowContext(ctx,
		`SELECT count(*) FROM bookings WHERE event_id = $1`, eventID).Scan(&n); err != nil {
		return err
	}
	if n >= capacity {
		return fmt.Errorf("événement %d : %w", eventID, ErrFull)
	}
	_, err := tx.ExecContext(ctx,
		`INSERT INTO bookings (event_id, user_id) VALUES ($1, $2)`, eventID, userID)
	return translate(err)
}

// BookNaive : (a) compter puis insérer, READ COMMITTED.
func BookNaive(ctx context.Context, db *sql.DB, eventID, userID int) error {
	return WithTx(ctx, db, nil, func(tx *sql.Tx) error {
		return countThenInsert(ctx, tx, eventID, userID, false)
	})
}

// BookLocked : (b) verrouiller l'événement d'abord.
func BookLocked(ctx context.Context, db *sql.DB, eventID, userID int) error {
	return WithTx(ctx, db, nil, func(tx *sql.Tx) error {
		return countThenInsert(ctx, tx, eventID, userID, true)
	})
}

// BookSerializable : (c) SERIALIZABLE + réessai.
func BookSerializable(ctx context.Context, db *sql.DB, eventID, userID int) error {
	return WithRetryTx(ctx, db, &sql.TxOptions{Isolation: sql.LevelSerializable}, func(tx *sql.Tx) error {
		return countThenInsert(ctx, tx, eventID, userID, false)
	})
}

// BookOneStatement : (d) une seule instruction, sans transaction explicite.
func BookOneStatement(ctx context.Context, db *sql.DB, eventID, userID int) error {
	res, err := db.ExecContext(ctx, `
		INSERT INTO bookings (event_id, user_id)
		SELECT $1, $2
		WHERE (SELECT count(*) FROM bookings WHERE event_id = $1)
		    < (SELECT capacity FROM events WHERE id = $1)`, eventID, userID)
	if err != nil {
		return translate(err)
	}
	if n, _ := res.RowsAffected(); n == 0 {
		return fmt.Errorf("événement %d : %w", eventID, ErrFull)
	}
	return nil
}
```

```go
// Fichier : txx/txx_test.go
package txx

import (
	"context"
	"database/sql"
	"errors"
	"os"
	"sync"
	"sync/atomic"
	"testing"

	"github.com/jackc/pgx/v5/pgconn"
	_ "github.com/jackc/pgx/v5/stdlib"
)

func openTest(t *testing.T) *sql.DB {
	t.Helper()
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		t.Skip("DATABASE_URL non définie")
	}
	db, err := sql.Open("pgx", dsn)
	if err != nil {
		t.Fatal(err)
	}
	db.SetMaxOpenConns(20)
	t.Cleanup(func() { db.Close() })
	return db
}

func TestRetryNaiveWithdrawals(t *testing.T) {
	db := openTest(t)
	ctx := t.Context()
	if _, err := db.ExecContext(ctx, `DROP TABLE IF EXISTS l4_acc;
		CREATE TABLE l4_acc (id int PRIMARY KEY, balance bigint NOT NULL);
		INSERT INTO l4_acc VALUES (1, 1000)`); err != nil {
		t.Fatal(err)
	}
	var calls atomic.Int64
	var failed atomic.Int64
	var wg sync.WaitGroup
	for range 100 {
		wg.Go(func() {
			err := WithRetryTx(ctx, db, &sql.TxOptions{Isolation: sql.LevelSerializable}, func(tx *sql.Tx) error {
				calls.Add(1)
				var b int64
				if err := tx.QueryRowContext(ctx, `SELECT balance FROM l4_acc WHERE id = 1`).Scan(&b); err != nil {
					return err
				}
				_, err := tx.ExecContext(ctx, `UPDATE l4_acc SET balance = $1 WHERE id = 1`, b-1)
				return err
			})
			if err != nil {
				failed.Add(1)
				var pgErr *pgconn.PgError
				if !errors.As(err, &pgErr) {
					t.Errorf("l'erreur finale doit rester inspectable : %v", err)
				}
			}
		})
	}
	wg.Wait()
	var b int64
	db.QueryRowContext(ctx, `SELECT balance FROM l4_acc WHERE id = 1`).Scan(&b)
	if b != 1000-100+failed.Load() {
		t.Fatalf("solde %d pour %d échecs : un retrait a été perdu", b, failed.Load())
	}
	t.Logf("solde %d, abandons %d, appels de fn : %d pour 100 retraits", b, failed.Load(), calls.Load())
	if calls.Load() <= 100 {
		t.Errorf("fn appelée %d fois : aucune transaction rejouée ?", calls.Load())
	}
}

func TestBookingVariants(t *testing.T) {
	db := openTest(t)
	variants := []struct {
		name string
		book func(context.Context, *sql.DB, int, int) error
		safe bool // les variantes non sûres sont mesurées, pas exigées
	}{
		{"a-naif", BookNaive, false},
		{"b-for-update", BookLocked, true},
		{"c-serializable", BookSerializable, true},
		{"d-une-instruction", BookOneStatement, false},
	}
	for _, v := range variants {
		t.Run(v.name, func(t *testing.T) {
			ctx := t.Context()
			if _, err := db.ExecContext(ctx, Schema+`; INSERT INTO events VALUES (1, 50)`); err != nil {
				t.Fatal(err)
			}
			var full, other atomic.Int64
			var wg sync.WaitGroup
			for user := range 500 {
				wg.Go(func() {
					err := v.book(ctx, db, 1, user)
					switch {
					case err == nil:
					case errors.Is(err, ErrFull):
						full.Add(1)
					default:
						other.Add(1)
					}
				})
			}
			wg.Wait()
			var n int
			db.QueryRowContext(ctx, `SELECT count(*) FROM bookings`).Scan(&n)
			t.Logf("%-18s réservations=%d complet=%d autres erreurs=%d", v.name, n, full.Load(), other.Load())
			if n > 50 && v.safe {
				t.Errorf("SURRÉSERVATION : %d places vendues pour 50", n)
			}
		})
	}
}

func TestDuplicateBooking(t *testing.T) {
	db := openTest(t)
	ctx := t.Context()
	if _, err := db.ExecContext(ctx, Schema+`; INSERT INTO events VALUES (1, 50)`); err != nil {
		t.Fatal(err)
	}
	var dup atomic.Int64
	var wg sync.WaitGroup
	for range 10 {
		wg.Go(func() {
			if err := BookLocked(ctx, db, 1, 7); errors.Is(err, ErrAlreadyBooked) {
				dup.Add(1)
			}
		})
	}
	wg.Wait()
	if dup.Load() != 9 {
		t.Fatalf("%d doublons détectés, attendu 9", dup.Load())
	}
}
```

Mesuré (événement de 50 places, 500 utilisateurs simultanés, pool de 20) :

| Variante | Réservations | Verdict |
|---|---|---|
| (a) compter puis insérer | **60 – 61** | surréservation |
| (b) `FOR UPDATE` sur l'événement | 50 | correct |
| (c) `SERIALIZABLE` + réessai | 50 (1 abandon après 5 tentatives) | correct |
| (d) une seule instruction `INSERT … SELECT … WHERE count < capacity` | **56 – 66** | surréservation |

**Pourquoi (d) échoue.** « Une instruction est atomique » est vrai, mais atomique ne veut pas dire
**isolé des autres** : en `READ COMMITTED`, chaque `INSERT … SELECT` compte les réservations **validées
au début de l'instruction**. Vingt instructions simultanées voient toutes 49, et toutes insèrent. Il n'y a
pas de ligne commune à verrouiller (les nouvelles réservations sont de **nouvelles** lignes), donc aucune
attente. C'est la différence avec l'`UPDATE … SET balance = balance - 1` d'E1, qui modifie **une ligne
existante** et prend donc son verrou. Règle : une contrainte qui porte sur un **ensemble** de lignes
(« pas plus de 50 ») demande soit un verrou sur une ligne qui la représente (l'événement, variante b ;
ou un compteur `seats_left` décrémenté atomiquement), soit `SERIALIZABLE`.

**Le doublon.** La contrainte `UNIQUE (event_id, user_id)` produit l'erreur `23505` ; `translate` la
convertit en `ErrAlreadyBooked`. Dix requêtes simultanées du même utilisateur : une réussit, neuf
reçoivent `ErrAlreadyBooked` (`TestDuplicateBooking`). La base garantit l'unicité même quand le code ne
vérifie rien : c'est le bon endroit pour cette règle.

## Quiz

1. Une transaction est attachée à une connexion ; `db` en prendrait une autre, hors transaction (et peut
   bloquer sur les verrous de la transaction).
2. Rien : elle rend `sql.ErrTxDone`, qu'on ignore.
3. `database/sql` annule la transaction ; le `Commit` suivant échoue.
4. Non : deux transactions peuvent lire la même valeur et écrire chacune la leur.
5. `UPDATE` atomique, `SELECT … FOR UPDATE`, isolation `REPEATABLE READ`/`SERIALIZABLE` avec réessai.
6. Échec de sérialisation : rejouer toute la transaction, avec limite et attente aléatoire.
7. Les valeurs lues au début ont pu changer : la décision doit être recalculée.
8. Verrouiller les lignes toujours dans le même ordre (`ORDER BY id FOR UPDATE`), et réessayer `40P01`.
9. Une transaction ouverte qui ne fait rien : elle retient une connexion et des verrous.
10. Les verrous et la connexion sont retenus pendant toute la durée de l'appel, et un appel rejoué
    (réessai) aurait plusieurs effets.
