# Leçon 2 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Le `NULL` de production

« Manuscrit anonyme » fait échouer la liste :

```
sql: Scan error on column index 2, name "year": converting NULL to int is unsupported
```

| Correction | Code | JSON (avec `json:"year"`) |
|---|---|---|
| `sql.Null[int]` | `b.Year.Valid` | `{"V":0,"Valid":false}` — **à ne pas exposer** tel quel ; convertir vers un type d'API |
| `*int` | `b.Year == nil` | `null` — directement exploitable |
| `COALESCE(year, 0)` | `b.Year == 0` | `0` — faux : « année 0 » n'est pas « année inconnue » |

`COALESCE` n'est correct que si la valeur par défaut a **le même sens** que l'absence (un compteur
absent = 0). Ici elle ment. Pour une struct qui sert aussi de réponse JSON, le pointeur est le plus
simple ; mais la leçon 7 et le projet 3 (D8) séparent de toute façon le type de stockage du type d'API.

## E2 — Le résultat partiel

Sans `rows.Err()`, l'appelant reçoit `[5 10]` et **aucune erreur**. Avec, il reçoit l'erreur
`division by zero`. Le test qui attrape l'oubli :

```go
got, err := readInts(ctx, db, `SELECT 10 / (3 - n) FROM generate_series(1, 5) AS n`)
if err == nil {
	t.Fatalf("got %v sans erreur : rows.Err() oublié ?", got)
}
```

Ce test est **déterministe** grâce à la division par zéro. Le même oubli dans la vraie vie se manifeste
lors d'une coupure réseau ou d'une annulation : rare, invisible, et des données silencieusement
tronquées.

## E3 — Une destination de trop

```
sql: expected 6 destination arguments in Scan, not 5
```

L'erreur n'apparaît qu'**à l'exécution**, à la première requête qui passe par `scanBook`. Rien à la
compilation (les requêtes sont des chaînes), rien au démarrage. Un test qui n'appelle jamais `getBook`
ne le voit pas : d'où l'intérêt d'un test d'intégration **par requête** du *repository* (leçon 7), et
des générateurs de code qui vérifient les requêtes contre le schéma (leçon 8).

## E4 — Les fuseaux

Pour un même `created_at` (`2026-11-03 10:00:00Z`), mesuré :

```
TZ=Europe/Paris      2026-11-03 11:00:00 +0100 CET  | .UTC() 2026-11-03 10:00:00 +0000 UTC
TZ=America/New_York  2026-11-03 05:00:00 -0500 EST  | .UTC() 2026-11-03 10:00:00 +0000 UTC
TZ=Asia/Tokyo        2026-11-03 19:00:00 +0900 JST  | .UTC() 2026-11-03 10:00:00 +0000 UTC
== false   Equal true
```

L'**instant** ne change pas ; seul son **affichage** dépend du fuseau de la machine. `==` compare aussi le
fuseau (et la lecture monotone, N4 L4) : faux. `Equal` compare l'instant : vrai. Conséquences : un test
qui compare des heures avec `==` passe sur le portable du développeur et échoue dans la CI (fuseau
UTC), ou l'inverse ; et une réponse d'API qui affiche `%v` change selon le serveur. Règle : `.UTC()` à
la lecture (dans `scan…`), `Equal` pour comparer, RFC 3339 en UTC dans les API.

## E5 — Un tableau PostgreSQL

L'écriture d'un `[]string` en paramètre fonctionne. La lecture directe échoue :

```
sql: Scan error on column index 0, name "authors": unsupported Scan, storing driver.Value type string into type *[]string
```

(pgx rend le texte brut `{Camus,Sartre}` à `database/sql`, qui ne sait pas le découper.) Avec
`pgtype.NewMap().SQLScanner(&authors)`, on obtient `[Camus Sartre]`.

`text[]` quand c'est une **liste de valeurs simples** qu'on veut interroger avec les opérateurs de
tableau (`'Camus' = ANY(authors)`) et indexer (GIN) ; `jsonb` quand la structure est **imbriquée** ou
variable. Dans les deux cas, se demander d'abord si ce n'est pas une table à part (`book_authors`) :
une liste qu'on filtre, trie ou modifie élément par élément est une relation.

## Exercice intermédiaire — Des types qui se défendent

