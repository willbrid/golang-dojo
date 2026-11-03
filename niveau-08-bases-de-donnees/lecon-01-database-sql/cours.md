# Leçon 1 — `database/sql`

> Presque tous les services Go parlent à une base relationnelle, et presque tous les incidents de
> production qui touchent une base viennent d'un malentendu sur **ce qu'est** l'objet qu'on manipule. On
> croit tenir « une connexion » ; on tient un **pool** de connexions, partagé par toutes les
> goroutines, qui s'ouvre paresseusement et dont chaque connexion a son propre état. Cette leçon pose ce
> modèle, avec PostgreSQL et le pilote pgx.

## Objectifs

1. Comprendre le partage des rôles entre `database/sql` (l'API) et un **pilote** (le protocole).
2. Ouvrir un `*sql.DB`, savoir qu'il s'agit d'un **pool**, et vérifier la connexion avec `PingContext`.
3. Exécuter une instruction avec `ExecContext` et des **paramètres** `$1, $2…`.
4. Savoir que l'état de session (table temporaire, `SET`) appartient à **une** connexion.
5. Situer pgx natif (`pgxpool`) par rapport à `database/sql`.

## Explication

### Deux couches : l'API et le pilote

Le paquet standard `database/sql` définit une API **générique** : ouvrir, exécuter, lire des lignes,
gérer des transactions et un pool. Il ne sait parler à **aucune** base. Le protocole réseau de chaque
base est implémenté par un **pilote** (*driver*), un module externe qui s'enregistre auprès de
`database/sql` sous un nom :

```go
import _ "github.com/jackc/pgx/v5/stdlib" // son init() appelle sql.Register("pgx", …)

db, err := sql.Open("pgx", dsn)            // "pgx" : le nom enregistré
```

L'import « blanc » `_` (N2 L8) n'importe le paquet **que** pour son `init`. L'oublier donne, à
l'exécution :

```
sql: unknown driver "pgx" (forgotten import?)
```

Pour PostgreSQL, le pilote de référence est **pgx** (`github.com/jackc/pgx/v5`). Il s'utilise de deux
façons : à travers `database/sql` (sous-paquet `stdlib`, ce que fait ce niveau), ou directement par son
API **native** (voir plus bas).

### Le DSN

La chaîne de connexion (*data source name*) de PostgreSQL est une URL :

```
postgres://utilisateur:motdepasse@hôte:5432/base?sslmode=disable
```

Le paramètre `sslmode` décide du chiffrement. Sans lui, pgx utilise `prefer` : il **tente** TLS, puis
se rabat sur une connexion en clair si le serveur refuse — on le voit dans les messages d'erreur, qui
montrent les deux tentatives. En production on exige `verify-full` (chiffrement **et** vérification du
certificat, N7 L11) ; en local, `disable`.

Le DSN contient un **secret** : il vient de l'environnement (`DATABASE_URL`), jamais du code source, et
ne s'écrit jamais dans un journal.

### `*sql.DB` est un pool, pas une connexion

`sql.Open` **ne se connecte pas**. Il valide le nom du pilote, prépare une structure, et rend un
`*sql.DB` **vide**. Les connexions sont ouvertes **à la demande** : à la première requête, puis chaque
fois que toutes les connexions existantes sont occupées.

Conséquences :

- `sql.Open` réussit avec un mot de passe faux. Pour vérifier la configuration **au démarrage**, il faut
  `db.PingContext(ctx)`, qui ouvre une vraie connexion.
- Un `*sql.DB` est **sûr en concurrence** (N6) : on en crée **un seul** pour tout le programme, et on le
  passe à qui en a besoin. Ouvrir un `*sql.DB` par requête HTTP, c'est créer un pool par requête :
  des milliers de connexions, et une base qui refuse les suivantes.
- `db.Close()` ferme **le pool** : on l'appelle une fois, à l'arrêt du programme.

`db.Stats()` montre l'état du pool (connexions ouvertes, utilisées, inactives, attentes). La leçon 5
y revient en détail.

### Trois façons d'envoyer du SQL

| Méthode | Pour | Rend |
|---|---|---|
| `ExecContext(ctx, sql, args...)` | `INSERT`, `UPDATE`, `DELETE`, DDL — sans lignes en retour | `sql.Result` |
| `QueryRowContext(ctx, sql, args...)` | une ligne exactement | `*sql.Row` (à `Scan`) |
| `QueryContext(ctx, sql, args...)` | zéro, une ou plusieurs lignes | `*sql.Rows` (à parcourir **et fermer**) |

Les versions **sans** `Context` (`Exec`, `Query`…) existent pour des raisons historiques : elles
utilisent `context.Background()` et ne peuvent pas être annulées. Ce niveau n'utilise que les versions
`Context` ; la leçon 2 détaille `QueryRow` et `Query`.

`sql.Result` donne `RowsAffected()` (lignes touchées par un `UPDATE` ou un `DELETE`, utile pour détecter
« rien à mettre à jour »). Son autre méthode, `LastInsertId()`, vient de MySQL et n'existe pas en
PostgreSQL :

```
LastInsertId is not supported by this driver
```

En PostgreSQL, on récupère l'identifiant généré avec `INSERT … RETURNING id` et `QueryRowContext`
(leçon 2).

### Les paramètres : `$1`, `$2`…

Une valeur venant de l'extérieur n'est **jamais** concaténée dans le SQL. Elle passe en **paramètre** :

```go
db.ExecContext(ctx, `INSERT INTO books (title, author) VALUES ($1, $2)`, title, author)
```

Le SQL et les valeurs voyagent **séparément** jusqu'au serveur : une valeur ne peut pas devenir du SQL,
quelles que soient les apostrophes qu'elle contient (leçon 3). La syntaxe des marqueurs dépend de la
base : `$1` pour PostgreSQL, `?` pour MySQL et SQLite. Un `?` envoyé à PostgreSQL donne :

```
ERROR: syntax error at or near ")" (SQLSTATE 42601)
```

Un `ExecContext` **sans** paramètre peut contenir plusieurs instructions séparées par `;` (pratique
pour un schéma). **Avec** paramètres, c'est refusé :

```
ERROR: cannot insert multiple commands into a prepared statement (SQLSTATE 42601)
```

### L'état de session appartient à une connexion

Une table temporaire, un `SET search_path`, un `SET statement_timeout`, un verrou consultatif : tout cela
vit dans **une** connexion PostgreSQL. Or deux appels successifs à `db.ExecContext` peuvent passer par
**deux connexions différentes** du pool. Mesuré :

```go
db.ExecContext(ctx, "CREATE TEMP TABLE brouillon(n int)")      // connexion n°1
rows, _ := db.QueryContext(ctx, "SELECT 1")                     // occupe la n°1
_, err := db.ExecContext(ctx, "INSERT INTO brouillon VALUES (1)") // ouvre la n°2
// ERROR: relation "brouillon" does not exist (SQLSTATE 42P01)
```

Ce code « marche » tant qu'une seule goroutine utilise la base, et casse en charge. Quand une suite
d'instructions doit partager une session, on **réserve** une connexion avec `db.Conn(ctx)` (et on la
rend avec `conn.Close()`), ou on utilise une **transaction** (leçon 4), qui est elle aussi attachée à
une connexion.

