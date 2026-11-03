# Leçon 6 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Sans verrou

Mesuré, trois exécutions sur trois :

```
migration 1 (books) : ERROR: duplicate key value violates unique constraint "pg_type_typname_nsp_index" (SQLSTATE 23505)
```

(Selon le moment exact, on peut aussi obtenir une violation de la clé primaire de `schema_migrations`,
ou `relation "…" already exists`.) `IF NOT EXISTS` est une vérification **puis** une action : deux
sessions vérifient en même temps que la table n'existe pas, puis la créent toutes les deux. La seconde
se heurte au catalogue de PostgreSQL (`pg_type` : chaque table crée aussi un type du même nom). C'est
encore le « vérifier puis agir » de N6 L6 ; seule une exclusion mutuelle (le verrou consultatif) règle le
problème.

## E2 — Le tri

`fs.Glob` rend les noms dans l'ordre **lexical** : `10_j.sql`, `1_a.sql`, `2_b.sql`… Le migrateur trie par
**numéro** (`Atoi`) : 1, 2, …, 10, correct. Un outil qui trierait les noms appliquerait `10_j` en
deuxième. D'où la convention de largeur fixe (`0001_`, `0002_`…), qui rend les deux tris identiques, et
le tri numérique dans le migrateur, qui ne dépend pas de la convention.

## E3 — L'état d'une base

La fonction `Status` est dans le code de l'exercice intermédiaire ci-dessous. Exemple de sortie, après
avoir modifié la migration 1, supprimé le fichier 2 et ajouté un fichier 3 :

```
0001 books : MODIFIÉE après application
0003 y : en attente
0002 : appliquée en base, ABSENTE des fichiers
```

La troisième ligne est le signe qu'une version du code qui n'existe plus a été déployée sur cette base
(une branche abandonnée) : son schéma est peut-être incompatible avec le code actuel.

## E4 — La file d'attente des verrous

Pendant le blocage, `pg_stat_activity` montre l'`ALTER TABLE` **et** le `SELECT` applicatif avec
`wait_event_type = Lock` (`wait_event = relation`), et `pg_locks` montre leurs demandes de verrou avec
`granted = false` : l'`ALTER` attend un `AccessExclusiveLock` que la longue transaction empêche (elle
tient un `AccessShareLock`), et le `SELECT` attend un `AccessShareLock`, **compatible** avec celui de la
longue transaction, mais placé derrière l'`ALTER` dans la file. Avec `lock_timeout = '500ms'`, l'`ALTER`
abandonne au bout de 0,5 s et le `SELECT` passe aussitôt (mesures du cours).

## E5 — Mesurer avant de migrer

Mesuré, un million de lignes :

| Opération | Durée | Verdict |
|---|---|---|
| `ADD COLUMN c1 text` (sans défaut) | 2 ms | sans risque |
| `ADD COLUMN … DEFAULT now()` | 1,4 ms | sans risque : `now()` est **stable** (une valeur par transaction), pas volatile |
| `ALTER COLUMN s TYPE text` depuis `varchar(50)` | 1,1 ms | sans risque : types binairement compatibles, pas de réécriture |
| `ADD CONSTRAINT ck1 CHECK (id > 0)` | 94 ms | dangereux : parcourt toute la table sous verrou **exclusif** |
| `ADD CONSTRAINT ck2 … NOT VALID` | 2,3 ms | sans risque : ne vérifie que les **nouvelles** lignes |
| `VALIDATE CONSTRAINT ck2` | 65 ms | acceptable : parcourt la table, mais sous un verrou qui **laisse passer** lectures et écritures |

Sont aussi dangereux (mesurés dans le cours) : défaut volatile (`random()`), changement de type qui
convertit les données, `CREATE INDEX` sans `CONCURRENTLY`. Retenir : ce n'est pas la commande qui compte,
c'est « réécrit-elle la table ? » et « quel verrou, combien de temps ? ». La documentation de
PostgreSQL (chapitre `ALTER TABLE`) l'indique pour chaque forme ; dans le doute, mesurer sur une copie.

## Exercice intermédiaire — Migrations sans transaction

