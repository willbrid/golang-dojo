# Leçon 3 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Rejouer l'attaque

Mesures de référence (concaténation) : 2 lignes pour `x' OR '1'='1` ; les deux secrets `a1` et `b2`
rendus comme des noms avec l'`UNION` ; `cannot insert multiple commands into a prepared statement` via
`QueryContext` ; table **vidée** via `ExecContext`. Avec `$1` : 0 ligne pour chacune des saisies, et la
table intacte. La différence entre `Query` et `Exec` tient au protocole choisi par pgx (préparé ou
simple), pas à une quelconque protection : ne jamais s'y fier.

## E2 — `Sprintf` n'est pas un paramètre

Avec `%d` et un `int`, le texte produit ne contient que des chiffres : pas injectable **aujourd'hui**.
Mais :

- le jour où l'identifiant devient une `string` (UUID, slug), `%d` devient `%s` dans le même commit, et
  la faille apparaît **sans** que la forme du code change ;
- en revue de code, `Sprintf` dans une requête oblige à vérifier le **type** de chaque argument, à
  chaque fois ; `$1` se vérifie d'un coup d'œil ;
- le plan de la requête n'est pas réutilisable (chaque identifiant donne un texte SQL différent, donc
  une entrée de plus dans le cache d'instructions de pgx).

Une règle sans exception est une règle qu'un outil (ou un relecteur pressé) peut appliquer.

## E3 — `ORDER BY $1`

Les deux requêtes rendent les lignes dans l'**ordre d'insertion** (mesuré), sans erreur. `ORDER BY $1`
trie par une **expression constante** (la chaîne `'year'`, la même pour toutes les lignes) : un tri sans
effet. `ORDER BY 2` (un **littéral** entier) désigne la deuxième colonne du `SELECT` ; `ORDER BY $1::int`
avec la valeur 2 est encore une expression constante, pas une position. Le piège est vicieux : le code
semble fonctionner tant qu'on regarde des données insérées dans l'ordre.

## E4 — Le joker oublié

Sans échappement : « 100% » donne le motif `%100%%`, qui trouve « 100% Go » **et** « 1000 recettes » ;
« _ » donne `%_%` (« au moins un caractère »), qui trouve **tous** les titres (mesuré : 6 sur 6). Avec
échappement, « a_b » ne trouve que `a_b` (et pas `axb`).

## E5 — Préparer, ou pas

