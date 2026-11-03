# Leçon 1 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Ce que `Open` ne vérifie pas

| Cas | Ligne qui échoue | Message (résumé) |
|---|---|---|
| Mot de passe faux | `Ping` | `failed SASL auth: FATAL: password authentication failed for user "dojo" (SQLSTATE 28P01)` |
| Port fermé | `Ping` | `dial error: dial tcp 127.0.0.1:1: connect: connection refused` |
| Pilote `"postgres"` | `Open` | `sql: unknown driver "postgres" (forgotten import?)` |
| Sans `sslmode` | aucune (si le reste est bon) | — mais une erreur de connexion liste alors **deux** tentatives : `tls error: server refused TLS connection`, puis la tentative en clair |

`Open` ne vérifie que ce qu'il peut vérifier **sans réseau** : le nom du pilote. Se connecter dans
`Open` n'aurait pas de sens pour un pool : combien de connexions ouvrir ? Et que faire si la base est
momentanément indisponible ? `database/sql` laisse ces décisions au programme : c'est le rôle de
`PingContext` (et de l'exercice intermédiaire).

Remarque utile : un DSN **vide** n'est pas refusé par pgx. Il applique les valeurs par défaut de
PostgreSQL (socket Unix, utilisateur du système) et échoue avec un message déroutant :
`dial unix /tmp/.s.PGSQL.5432: connect: no such file or directory`. D'où la vérification explicite.

## E2 — Le pool vu de l'intérieur

Mesuré, 20 goroutines × `pg_sleep(0.2)` :

```
pendant : OpenConnections:20 InUse:20 Idle:0
après   : OpenConnections:2 InUse:0 Idle:2 MaxIdleClosed:18
durée=200ms

avec SetMaxOpenConns(5) :
durée=800ms attentes=15 attente_totale=6.4s
```

- Sans limite, le pool ouvre **une connexion par goroutine** : 20. Avec 2 000 requêtes simultanées, il
  essaierait d'en ouvrir 2 000, et PostgreSQL (100 connexions par défaut) refuserait les suivantes.
- **Après** la rafale, il n'en reste que 2 : `MaxIdleConns` vaut **2** par défaut, les 18 autres ont été
  fermées (`MaxIdleClosed:18`). À la rafale suivante, il faudra les rouvrir (poignée de main TCP et
  authentification chacune). La leçon 5 règle ces valeurs.
- Avec 5 connexions : 4 vagues de 0,2 s = 0,8 s ; 15 requêtes ont attendu une connexion libre, pour un
  total cumulé de 6,4 s (5 × 0,2 + 5 × 0,4 + 5 × 0,6 + arrondis).

## E3 — Qui est connecté ?

Pendant la rafale, `pg_stat_activity` montre 20 lignes `state = active` avec
`query = SELECT pg_sleep(0.2)`. Après, les 2 connexions conservées par le pool apparaissent en
`state = idle` : la colonne `query` montre alors la **dernière** requête exécutée sur cette connexion,
pas une requête en cours. Une connexion `idle in transaction` serait, elle, une transaction oubliée
(leçon 4).

## E4 — La table temporaire perdue

```go
conn, err := db.Conn(ctx)
if err != nil {
	return err
}
defer conn.Close()
conn.ExecContext(ctx, "CREATE TEMP TABLE brouillon(n int)")
conn.ExecContext(ctx, "INSERT INTO brouillon VALUES (1)")
```

Fermer le `rows` plus tôt « corrige » l'exemple du cours, mais **pas** le problème : il suffit qu'une
autre goroutine prenne la connexion n°1 entre les deux appels pour que l'erreur revienne, au hasard et
seulement en charge. `db.Conn` (ou une transaction) est la seule correction robuste : elle garantit
**la même** connexion.

## E5 — `RowsAffected` comme information

```go
var ErrNotFound = errors.New("livre introuvable")

func Rename(ctx context.Context, db *sql.DB, id int64, title string) error {
	res, err := db.ExecContext(ctx, `UPDATE l1_books SET title = $1 WHERE id = $2`, title, id)
	if err != nil {
		return fmt.Errorf("rename %d : %w", id, err)
	}
	n, err := res.RowsAffected()
	if err != nil {
		return fmt.Errorf("rename %d : %w", id, err)
	}
	if n == 0 {
		return fmt.Errorf("rename %d : %w", id, ErrNotFound)
	}
	return nil
}
```

Un `UPDATE` qui ne touche aucune ligne n'est **pas** une erreur SQL : sans ce test, le client recevrait
un 200 pour un livre qui n'existe pas.

## Exercice intermédiaire — Démarrage robuste

```go
// Fichier : dbx/open.go
package dbx

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"io"
	"net/url"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
)

// OpenDB ouvre le pool et attend que la base réponde (5 tentatives, attente doublée).
func OpenDB(ctx context.Context, dsn string, log io.Writer) (*sql.DB, error) {
	if dsn == "" {
		return nil, errors.New("DSN vide")
	}
	u, err := url.Parse(dsn)
	if err != nil {
		return nil, errors.New("DSN illisible") // pas %w : le message d'url.Parse contient le DSN, donc le mot de passe
	}
	if u.Scheme != "postgres" && u.Scheme != "postgresql" {
		return nil, fmt.Errorf("DSN : schéma %q non supporté", u.Scheme)
	}
	db, err := sql.Open("pgx", dsn)
	if err != nil {
		return nil, fmt.Errorf("open : %w", err)
	}
	wait := 100 * time.Millisecond
	for attempt := 1; ; attempt++ {
		pctx, cancel := context.WithTimeout(ctx, 2*time.Second)
		err = db.PingContext(pctx)
		cancel()
		if err == nil {
			fmt.Fprintf(log, "base prête (%s), tentative %d\n", u.Redacted(), attempt)
			return db, nil
		}
		fmt.Fprintf(log, "tentative %d sur %s : %v\n", attempt, u.Redacted(), err)
		if attempt == 5 {
			break
		}
		t := time.NewTimer(wait)
		select {
		case <-ctx.Done():
			t.Stop()
			db.Close()
			return nil, ctx.Err()
		case <-t.C:
		}
		wait *= 2
	}
	db.Close() // sinon le pool (et ses éventuelles connexions) fuit
	return nil, fmt.Errorf("base injoignable après 5 tentatives : %w", err)
}
```

Points de vigilance :

- `url.Parse` échoue sur un mot de passe mal encodé (`%zz`) **et cite le DSN** dans son message : on ne
  l'enveloppe pas, sinon le mot de passe part dans le journal.
- `u.Redacted()` remplace le mot de passe par `xxxxx`.
- Chaque `Ping` a son propre délai (2 s) : une base qui ne répond pas du tout (paquets perdus) ne bloque
  pas la tentative indéfiniment.
- L'attente passe par `select` sur `ctx.Done()` : `SIGTERM` pendant le démarrage arrête tout de suite.
- En cas d'abandon, `db.Close()` : un `*sql.DB` abandonné garde ses éventuelles connexions ouvertes.

Les tests (`DATABASE_URL` requise pour les deux derniers, sinon `t.Skip`) :

```go
// Fichier : dbx/open_test.go
package dbx

import (
	"bytes"
	"context"
	"errors"
	"strings"
	"testing"
	"time"
)

func TestOpenDBInvalid(t *testing.T) {
	for _, dsn := range []string{"", "mysql://u:p@h/db", "postgres://u:%zz@h/db"} {
		if _, err := OpenDB(t.Context(), dsn, new(bytes.Buffer)); err == nil {
			t.Errorf("OpenDB(%q) : erreur attendue", dsn)
		} else if strings.Contains(err.Error(), "%zz") {
			t.Errorf("le mot de passe fuit : %v", err)
		}
	}
}

func TestOpenDBUnreachable(t *testing.T) {
	var log bytes.Buffer
	start := time.Now()
	_, err := OpenDB(t.Context(), "postgres://u:secret@127.0.0.1:1/db?sslmode=disable", &log)
	if err == nil {
		t.Fatal("erreur attendue")
	}
	if n := strings.Count(log.String(), "tentative"); n != 5 {
		t.Errorf("%d tentatives, attendu 5", n)
	}
	if strings.Contains(log.String(), "secret") {
		t.Error("mot de passe dans le journal")
	}
	if d := time.Since(start); d < 1500*time.Millisecond {
		t.Errorf("durée %v : les attentes (100+200+400+800 ms) n'ont pas eu lieu", d)
	}
}

func TestOpenDBCanceled(t *testing.T) {
	ctx, cancel := context.WithTimeout(t.Context(), 150*time.Millisecond)
	defer cancel()
	_, err := OpenDB(ctx, "postgres://u:p@127.0.0.1:1/db?sslmode=disable", new(bytes.Buffer))
	if !errors.Is(err, context.DeadlineExceeded) {
		t.Fatalf("err = %v", err)
	}
}
```

Le test « base arrêtée puis redémarrée » se fait à la main : `docker stop dojo-pg`, lancer le
programme, `docker start dojo-pg` pendant les tentatives. On voit les premières tentatives échouer puis
`base prête … tentative 4`. Si la base met plus de 1,5 s à démarrer, 5 tentatives ne suffisent pas :
en production, on rend le nombre de tentatives configurable, ou on laisse l'orchestrateur relancer le
service.

## Défi — Une session réservée, proprement

```go
// Fichier : dbx/conn.go
package dbx

import (
	"context"
	"database/sql"
)

// WithConn réserve une connexion, exécute fn, nettoie la session et rend la connexion,
// même si fn panique.
func WithConn(ctx context.Context, db *sql.DB, fn func(*sql.Conn) error) error {
	conn, err := db.Conn(ctx)
	if err != nil {
		return err
	}
	defer conn.Close() // rend la connexion au pool
	defer func() {
		// Remet la session à zéro : verrous consultatifs, tables temporaires, SET, préparées.
		// context.WithoutCancel : le nettoyage doit avoir lieu même si ctx est annulé.
		_, _ = conn.ExecContext(context.WithoutCancel(ctx), "DISCARD ALL")
	}()
	return fn(conn)
}
```

**Le verrou oublié.** Les verrous consultatifs de session sont **réentrants** : la session qui tient
déjà le verrou 42 l'obtient à nouveau immédiatement (mesuré : deux `pg_advisory_lock(7)` de suite dans
la même session rendent la main tout de suite). Donc :

