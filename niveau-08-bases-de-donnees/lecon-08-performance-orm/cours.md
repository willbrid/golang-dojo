# Leçon 8 — Performance et ORM

> Une requête lente coûte rarement à cause de Go. Elle coûte parce que la base lit un million de lignes
> pour en rendre vingt, parce que le programme envoie cent requêtes là où une suffisait, ou parce que
> chaque insertion attend son écriture sur disque. Cette leçon apprend à **lire ce que fait la base**
> (`EXPLAIN`), à repérer les deux défauts les plus coûteux côté application (N+1 et écritures une par
> une), puis à juger honnêtement les outils qui écrivent le SQL à notre place.

## Objectifs

1. Lire un plan d'exécution avec `EXPLAIN (ANALYZE, BUFFERS)`.
2. Choisir un index, y compris composite, en fonction de la requête (filtre **et** tri).
3. Reconnaître et supprimer le problème N+1.
4. Écrire en masse : transaction, `unnest`, `COPY`.
5. Situer SQL écrit à la main, générateur de code et ORM, avec leurs coûts réels.

## Explication

### Lire un plan : `EXPLAIN (ANALYZE, BUFFERS)`

`EXPLAIN` montre le **plan** que PostgreSQL a choisi ; `ANALYZE` exécute vraiment la requête et ajoute les
temps et nombres de lignes **réels** ; `BUFFERS` ajoute le nombre de pages de 8 Kio lues (`hit` : déjà
en mémoire ; `read` : lues sur disque). Le nombre de pages est souvent plus parlant que le temps, qui
dépend du cache.

Mesuré sur une table d'un million de livres (10 000 auteurs), pour la requête typique d'une API :

```sql
SELECT id, title FROM l8_books WHERE author = 'auteur 42' ORDER BY created_at DESC LIMIT 20;
```

**Sans index** :

```
Limit  (actual time=40.964..45.185 rows=20 loops=1)
  ->  Gather Merge
        ->  Sort  Sort Key: created_at DESC
              ->  Parallel Seq Scan on l8_books  (actual time=0.408..36.570 rows=33 loops=3)
                    Filter: (author = 'auteur 42'::text)
                    Rows Removed by Filter: 333300
                    Buffers: shared hit=9154
Execution Time: 45.220 ms
```

À lire de l'intérieur vers l'extérieur : un **parcours séquentiel** (*Seq Scan*) de toute la table,
réparti sur 3 processus, qui **rejette** 333 300 lignes chacun pour en garder 100, puis un tri, puis la
limite. 9 296 pages lues pour 20 lignes.

**Avec `CREATE INDEX ON l8_books (author)`** :

```
Limit  (actual time=0.154..0.157 rows=20 loops=1)
  ->  Sort  Sort Method: top-N heapsort
        ->  Bitmap Heap Scan on l8_books  (actual time=0.037..0.130 rows=100 loops=1)
              ->  Bitmap Index Scan on l8_author  Index Cond: (author = 'auteur 42'::text)
Execution Time: 0.170 ms
```

L'index trouve les 100 lignes de l'auteur ; il reste à les trier pour garder les 20 plus récentes.
103 pages.

**Avec `CREATE INDEX ON l8_books (author, created_at)`** :

```
Limit  (actual time=0.024..0.042 rows=20 loops=1)
  ->  Index Scan Backward using l8_author_created on l8_books  (rows=20 loops=1)
        Index Cond: (author = 'auteur 42'::text)
Execution Time: 0.053 ms
```

L'index composite range les lignes **par auteur, puis par date** : PostgreSQL le parcourt à l'envers à
partir du bon auteur et s'arrête après 20 lignes. Plus de tri. 23 pages.

| Index | Temps | Pages lues |
|---|---|---|
| aucun | 45 ms | 9 296 |
| `(author)` | 0,17 ms | 103 |
| `(author, created_at)` | 0,053 ms | 23 |

Règles qui en découlent :

- un index composite sert les colonnes **de gauche à droite** : `(author, created_at)` sert
  `WHERE author = …` et `WHERE author = … ORDER BY created_at`, mais pas `WHERE created_at > …` seul ;