Ordres de grandeur attendus (une connexion, réseau local) : cache par défaut, `Stmt` préparé et
protocole simple sont dans le bruit l'un de l'autre (≈ 0,1 – 0,2 ms, dominé par l'aller-retour réseau) ;
`Prepare` à chaque appel est 2,5 à 4 fois plus lent (deux allers-retours de plus : préparation et
fermeture). Avec `-count 6` et `benchstat`, les trois premiers apparaissent typiquement avec un `p`
supérieur à 0,05 (« ~ ») : pas de différence démontrée. Conclusion : avec pgx, la préparation
explicite n'est pas une optimisation par défaut.

## Exercice intermédiaire — Un constructeur de requêtes sûr

```go
// Fichier : sqlq/sqlq.go
// Package sqlq construit des SELECT dynamiques où aucune valeur ne touche le texte SQL.
package sqlq

import (
	"errors"
	"fmt"
	"strconv"
	"strings"
)

// fragment n'est pas exporté : depuis un autre paquet, seule une CONSTANTE chaîne
// (littérale) peut être convertie implicitement en fragment. Une variable string,
// ou une concaténation avec une variable, est refusée à la compilation.
type fragment string

type clause struct {
	frag string
	args []any
}

type Builder struct {
	cols, table string
	wheres      []clause
	order       string
	limit       int
	errs        []error
}

func Select(cols fragment) *Builder { return &Builder{cols: string(cols)} }

func (b *Builder) From(table fragment) *Builder {
	b.table = string(table)
	return b
}

// Where ajoute une condition ; chaque ? du fragment reçoit une valeur, dans l'ordre.
// Limite : l'opérateur jsonb « ? » n'est pas utilisable ici (écrire jsonb_exists(col, ?)).
func (b *Builder) Where(frag fragment, args ...any) *Builder {
	if n := strings.Count(string(frag), "?"); n != len(args) {
		b.errs = append(b.errs, fmt.Errorf("Where(%q) : %d marqueur(s) pour %d valeur(s)", frag, n, len(args)))
	}
	b.wheres = append(b.wheres, clause{string(frag), args})
	return b
}

// WhereIn ajoute « col = ANY(?) » avec une slice comme unique paramètre.
func (b *Builder) WhereIn(col fragment, values any) *Builder {
	return b.Where(col+" = ANY(?)", values)
}

// OrderBy trie par allowed[key] ; une clé absente de la liste blanche est une erreur.
func (b *Builder) OrderBy(key string, allowed map[string]string, desc bool) *Builder {
	col, ok := allowed[key]
	if !ok {
		b.errs = append(b.errs, fmt.Errorf("tri %q non autorisé", key))
		return b
	}
	if desc {
		// La direction s'applique à CHAQUE colonne : « year, id » → « year DESC, id DESC ».
		// Ajouter " DESC" à la fin n'inverserait que la dernière.
		cols := strings.Split(col, ",")
		for i, c := range cols {
			cols[i] = strings.TrimSpace(c) + " DESC"
		}
		col = strings.Join(cols, ", ")
	}
	b.order = col
	return b
}

// Limit borne n dans [1, max] (défaut : max).
func (b *Builder) Limit(n, max int) *Builder {
	if n < 1 || n > max {
		n = max
	}
	b.limit = n
	return b
}

func (b *Builder) Build() (string, []any, error) {
	if err := errors.Join(b.errs...); err != nil {
		return "", nil, err
	}
	if b.cols == "" || b.table == "" {
		return "", nil, errors.New("Select et From sont obligatoires")
	}
	var sb strings.Builder
	var args []any
	sb.WriteString("SELECT " + b.cols + " FROM " + b.table)
	for i, c := range b.wheres {
		if i == 0 {
			sb.WriteString(" WHERE ")
		} else {
			sb.WriteString(" AND ")
		}
		parts := strings.Split(c.frag, "?")
		sb.WriteString(parts[0])
		for j, p := range parts[1:] {
			args = append(args, c.args[j])
			sb.WriteString("$" + strconv.Itoa(len(args)) + p)
		}
	}
	if b.order != "" {
		sb.WriteString(" ORDER BY " + b.order)
	}
	if b.limit > 0 {
		args = append(args, b.limit)
		sb.WriteString(" LIMIT $" + strconv.Itoa(len(args)))
	}
	return sb.String(), args, nil
}
```

```go
// Fichier : sqlq/sqlq_test.go
package sqlq

import (
	"reflect"
	"strings"
	"testing"
)

var sorts = map[string]string{"title": "title, id", "year": "year, id"}

func TestBuild(t *testing.T) {
	tests := []struct {
		name     string
		b        *Builder
		wantSQL  string
		wantArgs []any
		wantErr  string
	}{
		{"simple", Select("id").From("books"), "SELECT id FROM books", nil, ""},
		{"complet",
			Select("id, title").From("books").
				Where("author = ?", "Camus").Where("year BETWEEN ? AND ?", 1940, 1950).
				WhereIn("tag", []string{"a", "b"}).OrderBy("year", sorts, true).Limit(500, 100),
			"SELECT id, title FROM books WHERE author = $1 AND year BETWEEN $2 AND $3 AND tag = ANY($4) ORDER BY year DESC, id DESC LIMIT $5",
			[]any{"Camus", 1940, 1950, []string{"a", "b"}, 100}, ""},
		{"marqueurs", Select("id").From("books").Where("a = ? AND b = ?", 1), "", nil, "2 marqueur(s) pour 1 valeur(s)"},
		{"tri", Select("id").From("books").OrderBy("id; DROP TABLE books", sorts, false), "", nil, "non autorisé"},
		{"sans from", Select("id"), "", nil, "obligatoires"},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			q, args, err := tt.b.Build()
			if tt.wantErr != "" {
				if err == nil || !strings.Contains(err.Error(), tt.wantErr) {
					t.Fatalf("err = %v, attendu %q", err, tt.wantErr)
				}
				return
			}
			if err != nil || q != tt.wantSQL || !reflect.DeepEqual(args, tt.wantArgs) {
				t.Fatalf("Build() =\n %q %v %v\nattendu\n %q %v", q, args, err, tt.wantSQL, tt.wantArgs)
			}
		})
	}
}

// Propriété : le texte SQL ne dépend JAMAIS de la valeur ; la valeur n'apparaît que dans args.
func FuzzWhereAndOrder(f *testing.F) {
	for _, s := range []string{"x", "x' OR '1'='1", "title", "?", "$1", "year; DROP TABLE books"} {
		f.Add(s)
	}
	f.Fuzz(func(t *testing.T, s string) {
		q, args, err := Select("id").From("books").Where("title = ?", s).Build()
		if err != nil || q != "SELECT id FROM books WHERE title = $1" || len(args) != 1 || args[0] != s {
			t.Fatalf("Where(%q) : %q %v %v", s, q, args, err)
		}
		q, _, err = Select("id").From("books").OrderBy(s, sorts, false).Build()
		if _, allowed := sorts[s]; allowed != (err == nil) {
			t.Fatalf("OrderBy(%q) : err = %v", s, err)
		}
		if err == nil && q != "SELECT id FROM books ORDER BY "+sorts[s] {
			t.Fatalf("OrderBy(%q) : %q", s, q)
		}
	})
}
```

```go
// Fichier : sqlq/pg_test.go
package sqlq

import (
	"database/sql"
	"os"
	"testing"

	_ "github.com/jackc/pgx/v5/stdlib"
)

func TestAgainstPostgres(t *testing.T) {
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		t.Skip("DATABASE_URL non définie")
	}
	db, err := sql.Open("pgx", dsn)
	if err != nil {
		t.Fatal(err)
	}
	defer db.Close()
	ctx := t.Context()
	if _, err := db.ExecContext(ctx, `
		DROP TABLE IF EXISTS sqlq_books;
		CREATE TABLE sqlq_books (id int PRIMARY KEY, title text, year int, tag text);
		INSERT INTO sqlq_books VALUES (1, 'A', 1942, 'a'), (2, 'B', 1947, 'b'), (3, 'C', 1931, 'c')`); err != nil {
		t.Fatal(err)
	}
	q, args, err := Select("title").From("sqlq_books").
		Where("year >= ?", 1940).WhereIn("tag", []string{"a", "b", "x' OR '1'='1"}).
		OrderBy("year", sorts, true).Limit(10, 100).Build()
	if err != nil {
		t.Fatal(err)
	}
	rows, err := db.QueryContext(ctx, q, args...)
	if err != nil {
		t.Fatal(err)
	}
	defer rows.Close()
	var got []string
	for rows.Next() {
		var s string
		if err := rows.Scan(&s); err != nil {
			t.Fatal(err)
		}
		got = append(got, s)
	}
	if err := rows.Err(); err != nil {
		t.Fatal(err)
	}
	if len(got) != 2 || got[0] != "B" || got[1] != "A" {
		t.Fatalf("got %v", got)
	}
}
```

`go test -race ./...` passe ; `go test -fuzz FuzzWhereAndOrder -fuzztime 15s` : environ 85 000
exécutions sans échec. La propriété fuzzée n'est pas « le SQL ne contient pas `s` » (faux positif dès que
`s` vaut `id` ou `FROM`), mais **« le texte SQL ne dépend pas de `s` »**, ce qui est exactement la
définition d'une requête non injectable.

