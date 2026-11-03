# Leçon 8 — Événementiel et messaging

> Quand un adhérent emprunte un livre, trois choses doivent suivre : un courriel de confirmation, la mise à
> jour des statistiques, la levée d'une éventuelle réservation. Les faire dans la requête HTTP, c'est
> ralentir l'emprunt et le faire échouer quand le serveur de courriel est en panne. Les confier à des
> **événements** traités ailleurs, plus tard, découple les parties — à condition de régler les deux
> problèmes qui font échouer la plupart des systèmes événementiels : l'événement **perdu** et l'événement
> traité **deux fois**. Cette leçon les règle avec le patron *outbox* et des consommateurs idempotents,
> contre une vraie base.

## Objectifs

1. Distinguer appel synchrone, file de messages et publication/abonnement.
2. Connaître les garanties de livraison (au plus une fois, au moins une fois) et ce que vaut « exactement
   une fois ».
3. Reconnaître le problème de la **double écriture** et le résoudre par l'*outbox* transactionnelle.
4. Écrire un relais qui publie l'*outbox* en parallèle sans doublon de traitement ni blocage.
5. Écrire un consommateur **idempotent**.
6. Savoir ce que deviennent l'ordre, les messages empoisonnés et les versions d'événements.

## Explication

### Synchrone, file, publication

| Style | Principe | Couplage |
|---|---|---|
| **Appel synchrone** | l'émetteur appelle et attend la réponse | fort : l'émetteur dépend de la disponibilité et de la vitesse du destinataire |
| **File** (*queue*) | un message déposé est traité par **un** consommateur parmi plusieurs | l'émetteur ne connaît pas qui traite ; la file absorbe les pointes (backpressure, N6 L11) |
| **Publication / abonnement** (*pub/sub*) | un événement est reçu par **tous** les abonnés | l'émetteur ne connaît même pas l'existence des abonnés |

Un **événement** dit ce qui s'est passé (« emprunt créé »), au passé ; une **commande** dit ce qu'il faut
faire (« envoie ce courriel »). Les événements de domaine de la leçon 6 sont la matière première.

### Les garanties de livraison

Entre un émetteur et un courtier (Kafka, RabbitMQ, NATS…), ou entre le courtier et un consommateur, un
message peut se perdre (panne avant l'envoi) ou être livré deux fois (l'accusé de réception se perd, le
message est renvoyé). On choisit :

- **au plus une fois** : on n'envoie qu'une fois, sans réessai ; les pertes sont possibles ;
- **au moins une fois** : on réessaie jusqu'à confirmation ; les **doublons** sont possibles.

« **Exactement une fois** », au sens de la livraison, n'existe pas à travers un réseau qui peut tomber.
Ce qui existe, c'est un **effet** exactement une fois : livraison au moins une fois **+** consommateur
idempotent, qui reconnaît et ignore les doublons. C'est la combinaison de cette leçon.

### La double écriture

Le code naïf :

```go
tx.ExecContext(ctx, `INSERT INTO loans …`)   // 1. la base
tx.Commit()
bus.Publish(ctx, LoanCreated{…})             // 2. le courtier
```

Deux systèmes, deux écritures, aucune transaction commune. Si le programme s'arrête entre 1 et 2 (un
déploiement, une panne, un `SIGKILL`), l'emprunt existe et l'événement n'a jamais été publié : le courriel
ne part pas, les statistiques sont fausses, **pour toujours**. Inverser l'ordre (publier, puis valider) donne
l'inverse : un événement pour un emprunt qui n'existe pas, si la transaction échoue.

### L'*outbox* transactionnelle

La solution : n'écrire **que dans la base**. L'événement est inséré dans une table `outbox`, **dans la
même transaction** que l'emprunt. Les deux existent ou aucun (`TestNoEventWithoutLoan`). Un **relais**,
ailleurs, lit l'*outbox*, publie, et marque publié.

```
requête ──► transaction { INSERT loans ; INSERT outbox } ──► COMMIT
                                                    │
relais (boucle) ──► SELECT … FOR UPDATE SKIP LOCKED ┘──► Publish ──► UPDATE published_at ──► COMMIT
```