- l'égalité d'abord, puis la colonne du tri ou de l'intervalle ;
- chaque index **ralentit les écritures** (il faut le mettre à jour) et occupe de la place : on indexe
  les requêtes **réelles**, pas toutes les colonnes ;
- l'estimation (`rows=42`) et le réel (`rows=33`) doivent être proches. Un grand écart signifie des
  statistiques périmées (`ANALYZE`) et des plans mal choisis.

En production, l'extension `pg_stat_statements` liste les requêtes qui consomment le plus de temps
**au total** : c'est là qu'on cherche, pas au hasard.

### Le problème N+1

Le défaut de performance le plus courant dans un service qui parle à une base :

```go
books := listBooks(ctx)                    // 1 requête
for _, b := range books {
	b.Reviews = listReviews(ctx, b.ID)     // + 1 requête PAR livre
}
```

Pour 100 livres, 101 requêtes. Chacune est rapide, mais chacune paie un aller-retour réseau. Mesuré
**en local** (aller-retour minimal) pour 100 livres et 500 avis :

| Méthode | Requêtes | Durée |
|---|---|---|
| N+1 | 101 | 7,8 ms |
| liste + `WHERE book_id = ANY($1)` | 2 | 0,48 ms |
| `LEFT JOIN` | 1 | 0,42 ms |

Seize fois plus lent ici ; avec une base à 1 ms de réseau, le N+1 dépasse les 100 ms, alors que les
deux autres restent sous 3 ms. Et le coût croît avec la taille de la page.

Deux corrections :