### pgx natif

pgx a aussi sa propre API, sans `database/sql` :

```go
import "github.com/jackc/pgx/v5/pgxpool"

pool, err := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
defer pool.Close()
err = pool.QueryRow(ctx, "SELECT title FROM books WHERE author = $1", "Camus").Scan(&title)
```

| | `database/sql` + `pgx/stdlib` | `pgxpool` (natif) |
|---|---|---|
| Portabilité | même API pour toutes les bases | PostgreSQL uniquement |
| Écosystème | accepté partout (outils de migration, générateurs, bibliothèques) | moins d'outils, mais les principaux le gèrent |
| Fonctions PostgreSQL | les bases : requêtes, transactions | en plus : `COPY`, *batch*, `LISTEN/NOTIFY`, types riches (tableaux, intervalles) |
| Performance | très bonne | un peu meilleure (moins de conversions) |

Les deux sont des choix professionnels. Pour un service **uniquement** PostgreSQL, `pgxpool` est un
excellent défaut. Ce niveau utilise `database/sql` parce que tout ce qu'on y apprend vaut pour **toutes**
les bases, et que le modèle (pool, lignes, transactions) est le même ; la leçon 8 montre ce que pgx natif
apporte en plus. Une fois choisi, on s'y tient dans tout le projet.

## Exemple

Lancer PostgreSQL et définir `DATABASE_URL` comme indiqué dans le [README du niveau](../README.md).

```go
package main

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"log"
	"os"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib" // enregistre le pilote "pgx" auprès de database/sql
)

func main() {
	if err := run(context.Background()); err != nil {
		log.Fatal(err)
	}
}

func run(ctx context.Context) error {
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		return errors.New("DATABASE_URL non définie")
	}

	// 1. Open ne se connecte PAS : il prépare un pool vide.
	db, err := sql.Open("pgx", dsn)
	if err != nil {
		return fmt.Errorf("open : %w", err)
	}
	defer db.Close() // ferme le pool à la fin du programme, pas après chaque requête

	// 2. Ping ouvre une vraie connexion : c'est ici qu'on découvre un mauvais mot de passe.
	pctx, cancel := context.WithTimeout(ctx, 3*time.Second)
	defer cancel()
	if err := db.PingContext(pctx); err != nil {
		return fmt.Errorf("ping : %w", err)
	}

	// 3. Exec : une instruction qui ne rend pas de lignes.
	const schema = `
		DROP TABLE IF EXISTS l1_books;
		CREATE TABLE l1_books (
			id     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
			title  text NOT NULL,
			author text NOT NULL
		)`
	if _, err := db.ExecContext(ctx, schema); err != nil {
		return fmt.Errorf("schéma : %w", err)
	}

	// 4. Paramètres $1, $2 : jamais de concaténation de chaînes.
	res, err := db.ExecContext(ctx,
		`INSERT INTO l1_books (title, author) VALUES ($1, $2), ($3, $4)`,
		"L'Étranger", "Camus", "Le Petit Prince", "Saint-Exupéry")
	if err != nil {
		return fmt.Errorf("insert : %w", err)
	}
	n, _ := res.RowsAffected()
	fmt.Println("lignes insérées :", n)

	// 5. QueryRow + Scan : une ligne, une ou plusieurs colonnes (détaillé en leçon 2).
	var count int
	if err := db.QueryRowContext(ctx, `SELECT count(*) FROM l1_books`).Scan(&count); err != nil {
		return fmt.Errorf("count : %w", err)
	}
	fmt.Println("livres :", count)

	s := db.Stats()
	fmt.Printf("pool : ouvertes=%d utilisées=%d inactives=%d\n", s.OpenConnections, s.InUse, s.Idle)
	return nil
}
```

