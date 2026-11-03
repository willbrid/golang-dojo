# Leçon 7 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

Le code part du module de l'exemple du cours (non répété ici) ; seuls les fichiers ajoutés figurent
ci-dessous.

## E1 — L'interface du mauvais côté

`book` doit importer `postgres` pour connaître `postgres.Repository`. Si `postgres` importe `book`
(pour `book.Book` et `book.ErrNotFound`), c'est un **cycle d'imports** : refusé par le compilateur
(`import cycle not allowed`). Pour en sortir, il faudrait déplacer les types du domaine dans `postgres`
ou dans un troisième paquet « fourre-tout », et le domaine dépendrait alors de la couche technique. Et
l'implémentation mémoire devrait importer `postgres` pour satisfaire son interface : le test du service
dépendrait (à la compilation) du pilote PostgreSQL. Tout cela disparaît quand le consommateur déclare
l'interface.

## E2 — Le contrat attrape un menteur

| Bug introduit | Sous-test qui échoue |
|---|---|
| (a) `errors.New("not found")` | `introuvable` (`errors.Is(err, book.ErrNotFound)` faux) |
| (b) pas de tri | **aucun** (vérifié) |
| (c) doublons acceptés | `ISBN en double` |

(b) passe parce que le contrat crée les livres **dans l'ordre de leurs dates** : une implémentation qui
rend l'ordre d'insertion donne, par coïncidence, le bon résultat. Un contrat ne détecte que ce que ses
cas rendent **observable**. Pour attraper (b), il faut un cas où l'ordre d'insertion et l'ordre attendu
diffèrent, par exemple une méthode de test qui fixe `CreatedAt`, ou un tri demandé sur un autre critère
(le titre). Leçon : en écrivant un contrat, se demander pour chaque cas « quelle implémentation fausse
passerait quand même ? ».

Le plus dangereux reste (a) : avec la version mémoire, les tests du **service** et des **handlers**
verraient un « introuvable » qui ne correspond pas à `ErrNotFound`, donc un 500 au lieu d'un 404, et on
corrigerait le handler pour s'adapter au faux au lieu de voir que le faux ment. Sans suite de contrat,
rien ne relie le comportement du faux à celui de la vraie implémentation.

## E3 — Une transaction par test

Le contrat du cours passe. Mais le sous-test ajouté échoue :

```
ERROR: current transaction is aborted, commands ignored until end of transaction block (SQLSTATE 25P02)
```