- **par lot** : charger la liste, puis toutes les données liées en **une** requête `= ANY($1)`, et les
  répartir en Go avec une `map` (l'exemple) ;
- **jointure** : une seule requête, mais les colonnes du livre sont répétées pour chaque avis, et un
  livre sans avis demande un `LEFT JOIN`. Pour des relations « un vers beaucoup » imbriquées sur
  plusieurs niveaux, la jointure multiplie les lignes ; le chargement par lot reste simple.

Le N+1 se cache aussi derrière des fonctions bien nommées (`book.Author()` qui interroge la base) et
derrière les ORM (voir plus bas). Un compteur de requêtes par requête HTTP (dans un *middleware*, N7 L3)
le rend visible : une page qui fait 101 requêtes n'a pas besoin d'un chronomètre pour être suspecte.

### Écrire en masse

Mesuré, 10 000 lignes de deux colonnes :

| Méthode | Durée |
|---|---|
| 10 000 `INSERT`, chacun dans sa propre transaction | 16,8 s |
| 10 000 `INSERT` dans **une** transaction | 728 ms |
| un seul `INSERT … SELECT * FROM unnest($1::int[], $2::text[])` | 12 ms |
| `COPY` (pgx natif, `CopyFrom`) | 7 ms |

- Hors transaction, chaque `INSERT` est validé, donc **écrit sur disque** (`fsync`) avant de rendre la
  main : c'est ce qui coûte 1,7 ms par ligne, pas l'insertion elle-même.
- Dans une transaction, il reste 10 000 allers-retours.
- `unnest` transforme des tableaux (un `[]int32`, un `[]string`, passés en **paramètres**) en lignes :
  une seule requête, toujours paramétrée, disponible avec `database/sql`.
- `COPY` est le protocole d'import en masse de PostgreSQL ; pgx l'expose par `conn.CopyFrom` (API
  native, leçon 1). C'est la méthode pour des centaines de milliers de lignes.

### Écrire le SQL soi-même… ou pas

Trois familles d'outils :

| Approche | Exemple dans l'écosystème Go | Principe |
|---|---|---|
| SQL écrit à la main | `database/sql`, pgx | Ce niveau. Tout est visible ; le code de `Scan` est répétitif. |
| Générateur de code | `sqlc` | On écrit les requêtes **en SQL** dans des fichiers ; l'outil les vérifie contre le schéma et **génère** les fonctions Go typées et leur `Scan`. |
| ORM (*object-relational mapping*) | GORM, ent | On manipule des structs ; la bibliothèque **construit** le SQL, souvent par réflexion. |

L'exemple montre GORM sur les mêmes données. Ce qu'il faut en retenir, mesuré :

- **Le SQL généré est visible, et il faut le regarder.** `Preload("Reviews")` produit bien deux requêtes
  (liste, puis `IN (…)`) : GORM évite le N+1 **si on le lui demande**. La boucle naïve, elle, en produit
  N+1 exactement comme le code à la main. Un ORM ne protège pas du N+1 ; il le rend moins visible.
- **`SELECT *`** par défaut : toutes les colonnes, même celles qu'on n'utilise pas (leçon 2).
- **Le journal de GORM affiche les valeurs insérées dans le SQL** (`IN (1,2,3)`) pour la lisibilité ; ce
  qui est envoyé à la base reste paramétré. Ne pas copier ces lignes de journal pour « rejouer » une
  requête en croyant qu'elles sont exactes, et attention aux données sensibles qu'elles contiennent.
- **Coût** : lire 10 000 lignes prend ≈ 7,2 ms et 69 800 allocations avec GORM, contre ≈ 4,2 ms et
  39 800 allocations avec `database/sql` (réflexion pour associer colonnes et champs). Négligeable pour
  une page de 20 lignes ; mesurable sur des lots.

Le choix, honnêtement :

- **SQL à la main** quand les requêtes sont peu nombreuses ou sophistiquées (verrous, CTE, `jsonb`,
  fenêtres) : c'est le défaut raisonnable en Go, et tout ce niveau le montre.
- **Générateur** (`sqlc`) quand les requêtes se multiplient : on garde le SQL **et** on supprime le code
  répétitif et les erreurs de `Scan` (leçon 2, E3), vérifiés à la génération plutôt qu'à l'exécution.
- **ORM** pour beaucoup de CRUD simple sur de nombreux types, en acceptant d'apprendre l'outil en plus de
  SQL, de surveiller le SQL qu'il produit, et de redescendre au SQL brut pour les requêtes importantes.

Dans tous les cas, **savoir lire le SQL produit et son plan** reste indispensable : aucun outil ne
dispense de comprendre ce que fait la base.

## Exemple

Le même chargement (livres et leurs notes) en N+1, par lot, puis avec GORM. Un petit compteur de
requêtes remplace le chronomètre : le résultat est le même, le nombre de requêtes non.

Dépendances : `go get github.com/jackc/pgx/v5 gorm.io/gorm gorm.io/driver/postgres`.

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"os"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
	"gorm.io/driver/postgres"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"
)

// counter compte les requêtes envoyées : le N+1 devient visible sans chronomètre.
type counter struct {
	db *sql.DB
	n  int
}

func (c *counter) QueryContext(ctx context.Context, q string, args ...any) (*sql.Rows, error) {
	c.n++
	return c.db.QueryContext(ctx, q, args...)
}

type BookReviews struct {
	Title   string
	Ratings []int
}

func ids(ctx context.Context, c *counter) ([]int64, []string, error) {
	rows, err := c.QueryContext(ctx, `SELECT id, title FROM l8_books_s ORDER BY id`)
	if err != nil {
		return nil, nil, err
	}
	defer rows.Close()
	var ids []int64
	var titles []string
	for rows.Next() {
		var id int64
		var t string
		if err := rows.Scan(&id, &t); err != nil {
			return nil, nil, err
		}
		ids, titles = append(ids, id), append(titles, t)
	}
	return ids, titles, rows.Err()
}

func ratings(rows *sql.Rows) (map[int64][]int, error) {
	defer rows.Close()
	m := map[int64][]int{}
	for rows.Next() {
		var id int64
		var r int
		if err := rows.Scan(&id, &r); err != nil {
			return nil, err
		}
		m[id] = append(m[id], r)
	}
	return m, rows.Err()
}