Si le relais s'arrête après avoir publié et avant de marquer, l'événement sera republié au tour suivant :
c'est la livraison **au moins une fois**, et c'est voulu. Ce qui ne peut plus arriver, c'est la **perte**.

`FOR UPDATE SKIP LOCKED` permet de faire tourner **plusieurs** relais : chacun verrouille un lot de lignes,
et les autres sautent les lignes verrouillées au lieu d'attendre. C'est aussi la base des files de travail
construites sur PostgreSQL (projet 7).

### Le consommateur idempotent

Le consommateur enregistre, **dans la même transaction que son effet**, qu'il a traité cet événement :

```go
INSERT INTO processed (consumer, event_id) VALUES ($1, $2) ON CONFLICT DO NOTHING
-- 0 ligne insérée : déjà traité → on s'arrête
-- 1 ligne : on applique l'effet, puis COMMIT des deux ensemble
```

Si la transaction échoue, ni la trace ni l'effet ne sont enregistrés : le message sera retraité. Si elle
réussit, un doublon ultérieur sera reconnu. C'est ce qui transforme « au moins une fois » en « effet
exactement une fois ». Les effets naturellement idempotents (« fixer le statut à rendu ») n'ont pas besoin
de la table ; les effets cumulatifs (« incrémenter un compteur », « envoyer un courriel ») en ont besoin.

### Un piège rencontré en écrivant cette leçon

La première version du relais **annulait tout le lot** dès qu'une publication échouait (« rien n'est
marqué, tout sera republié »). Avec un courtier qui échoue sur un message sur sept et des lots de dix, chaque
lot contenait un échec : le relais republiait les six premiers messages, échouait au septième, annulait…
**indéfiniment**. Mesuré : le test ne se terminait jamais, et chaque tour ajoutait six doublons. C'est un
*livelock* : le système travaille sans progresser. La correction : **valider ce qui a été publié** avant de
rendre l'erreur, pour que chaque tour fasse avancer la file. Règle générale : un traitement par lots qui
réessaie doit conserver sa progression partielle.

### Ordre, messages empoisonnés, versions

- **L'ordre.** Avec plusieurs relais et `SKIP LOCKED`, deux événements peuvent être publiés dans le
  désordre. Quand l'ordre compte **pour une même entité** (les événements d'un même adhérent), on
  partitionne : par clé (un seul relais par partition, ou la clé de partition du courtier), et le
  consommateur rejette ou met de côté un événement plus ancien que ce qu'il a déjà vu (numéro de version
  de l'agrégat, leçon 6). L'ordre **global** est rarement nécessaire, et très coûteux.
- **Les messages empoisonnés.** Un message que le consommateur ne pourra jamais traiter (JSON invalide,
  référence inexistante) ne doit pas bloquer la file : après N échecs, il part dans une file de rebut
  (*dead letter queue*) pour examen humain.
- **Les versions.** Un événement est un **contrat** entre équipes et entre versions du code : on ajoute des
  champs, on n'en supprime ni n'en renomme ; un changement incompatible est un nouveau type d'événement
  (`loan.created.v2`).

### Quand ne pas le faire

- **Une seule action, dans le même module, qui doit réussir avec la requête** : un appel de fonction dans
  la transaction suffit. Les événements ajoutent de la cohérence **à terme**, des tables, un relais, de la
  surveillance.
- **Un courtier de messages** pour un système qui tient dans un processus : une table *outbox* et des
  consommateurs dans le même programme suffisent souvent (le relais et le consommateur de l'exemple
  pourraient être une goroutine du service).
- **Des événements partout**, y compris pour des lectures (« demande de titre ») : un appel synchrone est
  plus simple quand on a besoin de la réponse tout de suite.

## Exemple

Un emprunt écrit avec son événement ; trois relais concurrents vers un courtier qui **échoue** une fois
sur sept et **duplique** un message sur trois ; un consommateur idempotent. Contre PostgreSQL, un schéma
par test (N8 L7).

