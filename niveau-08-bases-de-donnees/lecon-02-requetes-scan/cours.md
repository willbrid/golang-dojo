# Leçon 2 — Requêtes et scan

> Écrire une requête est facile ; **lire** son résultat correctement l'est moins. Une colonne `NULL`
> qui fait échouer un `Scan` en production, une erreur qui survient à la troisième ligne et qu'on ne
> voit pas, un `Rows` jamais fermé qui retient une connexion : ce sont les bugs de base de données les
> plus courants en Go. Cette leçon couvre la lecture d'une ligne, de plusieurs lignes, des `NULL`, et
> l'écriture de types personnalisés qui savent entrer et sortir de la base.

## Objectifs

1. Lire une ligne avec `QueryRowContext` et reconnaître `sql.ErrNoRows`.
2. Parcourir plusieurs lignes avec `QueryContext` **sans** oublier `Close` ni `Err`.
3. Lire les colonnes `NULL` avec `sql.Null[T]` ou un pointeur.
4. Connaître les correspondances de types PostgreSQL ↔ Go qui piègent (`numeric`, `timestamptz`,
   tableaux, `jsonb`).
5. Écrire un type qui implémente `sql.Scanner` et `driver.Valuer`.

## Explication

### Une ligne : `QueryRowContext` et `sql.ErrNoRows`

```go
var title string
err := db.QueryRowContext(ctx, `SELECT title FROM books WHERE id = $1`, id).Scan(&title)
```

`QueryRowContext` ne rend pas d'erreur : elle est **reportée** au `Scan`, qui est obligatoire (sans lui,
la connexion n'est pas rendue au pool). Si la requête ne produit aucune ligne, `Scan` rend la sentinelle
`sql.ErrNoRows` :

```
sql: no rows in result set
```

C'est le cas « introuvable » ; le *repository* la traduit en erreur du **domaine** (N7 L5), pour que les
couches supérieures ne dépendent pas de `database/sql` :

```go
if errors.Is(err, sql.ErrNoRows) {
	return Book{}, fmt.Errorf("livre %d : %w", id, ErrNotFound)
}
```

Si la requête produit **plusieurs** lignes, `QueryRow` lit la première et ignore les autres, sans
erreur. Une requête par identifiant ne devrait jamais en produire plusieurs ; si elle peut, c'est un
`Query`.

`Scan` prend **autant de destinations que de colonnes**, dans l'ordre du `SELECT` :

```
sql: expected 2 destination arguments in Scan, not 1
```

D'où la règle : jamais `SELECT *` dans du code (l'ajout d'une colonne à la table casse tous les `Scan`),
toujours la liste explicite des colonnes.

### Plusieurs lignes : le patron complet

```go
rows, err := db.QueryContext(ctx, `SELECT id, title FROM books ORDER BY id`)
if err != nil {
	return nil, err
}
defer rows.Close()

var books []Book
for rows.Next() {
	var b Book
	if err := rows.Scan(&b.ID, &b.Title); err != nil {
		return nil, err
	}
	books = append(books, b)
}
return books, rows.Err()
```

Chacune des quatre parties compte :

- **`defer rows.Close()`** : tant qu'un `Rows` est ouvert, il **tient une connexion** du pool. Le parcours
  complet ferme `rows` automatiquement, mais un `return` au milieu de la boucle (erreur de `Scan`,
  limite atteinte) le laisse ouvert. Le `defer` couvre tous les chemins. Un `Rows` oublié par requête
  HTTP, et le pool est vide au bout de quelques minutes (leçon 5).
- **`rows.Next()`** rend `false` à la fin des lignes… **ou** en cas d'erreur.
- **`rows.Scan`** dans la boucle, avec les mêmes règles que pour `QueryRow`.
- **`rows.Err()`** distingue les deux fins. Une erreur peut survenir **pendant** le parcours, après que
  des lignes ont déjà été reçues. Mesuré, avec une division par zéro à la troisième ligne :

  ```go
  rows, err := db.QueryContext(ctx, `SELECT 10 / (3 - n) FROM generate_series(1, 5) AS n`)
  // err == nil : la requête a démarré
  // la boucle reçoit 5 puis 10, puis Next rend false
  // rows.Err() : ERROR: division by zero (SQLSTATE 22012)
  ```

  Sans `rows.Err()`, la fonction rendrait **deux livres sur cinq** comme si c'était le résultat complet.
  La même chose arrive si la connexion tombe ou si le contexte est annulé au milieu.

### Un seul endroit sait lire une ligne