// N+1 : une requête pour la liste, puis UNE PAR LIVRE.
func loadNPlusOne(ctx context.Context, c *counter) ([]BookReviews, error) {
	ids, titles, err := ids(ctx, c)
	if err != nil {
		return nil, err
	}
	out := make([]BookReviews, len(ids))
	for i, id := range ids {
		rows, err := c.QueryContext(ctx,
			`SELECT book_id, rating FROM l8_reviews_s WHERE book_id = $1 ORDER BY rating`, id)
		if err != nil {
			return nil, err
		}
		m, err := ratings(rows)
		if err != nil {
			return nil, err
		}
		out[i] = BookReviews{titles[i], m[id]}
	}
	return out, nil
}

// Par lot : deux requêtes, quel que soit le nombre de livres.
func loadBatched(ctx context.Context, c *counter) ([]BookReviews, error) {
	ids, titles, err := ids(ctx, c)
	if err != nil {
		return nil, err
	}
	rows, err := c.QueryContext(ctx,
		`SELECT book_id, rating FROM l8_reviews_s WHERE book_id = ANY($1) ORDER BY book_id, rating`, ids)
	if err != nil {
		return nil, err
	}
	m, err := ratings(rows)
	if err != nil {
		return nil, err
	}
	out := make([]BookReviews, len(ids))
	for i, id := range ids {
		out[i] = BookReviews{titles[i], m[id]}
	}
	return out, nil
}

// Les mêmes données vues par GORM.
type Book struct {
	ID      int64
	Title   string
	Reviews []Review `gorm:"foreignKey:BookID"`
}

type Review struct {
	ID     int64
	BookID int64
	Rating int
}

func (Book) TableName() string   { return "l8_books_s" }
func (Review) TableName() string { return "l8_reviews_s" }

// sqlLogger affiche chaque requête générée par GORM, sans les durées.
type sqlLogger struct{ logger.Interface }

func (sqlLogger) Trace(_ context.Context, _ time.Time, fc func() (string, int64), _ error) {
	sql, _ := fc()
	fmt.Println("  GORM →", sql)
}