**Un bug trouvé en écrivant ce corrigé.** La première version de `OrderBy` faisait
`b.order = col + " DESC"`. Avec la liste blanche `"year": "year, id"` (un départage, comme dans le
cours), cela donne `ORDER BY year, id DESC` : seul `id` est inversé, `year` reste croissant. Le test de
table **attendait** cette chaîne fautive (on écrit souvent l'attendu en recopiant la sortie) ; c'est le
test **contre PostgreSQL**, qui vérifie le résultat (`B` avant `A`), qui l'a trouvé. Leçon : un test qui
compare le SQL produit vérifie la syntaxe ; seul un test contre la base vérifie le sens.

**Point 5 — empêcher la concaténation.** Le type `fragment` n'est pas exporté. Depuis un autre paquet :

```go
const byAuthor = "author = ?"
sqlq.Select("id").From("books").Where(byAuthor, a)                  // compile : constante
sqlq.Select("id").From("books").Where("title = '" + s + "'")        // refusé :
// cannot use "title = '" + s + "'" (value of type string) as sqlq.fragment value in argument to …Where
```

Une constante chaîne non typée est convertible implicitement en n'importe quel type de base `string`,
même non exporté ; une **valeur** de type `string` ne l'est pas, et l'appelant ne peut pas écrire la
conversion explicite `sqlq.fragment(x)` puisque le nom n'est pas exporté. Le compilateur garantit donc
que **tout** fragment SQL est écrit en dur dans le code source. (Contournable par réflexion ou `unsafe`,
mais plus par accident.)

## Défi — Audit d'un service existant

| # | Faille ou bug | Requête qui l'exploite | Effet |
|---|---|---|---|
| 1 | `cat` concaténé dans `IN (…)` | `?cat=x') OR ('1'='1` | `owner = $1 AND … AND category IN ('x') OR ('1'='1')` : `AND` est prioritaire sur `OR`, le `WHERE` devient vrai pour **toutes** les lignes → objets des **autres propriétaires** (vérifié : 2 lignes sur 2). |
| 2 | même point, `UNION` | `?cat=x') UNION SELECT id, password_hash FROM users --` | exfiltration d'une autre table. |
| 3 | `LIMIT %s` non validé | `?limit=(SELECT CASE WHEN substr((SELECT password_hash FROM users LIMIT 1),1,1)='a' THEN 1 ELSE 0 END)` | PostgreSQL accepte une sous-requête dans `LIMIT` (vérifié) : oracle booléen, le hachage s'extrait caractère par caractère (injection « à l'aveugle »). |
| 4 | `LIMIT` absent | `GET /items` sans `limit` | `LIMIT ` vide → `syntax error` → **toutes** les requêtes sans `limit` échouent en 500. |
| 5 | pas de borne | `?limit=100000000` | réponse énorme, mémoire et base saturées. |
| 6 | `sort` filtré par regexp | `?sort=password_hash` | tri par une colonne **secrète** : en observant l'ordre, on apprend des choses sur les valeurs cachées. `?sort=!!!` → `ORDER BY ` vide → 500. |
| 7 | `name` « échappé » à la main | `?name=%` ou `?name=_` | pas exploitable comme injection tant que `standard_conforming_strings` est actif (défaut de PostgreSQL), mais fragile, et les jokers ne sont pas échappés : `%` liste tout. |
| 8 | style | `regexp.MustCompile` à chaque requête ; variable `sql` qui masque le paquet `database/sql` | coût inutile ; confusion. |

