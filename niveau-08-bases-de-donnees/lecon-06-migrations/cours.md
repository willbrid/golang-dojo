# Leçon 6 — Migrations

> Le code d'un service évolue tous les jours ; son schéma de base aussi. Mais le code se remplace en
> redémarrant, alors que la base **garde ses données** et se modifie sous les pieds d'une application
> qui tourne. Une migration mal pensée bloque toutes les requêtes pendant des minutes, ou casse
> l'ancienne version du code encore en service pendant le déploiement. Cette leçon versionne le
> schéma, écrit un migrateur complet (pour comprendre ce que font les outils) et apprend à modifier une
> base **sans coupure**.

## Objectifs

1. Versionner le schéma dans des fichiers SQL numérotés, appliqués dans l'ordre et une seule fois.
2. Écrire un migrateur : table de suivi, transaction par migration, verrou contre les exécutions
   simultanées, détection des migrations modifiées.
3. Embarquer les migrations dans le binaire avec `embed`.
4. Savoir quelles opérations de schéma bloquent ou réécrivent une table, et les éviter.
5. Appliquer le patron *expand/contract* pour changer un schéma sans coupure.

## Explication

### Le principe

Chaque changement de schéma est un fichier, numéroté, **jamais modifié** une fois appliqué quelque part :

```
migrations/
├── 0001_books.sql
├── 0002_books_author.sql
└── 0003_reviews.sql
```

La base garde la liste des migrations appliquées dans une table (`schema_migrations`). Au démarrage (ou
dans une étape de déploiement), le migrateur applique, dans l'ordre, celles qui manquent. Conséquences :
toutes les bases (poste du développeur, CI, préproduction, production) passent par **les mêmes états**,
et l'on sait à tout moment dans quel état est une base.

Les fichiers sont embarqués dans le binaire (N4 L7) : le service et ses migrations voyagent ensemble.

```go
//go:embed migrations/*.sql
var migrationsFS embed.FS

sub, _ := fs.Sub(migrationsFS, "migrations")   // un fs.FS dont la racine est le dossier
ms, err := Load(sub)
```

En écrivant le migrateur contre l'interface `fs.FS` plutôt que contre `embed.FS`, on peut le tester avec
`testing/fstest.MapFS` : des fichiers en mémoire, construits par le test. C'est ce que fait l'exemple
pour rester en un seul fichier.

### Ce qu'un migrateur doit garantir

1. **Ordre et unicité** : tri par numéro ; deux fichiers avec le même numéro sont une erreur (cas
   fréquent : deux branches fusionnées qui ont chacune créé leur `0007_…`).
2. **Tout ou rien par migration** : PostgreSQL sait exécuter le DDL (`CREATE`, `ALTER`) **dans une
   transaction**. Une migration qui échoue au milieu est annulée entièrement, et l'enregistrement dans
   `schema_migrations` fait partie de la même transaction. (Ce n'est pas vrai de toutes les bases : MySQL
   valide implicitement chaque instruction DDL.)
3. **Une seule exécution à la fois** : dix instances qui démarrent ensemble ne doivent pas appliquer la
   même migration dix fois. Un verrou consultatif (leçon 1) sur une connexion réservée règle la question
   : les autres attendent, puis constatent qu'il n'y a plus rien à faire. Sans verrou (mesuré), deux
   instances simultanées échouent sur une erreur obscure :
   `duplicate key value violates unique constraint "pg_type_typname_nsp_index"`.
4. **Intégrité** : une empreinte (SHA-256) de chaque fichier est enregistrée. Si un fichier déjà appliqué
   a été modifié, le migrateur refuse de continuer : les bases déjà migrées ne recevront jamais la
   modification, et les environnements divergeraient en silence. Une correction est une **nouvelle**
   migration.

### Les outils

Ce que fait l'exemple est, en petit, ce que font les outils répandus de l'écosystème Go :
`golang-migrate`, `goose`, `tern` (de l'auteur de pgx), ou `atlas` (qui calcule aussi les migrations à
partir d'un schéma cible). Ils ajoutent : migrations écrites en Go, commandes en ligne (`up`, `down`,
`status`), gestion de plusieurs bases. Écrire le sien une fois permet de les juger ; en production, un
outil établi est généralement le bon choix.

### Revenir en arrière ?

Beaucoup d'outils proposent des migrations `down` (l'inverse de chaque `up`). En pratique :