func main() {
	ctx := context.Background()
	dsn := os.Getenv("DATABASE_URL")
	db, err := sql.Open("pgx", dsn)
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()
	if _, err := db.ExecContext(ctx, `
		DROP TABLE IF EXISTS l8_reviews_s, l8_books_s;
		CREATE TABLE l8_books_s (id bigint PRIMARY KEY, title text NOT NULL);
		CREATE TABLE l8_reviews_s (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
			book_id bigint NOT NULL REFERENCES l8_books_s(id), rating int NOT NULL);
		CREATE INDEX ON l8_reviews_s (book_id);
		INSERT INTO l8_books_s VALUES (1, 'L''Étranger'), (2, 'La Peste'), (3, 'Vol de nuit');
		INSERT INTO l8_reviews_s (book_id, rating) VALUES (1, 5), (1, 4), (2, 3), (3, 5), (3, 2), (3, 4)`); err != nil {
		log.Fatal(err)
	}

	c := &counter{db: db}
	a, err := loadNPlusOne(ctx, c)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("N+1      : %d requêtes → %v\n", c.n, a)

	c = &counter{db: db}
	b, err := loadBatched(ctx, c)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("par lot  : %d requêtes → %v\n", c.n, b)

	gdb, err := gorm.Open(postgres.Open(dsn), &gorm.Config{Logger: sqlLogger{logger.Discard}})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("GORM, Preload :")
	var books []Book
	if err := gdb.WithContext(ctx).Preload("Reviews").Order("id").Find(&books).Error; err != nil {
		log.Fatal(err)
	}
	fmt.Println("GORM, boucle naïve :")
	var bare []Book
	gdb.WithContext(ctx).Order("id").Find(&bare)
	for i := range bare {
		gdb.WithContext(ctx).Where("book_id = ?", bare[i].ID).Find(&bare[i].Reviews)
	}
	fmt.Printf("livres GORM : %d, avis du premier : %d\n", len(books), len(books[0].Reviews))
}
```

Sortie :

```
N+1      : 4 requêtes → [{L'Étranger [4 5]} {La Peste [3]} {Vol de nuit [2 4 5]}]
par lot  : 2 requêtes → [{L'Étranger [4 5]} {La Peste [3]} {Vol de nuit [2 4 5]}]
GORM, Preload :
  GORM → SELECT * FROM "l8_reviews_s" WHERE "l8_reviews_s"."book_id" IN (1,2,3)
  GORM → SELECT * FROM "l8_books_s" ORDER BY id
GORM, boucle naïve :
  GORM → SELECT * FROM "l8_books_s" ORDER BY id
  GORM → SELECT * FROM "l8_reviews_s" WHERE book_id = 1
  GORM → SELECT * FROM "l8_reviews_s" WHERE book_id = 2
  GORM → SELECT * FROM "l8_reviews_s" WHERE book_id = 3
livres GORM : 3, avis du premier : 2
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `counter` | Une enveloppe minimale qui compte les `QueryContext` : on peut l'utiliser dans un test pour **exiger** un nombre de requêtes. |
| `loadNPlusOne` | 1 + N requêtes ; N grandit avec la page. |
| `loadBatched` | 2 requêtes quel que soit N ; la `map` répartit les avis par livre. |
| `= ANY($1)` avec `ids` | Une liste de taille quelconque en un seul paramètre (leçon 3). |
| `ORDER BY book_id, rating` | L'ordre des avis est décidé par la base, pas par le hasard du parcours. |
| Résultats identiques | La correction du N+1 ne change pas le résultat, seulement le nombre d'allers-retours. |
| `TableName()` | Les conventions de nommage de GORM (`books`, `reviews`) remplacées par les vraies tables. |
| `sqlLogger` | Affiche le SQL généré par GORM ; à activer en développement pour **voir** ce que fait l'ORM. |
| Ordre des lignes `Preload` | La requête de chargement des avis s'affiche avant la requête principale : le journal est écrit à la **fin** de chaque requête, et le préchargement s'exécute pendant le traitement de la principale. |
| Boucle naïve GORM | 4 requêtes : le même N+1, avec une autre syntaxe. |

## Erreurs fréquentes

1. **Optimiser sans `EXPLAIN (ANALYZE, BUFFERS)`.**
2. **Un index par colonne** au lieu d'un index composite adapté à la requête.
3. **Un index composite dans le mauvais ordre** (colonne de tri avant la colonne d'égalité).
4. **Indexer tout** : les écritures ralentissent, le disque se remplit.
5. **Le N+1**, à la main ou à travers un ORM.
6. **Des insertions en masse une par une, hors transaction.**
7. **`OFFSET` élevé** pour paginer : PostgreSQL lit et jette toutes les lignes sautées (exercice E4).
8. **Croire qu'un ORM dispense de connaître SQL.**
9. **Copier le SQL d'un journal d'ORM** en croyant qu'il est exactement ce qui a été envoyé.

## Bonnes pratiques Go

- Chercher les requêtes coûteuses avec `pg_stat_statements`, puis `EXPLAIN (ANALYZE, BUFFERS)`.
- Index composites conçus pour les requêtes réelles : égalité, puis tri ou intervalle.
- Charger les relations par lot (`= ANY`) ou par jointure ; compter les requêtes par requête HTTP.
- Écritures en masse : transaction, `unnest`, ou `COPY` (pgx natif).
- SQL à la main par défaut ; générateur quand les requêtes se multiplient ; ORM en connaissance de cause.
- Mesurer avant et après (N5 L6) : un index qui ne change pas le plan ne sert à rien.

## Ce que je dois retenir

- `EXPLAIN (ANALYZE, BUFFERS)` : le plan, les lignes réelles, les pages lues.
- Un bon index composite peut diviser le temps par mille et supprimer le tri.
- N+1 = un aller-retour par élément ; se corrige par un chargement par lot ou une jointure.
- En masse : jamais une transaction par ligne ; `unnest` et `COPY` font 1 000 fois mieux.
- Un ORM écrit le SQL, mais n'en supprime ni le coût ni la nécessité de le comprendre.

---

➡️ [Exercices](exercices.md)
