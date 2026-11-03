# Leçon 8 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Les mesures des exercices faciles sont produites par le fichier de tests d'exploration suivant, ajouté au
paquet `outbox` du cours :

```go
// Fichier : outbox/explore_test.go
package outbox

import (
	"context"
	"database/sql"
	"encoding/json"
	"errors"
	"fmt"
	"sync"
	"testing"
	"time"
)

var errCrash = errors.New("panne entre la base et le courtier")

func borrowNaive(ctx context.Context, db *sql.DB, bus Publisher, member, book string, crash bool) error {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	defer tx.Rollback()
	if _, err := tx.ExecContext(ctx, `INSERT INTO loans (member, book) VALUES ($1, $2)`, member, book); err != nil {
		return err
	}
	if err := tx.Commit(); err != nil {
		return err
	}
	if crash {
		return errCrash
	}
	p, _ := json.Marshal(LoanCreated{member, book})
	return bus.Publish(ctx, Message{Payload: p})
}

func TestE1DoubleWrite(t *testing.T) {
	db := newDB(t)
	bus := &flakyBus{}
	for i := range 100 {
		borrowNaive(t.Context(), db, bus, "m", fmt.Sprint(i), i%10 == 0)
	}
	var loans int
	db.QueryRow(`SELECT count(*) FROM loans`).Scan(&loans)
	uniq := map[string]bool{}
	for _, m := range bus.delivered {
		uniq[string(m.Payload)] = true
	}
	t.Logf("E1 : emprunts=%d, événements distincts publiés=%d (perdus=%d)", loans, len(uniq), loans-len(uniq))
}

// L'ancienne version du relais, qui annule tout le lot.
func relayAllOrNothing(ctx context.Context, db *sql.DB, pub Publisher, batch int) (int, error) {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return 0, err
	}
	defer tx.Rollback()
	rows, err := tx.QueryContext(ctx, `SELECT id, topic, payload FROM outbox WHERE published_at IS NULL ORDER BY id LIMIT $1 FOR UPDATE SKIP LOCKED`, batch)
	if err != nil {
		return 0, err
	}
	var msgs []Message
	for rows.Next() {
		var m Message
		rows.Scan(&m.ID, &m.Topic, &m.Payload)
		msgs = append(msgs, m)
	}
	rows.Close()
	for _, m := range msgs {
		if err := pub.Publish(ctx, m); err != nil {
			return 0, err
		}
		tx.ExecContext(ctx, `UPDATE outbox SET published_at = now() WHERE id = $1`, m.ID)
	}
	return len(msgs), tx.Commit()
}

func TestE2Livelock(t *testing.T) {
	for _, every := range []int{7, 11} {
		db := newDB(t)
		for i := range 50 {
			Borrow(t.Context(), db, "m", fmt.Sprint(i))
		}
		bus := &flakyBus{failEvery: every}
		rounds, progressed := 0, true
		for ; rounds < 200; rounds++ {
			n, err := relayAllOrNothing(t.Context(), db, bus, 10)
			if err == nil && n == 0 {
				break
			}
		}
		var pending int
		db.QueryRow(`SELECT count(*) FROM outbox WHERE published_at IS NULL`).Scan(&pending)
		progressed = pending == 0
		t.Logf("E2 failEvery=%d lots de 10 : %d tours, en attente=%d, livrés=%d, terminé=%v", every, rounds, pending, len(bus.delivered), progressed)
	}
}

func TestE3NaiveConsumer(t *testing.T) {
	db := newDB(t)
	for i := range 100 {
		Borrow(t.Context(), db, "m", fmt.Sprint(i))
	}
	bus := &flakyBus{}
	for {
		n, _ := RelayOnce(t.Context(), db, bus, 10)
		if n == 0 {
			break
		}
	}
	for range bus.delivered { // sans table processed
		db.Exec(`INSERT INTO loan_stats (member, loans) VALUES ('m', 1) ON CONFLICT (member) DO UPDATE SET loans = loan_stats.loans + 1`)
	}
	var total int
	db.QueryRow(`SELECT loans FROM loan_stats`).Scan(&total)
	t.Logf("E3 : 100 événements, livrés=%d, total compté=%d", len(bus.delivered), total)
}

func relayNoSkip(ctx context.Context, db *sql.DB, pub Publisher, batch int) (int, error) {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return 0, err
	}
	defer tx.Rollback()
	rows, err := tx.QueryContext(ctx, `SELECT id, payload FROM outbox WHERE published_at IS NULL ORDER BY id LIMIT $1 FOR UPDATE`, batch)
	if err != nil {
		return 0, err
	}
	var msgs []Message
	for rows.Next() {
		var m Message
		rows.Scan(&m.ID, &m.Payload)
		msgs = append(msgs, m)
	}
	rows.Close()
	for _, m := range msgs {
		pub.Publish(ctx, m)
		time.Sleep(time.Millisecond) // un courtier qui met 1 ms
		tx.ExecContext(ctx, `UPDATE outbox SET published_at = now() WHERE id = $1`, m.ID)
	}
	return len(msgs), tx.Commit()
}

func TestE4SkipLocked(t *testing.T) {
	for name, relay := range map[string]func(context.Context, *sql.DB, Publisher, int) (int, error){
		"FOR UPDATE": relayNoSkip,
		"SKIP LOCKED": func(ctx context.Context, db *sql.DB, p Publisher, b int) (int, error) {
			return RelayOnce(ctx, db, slowBus{p}, b)
		},
	} {
		db := newDB(t)
		for i := range 300 {
			Borrow(t.Context(), db, "m", fmt.Sprint(i))
		}
		bus := &flakyBus{}
		start := time.Now()
		var wg sync.WaitGroup
		for range 3 {
			wg.Go(func() {
				for {
					if n, err := relay(t.Context(), db, bus, 10); err != nil || n == 0 {
						return
					}
				}
			})
		}
		wg.Wait()
		var pending int
		db.QueryRow(`SELECT count(*) FROM outbox WHERE published_at IS NULL`).Scan(&pending)
		t.Logf("E4 %s : 3 relais, 300 événements, %v, en attente=%d", name, time.Since(start).Round(10*time.Millisecond), pending)
	}
}

type slowBus struct{ Publisher }

func (s slowBus) Publish(ctx context.Context, m Message) error {
	time.Sleep(time.Millisecond)
	return s.Publisher.Publish(ctx, m)
}
```