```go
// Fichier : bookdb/types.go
package bookdb

import (
	"database/sql/driver"
	"errors"
	"fmt"
	"strconv"
	"strings"
)

var ErrInvalidISBN = errors.New("ISBN invalide")

// ISBN est un ISBN-13 dont le chiffre de contrôle a été vérifié.
type ISBN string

func ParseISBN(s string) (ISBN, error) {
	if len(s) != 13 {
		return "", fmt.Errorf("%w : %q (13 chiffres attendus)", ErrInvalidISBN, s)
	}
	sum := 0
	for i := range 13 {
		c := s[i]
		if c < '0' || c > '9' {
			return "", fmt.Errorf("%w : %q (caractère %q)", ErrInvalidISBN, s, c)
		}
		if i == 12 {
			break
		}
		d := int(c - '0')
		if i%2 == 1 {
			d *= 3
		}
		sum += d
	}
	if want := (10 - sum%10) % 10; int(s[12]-'0') != want {
		return "", fmt.Errorf("%w : %q (chiffre de contrôle %d attendu)", ErrInvalidISBN, s, want)
	}
	return ISBN(s), nil
}

// Value refuse d'envoyer un ISBN construit sans ParseISBN.
func (i ISBN) Value() (driver.Value, error) {
	if _, err := ParseISBN(string(i)); err != nil {
		return nil, err
	}
	return string(i), nil
}

// Scan refuse de lire une valeur invalide présente en base.
func (i *ISBN) Scan(src any) error {
	var s string
	switch v := src.(type) {
	case string:
		s = v
	case []byte:
		s = string(v)
	default:
		return fmt.Errorf("ISBN : type %T non géré", src)
	}
	p, err := ParseISBN(s)
	if err != nil {
		return err
	}
	*i = p
	return nil
}

// Money est un montant en centimes, stocké en numeric(12,2).
type Money int64

func (m Money) Value() (driver.Value, error) {
	sign, v := "", int64(m)
	if v < 0 {
		sign, v = "-", -v
	}
	return fmt.Sprintf("%s%d.%02d", sign, v/100, v%100), nil
}

func (m *Money) Scan(src any) error {
	var s string
	switch v := src.(type) {
	case string:
		s = v
	case []byte:
		s = string(v)
	default:
		return fmt.Errorf("Money : type %T non géré", src)
	}
	p, err := ParseMoney(s)
	if err != nil {
		return err
	}
	*m = p
	return nil
}

// ParseMoney lit "12", "12.3", "12.34", "-0.05". Aucun float64.
func ParseMoney(s string) (Money, error) {
	neg := strings.HasPrefix(s, "-")
	body := strings.TrimPrefix(s, "-")
	whole, frac, hasDot := strings.Cut(body, ".")
	// Chiffres uniquement : strconv.ParseInt accepterait un signe ("--5", "+5", "1.+5").
	if !digits(whole) || (hasDot && (!digits(frac) || len(frac) > 2)) {
		return 0, fmt.Errorf("montant %q : format invalide", s)
	}
	for len(frac) < 2 {
		frac += "0"
	}
	w, err := strconv.ParseInt(whole, 10, 64)
	if err != nil || w > (1<<63-1)/100-1 {
		return 0, fmt.Errorf("montant %q : trop grand", s)
	}
	f, _ := strconv.ParseInt(frac, 10, 64) // deux chiffres : ne peut pas échouer
	c := w*100 + f
	if neg {
		c = -c
	}
	return Money(c), nil
}

func digits(s string) bool {
	if s == "" {
		return false
	}
	for _, c := range []byte(s) {
		if c < '0' || c > '9' {
			return false
		}
	}
	return true
}
```

Remarques :

- `Value` **revérifie** : un `ISBN("123")` est constructible (conversion), seul `ParseISBN` garantit la
  validité. L'erreur survient avant tout envoi réseau.
- `database/sql` enveloppe l'erreur de `Scan` en citant la colonne : on n'a pas à le faire soi-même
  (`sql: Scan error on column index 0, name "isbn": ISBN invalide : "123" (13 chiffres attendus)`), et
  `errors.Is(err, ErrInvalidISBN)` reste vrai.
- `ParseMoney` vérifie que chaque partie ne contient **que des chiffres**. Une première version qui
  s'en remettait à `strconv.ParseInt` acceptait `"--5"` (lu comme `-(-5)` = **+5,00**) et `"+5"` : le
  test de table l'a montré dès qu'on y a ajouté ces cas. Les entrées hostiles d'un analyseur, on les
  écrit, ou on les fait trouver par le fuzzing (N5 L7).
- `numeric(12,2)` rend toujours deux décimales (`"12.30"`), mais `Scan` accepte les autres formes pour
  rester utilisable avec une colonne `numeric` sans échelle.