- la prochaine goroutine qui obtient **la même** connexion « prend » le verrou sans attendre, alors
  qu'une autre section critique est censée le tenir : **l'exclusion mutuelle est rompue**, en silence ;
- une goroutine qui obtient une **autre** connexion attend **indéfiniment**, puisque personne ne
  relâchera ce verrou tant que la connexion vit (jusqu'à `ConnMaxLifetime`, leçon 5, ou jamais).

Parade : `DISCARD ALL` avant de rendre la connexion. Il relâche les verrous consultatifs, supprime les
tables temporaires, annule les `SET` et les instructions préparées. Le test suivant échoue si on le
retire :

```go
// Fichier : dbx/conn_test.go
package dbx

import (
	"bytes"
	"context"
	"database/sql"
	"os"
	"strings"
	"sync"
	"testing"
	"time"
)

func openTest(t *testing.T) *sql.DB {
	t.Helper()
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		t.Skip("DATABASE_URL non définie")
	}
	db, err := OpenDB(t.Context(), dsn, new(bytes.Buffer))
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { db.Close() })
	return db
}

func TestAdvisoryLockSerializes(t *testing.T) {
	db := openTest(t)
	ctx := t.Context()
	var mu sync.Mutex
	var order []string
	var wg sync.WaitGroup
	for _, name := range []string{"A", "B"} {
		wg.Go(func() {
			err := WithConn(ctx, db, func(c *sql.Conn) error {
				if _, err := c.ExecContext(ctx, "SELECT pg_advisory_lock(42)"); err != nil {
					return err
				}
				mu.Lock()
				order = append(order, name+" entre")
				mu.Unlock()
				time.Sleep(200 * time.Millisecond)
				mu.Lock()
				order = append(order, name+" sort")
				mu.Unlock()
				_, err := c.ExecContext(ctx, "SELECT pg_advisory_unlock(42)")
				return err
			})
			if err != nil {
				t.Error(err)
			}
		})
	}
	wg.Wait()
	// Exclusion mutuelle : « X entre, X sort, Y entre, Y sort », jamais entrelacé.
	if len(order) != 4 || order[0][0] != order[1][0] || !strings.HasSuffix(order[1], "sort") {
		t.Fatalf("ordre = %v", order)
	}
}

func TestLeakedLockIsDiscarded(t *testing.T) {
	db := openTest(t)
	db.SetMaxOpenConns(1) // la même connexion revient forcément
	ctx := t.Context()
	err := WithConn(ctx, db, func(c *sql.Conn) error {
		_, err := c.ExecContext(ctx, "SELECT pg_advisory_lock(43)") // oubli volontaire de l'unlock
		return err
	})
	if err != nil {
		t.Fatal(err)
	}
	var n int
	if err := db.QueryRowContext(ctx,
		"SELECT count(*) FROM pg_locks WHERE locktype = 'advisory' AND objid = 43").Scan(&n); err != nil {
		t.Fatal(err)
	}
	if n != 0 {
		t.Fatalf("%d verrou(s) 43 encore tenus", n)
	}
}

func TestWithConnPanicReturnsConn(t *testing.T) {
	db := openTest(t)
	db.SetMaxOpenConns(1)
	func() {
		defer func() { _ = recover() }()
		_ = WithConn(t.Context(), db, func(*sql.Conn) error { panic("boum") })
	}()
	// Si la connexion n'était pas rendue, ce Ping bloquerait jusqu'au délai.
	ctx, cancel := context.WithTimeout(t.Context(), time.Second)
	defer cancel()
	if err := db.PingContext(ctx); err != nil {
		t.Fatalf("connexion non rendue : %v", err)
	}
}
```

`go test -race ./...` passe (3,2 s, surtout les attentes de `TestOpenDBUnreachable`). Remplacer
`DISCARD ALL` par une instruction neutre fait échouer `TestLeakedLockIsDiscarded`.

Alternative encore plus sûre pour un verrou : `pg_advisory_xact_lock` **dans une transaction** (leçon 4)
— le verrou est relâché automatiquement à la fin de la transaction, `Commit` ou `Rollback`.

## Quiz

1. `database/sql` fournit l'API générique (pool, requêtes, transactions) ; le pilote implémente le
   protocole d'une base donnée.
2. Le paquet est importé uniquement pour son `init`, qui enregistre le pilote ; on n'utilise aucun de ses
   identifiants.
3. `Open` vérifie le nom du pilote et prépare un pool vide, sans réseau ; `PingContext` ouvre (ou
   réutilise) une connexion et vérifie que la base répond.
4. Un seul, partagé par tout le programme : c'est un pool sûr en concurrence.
5. `Exec` : sans lignes en retour (`sql.Result`) ; `QueryRow` : exactement une ligne ; `Query` : un
   ensemble de lignes à parcourir et fermer.
6. Elles permettent l'annulation et les délais (client parti, arrêt du serveur).
7. `$1`, `$2`, … dans l'ordre des arguments.
8. `INSERT … RETURNING id` avec `QueryRowContext(…).Scan(&id)`.
9. Elle vit dans une connexion ; l'appel suivant peut passer par une autre connexion du pool.
10. Une connexion en clair : identifiants et données lisibles sur le réseau. À réserver au poste local.
