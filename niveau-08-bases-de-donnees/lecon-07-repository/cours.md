# Leçon 7 — Repository pattern

> Le SQL doit vivre quelque part. Dispersé dans les handlers, il rend chaque test dépendant d'une base
> et chaque changement de schéma dangereux. Enfermé derrière une abstraction trop générale
> (`Repository[T]` avec `FindAll`, `FindBy`, `Save`, `Delete` pour tout), il perd tout ce que
> PostgreSQL sait faire et ne simplifie rien. Cette leçon trouve le juste milieu : une petite interface
> définie par celui qui l'utilise, une implémentation qui assume son SQL, et des tests contre une
> **vraie** base, isolés et rapides.

## Objectifs

1. Placer l'interface du stockage du côté du **domaine**, avec les seules méthodes dont il a besoin.
2. Écrire une implémentation PostgreSQL qui traduit les erreurs et les types à la frontière.
3. Faire fonctionner le même *repository* dans et hors transaction (`DBTX`).
4. Écrire une **suite de contrat** exécutée contre toutes les implémentations.
5. Tester contre une vraie base, avec une isolation par test, sans laisser de déchets.

## Explication

### Le rôle d'un *repository*

Un *repository* est la frontière entre le domaine et le stockage. Il :

- expose des opérations **métier** (`Create`, `Get`, `ListByAuthor`), pas des tables ni du SQL ;
- contient **tout** le SQL qui concerne ces données, au même endroit ;
- **traduit** à la frontière : `sql.ErrNoRows` → `book.ErrNotFound`, erreur PostgreSQL `23505` →
  `book.ErrDuplicate`, `timestamptz` → `time.Time` en UTC.

Au-dessus de lui, plus personne n'importe `database/sql` ni `pgconn` : le service et les handlers
(N7 L5) travaillent avec les erreurs du domaine.

### L'interface est définie par celui qui l'utilise

C'est la règle des interfaces en Go (N2 L4, N5 L3) appliquée au stockage. Le paquet `book` déclare ce
dont **il** a besoin :

```go
type Repository interface {
	Create(ctx context.Context, b *Book) error
	Get(ctx context.Context, id int64) (Book, error)
	ListByAuthor(ctx context.Context, author string) ([]Book, error)
}
```

Trois méthodes, nommées d'après les cas d'usage. Le paquet `postgres` l'implémente **sans l'importer** :
il importe `book` pour les types et les erreurs, mais c'est le compilateur qui constate que `*postgres.Repo`
satisfait `book.Repository`. Le sens des dépendances va vers le domaine : `postgres → book`, jamais
l'inverse. Le niveau 10 généralise cette idée à toute l'architecture.

### Ne pas sur-abstraire

Signes qu'une abstraction coûte plus qu'elle ne rapporte :

| Symptôme | Problème |
|---|---|
| `Repository[T any]` avec `FindAll`, `FindBy(field, value)`, `Save`, `Delete` | Les requêtes réelles (jointures, pagination par curseur, `FOR UPDATE`, agrégats) ne rentrent pas ; on finit par charger trop de données et filtrer en Go. |
| Une interface de 25 méthodes | Aucun consommateur n'en utilise plus de trois ; chaque faux de test doit tout implémenter. |
| « Au cas où on changerait de base » | Ce changement n'arrive presque jamais, et l'abstraction aurait de toute façon fui (types, verrous, erreurs). |
| Une interface pour un seul usage, sans test qui la remplace | De l'indirection sans bénéfice : appeler la struct concrète directement. |
| Des *mocks* qui simulent le SQL (« quand `Query` reçoit telle chaîne, rendre telles lignes ») | Le test vérifie le texte de la requête, pas son effet ; il casse au moindre reformatage et ne trouve aucun vrai bug. |

Le *repository* **assume** PostgreSQL : il utilise `RETURNING`, `= ANY`, `FOR UPDATE`, `jsonb` quand il
en a besoin. L'abstraction protège le **domaine** du SQL, pas le SQL de PostgreSQL.

### Dans et hors transaction : `DBTX`

