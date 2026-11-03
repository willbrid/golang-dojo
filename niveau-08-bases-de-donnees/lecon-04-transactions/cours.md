# Leçon 4 — Transactions

> Un virement, c'est deux écritures : débiter un compte, créditer l'autre. Si le programme s'arrête
> entre les deux, de l'argent disparaît. Si deux virements lisent le même solde en même temps, l'un des
> deux débits est perdu. Les transactions règlent le premier problème **par construction** ; le second
> demande de comprendre l'**isolation**, et c'est là que se cachent les bugs les plus coûteux, parce
> qu'ils n'apparaissent qu'en charge. Cette leçon donne les deux, avec des mesures.

## Objectifs

1. Ouvrir, valider et annuler une transaction sans jamais en laisser une ouverte.
2. Écrire un `WithTx` qui garantit `Commit` ou `Rollback` sur tous les chemins.
3. Reconnaître la **mise à jour perdue** et la corriger (`FOR UPDATE`, `UPDATE` atomique, isolation).
4. Connaître les niveaux d'isolation de PostgreSQL et **réessayer** une transaction sérialisable.
5. Éviter les interblocages en verrouillant dans un ordre fixe.

## Explication

### Le cycle de vie

```go
tx, err := db.BeginTx(ctx, nil)   // réserve UNE connexion du pool, envoie BEGIN
if err != nil {
	return err
}
defer tx.Rollback()                // filet de sécurité sur tous les chemins

if _, err := tx.ExecContext(ctx, `UPDATE …`); err != nil {
	return err                     // → Rollback par le defer
}
return tx.Commit()                 // → COMMIT ; le Rollback différé devient sans effet
```

- Une transaction est attachée à **une** connexion, réservée de `BeginTx` jusqu'à `Commit` ou `Rollback`.
  Toutes les instructions passent par `tx`, **jamais** par `db` : `db.ExecContext` à l'intérieur
  utiliserait une **autre** connexion, hors transaction (et avec un pool d'une seule connexion,
  attendrait indéfiniment celle que la transaction retient).
- `defer tx.Rollback()` est **le** patron. Après un `Commit` réussi, `Rollback` ne fait rien et rend
  `sql.ErrTxDone` (`transaction has already been committed or rolled back`), qu'on ignore. Sur tous les
  autres chemins (erreur, `return` anticipé, **panique**), il annule.
- Si `ctx` est annulé pendant la transaction, `database/sql` l'annule lui-même ; le `Commit` suivant
  échoue (mesuré : `context canceled`, ou `ErrTxDone` si l'annulation a déjà eu lieu).
- Une transaction ne peut avoir qu'un seul résultat ouvert à la fois. Lancer une instruction alors qu'un
  `Rows` de la même transaction n'est pas entièrement lu donne `driver: bad connection` (mesuré) : on lit
  et ferme le résultat **avant** l'instruction suivante.

`tx.Commit()` peut échouer : contrainte vérifiée en fin de transaction, conflit de sérialisation,
connexion perdue. Son erreur se traite **toujours**.

### `WithTx` : le patron en une fonction

Plutôt que de répéter ces lignes dans chaque fonction, on les écrit une fois :

```go
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
```

Le code métier ne voit plus que `fn` : il rend une erreur pour annuler, `nil` pour valider. Une panique
dans `fn` remonte, mais le `defer` annule d'abord la transaction (N2 L7).

### Ce qu'on ne fait pas dans une transaction

Une transaction retient une connexion **et** des verrous. Pendant qu'elle est ouverte, d'autres
transactions attendent. Donc, à l'intérieur :

- **pas d'appel réseau** (HTTP vers un autre service, envoi d'e-mail) : s'il prend 30 s, les verrous
  sont tenus 30 s ;
- **pas d'attente d'un utilisateur** ;
- **pas de travail long** qui pourrait être fait avant.

Une transaction ouverte et inactive apparaît dans `pg_stat_activity` avec l'état
`idle in transaction` (mesuré). C'est un signal d'alarme en production. PostgreSQL peut les couper
automatiquement (`idle_in_transaction_session_timeout`).

### La mise à jour perdue

Un retrait écrit naïvement : lire le solde, calculer, écrire.

```go
tx.QueryRowContext(ctx, `SELECT balance FROM accounts WHERE id = 1`).Scan(&b)
tx.ExecContext(ctx, `UPDATE accounts SET balance = $1 WHERE id = 1`, b-1)
```

Mesuré : 100 goroutines font chacune un retrait de 1 sur un solde de 1 000. Résultat attendu : 900.