```go
// Fichier : migrate/migrate.go
package migrate

import (
	"context"
	"crypto/sha256"
	"database/sql"
	"encoding/hex"
	"errors"
	"fmt"
	"io/fs"
	"path"
	"slices"
	"strconv"
	"strings"
	"time"

	"github.com/jackc/pgx/v5/pgconn"
)

const (
	noTxMarker  = "-- migrate:no-transaction"
	splitMarker = "-- migrate:split"
	lockID      = 727_001
)

type Migration struct {
	Version int
	Name    string
	SQL     string
	Sum     string
	NoTx    bool
}

// Statements découpe une migration no-transaction : une instruction par Exec, car plusieurs
// instructions envoyées ensemble s'exécutent dans une transaction implicite.
func (m Migration) Statements() []string {
	var out []string
	for part := range strings.SplitSeq(m.SQL, splitMarker) {
		if s := strings.TrimSpace(strings.TrimPrefix(strings.TrimSpace(part), noTxMarker)); s != "" {
			out = append(out, s)
		}
	}
	return out
}

func Load(fsys fs.FS) ([]Migration, error) {
	files, err := fs.Glob(fsys, "*.sql")
	if err != nil {
		return nil, err
	}
	var ms []Migration
	for _, f := range files {
		prefix, name, ok := strings.Cut(strings.TrimSuffix(path.Base(f), ".sql"), "_")
		v, err := strconv.Atoi(prefix)
		if !ok || err != nil || v <= 0 {
			return nil, fmt.Errorf("%s : nom attendu NNNN_description.sql", f)
		}
		body, err := fs.ReadFile(fsys, f)
		if err != nil {
			return nil, err
		}
		sum := sha256.Sum256(body)
		ms = append(ms, Migration{
			Version: v, Name: name, SQL: string(body), Sum: hex.EncodeToString(sum[:8]),
			NoTx: strings.HasPrefix(strings.TrimSpace(string(body)), noTxMarker),
		})
	}
	slices.SortFunc(ms, func(a, b Migration) int { return a.Version - b.Version })
	for i := 1; i < len(ms); i++ {
		if ms[i].Version == ms[i-1].Version {
			return nil, fmt.Errorf("version %d en double", ms[i].Version)
		}
	}
	return ms, nil
}

type Options struct {
	LockTimeout time.Duration // 0 = 2 s
	Retries     int           // tentatives du migrateur entier sur 55P03 ; 0 = 3
	RetryWait   time.Duration // attente initiale, doublée à chaque fois ; 0 = 1 s
}

func isLockTimeout(err error) bool {
	var pgErr *pgconn.PgError
	return errors.As(err, &pgErr) && pgErr.Code == "55P03"
}

// Migrate applique les migrations manquantes ; sur 55P03 (lock_timeout), réessaie plus tard.
func Migrate(ctx context.Context, db *sql.DB, ms []Migration, o Options) (int, error) {
	if o.LockTimeout == 0 {
		o.LockTimeout = 2 * time.Second
	}
	if o.Retries == 0 {
		o.Retries = 3
	}
	if o.RetryWait == 0 {
		o.RetryWait = time.Second
	}
	total, wait := 0, o.RetryWait
	for attempt := 1; ; attempt++ {
		n, err := migrateOnce(ctx, db, ms, o.LockTimeout)
		total += n
		if err == nil || !isLockTimeout(err) || attempt == o.Retries {
			return total, err
		}
		select {
		case <-ctx.Done():
			return total, errors.Join(ctx.Err(), err)
		case <-time.After(wait):
		}
		wait *= 2
	}
}

func migrateOnce(ctx context.Context, db *sql.DB, ms []Migration, lockTimeout time.Duration) (int, error) {
	conn, err := db.Conn(ctx)
	if err != nil {
		return 0, err
	}
	defer conn.Close()
	if _, err := conn.ExecContext(ctx, `SELECT pg_advisory_lock($1)`, lockID); err != nil {
		return 0, err
	}
	defer conn.ExecContext(context.WithoutCancel(ctx), `SELECT pg_advisory_unlock($1)`, lockID)
	// SET n'accepte pas de paramètre $1 : la valeur est un entier formaté par nous, pas une entrée.
	if _, err := conn.ExecContext(ctx,
		fmt.Sprintf(`SET lock_timeout = %d`, lockTimeout.Milliseconds())); err != nil {
		return 0, err
	}
	defer conn.ExecContext(context.WithoutCancel(ctx), `RESET lock_timeout`)

	done, err := appliedOn(ctx, conn)
	if err != nil {
		return 0, err
	}
	applied := 0
	for _, m := range ms {
		if a, ok := done[m.Version]; ok {
			if a.Sum != m.Sum {
				return applied, fmt.Errorf("migration %d (%s) modifiée après application", m.Version, m.Name)
			}
			continue
		}
		if err := apply(ctx, conn, m); err != nil {
			return applied, fmt.Errorf("migration %d (%s) : %w", m.Version, m.Name, err)
		}
		applied++
	}
	return applied, nil
}

const record = `INSERT INTO schema_migrations (version, name, checksum) VALUES ($1, $2, $3)`

func apply(ctx context.Context, conn *sql.Conn, m Migration) error {
	if m.NoTx {
		for _, stmt := range m.Statements() {
			if _, err := conn.ExecContext(ctx, stmt); err != nil {
				return err
			}
		}
		// Enregistrée APRÈS coup : si on s'arrête entre les deux, la migration sera rejouée.
		// Ses instructions doivent donc être rejouables (IF EXISTS / IF NOT EXISTS).
		_, err := conn.ExecContext(ctx, record, m.Version, m.Name, m.Sum)
		return err
	}
	tx, err := conn.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	defer tx.Rollback()
	if _, err := tx.ExecContext(ctx, m.SQL); err != nil {
		return err
	}
	if _, err := tx.ExecContext(ctx, record, m.Version, m.Name, m.Sum); err != nil {
		return err
	}
	return tx.Commit()
}

type appliedMigration struct {
	Sum string
	At  time.Time
}

type querier interface {
	ExecContext(context.Context, string, ...any) (sql.Result, error)
	QueryContext(context.Context, string, ...any) (*sql.Rows, error)
}

func appliedOn(ctx context.Context, q querier) (map[int]appliedMigration, error) {
	if _, err := q.ExecContext(ctx, `CREATE TABLE IF NOT EXISTS schema_migrations (
		version    int PRIMARY KEY,
		name       text NOT NULL,
		checksum   text NOT NULL,
		applied_at timestamptz NOT NULL DEFAULT now())`); err != nil {
		return nil, err
	}
	rows, err := q.QueryContext(ctx, `SELECT version, checksum, applied_at FROM schema_migrations`)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	done := map[int]appliedMigration{}
	for rows.Next() {
		var v int
		var a appliedMigration
		if err := rows.Scan(&v, &a.Sum, &a.At); err != nil {
			return nil, err
		}
		done[v] = a
	}
	return done, rows.Err()
}

// Status (E3) : l'état de chaque migration, et les versions inconnues des fichiers.
func Status(ctx context.Context, db *sql.DB, ms []Migration) ([]string, error) {
	done, err := appliedOn(ctx, db)
	if err != nil {
		return nil, err
	}
	var out []string
	known := map[int]bool{}
	for _, m := range ms {
		known[m.Version] = true
		a, ok := done[m.Version]
		switch {
		case !ok:
			out = append(out, fmt.Sprintf("%04d %s : en attente", m.Version, m.Name))
		case a.Sum != m.Sum:
			out = append(out, fmt.Sprintf("%04d %s : MODIFIÉE après application", m.Version, m.Name))
		default:
			out = append(out, fmt.Sprintf("%04d %s : appliquée le %s", m.Version, m.Name, a.At.UTC().Format(time.DateTime)))
		}
	}
	var unknown []int
	for v := range done {
		if !known[v] {
			unknown = append(unknown, v)
		}
	}
	slices.Sort(unknown)
	for _, v := range unknown {
		out = append(out, fmt.Sprintf("%04d : appliquée en base, ABSENTE des fichiers", v))
	}
	return out, nil
}
```