```go
// Fichier : outbox/schema.go
package outbox

const Schema = `
	CREATE TABLE loans (
		id     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
		member text NOT NULL,
		book   text NOT NULL);
	CREATE TABLE outbox (
		id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
		topic        text NOT NULL,
		payload      jsonb NOT NULL,
		published_at timestamptz);
	CREATE INDEX outbox_pending ON outbox (id) WHERE published_at IS NULL;
	-- côté consommateur : les événements déjà traités, et l'effet du traitement
	CREATE TABLE processed (consumer text, event_id bigint, PRIMARY KEY (consumer, event_id));
	CREATE TABLE loan_stats (member text PRIMARY KEY, loans int NOT NULL);`
```

```go
// Fichier : outbox/borrow.go
package outbox

import (
	"context"
	"database/sql"
	"encoding/json"
)

type LoanCreated struct {
	Member string `json:"member"`
	Book   string `json:"book"`
}

// Borrow écrit l'emprunt ET son événement dans la MÊME transaction :
// les deux existent, ou aucun. Aucun appel réseau ici.
func Borrow(ctx context.Context, db *sql.DB, member, book string) error {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	defer tx.Rollback()
	if _, err := tx.ExecContext(ctx, `INSERT INTO loans (member, book) VALUES ($1, $2)`, member, book); err != nil {
		return err
	}
	payload, _ := json.Marshal(LoanCreated{member, book})
	if _, err := tx.ExecContext(ctx,
		`INSERT INTO outbox (topic, payload) VALUES ('loan.created', $1)`, payload); err != nil {
		return err
	}
	return tx.Commit()
}
```

```go
// Fichier : outbox/relay.go
package outbox

import (
	"context"
	"database/sql"
)

type Message struct {
	ID      int64
	Topic   string
	Payload []byte
}

// Publisher : le courtier de messages (Kafka, RabbitMQ, NATS…), vu par le relais.
type Publisher interface {
	Publish(ctx context.Context, m Message) error
}

// RelayOnce publie au plus batch événements en attente, et marque publiés ceux qui l'ont été.
// FOR UPDATE SKIP LOCKED : plusieurs relais en parallèle se partagent les lignes sans
// s'attendre. Si le relais s'arrête APRÈS Publish et AVANT le Commit, l'événement sera
// republié : livraison AU MOINS UNE FOIS.
func RelayOnce(ctx context.Context, db *sql.DB, pub Publisher, batch int) (int, error) {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return 0, err
	}
	defer tx.Rollback()
	rows, err := tx.QueryContext(ctx, `
		SELECT id, topic, payload FROM outbox
		WHERE published_at IS NULL ORDER BY id LIMIT $1
		FOR UPDATE SKIP LOCKED`, batch)
	if err != nil {
		return 0, err
	}
	var msgs []Message
	for rows.Next() {
		var m Message
		if err := rows.Scan(&m.ID, &m.Topic, &m.Payload); err != nil {
			rows.Close()
			return 0, err
		}
		msgs = append(msgs, m)
	}
	rows.Close()
	if err := rows.Err(); err != nil {
		return 0, err
	}
	done := 0
	var pubErr error
	for _, m := range msgs {
		if pubErr = pub.Publish(ctx, m); pubErr != nil {
			break // on s'arrête, mais on GARDE ce qui a été publié (voir ci-dessous)
		}
		if _, err := tx.ExecContext(ctx, `UPDATE outbox SET published_at = now() WHERE id = $1`, m.ID); err != nil {
			return 0, err
		}
		done++
	}
	// Valider même en cas d'échec de publication : sinon un lot qui contient toujours un
	// message en échec serait republié à l'infini, sans jamais progresser.
	if err := tx.Commit(); err != nil {
		return 0, err
	}
	return done, pubErr
}
```

```go
// Fichier : outbox/consumer.go
package outbox

import (
	"context"
	"database/sql"
	"encoding/json"
)

// Handle traite un message de façon IDEMPOTENTE : la trace « déjà traité » et l'effet
// sont écrits dans la même transaction. Un doublon est reconnu et ignoré.
func Handle(ctx context.Context, db *sql.DB, consumer string, m Message) (bool, error) {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return false, err
	}
	defer tx.Rollback()
	res, err := tx.ExecContext(ctx,
		`INSERT INTO processed (consumer, event_id) VALUES ($1, $2) ON CONFLICT DO NOTHING`, consumer, m.ID)
	if err != nil {
		return false, err
	}
	if n, _ := res.RowsAffected(); n == 0 {
		return false, nil // doublon : déjà traité
	}
	var e LoanCreated
	if err := json.Unmarshal(m.Payload, &e); err != nil {
		return false, err
	}
	if _, err := tx.ExecContext(ctx, `
		INSERT INTO loan_stats (member, loans) VALUES ($1, 1)
		ON CONFLICT (member) DO UPDATE SET loans = loan_stats.loans + 1`, e.Member); err != nil {
		return false, err
	}
	return true, tx.Commit()
}
```