En PostgreSQL, une erreur dans une transaction (ici la violation d'unicité) **empoisonne** la transaction
entière : toute instruction suivante est refusée jusqu'au `Rollback`. Avec un schéma par test, chaque
instruction est sa propre transaction, et le code de production se comporte de même. La stratégie « une
transaction par test » change donc le comportement du code testé ; elle ne convient pas aux tests qui
provoquent des erreurs, ni à ceux qui ouvrent eux-mêmes des transactions ou testent la concurrence.
(Contournement possible : des `SAVEPOINT` autour de chaque instruction — encore une différence avec la
production.)

## E4 — Le nettoyage qui n'a pas lieu

Après trois exécutions : une trentaine de schémas `test_%` restants (dix par exécution dans la mesure du
cours). Avec la vérification d'erreur, chaque test affiche :

```
nettoyage du schéma test_… : context canceled
```

`t.Context()` est annulé juste avant les fonctions de `Cleanup` : la requête `DROP SCHEMA` ne part même
pas.

## E5 — Faire évoluer le contrat

Fichiers touchés : `book/book.go` (interface), `book/booktest/contract.go` (deux sous-tests : supprimer
puis `Get` → `ErrNotFound` ; supprimer un identifiant absent → `ErrNotFound`), `memory/memory.go`,
`postgres/postgres.go`. Quatre fichiers, et **aucun** fichier de test d'implémentation : les deux
`TestContract` existants exécutent les nouveaux cas automatiquement. Côté PostgreSQL, `ErrNotFound` vient
de `RowsAffected() == 0` (leçon 1). Ajouter une méthode à l'interface **casse la compilation** de toute
implémentation qui ne la fournit pas encore : c'est voulu, et c'est une raison de garder les interfaces
petites.

## Exercice intermédiaire — Une opération atomique sur deux *repositories*

```go
// Fichier : book/review.go
package book

import (
	"context"
	"fmt"
)

type Review struct {
	BookID  int64
	Rating  int
	Comment string
}

type ReviewRepository interface {
	Add(ctx context.Context, r Review) error
	ListByBook(ctx context.Context, bookID int64) ([]Review, error)
}

// Store donne accès aux deux repositories ; WithinTx exécute fn de façon atomique.
type Store interface {
	Books() Repository
	Reviews() ReviewRepository
	WithinTx(ctx context.Context, fn func(books Repository, reviews ReviewRepository) error) error
}

func (s *Service) AddWithFirstReview(ctx context.Context, st Store, b Book, rating int, comment string) (Book, error) {
	err := st.WithinTx(ctx, func(books Repository, reviews ReviewRepository) error {
		if err := books.Create(ctx, &b); err != nil {
			return err
		}
		if rating < 1 || rating > 5 {
			return fmt.Errorf("%w : note %d hors de 1 à 5", ErrInvalid, rating)
		}
		return reviews.Add(ctx, Review{BookID: b.ID, Rating: rating, Comment: comment})
	})
	if err != nil {
		return Book{}, err
	}
	return b, nil
}
```

```go
// Fichier : book/booktest/store.go
package booktest

import (
	"errors"
	"testing"

	"exemple/book"
)

// RunStore : le contrat de l'atomicité, pour toute implémentation de book.Store.
func RunStore(t *testing.T, newStore func(t *testing.T) book.Store) {
	svc := book.NewService(nil)
	t.Run("tout ou rien", func(t *testing.T) {
		st := newStore(t)
		_, err := svc.AddWithFirstReview(t.Context(), st,
			book.Book{Title: "T", Author: "A", ISBN: "9780306406157"}, 9, "note invalide")
		if !errors.Is(err, book.ErrInvalid) {
			t.Fatalf("err = %v", err)
		}
		if got, _ := st.Books().ListByAuthor(t.Context(), "A"); len(got) != 0 {
			t.Fatalf("le livre a survécu à l'annulation : %+v", got)
		}
	})
	t.Run("réussite", func(t *testing.T) {
		st := newStore(t)
		b, err := svc.AddWithFirstReview(t.Context(), st,
			book.Book{Title: "T", Author: "A", ISBN: "9780306406157"}, 5, "bien")
		if err != nil {
			t.Fatal(err)
		}
		rs, err := st.Reviews().ListByBook(t.Context(), b.ID)
		if err != nil || len(rs) != 1 || rs[0].Rating != 5 {
			t.Fatalf("avis = %+v, %v", rs, err)
		}
	})
}
```

```go
// Fichier : memory/store.go
package memory

import (
	"context"
	"slices"
	"sync"
	"time"

	"exemple/book"
)

type reviews struct {
	mu   *sync.Mutex
	list []book.Review
}

func (r *reviews) Add(_ context.Context, rv book.Review) error {
	r.list = append(r.list, rv)
	return nil
}

func (r *reviews) ListByBook(_ context.Context, id int64) ([]book.Review, error) {
	var out []book.Review
	for _, rv := range r.list {
		if rv.BookID == id {
			out = append(out, rv)
		}
	}
	return out, nil
}

// Store en mémoire. WithinTx travaille sur des COPIES, et ne les publie qu'en cas de succès :
// c'est l'annulation la plus simple (copie, puis remplacement).
type Store struct {
	txMu    sync.Mutex // une « transaction » à la fois
	books   *Repo
	reviews *reviews
}

func NewStore(now func() time.Time) *Store {
	return &Store{books: New(now), reviews: &reviews{mu: new(sync.Mutex)}}
}

func (s *Store) Books() book.Repository         { return s.books }
func (s *Store) Reviews() book.ReviewRepository { return s.reviews }

func (s *Store) WithinTx(ctx context.Context, fn func(book.Repository, book.ReviewRepository) error) error {
	s.txMu.Lock()
	defer s.txMu.Unlock()
	s.books.mu.Lock()
	draftBooks := &Repo{books: slices.Clone(s.books.books), now: s.books.now}
	s.books.mu.Unlock()
	draftReviews := &reviews{mu: new(sync.Mutex), list: slices.Clone(s.reviews.list)}
	if err := fn(draftBooks, draftReviews); err != nil {
		return err // les brouillons sont abandonnés : rien n'a changé
	}
	s.books.mu.Lock()
	s.books.books = draftBooks.books
	s.books.mu.Unlock()
	s.reviews.list = draftReviews.list
	return nil
}
```

```go
// Fichier : memory/store_test.go
package memory

import (
	"testing"
	"time"

	"exemple/book"
	"exemple/book/booktest"
)

func TestStoreContract(t *testing.T) {
	booktest.RunStore(t, func(t *testing.T) book.Store {
		return NewStore(time.Now)
	})
}
```

```go
// Fichier : postgres/store.go
package postgres

import (
	"context"
	"database/sql"

	"exemple/book"
)

const ReviewSchema = `
	CREATE TABLE IF NOT EXISTS reviews (
		book_id bigint NOT NULL REFERENCES books(id) ON DELETE CASCADE,
		rating  int NOT NULL CHECK (rating BETWEEN 1 AND 5),
		comment text NOT NULL DEFAULT ''
	);`

type Reviews struct{ db DBTX }

func (r *Reviews) Add(ctx context.Context, rv book.Review) error {
	_, err := r.db.ExecContext(ctx,
		`INSERT INTO reviews (book_id, rating, comment) VALUES ($1, $2, $3)`, rv.BookID, rv.Rating, rv.Comment)
	return err
}

func (r *Reviews) ListByBook(ctx context.Context, id int64) ([]book.Review, error) {
	rows, err := r.db.QueryContext(ctx, `SELECT book_id, rating, comment FROM reviews WHERE book_id = $1`, id)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	var out []book.Review
	for rows.Next() {
		var rv book.Review
		if err := rows.Scan(&rv.BookID, &rv.Rating, &rv.Comment); err != nil {
			return nil, err
		}
		out = append(out, rv)
	}
	return out, rows.Err()
}

type Store struct{ db *sql.DB }

func NewStore(db *sql.DB) *Store { return &Store{db: db} }

func (s *Store) Books() book.Repository         { return New(s.db) }
func (s *Store) Reviews() book.ReviewRepository { return &Reviews{db: s.db} }

// WithinTx : deux repositories construits sur le MÊME *sql.Tx.
func (s *Store) WithinTx(ctx context.Context, fn func(book.Repository, book.ReviewRepository) error) error {
	tx, err := s.db.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	defer tx.Rollback()
	if err := fn(New(tx), &Reviews{db: tx}); err != nil {
		return err
	}
	return tx.Commit()
}
```

```go
// Fichier : postgres/store_test.go
package postgres

import (
	"errors"
	"testing"

	"github.com/jackc/pgx/v5/pgconn"

	"exemple/book"
	"exemple/book/booktest"
)

func TestStoreContract(t *testing.T) {
	t.Parallel()
	booktest.RunStore(t, func(t *testing.T) book.Store {
		db := newTestDB(t)
		if _, err := db.ExecContext(t.Context(), ReviewSchema); err != nil {
			t.Fatal(err)
		}
		return NewStore(db)
	})
}

// E3 : la stratégie « une transaction par test » et l'erreur qui empoisonne la transaction.
func TestTxPerTestAbortedTransaction(t *testing.T) {
	db := newTestDB(t)
	tx, err := db.BeginTx(t.Context(), nil)
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { tx.Rollback() })
	r := New(tx)
	a := book.Book{Title: "A", Author: "X", ISBN: "9780306406157"}
	b := a
	r.Create(t.Context(), &a)
	if err := r.Create(t.Context(), &b); !errors.Is(err, book.ErrDuplicate) {
		t.Fatalf("doublon : %v", err)
	}
	c := book.Book{Title: "C", Author: "X", ISBN: "9782070360024"}
	err = r.Create(t.Context(), &c)
	var pgErr *pgconn.PgError
	if !errors.As(err, &pgErr) || pgErr.Code != "25P02" {
		t.Fatalf("attendu 25P02 : %v", err)
	}
	t.Log(err)
}
```

`go test -race ./...` passe : les deux implémentations satisfont `RunStore`, et `TestTxPerTestAbortedTransaction`
montre l'erreur `25P02` d'E3.

Point 5 — **l'annulation en mémoire.** Il n'y a pas de journal à rejouer à l'envers. La solution simple :
`fn` travaille sur des **copies** (brouillons), et on ne les publie qu'en cas de succès ; un échec
abandonne simplement les copies. Coût : une copie de toutes les données à chaque transaction —
acceptable pour un faux de test, pas pour un vrai stockage. Le verrou `txMu` sérialise les
« transactions » : deux `WithinTx` simultanés ne publient pas des brouillons contradictoires.

Remarque de conception : la vérification de la note est faite **après** l'insertion du livre, exprès,
pour que le contrat teste vraiment l'annulation. Dans un vrai service, on validerait tout **avant**
d'ouvrir la transaction (plus court, moins de verrous) ; la base garde la contrainte `CHECK` en dernier
recours.

## Défi — Une base par test, depuis un modèle

```go
// Fichier : tmpldb/tmpldb_test.go
// Défi : une base par test, copiée d'un modèle préparé une fois dans TestMain.
package tmpldb

import (
	"context"
	"crypto/rand"
	"crypto/sha256"
	"database/sql"
	"encoding/hex"
	"fmt"
	"net/url"
	"os"
	"strings"
	"sync"
	"testing"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
)

// Les « migrations » : nTables tables, pour que leur application coûte quelque chose.
const nTables = 20

var migrations = func() []string {
	var ms []string
	for i := range nTables {
		ms = append(ms, fmt.Sprintf(`CREATE TABLE t%d (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
			name text NOT NULL, created_at timestamptz NOT NULL DEFAULT now());
			CREATE INDEX t%d_name ON t%d (name);`, i, i, i))
	}
	return ms
}()

var (
	adminDSN string
	template string
)

// withDB rend le DSN pointant vers une autre base.
func withDB(dsn, dbname string) string {
	u, _ := url.Parse(dsn)
	u.Path = "/" + dbname
	return u.String()
}

func TestMain(m *testing.M) {
	adminDSN = os.Getenv("DATABASE_URL")
	if adminDSN == "" {
		os.Exit(m.Run()) // les tests se sauteront eux-mêmes
	}
	sum := sha256.Sum256([]byte(strings.Join(migrations, "\n")))
	template = "tpl_" + hex.EncodeToString(sum[:6]) // le nom change si les migrations changent
	if err := ensureTemplate(); err != nil {
		fmt.Fprintln(os.Stderr, "modèle :", err)
		os.Exit(1)
	}
	os.Exit(m.Run())
}

func ensureTemplate() error {
	ctx := context.Background()
	admin, err := sql.Open("pgx", adminDSN)
	if err != nil {
		return err
	}
	defer admin.Close()
	var exists bool
	admin.QueryRowContext(ctx, `SELECT EXISTS (SELECT 1 FROM pg_database WHERE datname = $1)`, template).Scan(&exists)
	if exists {
		return nil
	}
	if _, err := admin.ExecContext(ctx, `CREATE DATABASE `+template); err != nil {
		return err
	}
	tpl, err := sql.Open("pgx", withDB(adminDSN, template))
	if err != nil {
		return err
	}
	defer tpl.Close() // AUCUNE connexion ne doit rester ouverte sur le modèle pendant les copies
	for _, m := range migrations {
		if _, err := tpl.ExecContext(ctx, m); err != nil {
			return err
		}
	}
	return nil
}

func randomName(prefix string) string { return prefix + strings.ToLower(rand.Text()[:12]) }

func newDBFromTemplate(t *testing.T) *sql.DB {
	t.Helper()
	if adminDSN == "" {
		t.Skip("DATABASE_URL non définie")
	}
	admin, err := sql.Open("pgx", adminDSN)
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { admin.Close() })
	name := randomName("test_")
	if _, err := admin.ExecContext(t.Context(), `CREATE DATABASE `+name+` TEMPLATE `+template); err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() {
		if _, err := admin.ExecContext(context.Background(), `DROP DATABASE `+name+` WITH (FORCE)`); err != nil {
			t.Errorf("suppression de %s : %v", name, err)
		}
	})
	db, err := sql.Open("pgx", withDB(adminDSN, name))
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { db.Close() })
	return db
}

func newDBWithSchema(t *testing.T) *sql.DB {
	t.Helper()
	if adminDSN == "" {
		t.Skip("DATABASE_URL non définie")
	}
	admin, err := sql.Open("pgx", adminDSN)
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { admin.Close() })
	schema := randomName("test_")
	if _, err := admin.ExecContext(t.Context(), `CREATE SCHEMA `+schema); err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { admin.ExecContext(context.Background(), `DROP SCHEMA `+schema+` CASCADE`) })
	db, err := sql.Open("pgx", adminDSN+"&search_path="+schema)
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { db.Close() })
	for _, m := range migrations {
		if _, err := db.ExecContext(t.Context(), m); err != nil {
			t.Fatal(err)
		}
	}
	return db
}

// 50 sous-tests parallèles par stratégie ; chacun insère une ligne et la relit.
func TestStrategies(t *testing.T) {
	var mu sync.Mutex
	durations := map[string]time.Duration{}
	for _, s := range []struct {
		name  string
		newDB func(*testing.T) *sql.DB
	}{{"base-modele", newDBFromTemplate}, {"schema-par-test", newDBWithSchema}} {
		start := time.Now()
		t.Run(s.name, func(t *testing.T) {
			for i := range 50 {
				t.Run(fmt.Sprint(i), func(t *testing.T) {
					t.Parallel()
					db := s.newDB(t)
					var id int64
					if err := db.QueryRowContext(t.Context(),
						`INSERT INTO t0 (name) VALUES ('x') RETURNING id`).Scan(&id); err != nil || id != 1 {
						t.Fatalf("id=%d err=%v (base non isolée ?)", id, err)
					}
				})
			}
		})
		mu.Lock()
		durations[s.name] = time.Since(start)
		mu.Unlock()
	}
	t.Logf("50 tests parallèles, %d tables : base-modele %v, schema-par-test %v", nTables,
		durations["base-modele"].Round(10*time.Millisecond), durations["schema-par-test"].Round(10*time.Millisecond))
}

// Point 4 : une connexion ouverte sur le modèle empêche la copie.
func TestTemplateBusy(t *testing.T) {
	if adminDSN == "" {
		t.Skip("DATABASE_URL non définie")
	}
	tpl, _ := sql.Open("pgx", withDB(adminDSN, template))
	defer tpl.Close()
	tpl.PingContext(t.Context()) // ouvre une connexion et la garde dans le pool
	admin, _ := sql.Open("pgx", adminDSN)
	defer admin.Close()
	name := randomName("test_")
	_, err := admin.ExecContext(t.Context(), `CREATE DATABASE `+name+` TEMPLATE `+template)
	if err == nil {
		admin.ExecContext(context.Background(), `DROP DATABASE `+name)
		t.Fatal("la copie aurait dû échouer")
	}
	t.Log(err)
}
```

Mesuré (50 tests parallèles, chaque table avec un index ; le premier passage inclut la création du
modèle) :

| Tables | Base copiée du modèle | Schéma par test |
|---|---|---|
| 20 | 6,2 s | 1,8 s |
| 60 | 8,0 s | 4,6 s |
| 120 | 17,7 s | 8,8 s |

Dans ces mesures, le **schéma par test gagne toujours**. Hypothèse (cohérente avec les chiffres, mais
non prouvée ici) : une copie de base duplique **tous** ses fichiers, catalogue système compris (plusieurs
mégaoctets même pour une base vide), et les écrit dans le journal de transactions ; sa suppression coûte
aussi. Le schéma ne crée que les objets demandés. La base modèle reprendrait l'avantage quand les
migrations font un travail **coûteux qui ne dépend pas de la taille des fichiers** : chargement de
données de référence volumineuses calculées, extensions longues à installer, fonctions compilées. La
leçon générale : mesurer avant de choisir, même une stratégie de test.

Point 4 : une connexion ouverte sur le modèle fait échouer la copie, après **5 secondes** d'attente
(mesuré) :

```
ERROR: source database "tpl_…" is being accessed by other users (SQLSTATE 55006)
```

Protection : fermer toute connexion au modèle dès la fin de sa construction (`defer tpl.Close()` dans
`ensureTemplate`), ne jamais se connecter au modèle depuis les tests, et le marquer
`ALTER DATABASE … WITH ALLOW_CONNECTIONS false` une fois prêt (`IS_TEMPLATE true` en prime).

Point 5 : le nom `tpl_<empreinte des migrations>` fait qu'un changement de migration produit un **nouveau**
modèle ; les anciens se nettoient à part (un `DROP DATABASE` des `tpl_%` qui ne correspondent pas à
l'empreinte courante).

## Quiz

1. Isoler le domaine du stockage : opérations métier, tout le SQL au même endroit, traduction des
   erreurs et des types.
2. C'est le consommateur qui sait ce dont il a besoin ; les dépendances vont vers le domaine, et il n'y a
   pas de cycle.
3. Méthodes génériques (`FindBy(field)`), interfaces énormes, « au cas où on changerait de base »,
   interface sans second usage, *mocks* du SQL.
4. Faire fonctionner le même *repository* avec `*sql.DB` et `*sql.Tx`, donc composer plusieurs
   *repositories* dans une même transaction.
5. Une suite de tests écrite contre l'interface et exécutée par chaque implémentation ; sans elle, le faux
   en mémoire peut diverger de la vraie implémentation et rendre les tests des couches supérieures faux.
6. Ils vérifient le texte des requêtes, pas leur effet ; aucun vrai bug SQL n'est trouvé.
7. Schéma par test (objets globaux partagés), transaction par test (pas de `Commit`, erreurs qui
   empoisonnent), base depuis un modèle (plus lent, modèle sans connexion), conteneur par paquet (Docker
   requis, quelques secondes).
8. Pour que `go test ./...` reste utilisable partout ; la CI, elle, fournit toujours la base.
9. Il est déjà annulé.
10. `now()` est l'heure de début de la transaction ; `clock_timestamp()` est l'heure réelle de
    l'instruction.