```go
// Fichier : migrate/migrate_test.go
package migrate

import (
	"context"
	"database/sql"
	"errors"
	"os"
	"strings"
	"testing"
	"testing/fstest"
	"time"

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
	t.Cleanup(func() { db.Close() })
	if _, err := db.ExecContext(t.Context(),
		`DROP TABLE IF EXISTS schema_migrations, m_books CASCADE`); err != nil {
		t.Fatal(err)
	}
	return db
}

func file(s string) *fstest.MapFile { return &fstest.MapFile{Data: []byte(s)} }

func TestLoadOrderAndDuplicates(t *testing.T) {
	fsys := fstest.MapFS{"10_j.sql": file("SELECT 1"), "2_b.sql": file("SELECT 1"), "1_a.sql": file("SELECT 1")}
	ms, err := Load(fsys)
	if err != nil || ms[0].Version != 1 || ms[1].Version != 2 || ms[2].Version != 10 {
		t.Fatalf("ordre : %+v %v", ms, err)
	}
	fsys["0002_bis.sql"] = file("SELECT 1")
	if _, err := Load(fsys); err == nil || !strings.Contains(err.Error(), "en double") {
		t.Fatalf("doublon non détecté : %v", err)
	}
}

func TestNoTxIndexAndInvalidIndexRecovery(t *testing.T) {
	db := openTest(t)
	ctx := t.Context()
	fsys := fstest.MapFS{
		"0001_books.sql": file(`CREATE TABLE m_books (id int PRIMARY KEY, isbn text);
			INSERT INTO m_books VALUES (1, 'a'), (2, 'a');`), // doublon volontaire
		// Version naïve : un échec laisse un index INVALIDE, et la relance échoue (déjà existant).
		"0002_isbn_idx.sql": file(noTxMarker + `
			CREATE UNIQUE INDEX CONCURRENTLY m_books_isbn ON m_books (isbn)`),
	}
	ms, _ := Load(fsys)
	if _, err := Migrate(ctx, db, ms, Options{}); err == nil {
		t.Fatal("l'index unique sur des doublons doit échouer")
	}
	var valid bool
	if err := db.QueryRowContext(ctx, `SELECT indisvalid FROM pg_index
		WHERE indexrelid = 'm_books_isbn'::regclass`).Scan(&valid); err != nil || valid {
		t.Fatalf("index invalide attendu : valid=%v err=%v", valid, err)
	}
	db.ExecContext(ctx, `DELETE FROM m_books WHERE id = 2`)
	if _, err := Migrate(ctx, db, ms, Options{}); err == nil || !strings.Contains(err.Error(), "already exists") {
		t.Fatalf("relance de la version naïve : %v", err)
	}
	// Version rejouable : supprimer un éventuel reste, puis créer. Pas encore enregistrée : on peut la changer.
	fsys["0002_isbn_idx.sql"] = file(noTxMarker + `
		DROP INDEX CONCURRENTLY IF EXISTS m_books_isbn;
		` + splitMarker + `
		CREATE UNIQUE INDEX CONCURRENTLY m_books_isbn ON m_books (isbn)`)
	ms, _ = Load(fsys)
	if n, err := Migrate(ctx, db, ms, Options{}); err != nil || n != 1 {
		t.Fatalf("relance rejouable : n=%d err=%v", n, err)
	}
	db.QueryRowContext(ctx, `SELECT indisvalid FROM pg_index WHERE indexrelid = 'm_books_isbn'::regclass`).Scan(&valid)
	if !valid {
		t.Fatal("index toujours invalide")
	}
	st, err := Status(ctx, db, ms)
	if err != nil || len(st) != 2 || !strings.Contains(st[1], "appliquée le") {
		t.Fatalf("status : %q %v", st, err)
	}
}

func TestLockTimeoutRetry(t *testing.T) {
	db := openTest(t)
	ctx := t.Context()
	fsys := fstest.MapFS{"0001_books.sql": file(`CREATE TABLE m_books (id int PRIMARY KEY)`)}
	ms, _ := Load(fsys)
	if _, err := Migrate(ctx, db, ms, Options{}); err != nil {
		t.Fatal(err)
	}
	fsys["0002_col.sql"] = file(`ALTER TABLE m_books ADD COLUMN title text`)
	ms, _ = Load(fsys)

	// Une longue transaction tient la table pendant 700 ms.
	long, err := db.BeginTx(ctx, nil)
	if err != nil {
		t.Fatal(err)
	}
	long.ExecContext(ctx, `SELECT * FROM m_books`)
	go func() { time.Sleep(700 * time.Millisecond); long.Rollback() }()

	start := time.Now()
	n, err := Migrate(ctx, db, ms, Options{LockTimeout: 200 * time.Millisecond, RetryWait: 300 * time.Millisecond})
	if err != nil || n != 1 {
		t.Fatalf("n=%d err=%v", n, err)
	}
	t.Logf("migration appliquée après %v (réessais sur lock_timeout)", time.Since(start).Round(100*time.Millisecond))

	// Abandon après épuisement : erreur 55P03 inspectable.
	fsys["0003_col.sql"] = file(`ALTER TABLE m_books ADD COLUMN author text`)
	ms, _ = Load(fsys)
	long2, _ := db.BeginTx(ctx, nil)
	defer long2.Rollback()
	long2.ExecContext(ctx, `SELECT * FROM m_books`)
	_, err = Migrate(ctx, db, ms, Options{LockTimeout: 100 * time.Millisecond, Retries: 2, RetryWait: 50 * time.Millisecond})
	if !isLockTimeout(err) {
		t.Fatalf("55P03 attendu : %v", err)
	}
}

func TestStatusUnknownAndModified(t *testing.T) {
	db := openTest(t)
	ctx := t.Context()
	fsys := fstest.MapFS{
		"0001_books.sql": file(`CREATE TABLE m_books (id int PRIMARY KEY)`),
		"0002_x.sql":     file(`SELECT 1`),
	}
	ms, _ := Load(fsys)
	if _, err := Migrate(ctx, db, ms, Options{}); err != nil {
		t.Fatal(err)
	}
	delete(fsys, "0002_x.sql")
	fsys["0001_books.sql"] = file(`CREATE TABLE m_books (id bigint PRIMARY KEY)`)
	fsys["0003_y.sql"] = file(`SELECT 1`)
	ms, _ = Load(fsys)
	st, err := Status(ctx, db, ms)
	if err != nil {
		t.Fatal(err)
	}
	want := []string{"MODIFIÉE", "en attente", "ABSENTE des fichiers"}
	for i, w := range want {
		if !strings.Contains(st[i], w) {
			t.Errorf("ligne %d = %q, attendu %q", i, st[i], w)
		}
	}
	if _, err := Migrate(ctx, db, ms, Options{}); err == nil || errors.Is(err, context.Canceled) {
		t.Fatalf("migration modifiée non refusée : %v", err)
	}
}
```

