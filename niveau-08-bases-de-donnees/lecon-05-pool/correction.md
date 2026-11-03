# Leçon 5 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Le coût de la réserve

La durée décroît à mesure que `MaxIdleConns` approche de 20, et `MaxIdleClosed` tombe à 0 quand la
réserve couvre la rafale (mesure du cours : 1,23 s et ≈ 870 fermetures à 2 ; 45 ms et 0 à 20). Dans
`pg_stat_activity`, la version à 2 montre un **va-et-vient** permanent : à chaque rafale, des processus
serveur apparaissent (nouveaux `pid`) puis disparaissent. Chaque connexion PostgreSQL est un **processus**
du système : créer et détruire des centaines de processus par seconde coûte aussi côté serveur.

## E2 — Les quatre fuites

Avec `MaxOpenConns(3)` : **3 appels** suffisent dans les quatre cas ; le quatrième appel, même correct,
attend une connexion jusqu'à son délai. Ce que montre la base (mesuré avec le vérificateur de
l'exercice intermédiaire) :

| Fuite | `state` dans `pg_stat_activity` | `query` |
|---|---|---|
| `Rows` lu partiellement | `idle` | `SELECT generate_series(1, 10)` |
| `QueryRow` sans `Scan` | `idle` | `SELECT 1` |
| transaction sans fin | **`idle in transaction`** | `SELECT 1` |
| `db.Conn` sans `Close` | `idle` | `SELECT 1` |

Seule la transaction est visible par son **état**. Pour les trois autres, le serveur a fini son travail
(il a envoyé toutes les lignes) : la connexion est inactive **côté base** mais retenue **côté Go**. La
base ne peut pas voir ces fuites ; seul `db.Stats().InUse` le peut. Indice précieux tout de même : pour
une connexion `idle`, la colonne `query` contient la **dernière** requête exécutée, donc la requête
coupable.

## E3 — Le filet de l'annulation

Mesuré, `MaxOpenConns(2)`, handler qui fuit avec `r.Context()` :

```
séquentiel : 0 échecs en 61ms ; concurrent : 0 échecs en 48ms ; attentes=97 attente totale=1.641s
```

Aucun échec : quand le handler rend la main, le serveur HTTP annule `r.Context()`, et `database/sql`
ferme les `Rows` associés. Mais en concurrent, **97 requêtes sur 100 ont attendu** une connexion
(1,6 s d'attente cumulée) : chaque requête garde sa connexion jusqu'à la toute fin du handler (écriture
de la réponse comprise) au lieu de la rendre dès la fin de la lecture. Avec un handler plus long (appel
à un autre service, gros JSON à écrire), la fuite « bénigne » devient une saturation. Et la même fonction
appelée depuis une tâche de fond fuirait pour de bon.

## E4 — Trop de connexions

Mesuré :

```
sans plafond : 164 erreurs en 1,7 s — FATAL: sorry, too many clients already (SQLSTATE 53300)
plafond 20   :   0 erreur  en 7,6 s
```

Sans plafond, plus de la moitié des requêtes échouent, et la base refuse aussi les connexions des
**autres** services et des administrateurs (au-delà des 3 réservées aux superutilisateurs). Avec 20
connexions, tout passe, en 15 vagues de 0,5 s.

Avec des requêtes HTTP à 2 s de délai, aucune des deux versions n'est satisfaisante : avec le plafond,
seules les 4 premières vagues (80 requêtes) passent dans les 2 s, les autres expirent en attendant
une connexion. Mais la version plafonnée échoue **proprement** : des délais côté service (→ 503), une
base qui reste disponible pour tout le monde, et un comportement prévisible. La vraie réponse est en
amont : la capacité ne suffit pas pour cette charge, il faut soit accélérer la requête, soit rejeter
tôt (N6 L11), soit ajouter de la capacité.

## E5 — Dimensionner sur le papier

- Requêtes SQL : 1 500 × 3 = 4 500/s, de 2 ms chacune → 4 500 × 0,002 = **9 connexions occupées en
  moyenne**, pour tout le service, soit 1,5 par instance.
- Budget : (200 − 20) / 6 = **30 connexions** au plus par instance.
- Choix : `MaxOpenConns = 10` par instance (60 au total). Plus de six fois la moyenne : de quoi absorber
  les pointes et la variance des durées, en restant à 60 sur 180 disponibles.
- Le jour où l'on passe à 12 instances : 12 × 10 = 120, encore sous 180. Mais une valeur de 30 (« tout
  le budget ») aurait donné 360 et des `too many clients already` au premier pic. D'où la règle :
  dimensionner pour le besoin, pas pour le budget, et recalculer à chaque changement du nombre
  d'instances (ou passer par un *pooler* externe comme PgBouncer).