Un cas d'usage écrit parfois dans plusieurs *repositories* d'un coup (créer une commande **et** décrémenter
un stock). Il faut alors que tous travaillent dans **la même** transaction (leçon 4). La technique la plus
simple : le *repository* ne dépend pas de `*sql.DB`, mais d'une petite interface que `*sql.DB` **et**
`*sql.Tx` satisfont tous les deux :

```go
type DBTX interface {
	ExecContext(ctx context.Context, query string, args ...any) (sql.Result, error)
	QueryContext(ctx context.Context, query string, args ...any) (*sql.Rows, error)
	QueryRowContext(ctx context.Context, query string, args ...any) *sql.Row
}

repo := postgres.New(db)                     // hors transaction
err := WithTx(ctx, db, nil, func(tx *sql.Tx) error {
	orders, stock := postgres.NewOrders(tx), postgres.NewStock(tx) // même transaction
	…
})
```

Si le **domaine** doit décider de l'atomicité (et pas seulement l'adaptateur), on ajoute à son interface
une méthode du type `WithinTx(ctx, func(Repository) error) error`, que l'implémentation PostgreSQL
réalise avec `WithTx` et l'implémentation mémoire avec un verrou.

### Une suite de contrat

Deux implémentations du même contrat (PostgreSQL pour la production, mémoire pour tester les couches
supérieures vite et sans base) doivent se comporter **de la même façon** : même erreur pour un
introuvable, même tri, même gestion des doublons. Sinon, les tests des couches supérieures, qui utilisent
la version mémoire, valident un comportement que la production n'a pas.

La réponse : une **suite de contrat**, écrite une fois contre l'interface, et exécutée par chaque
implémentation sur elle-même. Elle vit dans un paquet à part (`booktest`), importé uniquement par des
tests :

```go
func Run(t *testing.T, newRepo func(t *testing.T) book.Repository)
```

`newRepo` fournit une instance **vierge** pour chaque sous-test. Ajouter une implémentation (un cache,
une autre base) = une ligne de test qui appelle `booktest.Run`.

### Tester contre une vraie base

Les tests du *repository* lui-même **doivent** parler à PostgreSQL : c'est le SQL qu'on teste. Les
questions sont l'**isolation** (un test ne doit pas voir les données d'un autre, y compris en parallèle)
et le **coût**. Mesuré en local :

| Stratégie | Coût de mise en place | Isolation | Limite |
|---|---|---|---|
| Un **schéma** par test (`CREATE SCHEMA`, `search_path` dans le DSN) | ≈ 2 ms + création des tables | complète, tests parallèles | les objets globaux (rôles, extensions) sont partagés |
| Une **transaction** par test, annulée à la fin | quasi nul | complète | impossible de tester un `Commit`, la concurrence, ou du code qui ouvre ses propres transactions |
| Une **base** copiée d'un modèle (`CREATE DATABASE t TEMPLATE modele`) | ≈ 36 ms | totale | un peu plus lent ; le modèle ne doit avoir aucune connexion ouverte pendant la copie |
| Un **conteneur** démarré par le test (module `testcontainers-go`) | quelques secondes, une fois par paquet | totale, environnement reproductible | Docker requis sur la machine de test et en CI |

Ce cours utilise une base lancée **avant** les tests (le conteneur du README, ou un service de la CI) et
un schéma par test. Le test lit `DATABASE_URL` et se **saute** (`t.Skip`) si elle est absente :
`go test ./...` reste utilisable sans base, et la CI la fournit toujours. Le module `testcontainers-go`
automatise le démarrage du conteneur depuis le test lui-même (son sous-paquet `postgres` lance l'image
et rend le DSN du conteneur) : pratique quand on ne contrôle pas l'environnement de test, au prix d'une
dépendance et de quelques secondes par paquet. Il n'est pas utilisé ici ; la stratégie « base lancée à
l'extérieur + schéma par test » se transpose telle quelle.

### Un piège de nettoyage

`t.Context()` (Go 1.24) est **annulé juste avant** l'exécution des fonctions enregistrées par
`t.Cleanup`. Un nettoyage écrit avec `t.Context()` échoue donc toujours, en silence si l'on ignore
l'erreur. En écrivant l'exemple de cette leçon, c'est exactement ce qui s'est produit : après une
exécution des tests, **dix schémas de test** restaient dans la base. Dans un `Cleanup`, utiliser
`context.Background()` (avec un délai si nécessaire) et **vérifier** l'erreur.