`go test ./...` passe. Réponses :

**Point 2 — l'index invalide.** Un `CREATE UNIQUE INDEX CONCURRENTLY` qui échoue (ici, sur des doublons)
**laisse l'index** dans le catalogue, marqué `indisvalid = false` (vérifié par le test). Il n'est jamais
utilisé par les requêtes, mais il est **maintenu** à chaque écriture. À la relance, la même instruction
échoue : `relation "m_books_isbn" already exists`. Et `CREATE INDEX CONCURRENTLY IF NOT EXISTS` serait
pire : il **réussit** en silence en laissant l'index invalide, et la migration est enregistrée comme
appliquée. La version rejouable supprime d'abord tout reste
(`DROP INDEX CONCURRENTLY IF EXISTS`), puis crée l'index.

**Point 3 — une instruction par `Exec`.** Plusieurs instructions envoyées en un seul appel passent par le
protocole simple, et PostgreSQL les exécute dans une **transaction implicite** :

```
ERROR: DROP INDEX CONCURRENTLY cannot run inside a transaction block (SQLSTATE 25001)
```

D'où le découpage explicite sur `-- migrate:split`. Découper automatiquement sur `;` serait fragile
(points-virgules dans les chaînes, les commentaires, le corps des fonctions).

**Enregistrement après coup.** Une migration hors transaction est enregistrée **après** ses
instructions : un arrêt entre les deux la fera rejouer. Toute migration `no-transaction` doit donc être
rejouable (`IF EXISTS`, `IF NOT EXISTS`, suppression préalable), comme la version corrigée.