## Exercice intermédiaire — Un test qui attrape les fuites

```go
// Fichier : poolx/leaks.go
package poolx

import (
	"context"
	"database/sql"
)

// Quatre fonctions qui fuient une connexion chacune (E2).

func LeakRows(ctx context.Context, db *sql.DB) error {
	rows, err := db.QueryContext(ctx, `SELECT generate_series(1, 10)`)
	if err != nil {
		return err
	}
	rows.Next() // lu partiellement, jamais fermé
	return nil
}

func LeakRow(ctx context.Context, db *sql.DB) error {
	_ = db.QueryRowContext(ctx, `SELECT 1`) // Scan jamais appelé
	return nil
}

func LeakTx(ctx context.Context, db *sql.DB) error {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	_, err = tx.ExecContext(ctx, `SELECT 1`) // ni Commit ni Rollback
	return err
}

func LeakConn(ctx context.Context, db *sql.DB) error {
	conn, err := db.Conn(ctx)
	if err != nil {
		return err
	}
	_, err = conn.ExecContext(ctx, `SELECT 1`) // conn.Close() oublié
	return err
}
```

```go
// Fichier : poolx/check.go
package poolx

import (
	"context"
	"database/sql"
	"fmt"
	"strings"
	"testing"
	"time"
)

// LeakError rend une erreur si des connexions de db restent utilisées après grace, avec ce que
// la base sait des connexions de l'application appName (fixée par application_name dans le DSN).
func LeakError(db *sql.DB, probeDSN, appName string, grace time.Duration) error {
	deadline := time.Now().Add(grace)
	for db.Stats().InUse > 0 && time.Now().Before(deadline) {
		time.Sleep(10 * time.Millisecond) // les fermetures déclenchées par un contexte sont asynchrones
	}
	inUse := db.Stats().InUse
	if inUse == 0 {
		return nil
	}
	return fmt.Errorf("%d connexion(s) encore utilisée(s) ; côté base : [%s]",
		inUse, strings.Join(serverView(probeDSN, appName), " | "))
}

// serverView interroge pg_stat_activity par un pool NEUF : celui qu'on inspecte est peut-être plein.
func serverView(probeDSN, appName string) []string {
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()
	probe, err := sql.Open("pgx", probeDSN)
	if err != nil {
		return []string{err.Error()}
	}
	defer probe.Close()
	rows, err := probe.QueryContext(ctx, `
		SELECT state, left(query, 50) FROM pg_stat_activity
		WHERE application_name = $1 AND pid <> pg_backend_pid()
		ORDER BY state`, appName)
	if err != nil {
		return []string{err.Error()}
	}
	defer rows.Close()
	var out []string
	for rows.Next() {
		var state, q string
		if err := rows.Scan(&state, &q); err != nil {
			return append(out, err.Error())
		}
		out = append(out, state+": "+q)
	}
	return out
}

// CheckNoLeak enregistre la vérification de fin de test. Le pool doit être PROPRE au test.
func CheckNoLeak(t testing.TB, db *sql.DB, probeDSN, appName string) {
	t.Helper()
	t.Cleanup(func() {
		if err := LeakError(db, probeDSN, appName, 200*time.Millisecond); err != nil {
			t.Error(err)
		}
	})
}
```