`*sql.Row` et `*sql.Rows` ont tous deux une méthode `Scan(dest ...any) error`. Une petite interface
(côté consommateur, N2 L4) permet d'écrire **une** fonction de lecture, utilisée par la requête « par
identifiant » comme par la liste :

```go
type rowScanner interface{ Scan(dest ...any) error }

const bookColumns = `id, title, year, tags, created_at`

func scanBook(r rowScanner) (Book, error) {
	var b Book
	err := r.Scan(&b.ID, &b.Title, &b.Year, &b.Tags, &b.CreatedAt)
	return b, err
}
```

La liste des colonnes et l'ordre des destinations sont côte à côte : ajouter une colonne, c'est modifier
deux lignes voisines.

### Les `NULL`

En SQL, `NULL` signifie « pas de valeur ». En Go, une `string` ou un `int` ont **toujours** une valeur.
Lire un `NULL` dans une `string` échoue :

```
sql: Scan error on column index 0, name "text": converting NULL to string is unsupported
```

Trois solutions, à choisir par colonne :

| Destination | `NULL` devient | Quand |
|---|---|---|
| `sql.Null[T]` (Go 1.22+) | `{V: zéro, Valid: false}` | le cas général : explicite, sans allocation |
| `*T` (pointeur) | `nil` | pratique si la struct est aussi encodée en JSON (`null`) |
| `sql.NullString`, `sql.NullInt64`… | idem `Null[T]` | code antérieur à Go 1.22 ; même idée, un type par type de base |
| `COALESCE(col, '')` dans le SQL | la valeur par défaut | quand l'absence et la valeur zéro ont **le même sens** métier |

`sql.Null[T]` fonctionne dans les deux sens : comme destination de `Scan`, et comme **paramètre** (un
`Null[T]` avec `Valid: false` est envoyé comme `NULL`).

La meilleure solution reste en amont : **`NOT NULL` sur toute colonne qui n'a pas de raison d'être
nulle**. Chaque colonne nullable est un `if` de plus dans le code, pour toujours.

### Correspondances de types qui piègent

Mesuré avec pgx et `database/sql` :

| PostgreSQL | Go | Attention |
|---|---|---|
| `text`, `varchar` | `string` | — |
| `integer`, `bigint` | `int`, `int64` | une valeur trop grande pour la destination : `value out of range` |
| `numeric` | `string`, ou un type décimal | dans un `float64` : `12345678901234567890.123456789` devient `1.2345678901234567e+19`. **Jamais** de `float64` pour de l'argent (N4 L3). |
| `timestamptz` | `time.Time` | rendu dans le fuseau **local de la machine** ; comparer et stocker en UTC (`.UTC()`). |
| `timestamp` (sans fuseau) | `time.Time` | rendu en UTC, mais l'instant réel est ambigu : préférer `timestamptz` partout. |
| `jsonb` | `[]byte`, `string`, ou un type `Scanner` | pas directement dans une `map` ni une struct. |
| `text[]` | pas avec `database/sql` seul | `unsupported Scan … into type *[]string` ; voir ci-dessous. |

Pour les tableaux PostgreSQL, pgx fournit un adaptateur qui fait le pont :

```go
import "github.com/jackc/pgx/v5/pgtype"

m := pgtype.NewMap()
var tags []string
err := db.QueryRowContext(ctx, `SELECT tags FROM books WHERE id = $1`, id).Scan(m.SQLScanner(&tags))
```

Dans l'autre sens, un `[]string` passé en paramètre est accepté tel quel par pgx.

### `Scanner` et `Valuer` : apprendre la base à un type

Deux interfaces d'une méthode, que `database/sql` consulte avant toute conversion :

```go
// database/sql : lire une colonne dans le type.
type Scanner interface {
	Scan(src any) error
}

// database/sql/driver : convertir le type en valeur envoyable.
type Valuer interface {
	Value() (driver.Value, error)
}
```

`src` est la valeur brute rendue par le pilote : `int64`, `float64`, `bool`, `[]byte`, `string`,
`time.Time` ou `nil` (pour `NULL`). `driver.Value` doit être l'un de ces types. `Scan` est une méthode
à **récepteur pointeur** (elle modifie la valeur) ; `Value` une méthode à **récepteur valeur** (pour
fonctionner avec une valeur comme avec un pointeur, N2 L3).

Usages typiques : un identifiant typé, une valeur validée (un `ISBN` qui refuse d'être lu s'il est
invalide), un montant en centimes, une liste stockée en `jsonb`, une énumération stockée en texte.