**Point 4 — `lock_timeout` et réessai.** `SET` n'accepte pas de paramètre `$1` (c'est une commande
utilitaire, pas une requête) : la valeur est formatée par le programme, à partir d'un `time.Duration`, et
ne vient jamais d'une entrée extérieure. Le test `TestLockTimeoutRetry` mesure une migration bloquée
700 ms par une longue transaction : elle échoue sur `55P03`, attend, réessaie, et passe au bout d'environ
0,7 s — sans jamais avoir bloqué l'application plus de 200 ms.

## Défi — Renommer une colonne sans coupure

| Étape | Migration | Code déployé |
|---|---|---|
| 1. Expand | `ALTER TABLE books ADD COLUMN title text;` (instantané) | v1 : lit et écrit `name` (inchangé) |
| 2. Double écriture | — | v2 : écrit `name` **et** `title`, lit `name` |
| 3. Rattrapage | script par lots (ci-dessous), hors migration | v2 |
| 4. Contrainte | `ALTER TABLE books ADD CONSTRAINT title_nn CHECK (title IS NOT NULL) NOT VALID;` puis `VALIDATE CONSTRAINT title_nn;` puis `ALTER COLUMN title SET NOT NULL` (PostgreSQL 12+ réutilise la contrainte validée : pas de parcours sous verrou exclusif) | v3 : lit `title`, écrit les deux |
| 5. Arrêt de `name` | `ALTER TABLE books ALTER COLUMN name DROP NOT NULL;` | v4 : n'utilise plus `name` |
| 6. Contract | `ALTER TABLE books DROP COLUMN name;` | v4 |