## Exemple

Un module complet : le domaine et son interface, une implémentation mémoire, une implémentation
PostgreSQL, la suite de contrat, et les tests d'intégration avec un schéma par test.

```
exemple/
├── book/book.go                 domaine : Book, erreurs, Repository, Service
├── book/booktest/contract.go    la suite de contrat
├── memory/                      implémentation en mémoire + son test de contrat
└── postgres/                    implémentation PostgreSQL + tests d'intégration
```

```go
// Fichier : book/book.go
// Package book est le domaine : types, règles, et ce dont il a besoin du stockage.
package book

import (
	"context"
	"errors"
	"fmt"
	"strings"
	"time"
)

var (
	ErrNotFound  = errors.New("livre introuvable")
	ErrDuplicate = errors.New("ISBN déjà enregistré")
	ErrInvalid   = errors.New("livre invalide")
)

type Book struct {
	ID        int64
	Title     string
	Author    string
	ISBN      string
	CreatedAt time.Time
}

// Repository est défini ICI, par celui qui l'utilise : trois méthodes, celles dont le
// service a besoin, pas un CRUD générique.
type Repository interface {
	Create(ctx context.Context, b *Book) error // remplit ID et CreatedAt
	Get(ctx context.Context, id int64) (Book, error)
	ListByAuthor(ctx context.Context, author string) ([]Book, error)
}

type Service struct {
	repo Repository
}

func NewService(r Repository) *Service { return &Service{repo: r} }

func (s *Service) Add(ctx context.Context, title, author, isbn string) (Book, error) {
	b := Book{Title: strings.TrimSpace(title), Author: strings.TrimSpace(author), ISBN: isbn}
	if b.Title == "" || b.Author == "" || len(b.ISBN) != 13 {
		return Book{}, fmt.Errorf("%w : titre, auteur et ISBN-13 requis", ErrInvalid)
	}
	if err := s.repo.Create(ctx, &b); err != nil {
		return Book{}, fmt.Errorf("ajout de %q : %w", b.Title, err)
	}
	return b, nil
}
```

```go
// Fichier : book/booktest/contract.go
// Package booktest contient la suite de CONTRAT : ce que toute implémentation de
// book.Repository doit garantir. Chaque implémentation l'exécute contre elle-même.
package booktest

import (
	"errors"
	"testing"

	"exemple/book"
)

func Run(t *testing.T, newRepo func(t *testing.T) book.Repository) {
	t.Run("créer puis lire", func(t *testing.T) {
		r := newRepo(t)
		b := book.Book{Title: "L'Étranger", Author: "Camus", ISBN: "9782070360024"}
		if err := r.Create(t.Context(), &b); err != nil {
			t.Fatal(err)
		}
		if b.ID == 0 || b.CreatedAt.IsZero() {
			t.Fatalf("ID et CreatedAt doivent être remplis : %+v", b)
		}
		got, err := r.Get(t.Context(), b.ID)
		if err != nil || got.Title != b.Title || got.ISBN != b.ISBN || !got.CreatedAt.Equal(b.CreatedAt) {
			t.Fatalf("Get = %+v, %v ; attendu %+v", got, err, b)
		}
	})
	t.Run("introuvable", func(t *testing.T) {
		if _, err := newRepo(t).Get(t.Context(), 424242); !errors.Is(err, book.ErrNotFound) {
			t.Fatalf("err = %v, attendu ErrNotFound", err)
		}
	})
	t.Run("ISBN en double", func(t *testing.T) {
		r := newRepo(t)
		a := book.Book{Title: "A", Author: "X", ISBN: "9780306406157"}
		b := book.Book{Title: "B", Author: "Y", ISBN: "9780306406157"}
		if err := r.Create(t.Context(), &a); err != nil {
			t.Fatal(err)
		}
		if err := r.Create(t.Context(), &b); !errors.Is(err, book.ErrDuplicate) {
			t.Fatalf("err = %v, attendu ErrDuplicate", err)
		}
	})
	t.Run("liste par auteur, triée par date puis id", func(t *testing.T) {
		r := newRepo(t)
		for i, title := range []string{"La Peste", "L'Étranger", "Autre"} {
			author := "Camus"
			if title == "Autre" {
				author = "Sartre"
			}
			b := book.Book{Title: title, Author: author, ISBN: "978000000000" + string(rune('0'+i))}
			if err := r.Create(t.Context(), &b); err != nil {
				t.Fatal(err)
			}
		}
		got, err := r.ListByAuthor(t.Context(), "Camus")
		if err != nil || len(got) != 2 || got[0].Title != "La Peste" || got[1].Title != "L'Étranger" {
			t.Fatalf("ListByAuthor = %+v, %v", got, err)
		}
		if got, _ := r.ListByAuthor(t.Context(), "Personne"); len(got) != 0 {
			t.Fatalf("auteur inconnu : %+v", got)
		}
	})
}
```