## Exemple

```go
package main

import (
	"context"
	"database/sql"
	"database/sql/driver"
	"encoding/json"
	"errors"
	"fmt"
	"log"
	"os"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
)

// Tags est stocké dans une colonne jsonb : Valuer pour écrire, Scanner pour lire.
type Tags []string

func (t Tags) Value() (driver.Value, error) {
	if t == nil {
		t = Tags{} // jamais de NULL : un tableau JSON vide
	}
	return json.Marshal(t)
}

func (t *Tags) Scan(src any) error {
	switch v := src.(type) {
	case []byte:
		return json.Unmarshal(v, t)
	case string:
		return json.Unmarshal([]byte(v), t)
	case nil:
		*t = nil
		return nil
	default:
		return fmt.Errorf("Tags : type %T non géré", src)
	}
}

type Book struct {
	ID        int64
	Title     string
	Year      sql.Null[int] // colonne NULL autorisée : année inconnue
	Tags      Tags
	CreatedAt time.Time
}

var ErrNotFound = errors.New("livre introuvable")

// rowScanner est satisfaite par *sql.Row ET *sql.Rows : un seul endroit sait lire un livre.
type rowScanner interface {
	Scan(dest ...any) error
}

const bookColumns = `id, title, year, tags, created_at`

func scanBook(r rowScanner) (Book, error) {
	var b Book
	err := r.Scan(&b.ID, &b.Title, &b.Year, &b.Tags, &b.CreatedAt)
	return b, err
}

func insertBook(ctx context.Context, db *sql.DB, b *Book) error {
	return db.QueryRowContext(ctx,
		`INSERT INTO l2_books (title, year, tags) VALUES ($1, $2, $3)
		 RETURNING id, created_at`,
		b.Title, b.Year, b.Tags,
	).Scan(&b.ID, &b.CreatedAt)
}

func getBook(ctx context.Context, db *sql.DB, id int64) (Book, error) {
	b, err := scanBook(db.QueryRowContext(ctx,
		`SELECT `+bookColumns+` FROM l2_books WHERE id = $1`, id))
	if errors.Is(err, sql.ErrNoRows) {
		return Book{}, fmt.Errorf("livre %d : %w", id, ErrNotFound)
	}
	return b, err
}

func listBooks(ctx context.Context, db *sql.DB, tag string) ([]Book, error) {
	rows, err := db.QueryContext(ctx,
		`SELECT `+bookColumns+` FROM l2_books WHERE tags ? $1 ORDER BY id`, tag)
	if err != nil {
		return nil, err
	}
	defer rows.Close() // rend la connexion au pool, même en cas de retour anticipé

	var books []Book
	for rows.Next() {
		b, err := scanBook(rows)
		if err != nil {
			return nil, err
		}
		books = append(books, b)
	}
	return books, rows.Err() // une erreur PENDANT le parcours n'apparaît qu'ici
}

func main() {
	if err := run(context.Background()); err != nil {
		log.Fatal(err)
	}
}

func run(ctx context.Context) error {
	db, err := sql.Open("pgx", os.Getenv("DATABASE_URL"))
	if err != nil {
		return err
	}
	defer db.Close()

	if _, err := db.ExecContext(ctx, `
		DROP TABLE IF EXISTS l2_books;
		CREATE TABLE l2_books (
			id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
			title      text NOT NULL,
			year       int,
			tags       jsonb NOT NULL DEFAULT '[]',
			created_at timestamptz NOT NULL DEFAULT now()
		)`); err != nil {
		return err
	}

	books := []Book{
		{Title: "L'Étranger", Year: sql.Null[int]{V: 1942, Valid: true}, Tags: Tags{"roman", "classique"}},
		{Title: "Manuscrit anonyme", Tags: Tags{"roman"}}, // Year.Valid == false → NULL
		{Title: "Recueil", Tags: nil},
	}
	for i := range books {
		if err := insertBook(ctx, db, &books[i]); err != nil {
			return fmt.Errorf("insert %q : %w", books[i].Title, err)
		}
	}
	fmt.Println("identifiants :", books[0].ID, books[1].ID, books[2].ID)

	list, err := listBooks(ctx, db, "roman")
	if err != nil {
		return err
	}
	for _, b := range list {
		year := "inconnue"
		if b.Year.Valid {
			year = fmt.Sprint(b.Year.V)
		}
		fmt.Printf("%d %-18s année=%-8s tags=%v date=%t\n", b.ID, b.Title, year, b.Tags, !b.CreatedAt.IsZero())
	}

	b, err := getBook(ctx, db, 3)
	fmt.Printf("livre 3 : %q tags=%v (nil=%t) err=%v\n", b.Title, b.Tags, b.Tags == nil, err)

	_, err = getBook(ctx, db, 99)
	fmt.Println("livre 99 :", err, "→ ErrNotFound ?", errors.Is(err, ErrNotFound))
	return nil
}
```