```go
// Fichier : poolx/poolx_test.go
package poolx

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"net/http"
	"net/http/httptest"
	"os"
	"sync"
	"sync/atomic"
	"testing"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
)

func dsnFor(t *testing.T, app string) string {
	t.Helper()
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		t.Skip("DATABASE_URL non définie")
	}
	return dsn + "&application_name=" + app
}

func openPool(t *testing.T, dsn string, max int) *sql.DB {
	t.Helper()
	db, err := sql.Open("pgx", dsn)
	if err != nil {
		t.Fatal(err)
	}
	db.SetMaxOpenConns(max)
	db.SetMaxIdleConns(max)
	t.Cleanup(func() { db.Close() })
	return db
}

func TestLeaksAreDetected(t *testing.T) {
	leaks := []struct {
		name string
		fn   func(context.Context, *sql.DB) error
	}{{"rows", LeakRows}, {"row", LeakRow}, {"tx", LeakTx}, {"conn", LeakConn}}
	for _, l := range leaks {
		t.Run(l.name, func(t *testing.T) {
			app := "leaktest_" + l.name
			dsn := dsnFor(t, app)
			db := openPool(t, dsn, 3)
			for range 3 {
				if err := l.fn(context.Background(), db); err != nil {
					t.Fatal(err)
				}
			}
			// Le 4e appel, même correct, ne trouve plus de connexion.
			ctx, cancel := context.WithTimeout(context.Background(), 300*time.Millisecond)
			defer cancel()
			if err := db.PingContext(ctx); !errors.Is(err, context.DeadlineExceeded) {
				t.Fatalf("pool non bloqué après 3 fuites : %v", err)
			}
			err := LeakError(db, dsn, app, 100*time.Millisecond)
			if err == nil {
				t.Fatal("fuite non détectée")
			}
			t.Log(err)
		})
	}
}

func TestNoLeakPasses(t *testing.T) {
	dsn := dsnFor(t, "leaktest_ok")
	db := openPool(t, dsn, 3)
	CheckNoLeak(t, db, dsn, "leaktest_ok")
	for range 10 {
		var n int
		if err := db.QueryRowContext(t.Context(), `SELECT 1`).Scan(&n); err != nil {
			t.Fatal(err)
		}
	}
}

// E3 : une fuite de Rows dans un handler qui utilise r.Context().
func TestLeakInHandler(t *testing.T) {
	db := openPool(t, dsnFor(t, "leaktest_http"), 2)
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if err := LeakRows(r.Context(), db); err != nil {
			http.Error(w, err.Error(), 500)
		}
	}))
	defer srv.Close()
	get := func() bool {
		ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
		defer cancel()
		req, _ := http.NewRequestWithContext(ctx, "GET", srv.URL, nil)
		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return false
		}
		resp.Body.Close()
		return resp.StatusCode == 200
	}
	var seqFail int
	start := time.Now()
	for range 100 {
		if !get() {
			seqFail++
		}
	}
	seqDur := time.Since(start)
	var concFail atomic.Int64
	var wg sync.WaitGroup
	start = time.Now()
	for range 100 {
		wg.Go(func() {
			if !get() {
				concFail.Add(1)
			}
		})
	}
	wg.Wait()
	s := db.Stats()
	t.Logf("séquentiel : %d échecs en %v ; concurrent : %d échecs en %v ; attentes=%d attente totale=%v",
		seqFail, seqDur.Round(time.Millisecond), concFail.Load(), time.Since(start).Round(time.Millisecond),
		s.WaitCount, s.WaitDuration.Round(time.Millisecond))
}

// E4 : 300 goroutines, avec et sans plafond.
func TestTooManyClients(t *testing.T) {
	for _, max := range []int{0, 20} {
		t.Run(fmt.Sprint("max=", max), func(t *testing.T) {
			db := openPool(t, dsnFor(t, "leaktest_many"), max)
			var failed atomic.Int64
			var msg atomic.Value
			var wg sync.WaitGroup
			start := time.Now()
			for range 300 {
				wg.Go(func() {
					if _, err := db.ExecContext(t.Context(), `SELECT pg_sleep(0.5)`); err != nil {
						failed.Add(1)
						msg.Store(err.Error())
					}
				})
			}
			wg.Wait()
			t.Logf("max=%d : %d erreurs en %v ; exemple : %v", max, failed.Load(),
				time.Since(start).Round(100*time.Millisecond), msg.Load())
		})
	}
}
```

Messages obtenus pour les quatre fuites :

```
3 connexion(s) encore utilisée(s) ; côté base : [idle: SELECT generate_series(1, 10) | idle: SELECT generate_series(1, 10) | idle: SELECT generate_series(1, 10)]
3 connexion(s) encore utilisée(s) ; côté base : [idle: SELECT 1 | idle: SELECT 1 | idle: SELECT 1]
3 connexion(s) encore utilisée(s) ; côté base : [idle in transaction: SELECT 1 | idle in transaction: SELECT 1 | idle in transaction: SELECT 1]
3 connexion(s) encore utilisée(s) ; côté base : [idle: SELECT 1 | idle: SELECT 1 | idle: SELECT 1]
```