```go
// Fichier : memory/memory.go
// Package memory : une implémentation en mémoire, pour les tests des couches supérieures.
package memory

import (
	"context"
	"fmt"
	"slices"
	"sync"
	"time"

	"exemple/book"
)

type Repo struct {
	mu    sync.Mutex
	books []book.Book
	now   func() time.Time
}

func New(now func() time.Time) *Repo { return &Repo{now: now} }

func (r *Repo) Create(_ context.Context, b *book.Book) error {
	r.mu.Lock()
	defer r.mu.Unlock()
	for _, x := range r.books {
		if x.ISBN == b.ISBN {
			return fmt.Errorf("isbn %s : %w", b.ISBN, book.ErrDuplicate)
		}
	}
	b.ID = int64(len(r.books) + 1)
	b.CreatedAt = r.now()
	r.books = append(r.books, *b)
	return nil
}

func (r *Repo) Get(_ context.Context, id int64) (book.Book, error) {
	r.mu.Lock()
	defer r.mu.Unlock()
	for _, b := range r.books {
		if b.ID == id {
			return b, nil
		}
	}
	return book.Book{}, fmt.Errorf("livre %d : %w", id, book.ErrNotFound)
}

func (r *Repo) ListByAuthor(_ context.Context, author string) ([]book.Book, error) {
	r.mu.Lock()
	defer r.mu.Unlock()
	var out []book.Book
	for _, b := range r.books {
		if b.Author == author {
			out = append(out, b)
		}
	}
	slices.SortStableFunc(out, func(a, b book.Book) int { return a.CreatedAt.Compare(b.CreatedAt) })
	return out, nil
}
```

```go
// Fichier : memory/memory_test.go
package memory

import (
	"testing"
	"time"

	"exemple/book"
	"exemple/book/booktest"
)

func TestContract(t *testing.T) {
	booktest.Run(t, func(t *testing.T) book.Repository {
		clock := time.Date(2026, 1, 1, 0, 0, 0, 0, time.UTC)
		return New(func() time.Time { clock = clock.Add(time.Second); return clock })
	})
}
```