## E1 — La double écriture

Mesuré, 100 emprunts avec une panne simulée une fois sur dix entre la validation et la publication :

```
emprunts=100, événements distincts publiés=90 (perdus=10)
```

Dix emprunts n'auront **jamais** d'événement : aucun réessai ne peut les retrouver, puisque rien n'en garde
la trace. Dans l'autre ordre (publier, puis valider une transaction qui échoue), on obtient des événements
pour des emprunts qui n'existent pas : le consommateur enverra un courriel de confirmation pour un emprunt
annulé. Les deux ordres sont faux ; seule l'écriture **unique** dans la base (l'*outbox*) est juste.

## E2 — Retrouver le *livelock*

```
failEvery=7  lots de 10 : 200 tours, en attente=50, livrés=1600, terminé=false
failEvery=11 lots de 10 : 9 tours,   en attente=0,  livrés=67,   terminé=true
```

Avec un échec tous les 7 messages et des lots de 10, **chaque** lot contient un échec : tout est annulé, et le
compteur du courtier avance de 7 à chaque tour, si bien que l'échec retombe toujours dans le lot suivant.
Après 200 tours, rien n'a progressé et 1 600 messages ont été livrés (tous des doublons d'une poignée
d'événements). Avec un échec tous les 11, certains lots passent entiers : le système progresse, en
gaspillant des publications. Le *livelock* apparaît dès que la probabilité qu'un lot entier réussisse
devient nulle (ou très faible) ; sa trace de goroutines montre un relais **actif** (dans `Publish`, `Exec`
ou `Rollback`), jamais bloqué au même endroit — ce qui le distingue d'un interblocage.

## E3 — Le consommateur naïf

```
100 événements, livrés=133, total compté=133
```