Sortie :

```
lignes insérées : 2
livres : 2
pool : ouvertes=1 utilisées=0 inactives=1
```

Avec un mot de passe faux, `Open` réussit et c'est `Ping` qui échoue :

```
2026/11/03 06:20:31 ping : failed to connect to `user=dojo database=dojo`: 127.0.0.1:5432 (127.0.0.1): failed SASL auth: FATAL: password authentication failed for user "dojo" (SQLSTATE 28P01)
exit status 1
```

Sans `sslmode`, le même message liste **deux** tentatives (TLS refusé, puis en clair).

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `_ "github.com/jackc/pgx/v5/stdlib"` | Enregistre le pilote ; le nom `"pgx"` est celui qu'attend `sql.Open`. |
| `run(ctx) error` + `log.Fatal` dans `main` | Les `defer` de `run` s'exécutent avant la sortie (N2 L7) ; `log.Fatal` dans `run` les sauterait. |
| `sql.Open` | Ne fait **aucun** accès réseau. |
| `defer db.Close()` | Une seule fois : le pool vit aussi longtemps que le programme. |
| `PingContext` avec délai | Échec rapide et explicite au démarrage si la base est inaccessible ou mal configurée. |
| Schéma sans paramètres | Plusieurs instructions dans un même `Exec` : permis parce qu'il n'y a pas de paramètre. |
| `GENERATED ALWAYS AS IDENTITY` | La façon standard (SQL) de générer un identifiant ; `bigserial` est l'ancienne forme. |
| `$1…$4` | Les valeurs voyagent à part du SQL. |
| `RowsAffected` | Le nombre de lignes touchées, fourni par le serveur. |
| `Stats()` | Une seule connexion ouverte : chaque instruction l'a rendue au pool avant la suivante. |

## Erreurs fréquentes

1. **Oublier l'import du pilote** : `unknown driver "pgx" (forgotten import?)`.
2. **Croire que `Open` a vérifié la connexion** : le service démarre « bien » et échoue à la première
   requête. Toujours `PingContext` au démarrage.
3. **Un `sql.Open` par requête ou par fonction**, ou `defer db.Close()` dans un handler : un pool créé
   puis détruit à chaque fois.
4. **Concaténer des valeurs dans le SQL** (`"… WHERE name = '" + name + "'"`) : injection SQL (leçon 3).
5. **`?` au lieu de `$1`** avec PostgreSQL.
6. **`LastInsertId`** avec PostgreSQL : utiliser `RETURNING`.
7. **Compter sur l'état de session** (table temporaire, `SET`) à travers plusieurs appels au pool.
8. **Les méthodes sans `Context`** : impossible d'annuler une requête lente quand le client HTTP est
   parti (N7 L7).
9. **Écrire le DSN dans le code** ou dans un journal : il contient un mot de passe.

## Bonnes pratiques Go

- Un `*sql.DB` par base et par programme, créé dans `main`, passé par injection (N5 L3).
- `PingContext` avec délai au démarrage ; échec = le programme ne démarre pas.
- DSN depuis l'environnement ; `sslmode=verify-full` en production.
- Uniquement les méthodes `…Context`.
- Paramètres `$n` pour **toute** valeur.
- `db.Conn(ctx)` ou une transaction quand plusieurs instructions doivent partager une session.
- Choisir `database/sql` ou `pgxpool` une fois pour tout le projet.

## Ce que je dois retenir

- `database/sql` = l'API ; le pilote (pgx) = le protocole, enregistré par un import blanc.
- `*sql.DB` est un **pool** paresseux et partagé ; `Open` ne se connecte pas, `Ping` si.
- `Exec` (sans lignes), `QueryRow` (une ligne), `Query` (plusieurs) — toujours en version `Context`.
- PostgreSQL : paramètres `$1`, identifiant par `RETURNING`.
- L'état de session vit dans **une** connexion ; deux appels au pool peuvent en utiliser deux.

---

➡️ [Exercices](exercices.md)