```go
// Fichier : postgres/postgres.go
// Package postgres implémente book.Repository avec PostgreSQL.
package postgres

import (
	"context"
	"database/sql"
	"errors"
	"fmt"

	"github.com/jackc/pgx/v5/pgconn"

	"exemple/book"
)

const Schema = `
	CREATE TABLE IF NOT EXISTS books (
		id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
		title      text NOT NULL,
		author     text NOT NULL,
		isbn       text NOT NULL UNIQUE,
		created_at timestamptz NOT NULL DEFAULT clock_timestamp()
	);
	CREATE INDEX IF NOT EXISTS books_author ON books (author, created_at, id);`

// DBTX est satisfaite par *sql.DB et *sql.Tx : le même Repo fonctionne dans et hors transaction.
type DBTX interface {
	ExecContext(ctx context.Context, query string, args ...any) (sql.Result, error)
	QueryContext(ctx context.Context, query string, args ...any) (*sql.Rows, error)
	QueryRowContext(ctx context.Context, query string, args ...any) *sql.Row
}

type Repo struct{ db DBTX }

func New(db DBTX) *Repo { return &Repo{db: db} }

const columns = `id, title, author, isbn, created_at`

type scanner interface{ Scan(dest ...any) error }

func scanBook(s scanner) (book.Book, error) {
	var b book.Book
	err := s.Scan(&b.ID, &b.Title, &b.Author, &b.ISBN, &b.CreatedAt)
	b.CreatedAt = b.CreatedAt.UTC()
	return b, err
}

func (r *Repo) Create(ctx context.Context, b *book.Book) error {
	err := r.db.QueryRowContext(ctx,
		`INSERT INTO books (title, author, isbn) VALUES ($1, $2, $3) RETURNING id, created_at`,
		b.Title, b.Author, b.ISBN).Scan(&b.ID, &b.CreatedAt)
	var pgErr *pgconn.PgError
	if errors.As(err, &pgErr) && pgErr.Code == "23505" {
		return fmt.Errorf("isbn %s : %w", b.ISBN, book.ErrDuplicate)
	}
	b.CreatedAt = b.CreatedAt.UTC()
	return err
}

func (r *Repo) Get(ctx context.Context, id int64) (book.Book, error) {
	b, err := scanBook(r.db.QueryRowContext(ctx, `SELECT `+columns+` FROM books WHERE id = $1`, id))
	if errors.Is(err, sql.ErrNoRows) {
		return book.Book{}, fmt.Errorf("livre %d : %w", id, book.ErrNotFound)
	}
	return b, err
}

func (r *Repo) ListByAuthor(ctx context.Context, author string) ([]book.Book, error) {
	rows, err := r.db.QueryContext(ctx,
		`SELECT `+columns+` FROM books WHERE author = $1 ORDER BY created_at, id`, author)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	var out []book.Book
	for rows.Next() {
		b, err := scanBook(rows)
		if err != nil {
			return nil, err
		}
		out = append(out, b)
	}
	return out, rows.Err()
}
```

```go
// Fichier : postgres/postgres_test.go
package postgres

import (
	"context"
	"crypto/rand"
	"database/sql"
	"fmt"
	"os"
	"strings"
	"testing"

	_ "github.com/jackc/pgx/v5/stdlib"

	"exemple/book"
	"exemple/book/booktest"
)

// newTestDB crée un SCHÉMA propre au test, y applique le schéma de l'application, et le
// supprime à la fin. Les tests sont isolés et peuvent s'exécuter en parallèle.
func newTestDB(t *testing.T) *sql.DB {
	t.Helper()
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		t.Skip("DATABASE_URL non définie : test d'intégration ignoré")
	}
	admin, err := sql.Open("pgx", dsn)
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { admin.Close() })
	// 12 caractères aléatoires, en minuscules (PostgreSQL replie les identifiants non cités).
	schema := "test_" + strings.ToLower(rand.Text()[:12])
	if _, err := admin.ExecContext(t.Context(), `CREATE SCHEMA `+schema); err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() {
		// PAS t.Context() : il est déjà annulé quand les fonctions de Cleanup s'exécutent.
		if _, err := admin.ExecContext(context.Background(), `DROP SCHEMA `+schema+` CASCADE`); err != nil {
			t.Errorf("nettoyage du schéma %s : %v", schema, err)
		}
	})
	db, err := sql.Open("pgx", fmt.Sprintf("%s&search_path=%s", dsn, schema))
	if err != nil {
		t.Fatal(err)
	}
	db.SetMaxOpenConns(4)
	t.Cleanup(func() { db.Close() })
	if _, err := db.ExecContext(t.Context(), Schema); err != nil {
		t.Fatal(err)
	}
	return db
}

func TestContract(t *testing.T) {
	t.Parallel()
	booktest.Run(t, func(t *testing.T) book.Repository { return New(newTestDB(t)) })
}

// Le même Repo dans une transaction : une écriture annulée ne laisse aucune trace.
func TestInsideTransaction(t *testing.T) {
	t.Parallel()
	db := newTestDB(t)
	tx, err := db.BeginTx(t.Context(), nil)
	if err != nil {
		t.Fatal(err)
	}
	b := book.Book{Title: "T", Author: "A", ISBN: "9780306406157"}
	if err := New(tx).Create(t.Context(), &b); err != nil {
		t.Fatal(err)
	}
	tx.Rollback()
	if _, err := New(db).Get(t.Context(), b.ID); err == nil {
		t.Fatal("le livre ne devrait pas exister après Rollback")
	}
}
```

`go test -race -v ./...` (avec `DATABASE_URL`) :