Chaque doublon est compté. Le résultat **varie** d'une exécution à l'autre (ici, le courtier de test
duplique de façon déterministe, mais en vrai les doublons dépendent des pannes et des délais) : des
statistiques fausses, et fausses différemment chaque jour.

## E4 — Sans `SKIP LOCKED`

```
FOR UPDATE  : 3 relais, 300 événements, 770ms
SKIP LOCKED : 3 relais, 300 événements, 310ms
```

Sans `SKIP LOCKED`, les trois relais demandent les **mêmes** dix premières lignes : deux attendent que le
premier valide, puis relisent (les lignes sont désormais publiées) et prennent le lot suivant. Ça
fonctionne — aucun doublon, aucune perte — mais les relais travaillent **à tour de rôle**. Avec
`SKIP LOCKED`, chacun prend des lignes différentes et ils travaillent en parallèle.

## E5 — L'âge de l'*outbox*

```sql
ALTER TABLE outbox ADD COLUMN created_at timestamptz NOT NULL DEFAULT now();

SELECT count(*) AS en_attente, now() - min(created_at) AS age_du_plus_ancien
FROM outbox WHERE published_at IS NULL;
```

C'est la métrique qui résume la santé de tout le dispositif : si le relais est arrêté, le courtier en
panne, ou un message empoisonné bloque la tête de la file, l'âge grandit. Le nombre d'événements en
attente seul ne suffit pas (il peut être élevé pendant une pointe normale) ; un âge qui dépasse quelques
secondes est un incident. On l'exporte comme métrique et on alerte dessus (leçon 10).

## Exercice intermédiaire — Ordre par adhérent et messages empoisonnés

```go
// Fichier : ordered/ordered.go
// Exercice intermédiaire : séquence par adhérent, consommateur ordonné, file de rebut.
package ordered

import (
	"context"
	"database/sql"
	"encoding/json"
	"errors"
)

const Schema = `
	CREATE TABLE member_seq (member text PRIMARY KEY, last int NOT NULL);
	CREATE TABLE events (
		id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
		member  text NOT NULL,
		seq     int NOT NULL,
		payload jsonb NOT NULL,
		UNIQUE (member, seq));
	CREATE TABLE consumer_pos (consumer text, member text, last_seq int NOT NULL,
		PRIMARY KEY (consumer, member));
	CREATE TABLE applied (consumer text, member text, seq int);
	CREATE TABLE dead_letters (event_id bigint PRIMARY KEY, payload text, error text NOT NULL);`

type Event struct {
	ID      int64
	Member  string
	Seq     int
	Payload []byte
}

// Emit attribue le numéro de séquence de l'adhérent DANS la transaction : l'UPSERT
// verrouille la ligne de l'adhérent, deux émissions simultanées obtiennent n et n+1.
func Emit(ctx context.Context, db *sql.DB, member string, payload any) error {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	defer tx.Rollback()
	var seq int
	if err := tx.QueryRowContext(ctx, `
		INSERT INTO member_seq (member, last) VALUES ($1, 1)
		ON CONFLICT (member) DO UPDATE SET last = member_seq.last + 1
		RETURNING last`, member).Scan(&seq); err != nil {
		return err
	}
	p, err := json.Marshal(payload)
	if err != nil {
		return err
	}
	if _, err := tx.ExecContext(ctx,
		`INSERT INTO events (member, seq, payload) VALUES ($1, $2, $3)`, member, seq, p); err != nil {
		return err
	}
	return tx.Commit()
}

var (
	ErrTooEarly = errors.New("événement arrivé trop tôt : à relivrer")
	ErrPoison   = errors.New("message illisible")
)

// Apply : rejette ce qui est trop tôt, ignore les doublons, applique dans l'ordre.
func Apply(ctx context.Context, db *sql.DB, consumer string, e Event) (applied bool, err error) {
	var body map[string]any
	if err := json.Unmarshal(e.Payload, &body); err != nil {
		return false, errors.Join(ErrPoison, err)
	}
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return false, err
	}
	defer tx.Rollback()
	// Verrouiller la position de ce consommateur pour cet adhérent (créée à 0 au besoin).
	if _, err := tx.ExecContext(ctx, `INSERT INTO consumer_pos VALUES ($1, $2, 0) ON CONFLICT DO NOTHING`,
		consumer, e.Member); err != nil {
		return false, err
	}
	var last int
	if err := tx.QueryRowContext(ctx, `SELECT last_seq FROM consumer_pos
		WHERE consumer = $1 AND member = $2 FOR UPDATE`, consumer, e.Member).Scan(&last); err != nil {
		return false, err
	}
	switch {
	case e.Seq <= last:
		return false, nil // doublon
	case e.Seq > last+1:
		return false, ErrTooEarly
	}
	if _, err := tx.ExecContext(ctx, `INSERT INTO applied VALUES ($1, $2, $3)`, consumer, e.Member, e.Seq); err != nil {
		return false, err
	}
	if _, err := tx.ExecContext(ctx, `UPDATE consumer_pos SET last_seq = $3 WHERE consumer = $1 AND member = $2`,
		consumer, e.Member, e.Seq); err != nil {
		return false, err
	}
	return true, tx.Commit()
}

func DeadLetter(ctx context.Context, db *sql.DB, e Event, cause error) error {
	_, err := db.ExecContext(ctx, `INSERT INTO dead_letters VALUES ($1, $2, $3) ON CONFLICT DO NOTHING`,
		e.ID, string(e.Payload), cause.Error())
	return err
}
```