- une migration `down` est **rarement testée**, donc souvent fausse le jour où on en a besoin ;
- elle **détruit des données** (`DROP COLUMN` d'une colonne remplie depuis le déploiement) ;
- le code déployé **après** la migration a peut-être déjà écrit des données qui en dépendent.

La pratique courante en production est **en avant seulement** (*forward-only*) : on corrige par une
nouvelle migration, et on rend chaque migration **compatible avec la version précédente du code**, pour
pouvoir revenir au code précédent **sans** toucher au schéma. C'est l'objet de la suite.

### Ce que coûte un changement de schéma

Toute modification de table prend un verrou. Certaines sont instantanées, d'autres **réécrivent** la
table entière pendant que tout est bloqué. Mesuré sur une table d'un million de lignes :

| Opération | Durée | Remarque |
|---|---|---|
| `ADD COLUMN a text NOT NULL DEFAULT 'x'` | 1,7 ms | défaut **constant** : métadonnées seulement (PostgreSQL 11+) |
| `ADD COLUMN b float8 NOT NULL DEFAULT random()` | 925 ms | défaut **volatile** : réécriture de toute la table |
| `ALTER COLUMN id TYPE bigint` | 857 ms | réécriture |
| `CREATE INDEX` | 1,8 s | bloque les **écritures** pendant toute la construction |
| `CREATE INDEX CONCURRENTLY` | 1,9 s | n'empêche pas les écritures ; **interdit dans une transaction** |

Sur une table de cent millions de lignes, multiplier par cent : les réécritures deviennent des minutes
d'indisponibilité.

### La file d'attente des verrous

Le danger le plus sournois n'est pas la durée de l'`ALTER`, mais l'**attente** de son verrou. Un
`ALTER TABLE` demande un verrou exclusif ; s'il doit attendre (une longue transaction lit la table), il se
met dans la file… et **toutes les requêtes qui arrivent après lui** attendent derrière, même un simple
`SELECT`. Mesuré :

| `lock_timeout` de la migration | `SELECT` ordinaire de l'application | `ALTER TABLE` |
|---|---|---|
| aucun | **bloqué**, expire au bout de son délai de 2 s | réussit après 2,2 s, quand la longue transaction se termine |
| `500ms` | réussit après 0,3 s | abandonne : `canceling statement due to lock timeout (SQLSTATE 55P03)` |

Une migration qui « ne fait presque rien » a bloqué toute l'application. Règle : **toute migration
s'exécute avec un `lock_timeout` court** (`SET lock_timeout = '2s'`), et se réessaie plus tard en cas
d'échec. Un échec de migration est un incident mineur ; une application gelée est une panne.

### Sans coupure : *expand / contract*

Pendant un déploiement progressif, **l'ancienne et la nouvelle version du code tournent en même temps**
sur le même schéma. Chaque migration doit donc être compatible avec les deux. Renommer une colonne
`name` en `title` d'un seul coup casse l'ancienne version à l'instant de la migration. On procède en
étapes, chacune déployée séparément :

1. **Expand** (élargir) : ajouter la nouvelle colonne `title`, nullable. Le code n'y touche pas encore.
2. **Écrire les deux** : la nouvelle version du code écrit `name` **et** `title`, lit `name`.
3. **Rattraper** : recopier `name` dans `title` pour les lignes existantes, **par lots** (pas un seul
   `UPDATE` d'un million de lignes qui verrouille tout).
4. **Basculer la lecture** : le code lit `title`, écrit encore les deux. On peut maintenant ajouter
   `NOT NULL` à `title`.
5. **Contract** (resserrer) : le code n'utilise plus `name` ; une dernière migration supprime la colonne.

Cinq déploiements au lieu d'un : c'est le prix de zéro seconde de coupure. Le même patron s'applique à
un changement de type, à un découpage de table, à l'ajout d'une contrainte `NOT NULL` sur une colonne
existante.

## Exemple

Un migrateur complet, testé sur trois scénarios : deux instances qui démarrent ensemble, une migration
modifiée après coup, une migration qui échoue au milieu.

```go
package main

import (
	"context"
	"crypto/sha256"
	"database/sql"
	"encoding/hex"
	"fmt"
	"io/fs"
	"log"
	"os"
	"path"
	"slices"
	"strconv"
	"strings"
	"sync"
	"testing/fstest"

	_ "github.com/jackc/pgx/v5/stdlib"
)

type Migration struct {
	Version int
	Name    string
	SQL     string
	Sum     string // empreinte du contenu : détecte une migration modifiée après coup
}

// Load lit les fichiers NNNN_nom.sql d'un fs.FS (un embed.FS en production).
func Load(fsys fs.FS) ([]Migration, error) {
	files, err := fs.Glob(fsys, "*.sql")
	if err != nil {
		return nil, err
	}
	var ms []Migration
	for _, f := range files {
		prefix, name, ok := strings.Cut(strings.TrimSuffix(path.Base(f), ".sql"), "_")
		v, err := strconv.Atoi(prefix)
		if !ok || err != nil || v <= 0 {
			return nil, fmt.Errorf("%s : nom attendu NNNN_description.sql", f)
		}
		body, err := fs.ReadFile(fsys, f)
		if err != nil {
			return nil, err
		}
		sum := sha256.Sum256(body)
		ms = append(ms, Migration{v, name, string(body), hex.EncodeToString(sum[:8])})
	}
	slices.SortFunc(ms, func(a, b Migration) int { return a.Version - b.Version })
	for i := 1; i < len(ms); i++ {
		if ms[i].Version == ms[i-1].Version {
			return nil, fmt.Errorf("version %d en double", ms[i].Version)
		}
	}
	return ms, nil
}

const lockID = 727_001 // identifiant arbitraire, propre à ce service

// Migrate applique les migrations manquantes, chacune dans sa transaction, sous un verrou
// consultatif : deux instances qui démarrent ensemble ne migrent pas en même temps.
func Migrate(ctx context.Context, db *sql.DB, ms []Migration) (applied int, err error) {
	conn, err := db.Conn(ctx) // le verrou de session exige UNE connexion
	if err != nil {
		return 0, err
	}
	defer conn.Close()
	if _, err := conn.ExecContext(ctx, `SELECT pg_advisory_lock($1)`, lockID); err != nil {
		return 0, err
	}
	defer conn.ExecContext(context.WithoutCancel(ctx), `SELECT pg_advisory_unlock($1)`, lockID)

	if _, err := conn.ExecContext(ctx, `CREATE TABLE IF NOT EXISTS schema_migrations (
		version    int PRIMARY KEY,
		name       text NOT NULL,
		checksum   text NOT NULL,
		applied_at timestamptz NOT NULL DEFAULT now())`); err != nil {
		return 0, err
	}
	done := map[int]string{}
	rows, err := conn.QueryContext(ctx, `SELECT version, checksum FROM schema_migrations`)
	if err != nil {
		return 0, err
	}
	for rows.Next() {
		var v int
		var sum string
		if err := rows.Scan(&v, &sum); err != nil {
			rows.Close()
			return 0, err
		}
		done[v] = sum
	}
	rows.Close()
	if err := rows.Err(); err != nil {
		return 0, err
	}

	for _, m := range ms {
		if sum, ok := done[m.Version]; ok {
			if sum != m.Sum {
				return applied, fmt.Errorf("migration %d (%s) modifiée après application", m.Version, m.Name)
			}
			continue
		}
		err := func() error {
			tx, err := conn.BeginTx(ctx, nil)
			if err != nil {
				return err
			}
			defer tx.Rollback()
			if _, err := tx.ExecContext(ctx, m.SQL); err != nil {
				return err
			}
			if _, err := tx.ExecContext(ctx,
				`INSERT INTO schema_migrations (version, name, checksum) VALUES ($1, $2, $3)`,
				m.Version, m.Name, m.Sum); err != nil {
				return err
			}
			return tx.Commit()
		}()
		if err != nil {
			return applied, fmt.Errorf("migration %d (%s) : %w", m.Version, m.Name, err)
		}
		applied++
	}
	return applied, nil
}

func main() {
	ctx := context.Background()
	db, err := sql.Open("pgx", os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()
	db.ExecContext(ctx, `DROP TABLE IF EXISTS schema_migrations, l6_reviews, l6_books`) // repartir de zéro

	// En production : //go:embed migrations/*.sql  puis  fs.Sub(migrationsFS, "migrations").
	fsys := fstest.MapFS{
		"0001_books.sql": {Data: []byte(`
			CREATE TABLE l6_books (
				id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
				title text NOT NULL
			);`)},
		"0002_books_author.sql": {Data: []byte(`
			ALTER TABLE l6_books ADD COLUMN author text NOT NULL DEFAULT '';`)},
		"0003_reviews.sql": {Data: []byte(`
			CREATE TABLE l6_reviews (
				book_id bigint NOT NULL REFERENCES l6_books(id) ON DELETE CASCADE,
				rating  int NOT NULL CHECK (rating BETWEEN 1 AND 5)
			);`)},
	}
	ms, err := Load(fsys)
	if err != nil {
		log.Fatal(err)
	}

	// Deux instances démarrent en même temps.
	var wg sync.WaitGroup
	results := make([]int, 2)
	for i := range 2 {
		wg.Go(func() {
			n, err := Migrate(ctx, db, ms)
			if err != nil {
				log.Fatal(err)
			}
			results[i] = n
		})
	}
	wg.Wait()
	fmt.Println("appliquées par les deux instances :", results[0]+results[1])

	n, err := Migrate(ctx, db, ms)
	fmt.Println("troisième démarrage :", n, err)

	// Quelqu'un « corrige » une migration déjà appliquée.
	fsys["0002_books_author.sql"] = &fstest.MapFile{Data: []byte(
		`ALTER TABLE l6_books ADD COLUMN author text NOT NULL DEFAULT 'inconnu';`)}
	ms, _ = Load(fsys)
	_, err = Migrate(ctx, db, ms)
	fmt.Println("après modification :", err)

	// Une migration qui échoue est annulée entièrement.
	fsys["0002_books_author.sql"] = &fstest.MapFile{Data: []byte(`
			ALTER TABLE l6_books ADD COLUMN author text NOT NULL DEFAULT '';`)}
	fsys["0004_bad.sql"] = &fstest.MapFile{Data: []byte(`
		ALTER TABLE l6_books ADD COLUMN isbn text;
		ALTER TABLE l6_books ADD COLUMN isbn text;`)}
	ms, _ = Load(fsys)
	_, err = Migrate(ctx, db, ms)
	fmt.Println("migration 4 :", err)
	var cols int
	db.QueryRowContext(ctx, `SELECT count(*) FROM information_schema.columns
		WHERE table_name = 'l6_books' AND column_name = 'isbn'`).Scan(&cols)
	fmt.Println("colonne isbn créée :", cols == 1)
}
```

Sortie :

```
appliquées par les deux instances : 3
troisième démarrage : 0 <nil>
après modification : migration 2 (books_author) modifiée après application
migration 4 : migration 4 (bad) : ERROR: column "isbn" of relation "l6_books" already exists (SQLSTATE 42701)
colonne isbn créée : false
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `Load(fsys fs.FS)` | Fonctionne avec `embed.FS` en production et `fstest.MapFS` en test. |
| Nom `NNNN_description.sql` | Le numéro donne l'ordre ; le tri est **numérique** (`Atoi`), pas alphabétique. |
| Version en double → erreur | Deux branches qui ont créé la même version sont détectées avant toute exécution. |
| `db.Conn` + `pg_advisory_lock` | Le verrou de session vit sur **une** connexion ; toutes les étapes passent par elle. |
| `defer … pg_advisory_unlock` avec `WithoutCancel` | Le verrou est relâché même si `ctx` est annulé. |
| `CREATE TABLE IF NOT EXISTS schema_migrations` | Le migrateur crée sa propre table de suivi. |
| Une transaction par migration, avec l'`INSERT` de suivi | Migration et enregistrement réussissent ou échouent **ensemble**. |
| Empreinte comparée | « Modifiée après application » : on refuse au lieu de diverger. |
| `appliquées … : 3` | Les deux instances se partagent le travail **sans** l'appliquer deux fois (laquelle fait quoi dépend de l'ordonnancement). |
| Migration 4 en échec, colonne `isbn` absente | La première instruction a été annulée avec la seconde. |

## Erreurs fréquentes

1. **Modifier une migration déjà appliquée** au lieu d'en écrire une nouvelle.
2. **Tri alphabétique** des fichiers sans numéros de largeur fixe (`10_…` avant `2_…`).
3. **Pas de verrou** : plusieurs instances migrent en même temps.
4. **Migrations hors transaction** quand la base le permet.
5. **Pas de `lock_timeout`** : la migration gèle l'application dans la file des verrous.
6. **Une migration incompatible avec le code encore déployé** (renommage, suppression, `NOT NULL` d'un coup).
7. **`CREATE INDEX` sans `CONCURRENTLY`** sur une grosse table en production (et oublier que
   `CONCURRENTLY` ne peut pas être dans une transaction).
8. **Un `UPDATE` massif** dans une migration : réécriture et verrous sur toute la table.
9. **Compter sur les migrations `down`** comme plan de retour arrière.
10. **Le schéma modifié à la main** en production : il diverge de ce que disent les migrations.

## Bonnes pratiques Go

- Migrations SQL embarquées (`embed`), migrateur écrit contre `fs.FS`.
- Une transaction par migration ; verrou consultatif ; empreintes vérifiées.
- `SET lock_timeout` court dans chaque migration ; réessayer plus tard plutôt que geler l'application.
- *Expand/contract* pour tout changement incompatible ; chaque migration compatible avec la version N−1.
- Index par `CREATE INDEX CONCURRENTLY`, hors transaction ; gros rattrapages par lots, hors migration.
- En avant seulement ; la sauvegarde et la restauration sont le vrai plan de secours.
- Un test qui applique **toutes** les migrations sur une base vide, à chaque intégration continue.

## Ce que je dois retenir

- Schéma = fichiers numérotés, appliqués une fois, jamais modifiés ; table de suivi dans la base.
- Migrateur : ordre, transaction par migration, verrou consultatif, empreintes.
- Certaines opérations réécrivent la table ou bloquent les écritures ; mesurer, éviter, ou `CONCURRENTLY`.
- Un `ALTER` qui attend son verrou bloque tout ce qui arrive derrière lui : `lock_timeout`.
- Sans coupure = *expand/contract* : plusieurs déploiements, chacun compatible avec le précédent.

---

➡️ [Exercices](exercices.md)