```
ok   exemple/memory      (4 sous-tests du contrat)
ok   exemple/postgres    (les mêmes 4 sous-tests, plus TestInsideTransaction)
```

Sans `DATABASE_URL`, les tests du paquet `postgres` sont signalés `SKIP` et les autres passent.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `book.Repository` dans le paquet du domaine | Défini par le consommateur ; trois méthodes, celles des cas d'usage. |
| `book` n'importe ni `database/sql` ni `pgconn` | Le domaine ne sait pas qu'il existe une base. |
| `booktest.Run(t, newRepo)` | Le contrat, écrit une fois, exécuté par chaque implémentation. |
| `newRepo(t)` à chaque sous-test | Une instance vierge par cas : aucun test ne dépend d'un autre. |
| `memory.Repo` avec horloge injectée | Des dates distinctes et prévisibles, donc un tri vérifiable (N5 L3). |
| `postgres.DBTX` | `*sql.DB` ou `*sql.Tx` : `TestInsideTransaction` utilise le même `Repo` dans une transaction annulée. |
| Traduction `23505` → `ErrDuplicate`, `ErrNoRows` → `ErrNotFound` | Les deux implémentations rendent les **mêmes** erreurs du domaine : le contrat le vérifie. |
| `.UTC()` à la lecture | Le contrat compare les dates avec `Equal`, et l'implémentation ne dépend pas du fuseau de la machine (leçon 2). |
| `DEFAULT clock_timestamp()` | `now()` vaut l'heure de **début de transaction** : plusieurs insertions dans une même transaction auraient la même date. `clock_timestamp()` est l'heure réelle. Le tri garde `id` en second critère de toute façon. |
| `UNIQUE` sur `isbn` | La règle « un ISBN une seule fois » est garantie par la base, même sous concurrence (leçon 4). |
| Index `(author, created_at, id)` | Sert exactement la requête `ListByAuthor` (leçon 8). |
| `newTestDB` : schéma aléatoire, `search_path`, `Cleanup` en `context.Background()` | Isolation, parallélisme (`t.Parallel`), et rien ne reste après les tests. |

## Erreurs fréquentes

1. **Le SQL dans les handlers.**
2. **L'interface dans le paquet de l'implémentation** (`postgres.BookRepository`), que le domaine doit
   alors importer.
3. **Un `Repository[T]` générique** qui force toutes les requêtes dans le même moule.
4. **Laisser fuir `sql.ErrNoRows`** ou les erreurs `pgconn` hors du *repository*.
5. **Tester le *repository* avec des *mocks* de `database/sql`** au lieu d'une vraie base.
6. **Une implémentation mémoire qui ne respecte pas le même contrat** (tri, doublons, erreurs) que la
   vraie : les tests des couches supérieures mentent.
7. **Des tests d'intégration qui partagent des données** : ordre d'exécution significatif, échecs
   aléatoires en parallèle.
8. **`t.Context()` dans `t.Cleanup`** : le nettoyage échoue toujours.
9. **Un *repository* qui ouvre lui-même ses transactions** : impossible de composer deux opérations
   atomiquement.

## Bonnes pratiques Go

- Interface côté domaine, petite, nommée d'après les usages ; implémentation qui assume son SQL.
- Traduction des erreurs et des types **à la frontière**, et nulle part ailleurs.
- `DBTX` pour que les *repositories* se composent dans une transaction décidée par l'appelant.
- Une suite de contrat par interface de stockage, exécutée par chaque implémentation.
- Tests d'intégration contre une vraie base, isolés (schéma, base modèle ou conteneur), parallèles,
  ignorés proprement sans `DATABASE_URL`.
- Nettoyage en `context.Background()`, erreur vérifiée.

## Ce que je dois retenir

- Le *repository* isole le **domaine** du SQL ; il n'isole pas le SQL de PostgreSQL.
- Interface définie par le consommateur, peu de méthodes, erreurs du domaine.
- `DBTX` : le même code dans et hors transaction.
- Suite de contrat : toutes les implémentations se comportent pareil, et on le prouve.
- Le SQL se teste contre une vraie base ; un schéma par test donne isolation et parallélisme.

---

➡️ [Exercices](exercices.md)