| Variante | Solde final | Transactions en échec |
|---|---|---|
| Naïve (`READ COMMITTED`, défaut) | **993** | 0 |
| `SELECT … FOR UPDATE` | 900 | 0 |
| Isolation `SERIALIZABLE` | 989 | **89** (`40001 could not serialize access due to concurrent update`) |
| Isolation `REPEATABLE READ` | 989 | **89** (même erreur) |

**La transaction ne suffit pas.** Dans la version naïve, chaque transaction est atomique, et pourtant 93
retraits sur 100 ont disparu : deux transactions lisent le même solde (1 000), écrivent chacune 999, et
la seconde écrase la première. C'est le « vérifier puis agir » de N6 L6, entre processus.

Trois corrections, dans l'ordre de préférence :

1. **Laisser la base faire le calcul**, en une instruction atomique :
   `UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1`, puis
   `RowsAffected() == 0` signifie « solde insuffisant ». Pas de lecture préalable, pas de course.
2. **Verrouiller la ligne lue** : `SELECT … FOR UPDATE`. La seconde transaction **attend** que la première
   se termine, puis lit la valeur à jour. Indispensable quand la décision demande plusieurs lectures
   (l'exemple ci-dessous).
3. **Monter l'isolation** à `SERIALIZABLE` (ou `REPEATABLE READ`) : PostgreSQL **détecte** le conflit
   et fait échouer l'une des transactions (code `40001`). Rien n'est perdu, mais la transaction refusée
   doit être **rejouée** par le programme. Sans boucle de réessai, 89 retraits sur 100 sont refusés.

### Les niveaux d'isolation de PostgreSQL

| Niveau | Ce que voit chaque instruction | Usage |
|---|---|---|
| `READ COMMITTED` (défaut) | ce qui était validé **au début de l'instruction** | le cas courant, avec `FOR UPDATE` ou des `UPDATE` atomiques |
| `REPEATABLE READ` | ce qui était validé **au début de la transaction** ; écrire une ligne modifiée entre-temps → `40001` | rapports cohérents (plusieurs lectures sur un même instantané) |
| `SERIALIZABLE` | comme si les transactions s'exécutaient **l'une après l'autre** ; tout conflit → `40001` | règles métier complexes qu'aucun verrou simple n'exprime |

En Go : `db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})`. `ReadOnly: true` déclare
une transaction en lecture seule (PostgreSQL refuse alors toute écriture).

### Reconnaître et réessayer

Les erreurs PostgreSQL arrivent comme des `*pgconn.PgError` (paquet `github.com/jackc/pgx/v5/pgconn`),
qui portent un **code SQLSTATE** stable :

```go
var pgErr *pgconn.PgError
if errors.As(err, &pgErr) {
	switch pgErr.Code {
	case "40001": // serialization_failure : rejouer la transaction
	case "40P01": // deadlock_detected : rejouer la transaction
	case "23505": // unique_violation : conflit métier (doublon), → 409 (N7 L5)
	case "23503": // foreign_key_violation : référence inexistante
	}
}
```

Rejouer, c'est relancer **toute** la fonction `WithTx`, lectures comprises (les valeurs lues ont pu
changer), avec un nombre de tentatives borné et une petite attente aléatoire (*jitter*, niveau 10) :

```go
func retryable(err error) bool {
	var pgErr *pgconn.PgError
	return errors.As(err, &pgErr) && (pgErr.Code == "40001" || pgErr.Code == "40P01")
}

for attempt := 1; ; attempt++ {
	err = WithTx(ctx, db, &sql.TxOptions{Isolation: sql.LevelSerializable}, fn)
	if err == nil || !retryable(err) || attempt == 5 {
		return err
	}
	// attendre quelques millisecondes aléatoires, en respectant ctx
}
```