```go
// Fichier : bookdb/types_test.go
package bookdb

import (
	"errors"
	"testing"
)

func TestParseISBN(t *testing.T) {
	for _, s := range []string{"9780306406157", "9782070360024"} {
		if _, err := ParseISBN(s); err != nil {
			t.Errorf("ParseISBN(%q) = %v", s, err)
		}
	}
	for _, s := range []string{"", "123", "9780306406158", "97803064061X7", "97803064061570"} {
		if _, err := ParseISBN(s); !errors.Is(err, ErrInvalidISBN) {
			t.Errorf("ParseISBN(%q) = %v, attendu ErrInvalidISBN", s, err)
		}
	}
}

func TestISBNValueRefusesInvalid(t *testing.T) {
	if _, err := ISBN("123").Value(); !errors.Is(err, ErrInvalidISBN) {
		t.Fatalf("err = %v", err)
	}
}

func TestMoney(t *testing.T) {
	ok := map[string]Money{"12": 1200, "12.3": 1230, "12.34": 1234, "0.05": 5, "-0.05": -5, "-12.30": -1230}
	for s, want := range ok {
		var m Money
		if err := m.Scan(s); err != nil || m != want {
			t.Errorf("Scan(%q) = %d, %v ; attendu %d", s, m, err, want)
		}
	}
	for _, s := range []string{"12.345", "", ".5", "12.", "abc", "1.-5", "1.+5", "--5", "+5", "-", "99999999999999999999"} {
		var m Money
		if err := m.Scan(s); err == nil {
			t.Errorf("Scan(%q) = %d, erreur attendue", s, m)
		}
	}
	for m, want := range map[Money]string{1234: "12.34", 5: "0.05", -5: "-0.05", 0: "0.00"} {
		if v, _ := m.Value(); v != want {
			t.Errorf("Value(%d) = %v, attendu %s", m, v, want)
		}
	}
}
```

Le test d'aller-retour contre PostgreSQL est dans le fichier de test du défi (`TestRoundTrip`) : il
vérifie qu'un ISBN invalide est refusé **à l'écriture** (`Valuer`) et **à la lecture** (`Scanner`,
avec une ligne insérée en SQL brut).

## Défi — Un scan générique

```go
// Fichier : bookdb/query.go
package bookdb

import (
	"context"
	"database/sql"
	"errors"
	"iter"
)

// RowScanner est satisfaite par *sql.Row et *sql.Rows.
type RowScanner interface {
	Scan(dest ...any) error
}

var ErrTooManyRows = errors.New("plus d'une ligne")

// QueryAll lit toutes les lignes ; le patron Close/Err est écrit une fois pour toutes.
func QueryAll[T any](ctx context.Context, db *sql.DB, scan func(RowScanner) (T, error),
	query string, args ...any) ([]T, error) {
	rows, err := db.QueryContext(ctx, query, args...)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	var out []T
	for rows.Next() {
		v, err := scan(rows)
		if err != nil {
			return nil, err
		}
		out = append(out, v)
	}
	return out, rows.Err()
}

// QueryOne exige exactement une ligne.
func QueryOne[T any](ctx context.Context, db *sql.DB, scan func(RowScanner) (T, error),
	query string, args ...any) (T, error) {
	var zero T
	rows, err := db.QueryContext(ctx, query, args...)
	if err != nil {
		return zero, err
	}
	defer rows.Close()
	if !rows.Next() {
		if err := rows.Err(); err != nil {
			return zero, err
		}
		return zero, sql.ErrNoRows
	}
	v, err := scan(rows)
	if err != nil {
		return zero, err
	}
	if rows.Next() {
		return zero, ErrTooManyRows
	}
	return v, rows.Err()
}

// QuerySeq rend les lignes une à une ; rows est fermé quand la boucle de l'appelant s'arrête.
func QuerySeq[T any](ctx context.Context, db *sql.DB, scan func(RowScanner) (T, error),
	query string, args ...any) iter.Seq2[T, error] {
	return func(yield func(T, error) bool) {
		var zero T
		rows, err := db.QueryContext(ctx, query, args...)
		if err != nil {
			yield(zero, err)
			return
		}
		defer rows.Close() // exécuté aussi quand yield rend false (break de l'appelant)
		for rows.Next() {
			v, err := scan(rows)
			if !yield(v, err) || err != nil {
				return
			}
		}
		if err := rows.Err(); err != nil {
			yield(zero, err)
		}
	}
}
```