```go
// Fichier : outbox/outbox_test.go
package outbox

import (
	"context"
	"crypto/rand"
	"database/sql"
	"errors"
	"fmt"
	"os"
	"strings"
	"sync"
	"sync/atomic"
	"testing"

	_ "github.com/jackc/pgx/v5/stdlib"
)

func newDB(t *testing.T) *sql.DB {
	t.Helper()
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		t.Skip("DATABASE_URL non définie")
	}
	admin, err := sql.Open("pgx", dsn)
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { admin.Close() })
	schema := "test_" + strings.ToLower(rand.Text()[:12])
	if _, err := admin.ExecContext(t.Context(), "CREATE SCHEMA "+schema); err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { admin.ExecContext(context.Background(), "DROP SCHEMA "+schema+" CASCADE") })
	db, err := sql.Open("pgx", dsn+"&search_path="+schema)
	if err != nil {
		t.Fatal(err)
	}
	db.SetMaxOpenConns(20)
	t.Cleanup(func() { db.Close() })
	if _, err := db.ExecContext(t.Context(), Schema); err != nil {
		t.Fatal(err)
	}
	return db
}

// flakyBus : un courtier en mémoire qui DUPLIQUE un message sur trois (cas réel :
// accusé de réception perdu, réémission) et fait échouer certaines publications.
type flakyBus struct {
	mu        sync.Mutex
	delivered []Message
	n         int
	failEvery int
}

var errBroker = errors.New("courtier indisponible")

func (b *flakyBus) Publish(_ context.Context, m Message) error {
	b.mu.Lock()
	defer b.mu.Unlock()
	b.n++
	if b.failEvery > 0 && b.n%b.failEvery == 0 {
		return errBroker
	}
	b.delivered = append(b.delivered, m)
	if b.n%3 == 0 {
		b.delivered = append(b.delivered, m) // doublon
	}
	return nil
}

func TestOutboxEndToEnd(t *testing.T) {
	db := newDB(t)
	ctx := t.Context()

	// 1. 200 emprunts, répartis entre 4 adhérents, écrits en parallèle.
	var wg sync.WaitGroup
	for i := range 200 {
		wg.Go(func() {
			if err := Borrow(ctx, db, fmt.Sprintf("m%d", i%4), fmt.Sprintf("b%d", i)); err != nil {
				t.Error(err)
			}
		})
	}
	wg.Wait()

	// 2. Trois relais concurrents, avec un courtier qui échoue une fois sur 7.
	bus := &flakyBus{failEvery: 7}
	var failures atomic.Int64
	for range 3 {
		wg.Go(func() {
			for {
				n, err := RelayOnce(ctx, db, bus, 10)
				if errors.Is(err, errBroker) {
					failures.Add(1)
					continue // la progression est conservée ; en vrai : attendre un peu (leçon 9)
				}
				if err != nil {
					t.Error(err)
					return
				}
				if n == 0 {
					return
				}
			}
		})
	}
	wg.Wait()
	var pending int
	db.QueryRowContext(ctx, `SELECT count(*) FROM outbox WHERE published_at IS NULL`).Scan(&pending)

	// 3. Le consommateur reçoit tout, doublons compris.
	applied, dups := 0, 0
	for _, m := range bus.delivered {
		ok, err := Handle(ctx, db, "stats", m)
		if err != nil {
			t.Fatal(err)
		}
		if ok {
			applied++
		} else {
			dups++
		}
	}
	var total int
	db.QueryRowContext(ctx, `SELECT coalesce(sum(loans), 0) FROM loan_stats`).Scan(&total)
	t.Logf("livrés=%d (dont doublons ignorés=%d), appliqués=%d, échecs de publication=%d, en attente=%d, total des stats=%d",
		len(bus.delivered), dups, applied, failures.Load(), pending, total)
	if pending != 0 || applied != 200 || total != 200 {
		t.Fatalf("chaque événement doit être appliqué exactement une fois")
	}
}

func TestNoEventWithoutLoan(t *testing.T) {
	db := newDB(t)
	ctx, cancel := context.WithCancel(t.Context())
	cancel() // la transaction ne peut pas aboutir
	_ = Borrow(ctx, db, "ada", "x")
	var loans, events int
	db.QueryRowContext(t.Context(), `SELECT (SELECT count(*) FROM loans), (SELECT count(*) FROM outbox)`).Scan(&loans, &events)
	if loans != 0 || events != 0 {
		t.Fatalf("emprunt=%d événements=%d : jamais l'un sans l'autre", loans, events)
	}
}
```
`go test -race ./...` (avec `DATABASE_URL`) :