Sortie :

```
identifiants : 1 2 3
1 L'Étranger         année=1942     tags=[roman classique] date=true
2 Manuscrit anonyme  année=inconnue tags=[roman] date=true
livre 3 : "Recueil" tags=[] (nil=false) err=<nil>
livre 99 : livre 99 : livre introuvable → ErrNotFound ? true
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `Tags.Value` (récepteur valeur) | Un `Tags` nil devient `[]` : la colonne est `NOT NULL`, le type le garantit. |
| `Tags.Scan` (récepteur pointeur) | Gère `[]byte` (ce que pgx rend pour `jsonb`), `string` et `nil`, et refuse le reste avec un message clair. |
| `sql.Null[int]` | L'année inconnue est un `NULL` en base, `Valid: false` en Go — dans les deux sens. |
| `rowScanner` + `scanBook` | Une seule fonction de lecture pour `QueryRow` et `Query`. |
| `bookColumns` | La liste des colonnes, jamais `SELECT *`. |
| `RETURNING id, created_at` | Les valeurs calculées par la base (identité, `DEFAULT now()`) reviennent dans la même requête. |
| `errors.Is(err, sql.ErrNoRows)` → `ErrNotFound` | Le reste du programme ne voit pas `database/sql`. |
| `tags ? $1` | `?` est ici l'opérateur `jsonb` « contient la clé » ; aucun conflit avec les paramètres `$n`. |
| `defer rows.Close()` + `rows.Err()` | Les deux garde-fous du parcours. |
| « Recueil » relu avec `tags=[]` | Le nil d'origine a été stocké `[]` et relu comme slice **vide** non nil : un aller-retour en base ne préserve pas toujours la distinction nil/vide. |

## Erreurs fréquentes

1. **Oublier `rows.Close()`** sur un chemin de sortie : connexion retenue, pool épuisé.
2. **Oublier `rows.Err()`** : un résultat partiel pris pour un résultat complet.
3. **Ne pas appeler `Scan` après `QueryRow`** : la connexion n'est pas rendue.
4. **Comparer `err == sql.ErrNoRows`** après avoir enveloppé l'erreur : `errors.Is`.
5. **Laisser `sql.ErrNoRows` remonter** jusqu'au handler, qui doit alors importer `database/sql`.
6. **`SELECT *`** : l'ordre et le nombre des colonnes changent avec la table.
7. **Lire une colonne nullable dans une `string`** : marche en test (aucun `NULL`), casse en production.
8. **`numeric` dans un `float64`** pour un montant.
9. **Comparer des `time.Time` lus en base avec `==`** : fuseau et monotonie diffèrent (N4 L4) ;
   `t.Equal(u)`.
10. **`Scan` avec récepteur valeur** : la méthode modifie une copie, et le type ne satisfait pas
    `sql.Scanner`.

## Bonnes pratiques Go

- `QueryRow` + `Scan` immédiat ; `sql.ErrNoRows` traduit en erreur de domaine dans la couche d'accès.
- Le patron `Query` complet, toujours : `defer Close`, `Next`, `Scan`, `Err`.
- Liste de colonnes en constante, à côté de la fonction `scan…` qui la lit.
- `NOT NULL` par défaut dans le schéma ; `sql.Null[T]` pour les vraies absences.
- `timestamptz` en base, UTC dans le code.
- Types `Scanner`/`Valuer` pour les valeurs métier qui ont des règles.

## Ce que je dois retenir

- `QueryRow` reporte l'erreur au `Scan` ; « aucune ligne » = `sql.ErrNoRows`.
- `Query` : `defer rows.Close()`, boucle `Next`/`Scan`, puis **`rows.Err()`**.
- Un `NULL` ne rentre pas dans un type Go de base : `sql.Null[T]`, pointeur, ou `NOT NULL`.
- `Scanner` (récepteur pointeur) pour lire, `Valuer` (récepteur valeur) pour écrire.
- Pièges de types : `numeric`, `timestamptz` (fuseau local), tableaux, `jsonb`.

---

➡️ [Exercices](exercices.md)