```go
// Fichier : bookdb/query_test.go
package bookdb

import (
	"context"
	"database/sql"
	"errors"
	"os"
	"strings"
	"testing"
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
	return db
}

func scanInt(r RowScanner) (int, error) {
	var n int
	err := r.Scan(&n)
	return n, err
}

func TestQueryOne(t *testing.T) {
	db := openTest(t)
	ctx := t.Context()
	if n, err := QueryOne(ctx, db, scanInt, `SELECT 42`); n != 42 || err != nil {
		t.Errorf("une ligne : %d, %v", n, err)
	}
	if _, err := QueryOne(ctx, db, scanInt, `SELECT 1 WHERE false`); !errors.Is(err, sql.ErrNoRows) {
		t.Errorf("zéro ligne : %v", err)
	}
	if _, err := QueryOne(ctx, db, scanInt, `SELECT generate_series(1, 2)`); !errors.Is(err, ErrTooManyRows) {
		t.Errorf("deux lignes : %v", err)
	}
}

func TestQueryAllPartial(t *testing.T) {
	db := openTest(t)
	got, err := QueryAll(t.Context(), db, scanInt, `SELECT 10 / (3 - n) FROM generate_series(1, 5) AS n`)
	if err == nil || !strings.Contains(err.Error(), "division by zero") {
		t.Fatalf("got %v, err %v : l'erreur en cours de parcours doit remonter", got, err)
	}
}

func TestQuerySeqBreakReleasesConn(t *testing.T) {
	db := openTest(t)
	db.SetMaxOpenConns(1)
	for n, err := range QuerySeq(t.Context(), db, scanInt, `SELECT generate_series(1, 1000)`) {
		if err != nil {
			t.Fatal(err)
		}
		if n == 3 {
			break
		}
	}
	ctx, cancel := context.WithTimeout(t.Context(), time.Second)
	defer cancel()
	if err := db.PingContext(ctx); err != nil {
		t.Fatalf("connexion non rendue après break : %v", err)
	}
}

func TestRoundTrip(t *testing.T) {
	db := openTest(t)
	ctx := t.Context()
	if _, err := db.ExecContext(ctx, `
		DROP TABLE IF EXISTS l2_corr;
		CREATE TABLE l2_corr (isbn text NOT NULL, price numeric(12,2) NOT NULL)`); err != nil {
		t.Fatal(err)
	}
	isbn, _ := ParseISBN("9780306406157")
	if _, err := db.ExecContext(ctx, `INSERT INTO l2_corr VALUES ($1, $2)`, isbn, Money(1230)); err != nil {
		t.Fatal(err)
	}
	// Valuer : erreur AVANT l'envoi.
	if _, err := db.ExecContext(ctx, `INSERT INTO l2_corr VALUES ($1, $2)`, ISBN("123"), Money(1)); !errors.Is(err, ErrInvalidISBN) {
		t.Errorf("insert invalide : %v", err)
	}
	var gotISBN ISBN
	var gotPrice Money
	if err := db.QueryRowContext(ctx, `SELECT isbn, price FROM l2_corr`).Scan(&gotISBN, &gotPrice); err != nil {
		t.Fatal(err)
	}
	if gotISBN != isbn || gotPrice != 1230 {
		t.Errorf("relu %q %d", gotISBN, gotPrice)
	}
	// Scanner : une valeur invalide insérée « à la main » est refusée à la lecture.
	db.ExecContext(ctx, `INSERT INTO l2_corr VALUES ('123', 1)`)
	_, err := QueryAll(ctx, db, func(r RowScanner) (ISBN, error) {
		var i ISBN
		err := r.Scan(&i)
		return i, err
	}, `SELECT isbn FROM l2_corr ORDER BY isbn`)
	if !errors.Is(err, ErrInvalidISBN) {
		t.Fatalf("lecture invalide : %v", err)
	}
	t.Log(err)
}
```

`go test -race ./...` passe. Le test `TestQuerySeqBreakReleasesConn` est la preuve demandée : avec
une seule connexion dans le pool, le `Ping` après le `break` ne réussit que si `rows` a été fermé. Le
`defer rows.Close()` dans la fonction itérateur s'exécute quand `yield` rend `false` (N3 L5).

**pgx natif.** `pgx.CollectRows(rows, pgx.RowToStructByName[Book])` associe les colonnes aux champs
**par nom** (balise `db:"..."`) :

- avantage : l'ordre des colonnes n'importe plus, et une colonne manquante ou en trop est signalée
  **par son nom** ; le code de `scan…` disparaît ;
- coûts : de la réflexion (niveau 9) à chaque ligne (plus lent qu'un `Scan` positionnel, mesurable sur de
  gros volumes), une dépendance à pgx natif, et un couplage entre les noms de colonnes et les noms de
  champs : renommer un champ Go peut casser une requête.

## Quiz

1. Dans le `Scan` du `*sql.Row` rendu.
2. `sql.ErrNoRows`.
3. Il lit la première et ignore les autres, sans erreur.
4. Pour couvrir les sorties anticipées (erreur de `Scan`, `return` au milieu) ; sinon la connexion reste
   retenue.
5. `rows.Err()` : `nil` = fin normale.
6. Le nombre et l'ordre des colonnes changent avec la table ; le `Scan` positionnel casse.
7. `sql.Null[T]`, un pointeur `*T`, `COALESCE` dans le SQL (si la valeur par défaut a le même sens).
8. Un entier de centimes (avec `Scanner`/`Valuer`), une `string`, ou un type décimal ; jamais `float64`.
9. Dans le fuseau local de la machine.
10. `Scan` : récepteur pointeur (il modifie la valeur). `Value` : récepteur valeur (utilisable sur une
    valeur comme sur un pointeur).