Détails de conception :

- `application_name` dans le DSN identifie les connexions du test dans `pg_stat_activity`.
- Le sondage passe par un pool **neuf** : le pool inspecté est peut-être plein, et une requête sur lui
  bloquerait. Il s'exclut lui-même (`pid <> pg_backend_pid()`) : sans cela, sa propre requête, sur
  plusieurs lignes, polluait le message.
- L'attente de 200 ms couvre les fermetures asynchrones déclenchées par l'annulation d'un contexte.
- Dans la suite de tests de la correction, les fuites sont testées par `LeakError` (le test **doit**
  détecter la fuite et réussit), et `CheckNoLeak` est appliqué à un test sain (`TestNoLeakPasses`). Dans
  ton propre code, on appelle `CheckNoLeak` au début de **chaque** test qui touche la base.

Point 3 : `InUse` est une propriété du **pool**. Si deux tests parallèles partagent un pool, la
connexion utilisée par l'un est vue par la vérification de l'autre : faux positifs aléatoires, ou une
vraie fuite attribuée au mauvais test. Un pool par test (petit : 2 ou 3 connexions) rend la vérification
exacte.

## Défi — Un tableau de bord du pool

Esquisse du calcul par intervalle (les compteurs de `Stats` sont cumulatifs) :

```go
type PoolSample struct {
	Waits       int64         `json:"waits"`        // requêtes ayant attendu pendant l'intervalle
	AvgWait     time.Duration `json:"avg_wait_ns"`  // attente moyenne de ces requêtes
	Utilization float64       `json:"utilization"`  // InUse / MaxOpenConnections
}

func sample(prev, cur sql.DBStats) PoolSample {
	s := PoolSample{Waits: cur.WaitCount - prev.WaitCount}
	if s.Waits > 0 {
		s.AvgWait = (cur.WaitDuration - prev.WaitDuration) / time.Duration(s.Waits)
	}
	if cur.MaxOpenConnections > 0 {
		s.Utilization = float64(cur.InUse) / float64(cur.MaxOpenConnections)
	}
	return s
}
```

La boucle : un `time.Ticker` d'une seconde dans une goroutine, `select` sur `ctx.Done()` et `ticker.C`
(N6 L4), dernier échantillon protégé par un `sync.Mutex` (ou un `atomic.Pointer[PoolSample]`, N6 L7)
et servi en JSON par le handler.

Prédiction pour la rampe : avec des requêtes de ≈ 5 ms (plus l'aller-retour, disons 5,5 ms), une
connexion traite au plus ≈ 180 requêtes/s. Saturation attendue vers 900/s pour 5 connexions, 1 800/s
pour 10, et au-delà de la rampe pour 20. En pratique, l'attente apparaît **avant** le point théorique
(dès 70–80 % d'occupation), parce que les arrivées ne sont pas régulières : c'est exactement ce que
modélise la théorie des files d'attente (niveau 11).

« Pendant 3 secondes » : une seule seconde d'attente élevée est normale (rafale, ramasse-miettes,
`checkpoint` de la base). Une alerte qui se déclenche sur chaque pointe isolée finit ignorée ; on alerte
sur une condition **soutenue**.

## Quiz

1. Elle attend qu'une connexion soit rendue, ou que son contexte se termine.
2. Illimité et 2 : la base est saturée en pointe, et les connexions sont fermées puis rouvertes sans
   cesse.
3. Pour que les connexions ouvertes pendant une pointe restent disponibles pour la suivante.
4. À renouveler les connexions : répartition après un changement d'infrastructure, mémoire côté serveur.
5. `instances × MaxOpenConns`, à comparer à `max_connections` moins une marge.
6. Des requêtes ont dû attendre une connexion : le pool est saturé.
7. Fuite : `InUse` au plafond, base calme ; saturation : base occupée, requêtes lentes.
8. `Rows` non fermé, `Row` non scanné, transaction non terminée, `Conn` non fermé.
9. Le contexte de la requête est annulé à la fin du handler, ce qui libère la connexion ; une tâche de
   fond en `context.Background()` la retient pour toujours.
10. C'est de la backpressure : les requêtes attendent (avec un délai) au lieu de surcharger la base.