```
livrés=266 (dont doublons ignorés=66), appliqués=200, échecs de publication=33, en attente=0, total des stats=200
--- PASS: TestOutboxEndToEnd
--- PASS: TestNoEventWithoutLoan
```

(Les nombres de doublons et d'échecs varient d'une exécution à l'autre ; « appliqués=200 » et « en
attente=0 » ne varient jamais.)

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `Borrow` : deux `INSERT`, une transaction | Pas de double écriture : l'événement n'existe que si l'emprunt existe. |
| Aucun appel au courtier dans `Borrow` | L'emprunt ne dépend plus de la disponibilité du courtier. |
| Index partiel `WHERE published_at IS NULL` | Le relais ne lit que les événements en attente, même avec des millions d'événements publiés. |
| `FOR UPDATE SKIP LOCKED` | Plusieurs relais sans attente ni double publication **simultanée**. |
| Commit malgré l'échec de publication | La progression partielle est conservée : pas de *livelock*. |
| `Publish` puis `UPDATE` | Une panne entre les deux produit un doublon, jamais une perte. |
| `processed` + `ON CONFLICT DO NOTHING` | Le doublon est reconnu dans la même transaction que l'effet. |
| `flakyBus` | Échecs et doublons injectés : le test vérifie ce qui doit **toujours** tenir. |
| `appliqués=200`, `total=200` | Au moins une fois + idempotence = effet exactement une fois. |

## Erreurs fréquentes

1. **La double écriture** : écrire en base puis publier (ou l'inverse) sans *outbox*.
2. **Publier depuis la transaction** (appel réseau dans une transaction, N8 L4).
3. **Croire à la livraison « exactement une fois »** et écrire un consommateur non idempotent.
4. **Une trace « déjà traité » écrite hors de la transaction de l'effet.**
5. **Un relais qui annule tout le lot** à chaque échec : *livelock*.
6. **Compter sur l'ordre global** des événements.
7. **Un message empoisonné qui bloque la file.**
8. **Modifier le format d'un événement** de façon incompatible.
9. **Une *outbox* qui grossit sans fin** : purger les événements publiés anciens.

## Bonnes pratiques Go

- Événement écrit dans l'*outbox*, dans la transaction de l'agrégat.
- Relais : lots bornés, `FOR UPDATE SKIP LOCKED`, progression partielle conservée, attente en cas d'erreur.
- Consommateurs idempotents : table de traces dans la transaction de l'effet, ou effet naturellement
  idempotent.
- Partitionner par clé quand l'ordre compte pour une entité ; versionner les événements.
- File de rebut pour les messages empoisonnés ; métriques sur l'âge du plus vieil événement en attente
  (leçon 10).
- Tester avec un courtier qui échoue et duplique.

## Ce que je dois retenir

- Événement = ce qui s'est passé ; file = un consommateur ; pub/sub = tous les abonnés.
- Au moins une fois + idempotence = effet exactement une fois ; la livraison exactement une fois n'existe
  pas.
- La double écriture perd des événements ; l'*outbox* transactionnelle l'empêche.
- `SKIP LOCKED` pour paralléliser un relais ; conserver la progression partielle.
- Ordre par entité seulement, messages empoisonnés mis de côté, événements versionnés.

---

➡️ [Exercices](exercices.md)
