# Leçon 8 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Le mauvais index

Avec seulement `(created_at)`, PostgreSQL choisit `Parallel Index Scan Backward using l8_created` : il
parcourt l'index **par date décroissante** et filtre l'auteur ligne par ligne, en espérant trouver vite
20 lignes. Mesuré :

| Auteur | Lignes rejetées (× 3 processus) | Pages lues | Durée |
|---|---|---|---|
| `auteur 42` | ≈ 140 000 | ≈ 7 100 | ≈ 25 ms |
| `auteur 9999` | ≈ 137 000 | ≈ 6 900 | ≈ 24 ms |
| inexistant | **333 333** (toute la table) | ≈ 17 300 | ≈ 56 ms |

Le plan repose sur un pari : « les lignes de cet auteur sont réparties uniformément, j'en trouverai 20
rapidement ». Pour un auteur **inexistant**, le pari est perdu au maximum : il parcourt **tout** l'index
pour ne rien trouver. C'est le cas typique d'une requête rapide pour les données courantes et
catastrophique pour un cas limite (un utilisateur qui cherche quelque chose qui n'existe pas). Avec
l'index `(author, created_at)`, un auteur inexistant coûte trois pages.

## E2 — L'ordre des colonnes

| Requête | `(author, created_at)` | `(created_at, author)` |
|---|---|---|
| un auteur, 20 plus récents | ≈ 0,05 – 0,16 ms, quelques dizaines de pages | ≈ 10 ms, ≈ 1 000 pages (parcours par date, filtre sur `author` **dans** l'index) |
| un intervalle de dates seul (7 840 lignes) | **parcours séquentiel** de la table (≈ 28 ms, 9 154 pages) | parcours de l'index seul, ≈ 1,2 ms, 43 pages |

Règle : un index composite se lit **de gauche à droite**. Il sert un filtre sur une **préfixe** de ses
colonnes ; les colonnes d'égalité d'abord, la colonne de tri ou d'intervalle ensuite. Un index ne sert
pas une requête qui ne contraint pas sa première colonne.

## E3 — Un test qui compte

```go
func TestPageQueryCount(t *testing.T) {
	db := openTest(t) // + données : 50 livres, des avis
	c := &counter{db: db}
	if _, err := loadBatched(t.Context(), c); err != nil {
		t.Fatal(err)
	}
	if c.n > 2 {
		t.Fatalf("%d requêtes pour une page : N+1 ?", c.n)
	}
}
```

Avec `loadNPlusOne` : `51 requêtes pour une page : N+1 ?`. Ce test protège contre la régression la plus
courante : quelqu'un qui ajoute plus tard « juste un petit appel » dans la boucle.

## E4 — `OFFSET` contre curseur

Mesuré, auteur de 200 000 livres, index `(author, created_at, id)` :

| Page | `LIMIT 20 OFFSET n` | Curseur `(created_at, id) < ($1, $2)` |
|---|---|---|
| 1 | 0,09 ms, 5 pages | 0,09 ms, 5 pages |
| 100 | 1,6 ms, 34 pages | 0,10 ms, 5 pages |
| 10 000 | **117 ms**, 2 778 pages | 0,09 ms, 5 pages |

`OFFSET n` oblige PostgreSQL à **produire puis jeter** les n premières lignes : le coût croît
linéairement avec la page. Le curseur saute directement à la bonne position dans l'index : coût
constant. Autre défaut de `OFFSET` : une insertion pendant la navigation décale les pages (un livre vu
deux fois, ou jamais). L'index nécessaire pour le curseur : `(author, created_at, id)`, **dans l'ordre
du tri**, avec l'identifiant pour départager (c'est la pagination demandée par le projet 3, D2).

## E5 — Le coût du `fsync`

Mesuré : 10 000 `INSERT` hors transaction, **18,3 s** avec `synchronous_commit = on`, **0,79 s** avec
`off`. Avec `off`, le `COMMIT` rend la main **avant** que le journal ne soit écrit sur disque. En cas de
panne du **serveur** (pas d'un simple arrêt du programme), les transactions validées dans les dernières
centaines de millisecondes environ sont **perdues**, alors que le client a reçu « succès ». La base
reste cohérente (pas de corruption), mais ces écritures n'ont jamais existé.

Acceptable pour des données qu'on peut perdre ou reconstruire (journaux d'activité, statistiques,
caches, événements de télémétrie), réglé **par transaction** (`SET LOCAL synchronous_commit = off`) ;
inacceptable pour de l'argent, des commandes, des comptes. Le meilleur remède reste de grouper les
écritures (transaction, `unnest`, `COPY`), qui ne coûte rien en durabilité.

## Exercice intermédiaire — Un import en masse

```go
// Fichier : catalog/catalog.go
package catalog

import (
	"context"
	"encoding/csv"
	"errors"
	"fmt"
	"io"
	"strconv"

	"github.com/jackc/pgx/v5"
)

const Schema = `CREATE TABLE IF NOT EXISTS catalog (
	isbn   text PRIMARY KEY,
	title  text NOT NULL,
	author text NOT NULL,
	year   int  NOT NULL)`

type Row struct {
	ISBN, Title, Author string
	Year                int32
}

type LineError struct {
	Line int
	Err  error
}

func (e LineError) Error() string { return fmt.Sprintf("ligne %d : %v", e.Line, e.Err) }

// Parse lit et valide TOUT le fichier avant le moindre accès à la base.
func Parse(r io.Reader) ([]Row, []LineError, error) {
	cr := csv.NewReader(r)
	cr.FieldsPerRecord = 4
	header, err := cr.Read()
	if err != nil {
		return nil, nil, fmt.Errorf("en-tête : %w", err)
	}
	if fmt.Sprint(header) != "[isbn title author year]" {
		return nil, nil, fmt.Errorf("en-tête inattendu : %v", header)
	}
	var rows []Row
	var bad []LineError
	for line := 2; ; line++ {
		rec, err := cr.Read()
		if errors.Is(err, io.EOF) {
			break
		}
		if err != nil {
			bad = append(bad, LineError{line, err})
			continue
		}
		year, err := strconv.ParseInt(rec[3], 10, 32)
		switch {
		case len(rec[0]) != 13:
			bad = append(bad, LineError{line, fmt.Errorf("ISBN %q : 13 chiffres attendus", rec[0])})
		case err != nil:
			bad = append(bad, LineError{line, fmt.Errorf("année %q invalide", rec[3])})
		default:
			rows = append(rows, Row{rec[0], rec[1], rec[2], int32(year)})
		}
	}
	return rows, bad, nil
}

// Beginner est satisfaite par *pgx.Conn et *pgxpool.Pool.
type Beginner interface {
	Begin(ctx context.Context) (pgx.Tx, error)
}

// Import : COPY dans une table temporaire, puis fusion rejouable, dans UNE transaction.
func Import(ctx context.Context, db Beginner, rows []Row) (int64, error) {
	tx, err := db.Begin(ctx)
	if err != nil {
		return 0, err
	}
	defer tx.Rollback(ctx)
	// Table temporaire : propre à la session ET détruite au Commit. Pas de piège de pool ici :
	// tout se passe dans la même transaction, donc sur la même connexion.
	if _, err := tx.Exec(ctx, `CREATE TEMP TABLE tmp_catalog (LIKE catalog) ON COMMIT DROP`); err != nil {
		return 0, err
	}
	if _, err := tx.CopyFrom(ctx, pgx.Identifier{"tmp_catalog"},
		[]string{"isbn", "title", "author", "year"},
		pgx.CopyFromSlice(len(rows), func(i int) ([]any, error) {
			r := rows[i]
			return []any{r.ISBN, r.Title, r.Author, r.Year}, nil
		})); err != nil {
		return 0, fmt.Errorf("copy : %w", err)
	}
	// DISTINCT ON : un fichier qui contient deux fois le même ISBN ne fait pas échouer l'upsert
	// (« ON CONFLICT DO UPDATE command cannot affect row a second time »).
	tag, err := tx.Exec(ctx, `
		INSERT INTO catalog (isbn, title, author, year)
		SELECT DISTINCT ON (isbn) isbn, title, author, year FROM tmp_catalog ORDER BY isbn
		ON CONFLICT (isbn) DO UPDATE
		SET title = EXCLUDED.title, author = EXCLUDED.author, year = EXCLUDED.year
		WHERE (catalog.title, catalog.author, catalog.year)
		   IS DISTINCT FROM (EXCLUDED.title, EXCLUDED.author, EXCLUDED.year)`)
	if err != nil {
		return 0, err
	}
	return tag.RowsAffected(), tx.Commit(ctx)
}

// ImportUnnest : la variante sans COPY, par lots, pour comparaison.
func ImportUnnest(ctx context.Context, db Beginner, rows []Row, batch int) error {
	tx, err := db.Begin(ctx)
	if err != nil {
		return err
	}
	defer tx.Rollback(ctx)
	for start := 0; start < len(rows); start += batch {
		end := min(start+batch, len(rows))
		var isbn, title, author []string
		var year []int32
		for _, r := range rows[start:end] {
			isbn, title, author, year = append(isbn, r.ISBN), append(title, r.Title), append(author, r.Author), append(year, r.Year)
		}
		if _, err := tx.Exec(ctx, `
			INSERT INTO catalog SELECT * FROM unnest($1::text[], $2::text[], $3::text[], $4::int[])
			ON CONFLICT (isbn) DO UPDATE SET title = EXCLUDED.title, author = EXCLUDED.author, year = EXCLUDED.year`,
			isbn, title, author, year); err != nil {
			return err
		}
	}
	return tx.Commit(ctx)
}
```

```go
// Fichier : catalog/catalog_test.go
package catalog

import (
	"fmt"
	"os"
	"strings"
	"testing"
	"time"

	"github.com/jackc/pgx/v5"
)

func csvFile(n int, extra string) string {
	var b strings.Builder
	b.WriteString("isbn,title,author,year\n")
	for i := range n {
		fmt.Fprintf(&b, "978%010d,Titre %d,Auteur %d,%d\n", i, i, i%500, 1900+i%120)
	}
	b.WriteString(extra)
	return b.String()
}

func TestParseErrors(t *testing.T) {
	_, bad, err := Parse(strings.NewReader(csvFile(3, "123,Court,X,2000\n9780000000009,Année,X,deux mille\n")))
	if err != nil || len(bad) != 2 || bad[0].Line != 5 || bad[1].Line != 6 {
		t.Fatalf("bad = %v, err = %v", bad, err)
	}
	t.Log(bad)
}

func connect(t *testing.T) *pgx.Conn {
	t.Helper()
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		t.Skip("DATABASE_URL non définie")
	}
	conn, err := pgx.Connect(t.Context(), dsn)
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { conn.Close(t.Context()) })
	if _, err := conn.Exec(t.Context(), `DROP TABLE IF EXISTS catalog; `+Schema); err != nil {
		t.Fatal(err)
	}
	return conn
}

func TestImportReplayable(t *testing.T) {
	conn := connect(t)
	ctx := t.Context()
	rows, bad, _ := Parse(strings.NewReader(csvFile(100_000, "")))
	if len(bad) != 0 {
		t.Fatal(bad)
	}
	start := time.Now()
	n, err := Import(ctx, conn, rows)
	if err != nil || n != 100_000 {
		t.Fatalf("n=%d err=%v", n, err)
	}
	t.Logf("COPY + fusion, 100 000 lignes : %v", time.Since(start).Round(time.Millisecond))

	// Rejouer le même fichier : aucune ligne modifiée.
	if n, err := Import(ctx, conn, rows); err != nil || n != 0 {
		t.Fatalf("rejeu : n=%d err=%v", n, err)
	}
	// Doublons dans le fichier : pas d'erreur.
	if _, err := Import(ctx, conn, append(rows[:2:2], rows[0])); err != nil {
		t.Fatalf("doublons : %v", err)
	}

	conn.Exec(ctx, `TRUNCATE catalog`) // même point de départ que COPY : table vide
	start = time.Now()
	if err := ImportUnnest(ctx, conn, rows, 5000); err != nil {
		t.Fatal(err)
	}
	t.Logf("unnest par lots de 5 000 : %v", time.Since(start).Round(time.Millisecond))
}

func TestImportAllOrNothing(t *testing.T) {
	conn := connect(t)
	rows, _, _ := Parse(strings.NewReader(csvFile(1000, "")))
	rows[999].Title = strings.Repeat("x", 10) + "\x00" // un octet nul : refusé par PostgreSQL pendant COPY
	if _, err := Import(t.Context(), conn, rows); err == nil {
		t.Fatal("erreur attendue")
	} else {
		t.Log(err)
	}
	var count int
	conn.QueryRow(t.Context(), `SELECT count(*) FROM catalog`).Scan(&count)
	if count != 0 {
		t.Fatalf("%d lignes importées malgré l'échec", count)
	}
}
```

`go test ./...` passe. Mesures (100 000 lignes, table vide au départ) :

```
COPY + fusion, 100 000 lignes : 1,1 – 2,0 s
unnest par lots de 5 000      : 1,1 s
```

**Les deux se valent ici.** Le transfert (où `COPY` excelle) n'est plus le facteur limitant : le coût
est dans la **fusion** (`ON CONFLICT`, mise à jour de l'index de clé primaire). Mesurer a évité de
conclure trop vite que `COPY` « est toujours plus rapide ». `COPY` reste le bon outil pour charger des
millions de lignes **sans** fusion, ou quand le réseau est lent.

Points de conception :

- **Validation d'abord** : `Parse` refuse les lignes invalides avec leur numéro **avant** toute écriture
  (`ligne 5 : ISBN "123" : 13 chiffres attendus`). Choix fait ici : rendre les erreurs, et laisser
  l'appelant décider (tout refuser, ou importer `rows` et publier le rapport).
- **Tout ou rien** : une erreur pendant `COPY` (ici un octet nul, refusé par PostgreSQL :
  `invalid byte sequence for encoding "UTF8": 0x00`) annule la transaction ; la table reste vide.
- **Rejouable** : `ON CONFLICT … DO UPDATE … WHERE … IS DISTINCT FROM …` ne touche que les lignes qui
  ont réellement changé. Réimporter le même fichier modifie **0** ligne (vérifié), et ne génère ni
  écriture inutile ni déclenchement de *trigger*.
- **Doublons dans le fichier** : sans `DISTINCT ON`, PostgreSQL refuse de mettre à jour deux fois la
  même ligne dans une instruction (`ON CONFLICT DO UPDATE command cannot affect row a second time`).

**Point 5 — la table temporaire.** Le piège de la leçon 1 venait de **plusieurs appels au pool** : la
table était créée sur une connexion et utilisée sur une autre. Ici, tout se passe **dans une
transaction**, donc sur une seule connexion, et `ON COMMIT DROP` la supprime à la fin. Une table
temporaire est sûre quand sa durée de vie est contenue dans une transaction ou un `*sql.Conn`.

## Défi — Les trois derniers avis de chaque livre

Mesuré, 2 millions d'avis (100 par livre), page de 20 livres :

| Version | Requêtes | Lignes transférées | Index `(book_id)` | Index `(book_id, created_at DESC)` |
|---|---|---|---|---|
| (a) tous les avis par `= ANY`, filtrés en Go | 2 | **2 000** | 0,8 ms, 116 pages | 0,8 ms, 122 pages |
| (b) `ROW_NUMBER() … <= 3` | 2 | 60 | 1,5 ms, 122 pages | 1,6 ms, 128 pages |
| (c) `CROSS JOIN LATERAL (… LIMIT 3)` | 2 | 60 | 1,7 ms, **2 063** pages | **0,17 ms, 121 pages** |

La requête (c) :

```sql
SELECT b.id, r.rating, r.created_at
FROM unnest($1::bigint[]) AS b(id)
CROSS JOIN LATERAL (
	SELECT rating, created_at FROM reviews
	WHERE book_id = b.id ORDER BY created_at DESC LIMIT 3
) r;
```

- (a) transfère 33 fois trop de lignes : acceptable avec 100 avis par livre, intenable avec 10 000.
- (b) lit **tous** les avis des 20 livres pour en garder 3 par livre : l'index composite ne l'aide guère.
- (c) est **pire** que les autres sans le bon index (pour chaque livre, il lit tous ses avis puis trie),
  et **dix fois meilleur** avec l'index `(book_id, created_at DESC)` : pour chaque livre, trois lectures
  d'index et c'est fini. Une requête n'est pas rapide ou lente en soi ; elle l'est **avec** un index.

**GORM.** La tentative naturelle
`Preload("Reviews", func(db *gorm.DB) *gorm.DB { return db.Order("created_at DESC").Limit(3) })`
produit (vérifié) :

```sql
SELECT * FROM "l8d_reviews" WHERE "l8d_reviews"."book_id" IN (101,…,120) ORDER BY created_at DESC LIMIT 3
```

Le `LIMIT 3` s'applique à **toute** la requête de préchargement : **3 avis au total** pour 20 livres, sans
la moindre erreur. GORM ne sait pas exprimer « 3 par livre » avec `Preload` ; il faut écrire le SQL (b) ou
(c) avec `Raw` et répartir soi-même. C'est l'illustration exacte du cours : un ORM simplifie le cas
courant, et le cas un peu plus riche demande de redescendre au SQL — à condition de remarquer que le
résultat est faux.

## Quiz

1. `ANALYZE` exécute la requête et donne temps et lignes réels ; `BUFFERS` donne les pages lues (en
   mémoire ou sur disque).
2. La table est lue entièrement et presque tout est jeté : il manque un index adapté au filtre.
3. Les colonnes d'égalité d'abord, puis celle du tri ou de l'intervalle.
4. Chaque index ralentit les écritures et occupe de la place ; seuls ceux qui servent des requêtes réelles
   se justifient.
5. Une requête par élément d'une liste ; chacune paie un aller-retour réseau, bien plus long en
   production qu'en local.
6. Chargement par lot (`= ANY($1)`) ou jointure.
7. Chaque `COMMIT` attend l'écriture du journal sur disque.
8. Le protocole d'import en masse de PostgreSQL ; `CopyFrom` de pgx (API native).
9. On garde le SQL, vérifié contre le schéma à la génération, sans écrire le code de `Scan` à la main.
10. Non : il peut le rendre moins visible ; `Preload` l'évite si on le demande, la boucle naïve le
    reproduit.