Chaque ligne est compatible avec la version de code qui la précède **et** celle qui la suit. L'étape 5
existe parce que v4 n'écrit plus `name` : tant que la colonne est `NOT NULL`, les insertions de v4
échoueraient.

Le rattrapage par lots :

```go
for last := int64(0); ; {
	var n int64
	err := db.QueryRowContext(ctx, `
		WITH batch AS (
			SELECT id FROM books WHERE id > $1 ORDER BY id LIMIT 1000
		), upd AS (
			UPDATE books b SET title = b.name
			FROM batch WHERE b.id = batch.id AND b.title IS NULL
		)
		SELECT coalesce(max(id), 0) FROM batch`, last).Scan(&n)
	if err != nil {
		return err
	}
	if n == 0 {
		return nil // plus rien après last
	}
	last = n
	time.Sleep(10 * time.Millisecond) // laisser respirer la base et la réplication
}
```

Chaque lot est une instruction (donc une transaction) qui ne verrouille que 1 000 lignes, quelques
millisecondes. `title IS NULL` rend le rattrapage rejouable et ne réécrit pas les lignes déjà remplies
par la double écriture de v2.

**La vue.** `CREATE VIEW books_v2 AS SELECT id, name AS title FROM books` permet au nouveau code de
**lire** sous le nouveau nom immédiatement (une vue aussi simple est même modifiable par `UPDATE` et
`INSERT`). Elle ne règle pas le fond : la colonne physique s'appelle toujours `name`, les index et
contraintes aussi, et il faudra un jour faire le vrai changement. Utile comme étape de transition ou
quand plusieurs applications lisent la même base ; pas comme fin en soi.

## Quiz

1. Les bases déjà migrées ne recevraient jamais la modification : les environnements divergent sans
   que rien ne le signale.
2. La version (clé), le nom, l'empreinte du fichier et la date d'application.
3. Pour qu'une migration qui échoue au milieu soit annulée entièrement, enregistrement compris.
4. L'exécution simultanée des migrations par plusieurs instances qui démarrent ensemble.
5. Pour l'utiliser avec `embed.FS` en production et `fstest.MapFS` en test.
6. Rarement testées, elles détruisent des données et ignorent ce que le code a écrit entre-temps.
7. Défaut volatile, changement de type avec conversion, entre autres ; `CREATE INDEX` bloque les écritures.
8. Il se place dans la file d'attente des verrous, et toutes les requêtes suivantes attendent derrière
   lui.
9. Qu'une instruction qui n'obtient pas son verrou dans le délai abandonne au lieu d'attendre (et de
   bloquer les autres).
10. Ajouter la nouvelle colonne, écrire les deux, rattraper par lots, lire la nouvelle, ajouter la
    contrainte, cesser d'écrire l'ancienne, la supprimer — un déploiement par étape.