Réponse au point 2 : oui, par la faille 1 (précédence `AND`/`OR`) ou 2 (`UNION`). Un filtre de
sécurité (`owner = $1`) placé dans un `WHERE` construit par concaténation ne protège rien.

Réécriture avec `sqlq` :

```go
var itemSorts = map[string]string{"name": "name, id", "created": "created_at, id"}

func (s *Store) Search(ctx context.Context, owner int64, f ItemFilter) ([]Item, error) {
	b := sqlq.Select("id, name").From("items").Where("owner = ?", owner)
	if f.Name != "" {
		b.Where("name LIKE ?", "%"+escapeLike(f.Name)+"%")
	}
	if len(f.Categories) > 0 {
		b.WhereIn("category", f.Categories)
	}
	if f.Sort == "" {
		f.Sort = "name"
	}
	q, args, err := b.OrderBy(f.Sort, itemSorts, f.Desc).Limit(f.Limit, 100).Build()
	if err != nil {
		return nil, fmt.Errorf("%w : %v", ErrInvalidFilter, err) // → 400, pas 500
	}
	// QueryContext + patron complet (leçon 2)
}
```

Autres changements : l'analyse de `*http.Request` sort du *store* (le handler construit un `ItemFilter`
typé, avec `limit` converti par `strconv.Atoi` et refusé s'il est invalide) ; une erreur de filtre
devient un 400 (N7 L5) ; un test par ligne du tableau ci-dessus, chacun vérifiant que l'utilisateur 1 ne
voit **jamais** un objet de l'utilisateur 2.

## Quiz

1. D'un texte SQL construit avec des valeurs (concaténation, `Sprintf`).
2. Le SQL est analysé (*Parse*) avant que les valeurs n'arrivent (*Bind*) : elles ne passent jamais par
   l'analyseur.
3. C'est réimplémenter l'analyseur du serveur, avec ses cas limites (encodages, chaînes `E'…'`, réglages).
4. Des valeurs : oui. Identifiants, mots-clés, structure : non.
5. Une liste blanche : clé fournie par le client → fragment écrit en dur ; clé inconnue = erreur.
6. `= ANY($1)` avec une slice.
7. Ce sont des jokers dans la valeur : sans échappement, la recherche est fausse (et peut coûter cher).
8. Elle les prépare et les garde dans un cache par connexion.
9. Boucle serrée mesurée, transaction, pilote sans cache.
10. Pour un tri total et stable (pagination déterministe).