Conséquence : `fn` peut s'exécuter **plusieurs fois**. Elle ne doit avoir aucun effet hors de la base
(pas d'e-mail envoyé, pas de compteur en mémoire incrémenté) : ses seuls effets sont dans la
transaction, annulés à chaque échec.

### Les interblocages

Deux virements croisés, sans précaution : A verrouille le compte 1 puis attend le 2 ; B verrouille le 2
puis attend le 1. Aucun ne peut avancer. PostgreSQL le détecte (au bout de `deadlock_timeout`, 1 s par
défaut) et tue l'une des transactions. Mesuré, 20 virements croisés simultanés : **15** échecs
`40P01 deadlock detected`.

La correction : **verrouiller toujours dans le même ordre**. `SELECT … WHERE id = ANY($1) ORDER BY id
FOR UPDATE` prend les verrous par identifiant croissant, quel que soit le sens du virement. Les
interblocages restent possibles dans un système complexe : `40P01` fait partie des erreurs à réessayer.

## Exemple

Des virements tout-ou-rien, concurrents, entre trois comptes.

```go
package main

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"log"
	"os"
	"sync"

	_ "github.com/jackc/pgx/v5/stdlib"
)

var (
	ErrInsufficient = errors.New("solde insuffisant")
	ErrNoAccount    = errors.New("compte introuvable")
)

// WithTx exécute fn dans une transaction : Commit si fn réussit, Rollback sinon — y compris
// si fn panique, puisque les defer s'exécutent pendant la remontée d'une panique.
func WithTx(ctx context.Context, db *sql.DB, opts *sql.TxOptions, fn func(*sql.Tx) error) error {
	tx, err := db.BeginTx(ctx, opts)
	if err != nil {
		return err
	}
	defer tx.Rollback() // sans effet après un Commit réussi (rend sql.ErrTxDone, ignoré)
	if err := fn(tx); err != nil {
		return err
	}
	return tx.Commit()
}

// Transfer déplace amount de from vers to, tout ou rien.
func Transfer(ctx context.Context, db *sql.DB, from, to int, amount int64) error {
	return WithTx(ctx, db, nil, func(tx *sql.Tx) error {
		// Verrouiller les DEUX lignes, toujours dans le même ordre (ORDER BY id) :
		// deux transferts croisés ne peuvent pas s'interbloquer.
		rows, err := tx.QueryContext(ctx,
			`SELECT id, balance FROM l4_accounts WHERE id = ANY($1) ORDER BY id FOR UPDATE`,
			[]int{from, to})
		if err != nil {
			return err
		}
		defer rows.Close()
		balances := map[int]int64{}
		for rows.Next() {
			var id int
			var b int64
			if err := rows.Scan(&id, &b); err != nil {
				return err
			}
			balances[id] = b
		}
		if err := rows.Err(); err != nil {
			return err
		}
		rows.Close() // libérer le résultat avant d'envoyer d'autres instructions sur la même connexion

		if _, ok := balances[to]; !ok {
			return fmt.Errorf("compte %d : %w", to, ErrNoAccount)
		}
		b, ok := balances[from]
		if !ok {
			return fmt.Errorf("compte %d : %w", from, ErrNoAccount)
		}
		if b < amount {
			return fmt.Errorf("compte %d (%d) : %w", from, b, ErrInsufficient)
		}
		if _, err := tx.ExecContext(ctx,
			`UPDATE l4_accounts SET balance = balance - $1 WHERE id = $2`, amount, from); err != nil {
			return err
		}
		_, err = tx.ExecContext(ctx,
			`UPDATE l4_accounts SET balance = balance + $1 WHERE id = $2`, amount, to)
		return err
	})
}

func balances(ctx context.Context, db *sql.DB) (string, int64) {
	rows, err := db.QueryContext(ctx, `SELECT id, balance FROM l4_accounts ORDER BY id`)
	if err != nil {
		log.Fatal(err)
	}
	defer rows.Close()
	var s string
	var total int64
	for rows.Next() {
		var id int
		var b int64
		rows.Scan(&id, &b)
		s += fmt.Sprintf("%d:%d ", id, b)
		total += b
	}
	return s, total
}

func main() {
	ctx := context.Background()
	db, err := sql.Open("pgx", os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()
	// Sans limite, 300 goroutines ouvriraient 300 connexions, et PostgreSQL (100 par défaut)
	// refuserait les suivantes : « too many clients already ». Le réglage du pool est la leçon 5.
	db.SetMaxOpenConns(20)
	if _, err := db.ExecContext(ctx, `
		DROP TABLE IF EXISTS l4_accounts;
		CREATE TABLE l4_accounts (id int PRIMARY KEY, balance bigint NOT NULL CHECK (balance >= 0));
		INSERT INTO l4_accounts VALUES (1, 1000), (2, 1000), (3, 1000)`); err != nil {
		log.Fatal(err)
	}

	// 1. Échec métier : rien n'est modifié.
	err = Transfer(ctx, db, 1, 2, 5000)
	fmt.Println("5000 de 1 vers 2 :", err)
	// 2. Échec au milieu : le débit déjà calculé n'est jamais appliqué.
	err = Transfer(ctx, db, 1, 99, 10)
	fmt.Println("10 de 1 vers 99  :", err)
	s, _ := balances(ctx, db)
	fmt.Println("soldes :", s)

	// 3. 300 transferts concurrents dans tous les sens.
	var wg sync.WaitGroup
	var mu sync.Mutex
	var failures []error
	for i := range 300 {
		wg.Go(func() {
			from, to := i%3+1, (i+1)%3+1
			if i%2 == 0 {
				from, to = to, from
			}
			if err := Transfer(ctx, db, from, to, int64(i%7+1)); err != nil {
				mu.Lock()
				failures = append(failures, err)
				mu.Unlock()
			}
		})
	}
	wg.Wait()
	s, total := balances(ctx, db)
	fmt.Println("après 300 transferts concurrents : échecs =", len(failures), "total =", total)
	fmt.Println("soldes :", s)
}
```

Sortie :

```
5000 de 1 vers 2 : compte 1 (1000) : solde insuffisant
10 de 1 vers 99  : compte 99 : compte introuvable
soldes : 1:1000 2:1000 3:1000 
après 300 transferts concurrents : échecs = 0 total = 3000
soldes : 1:1000 2:1006 3:994 
```

Les soldes finaux sont toujours les mêmes, d'une exécution à l'autre : chaque virement a un montant
fixe, et l'ordre dans lequel ils s'exécutent ne change pas la somme reçue par chaque compte.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `defer tx.Rollback()` dans `WithTx` | Annule sur **tous** les chemins, panique comprise ; sans effet après `Commit`. |
| `CHECK (balance >= 0)` | Dernière ligne de défense, dans la base : même un bug Go ne peut pas rendre un solde négatif. |
| `ORDER BY id FOR UPDATE` | Verrouille les deux lignes, **dans un ordre fixe** : pas d'interblocage entre virements croisés. |
| Vérification du solde **après** le verrou | La valeur lue ne peut plus changer avant le `Commit` : pas de mise à jour perdue. |
| `rows.Close()` explicite avant les `UPDATE` | Une transaction = une connexion = un seul résultat ouvert à la fois. |
| `return fmt.Errorf(… ErrInsufficient)` | Une erreur métier suffit à tout annuler : `WithTx` fait le `Rollback`. |
| Virement vers 99 | Le compte 1 a été lu et verrouillé, mais rien n'a été écrit : soldes intacts. |
| `SetMaxOpenConns(20)` | Sans limite, 300 goroutines = 300 connexions, et PostgreSQL refuse au-delà de 100 (leçon 5). |
| `total = 3000` | L'invariant du système, vérifié après la tempête. |

## Erreurs fréquentes

1. **Oublier `defer tx.Rollback()`** : une erreur au milieu laisse la transaction ouverte, la connexion
   retenue et les verrous tenus (`idle in transaction`).
2. **Utiliser `db` au lieu de `tx`** à l'intérieur de la transaction.
3. **Ignorer l'erreur de `Commit`.**
4. **Lire puis écrire sans verrou** en `READ COMMITTED` : la mise à jour perdue, invisible en test.
5. **Monter en `SERIALIZABLE` sans boucle de réessai.**
6. **Réessayer seulement le `Commit`**, ou seulement l'instruction qui a échoué, au lieu de toute la
   transaction.
7. **Des effets de bord dans une transaction rejouable** (e-mail, appel HTTP, compteur Go).
8. **Verrouiller dans un ordre qui dépend des données** (`from` puis `to`).
9. **Des transactions longues** : appel réseau, attente, traitement lourd à l'intérieur.
10. **Une transaction pour une seule instruction** : chaque instruction est déjà atomique.

## Bonnes pratiques Go

- `WithTx(ctx, db, opts, fn)` : un seul endroit qui ouvre, valide et annule.
- Les fonctions d'accès aux données prennent une petite interface (leçon 7) satisfaite par `*sql.DB` et
  `*sql.Tx`, pour être utilisables dans ou hors transaction.
- Préférer l'`UPDATE` atomique ; sinon `FOR UPDATE` ; `SERIALIZABLE` + réessai pour les règles complexes.
- Verrous dans un ordre fixe ; `40001` et `40P01` réessayés avec attente aléatoire et limite.
- Transactions courtes, sans entrée/sortie externe.
- Contraintes en base (`CHECK`, `UNIQUE`, clés étrangères) comme filet de sécurité.

## Ce que je dois retenir

- Transaction = une connexion réservée ; tout passe par `tx` ; `defer tx.Rollback()` toujours.
- Atomicité ≠ isolation : une transaction n'empêche **pas** la mise à jour perdue en `READ COMMITTED`.
- Corrections : `UPDATE` atomique, `FOR UPDATE`, ou isolation plus forte **avec réessai**.
- Erreurs PostgreSQL : `*pgconn.PgError` et son code (`40001`, `40P01`, `23505`).
- Interblocages : verrouiller dans un ordre fixe, et réessayer `40P01`.

---

➡️ [Exercices](exercices.md)