```go
// Fichier : ordered/ordered_test.go
package ordered

import (
	"context"
	"crypto/rand"
	"database/sql"
	"errors"
	"fmt"
	mrand "math/rand/v2"
	"os"
	"strings"
	"sync"
	"testing"

	_ "github.com/jackc/pgx/v5/stdlib"
)

func newDB(t *testing.T) *sql.DB {
	t.Helper()
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		t.Skip("DATABASE_URL non définie")
	}
	admin, _ := sql.Open("pgx", dsn)
	t.Cleanup(func() { admin.Close() })
	schema := "test_" + strings.ToLower(rand.Text()[:12])
	if _, err := admin.ExecContext(t.Context(), "CREATE SCHEMA "+schema); err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { admin.ExecContext(context.Background(), "DROP SCHEMA "+schema+" CASCADE") })
	db, _ := sql.Open("pgx", dsn+"&search_path="+schema)
	db.SetMaxOpenConns(20)
	t.Cleanup(func() { db.Close() })
	if _, err := db.ExecContext(t.Context(), Schema); err != nil {
		t.Fatal(err)
	}
	return db
}

func TestOrderedWithShuffleDuplicatesAndPoison(t *testing.T) {
	db := newDB(t)
	ctx := t.Context()
	var wg sync.WaitGroup
	for i := range 200 {
		wg.Go(func() {
			if err := Emit(ctx, db, fmt.Sprintf("m%d", i%4), map[string]int{"n": i}); err != nil {
				t.Error(err)
			}
		})
	}
	wg.Wait()
	// Un message empoisonné, ajouté directement (JSON valide pour jsonb, mais tronqué à la livraison).
	db.ExecContext(ctx, `INSERT INTO events (member, seq, payload) VALUES ('poison', 1, '{}')`)

	rows, _ := db.QueryContext(ctx, `SELECT id, member, seq, payload FROM events`)
	var queue []Event
	for rows.Next() {
		var e Event
		rows.Scan(&e.ID, &e.Member, &e.Seq, &e.Payload)
		if e.Member == "poison" {
			e.Payload = []byte(`{"n":`) // la corruption arrive « en transit »
		}
		queue = append(queue, e)
		if mrand.IntN(4) == 0 {
			queue = append(queue, e) // doublon
		}
	}
	rows.Close()
	mrand.Shuffle(len(queue), func(i, j int) { queue[i], queue[j] = queue[j], queue[i] })

	attempts := map[int64]int{}
	dead := map[int64]bool{} // en vrai : consulter la table dead_letters
	stats := map[string]int{}
	for len(queue) > 0 {
		e := queue[0]
		queue = queue[1:]
		if dead[e.ID] {
			stats["doublons"]++ // copie d'un message déjà mis au rebut
			continue
		}
		ok, err := Apply(ctx, db, "stats", e)
		switch {
		case errors.Is(err, ErrTooEarly):
			stats["relivrés"]++
			queue = append(queue, e) // relivré plus tard
		case errors.Is(err, ErrPoison):
			if attempts[e.ID]++; attempts[e.ID] < 3 {
				queue = append(queue, e)
			} else {
				DeadLetter(ctx, db, e, err)
				dead[e.ID] = true
				stats["rebut"]++
			}
		case err != nil:
			t.Fatal(err)
		case ok:
			stats["appliqués"]++
		default:
			stats["doublons"]++
		}
	}
	t.Logf("%v", stats)
	if stats["appliqués"] != 200 || stats["rebut"] != 1 {
		t.Fatalf("%v", stats)
	}
	// L'ordre : pour chaque adhérent, les seq appliquées sont 1, 2, 3… dans l'ordre d'application.
	rows, _ = db.QueryContext(ctx, `SELECT member, seq FROM applied ORDER BY ctid`)
	next := map[string]int{}
	for rows.Next() {
		var m string
		var s int
		rows.Scan(&m, &s)
		if next[m]++; s != next[m] {
			t.Fatalf("%s : seq %d appliquée au rang %d", m, s, next[m])
		}
	}
	rows.Close()
}
```

`go test -race ./...` passe. Mesuré :

```
map[appliqués:200 doublons:53 rebut:1 relivrés:2812]
```

- **La séquence** est attribuée par un `INSERT … ON CONFLICT DO UPDATE … RETURNING` sur la ligne de
  l'adhérent : l'*upsert* verrouille la ligne, deux émissions simultanées obtiennent `n` et `n+1`, sans
  mise à jour perdue (N8 L4). La contrainte `UNIQUE (member, seq)` en est le filet.
- **L'ordre** est vérifié : pour chaque adhérent, les séquences appliquées sont 1, 2, 3… dans l'ordre
  d'application.
- **Le prix de l'ordre** : avec un ordre de livraison **complètement** mélangé, 200 événements ont demandé
  2 812 relivraisons (14 par événement en moyenne). Un vrai courtier livre presque dans l'ordre par
  partition, et les relivraisons sont rares ; mais la mesure montre pourquoi on ne demande l'ordre que là
  où il est nécessaire, et seulement **par entité**.
- **Le message empoisonné** est mis au rebut après 3 tentatives, avec son erreur, et ne bloque rien.
  Piège trouvé par le test : la première version comptait parfois **deux** messages au rebut, quand le
  message empoisonné avait été dupliqué en transit — la seconde copie refaisait ses trois tentatives puis
  repartait au rebut. Un message déjà mis au rebut doit être reconnu comme un doublon (ici une `map` ; en
  vrai, la table `dead_letters`).

## Défi — Un relais qui tourne en permanence

```go
// Fichier : outbox/run.go
// Défi : un relais permanent. Réveil par NOTIFY, sinon interrogation périodique ;
// attente croissante sur erreur ; arrêt propre à l'annulation.
package outbox

import (
	"context"
	"database/sql"
	"time"
)

// Waker réveille le relais quand de nouveaux événements arrivent (ou rend après un délai).
type Waker interface {
	Wait(ctx context.Context, max time.Duration)
}

// PollWaker : pas de notification, on attend simplement max.
type PollWaker struct{}

func (PollWaker) Wait(ctx context.Context, max time.Duration) {
	select {
	case <-ctx.Done():
	case <-time.After(max):
	}
}

func Run(ctx context.Context, db *sql.DB, pub Publisher, w Waker, idle time.Duration) error {
	backoff := 10 * time.Millisecond
	for ctx.Err() == nil {
		// Contexte détaché pour le lot : un lot commencé se termine même si ctx est annulé.
		n, err := RelayOnce(context.WithoutCancel(ctx), db, pub, 50)
		switch {
		case err != nil:
			w.Wait(ctx, backoff)
			backoff = min(2*backoff, 2*time.Second)
		case n == 0:
			backoff = 10 * time.Millisecond
			w.Wait(ctx, idle)
		default:
			backoff = 10 * time.Millisecond // il reste peut-être du travail : on enchaîne
		}
	}
	return nil
}

// Purge supprime par lots les événements publiés depuis plus de age.
func Purge(ctx context.Context, db *sql.DB, age time.Duration) (int64, error) {
	var total int64
	for {
		res, err := db.ExecContext(ctx, `
			DELETE FROM outbox WHERE id IN (
				SELECT id FROM outbox
				WHERE published_at < now() - make_interval(secs => $1)
				LIMIT 1000)`, age.Seconds())
		if err != nil {
			return total, err
		}
		n, _ := res.RowsAffected()
		total += n
		if n < 1000 {
			return total, nil
		}
	}
}
```

```go
// Fichier : outbox/notify.go
package outbox

import (
	"context"
	"time"

	"github.com/jackc/pgx/v5"
)

// NotifyWaker écoute un canal PostgreSQL (LISTEN) sur une connexion pgx NATIVE dédiée.
type NotifyWaker struct{ Conn *pgx.Conn }

func (n NotifyWaker) Wait(ctx context.Context, max time.Duration) {
	wctx, cancel := context.WithTimeout(ctx, max)
	defer cancel()
	_, _ = n.Conn.WaitForNotification(wctx) // une notification, ou le délai
}
```

```go
// Fichier : outbox/run_test.go
package outbox

import (
	"context"
	"fmt"
	"os"
	"sync"
	"testing"
	"time"

	"github.com/jackc/pgx/v5"
)

// timedBus note l'heure de publication de chaque message.
type timedBus struct {
	mu sync.Mutex
	at map[int64]time.Time
}

func (b *timedBus) Publish(_ context.Context, m Message) error {
	b.mu.Lock()
	defer b.mu.Unlock()
	if _, ok := b.at[m.ID]; !ok {
		b.at[m.ID] = time.Now()
	}
	return nil
}

func latency(t *testing.T, notify bool) time.Duration {
	db := newDB(t)
	ctx, cancel := context.WithCancel(t.Context())
	var w Waker = PollWaker{}
	if notify {
		var sp string
		db.QueryRowContext(ctx, `SHOW search_path`).Scan(&sp)
		conn, err := pgx.Connect(ctx, os.Getenv("DATABASE_URL"))
		if err != nil {
			t.Fatal(err)
		}
		defer conn.Close(context.Background())
		if _, err := conn.Exec(ctx, "LISTEN outbox_"+sp); err != nil {
			t.Fatal(err)
		}
		w = NotifyWaker{conn}
		// le déclencheur NOTIFY, dans la transaction de l'INSERT (envoyé au COMMIT)
		db.ExecContext(ctx, fmt.Sprintf(`
			CREATE FUNCTION notify_outbox() RETURNS trigger AS $$
			BEGIN PERFORM pg_notify('outbox_%s', ''); RETURN NULL; END $$ LANGUAGE plpgsql;
			CREATE TRIGGER outbox_notify AFTER INSERT ON outbox FOR EACH STATEMENT EXECUTE FUNCTION notify_outbox();`, sp))
	}
	bus := &timedBus{at: map[int64]time.Time{}}
	done := make(chan error)
	go func() { done <- Run(ctx, db, bus, w, 200*time.Millisecond) }()
	time.Sleep(300 * time.Millisecond) // le relais attend, outbox vide

	var total time.Duration
	for range 10 {
		start := time.Now()
		if err := Borrow(ctx, db, "ada", "x"); err != nil {
			t.Fatal(err)
		}
		var id int64
		db.QueryRowContext(ctx, `SELECT max(id) FROM outbox`).Scan(&id)
		for {
			bus.mu.Lock()
			at, ok := bus.at[id]
			bus.mu.Unlock()
			if ok {
				total += at.Sub(start)
				break
			}
			time.Sleep(time.Millisecond)
		}
		time.Sleep(time.Duration(50+id%7*20) * time.Millisecond) // décaler par rapport au cycle d'attente
	}
	cancel()
	if err := <-done; err != nil {
		t.Fatal(err)
	}
	var locked int
	db.QueryRowContext(t.Context(), `SELECT count(*) FROM outbox WHERE published_at IS NULL`).Scan(&locked)
	if locked != 0 {
		t.Fatalf("%d événements non publiés après l'arrêt", locked)
	}
	return total / 10
}

func TestRunLatency(t *testing.T) {
	poll, notif := latency(t, false), latency(t, true)
	t.Logf("latence moyenne Borrow → publication : interrogation 200 ms = %v ; NOTIFY = %v",
		poll.Round(time.Millisecond), notif.Round(100*time.Microsecond))
	if notif*5 > poll {
		t.Errorf("NOTIFY devrait réduire nettement la latence")
	}
}

func TestPurge(t *testing.T) {
	db := newDB(t)
	for i := range 2500 {
		Borrow(t.Context(), db, "m", fmt.Sprint(i))
	}
	db.Exec(`UPDATE outbox SET published_at = now() - interval '8 days' WHERE id <= 2400`)
	n, err := Purge(t.Context(), db, 7*24*time.Hour)
	var left int
	db.QueryRow(`SELECT count(*) FROM outbox`).Scan(&left)
	if err != nil || n != 2400 || left != 100 {
		t.Fatalf("purgés=%d restants=%d err=%v", n, left, err)
	}
}
```

Mesuré :

```
latence moyenne Borrow → publication : interrogation 200 ms = 102ms ; NOTIFY = 8.5ms
```

- **L'interrogation périodique** donne une latence moyenne d'environ la moitié de la période (100 ms
  pour 200 ms) ; la raccourcir charge la base de requêtes vides.
- **`LISTEN`/`NOTIFY`** réveille le relais dès le `COMMIT` (la notification émise par le déclencheur n'est
  envoyée qu'à la validation : jamais pour une transaction annulée). Il faut une connexion pgx **native**
  dédiée, car `WaitForNotification` n'existe pas dans `database/sql`. La notification n'est qu'un
  **réveil** : on garde l'interrogation périodique, au cas où une notification serait perdue (connexion
  coupée).
- **L'arrêt propre** : chaque lot s'exécute avec `context.WithoutCancel(ctx)`, donc un lot commencé se
  termine ; l'annulation n'interrompt que l'attente. Le test vérifie qu'aucun événement ne reste non
  publié après l'arrêt.
- **La purge** supprime par lots de 1 000 : chaque `DELETE` est court et ne verrouille qu'une petite partie
  de la table (2 400 événements purgés, 100 conservés dans le test).
- **L'attente croissante** sur erreur (10 ms, 20, 40… jusqu'à 2 s) évite de marteler un courtier en panne ;
  la leçon 9 y ajoute l'aléa.

## Quiz

1. Une file : un consommateur parmi plusieurs traite chaque message ; pub/sub : tous les abonnés reçoivent
   chaque événement.
2. Un événement dit ce qui s'est passé ; une commande, ce qu'il faut faire.
3. Au plus une fois : pas de doublon, pertes possibles ; au moins une fois : pas de perte, doublons possibles.
4. Parce qu'on ne peut pas distinguer, à travers le réseau, une perte d'un accusé perdu ; on obtient un
   effet exactement une fois avec au moins une fois + idempotence.
5. Écrire dans la base et publier au courtier sans transaction commune : une panne entre les deux perd (ou
   invente) un événement.
6. L'événement est écrit dans la base, dans la même transaction ; un relais le publie ensuite, au moins une
   fois.
7. À faire travailler plusieurs relais en parallèle sur des lignes différentes, sans attente.
8. Enregistrer « déjà traité » dans la même transaction que l'effet (ou rendre l'effet naturellement
   idempotent).
9. Si chaque lot contient un échec, tout est annulé à chaque tour : il ne conserve jamais de progression.
10. Le mettre au rebut (*dead letter*) après quelques tentatives, avec son erreur, pour examen.
