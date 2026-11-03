# Leçon 3 — Requêtes préparées

> L'injection SQL a plus de vingt-cinq ans et figure toujours parmi les failles les plus exploitées du
> Web. Elle ne demande aucune compétence : une apostrophe dans un champ de formulaire. Elle ne vient
> jamais du pilote ni de la base, mais toujours d'un **texte SQL construit avec des valeurs**. Cette
> leçon la montre en vrai, explique pourquoi les paramètres l'empêchent **par construction**, puis
> traite les cas où les paramètres ne suffisent pas : noms de colonnes, tri, listes, motifs `LIKE`.

## Objectifs

1. Voir une injection SQL réussir, et comprendre pourquoi.
2. Comprendre ce qu'un paramètre change dans le protocole : le SQL et les valeurs voyagent séparément.
3. Savoir ce qui **ne peut pas** être un paramètre, et le traiter par liste blanche.
4. Construire une requête dynamique (filtres optionnels) sans jamais concaténer une valeur.
5. Savoir quand préparer explicitement une requête (`PrepareContext`), et quand c'est inutile.

## Explication

### L'injection, en vrai

Une fonction de recherche « naïve » :

```go
q := "SELECT name FROM l3_users WHERE name = '" + input + "'"
rows, err := db.QueryContext(ctx, q)
```

Ce que l'attaquant saisit, et ce qui se passe (mesuré, PostgreSQL 17, pgx 5) :

| Saisie | SQL obtenu (extrait) | Effet |
|---|---|---|
| `x' OR '1'='1` | `WHERE name = 'x' OR '1'='1'` | **toutes** les lignes rendues : le filtre est contourné |
| `x' UNION SELECT secret FROM l3_users --` | `… UNION SELECT secret …` | les **secrets** de la table rendus comme des noms : exfiltration |
| `x'; DELETE FROM l3_users; --` (via `QueryContext`) | deux instructions | refusé : `cannot insert multiple commands into a prepared statement` |
| la même chose via `ExecContext` | deux instructions | **exécuté** : la table est vidée |

Le troisième cas n'est pas une protection : il s'agit d'un détail d'implémentation de pgx (une requête
`Query` est préparée, ce qui interdit plusieurs instructions ; un `Exec` sans paramètre passe par le
protocole simple, qui les accepte). Les deux premières attaques fonctionnent dans tous les cas. Un
`UPDATE` ou un `DELETE` construit par concaténation peut, lui, tout effacer.

Échapper soi-même les apostrophes (`strings.ReplaceAll(input, "'", "''")`) n'est pas une solution :
c'est réimplémenter l'analyseur SQL du serveur, avec ses jeux de caractères, ses chaînes `E'...'` et
ses cas limites. Les tentatives de ce genre ont une longue histoire de contournements.

### Pourquoi un paramètre protège par construction

```go
rows, err := db.QueryContext(ctx, `SELECT name FROM l3_users WHERE name = $1`, input)
```

Avec un paramètre, le protocole PostgreSQL envoie **deux messages distincts** :

1. *Parse* : le texte SQL avec ses marqueurs `$1`, analysé et compilé en plan **avant** toute valeur ;
2. *Bind* : les valeurs, sous forme de données typées.

La valeur arrive quand l'analyse est **terminée** : elle ne peut plus changer la structure de la requête.
`x' OR '1'='1` est cherché littéralement comme un nom (mesuré : 0 ligne). Ce n'est pas un
échappement mieux fait : il n'y a **rien à échapper**, parce que la valeur ne passe jamais par
l'analyseur.

Règle, sans exception : **toute valeur venant de l'extérieur du code est un paramètre.** Y compris les
valeurs « sûres » (un identifiant numérique déjà validé), pour que la règle reste vérifiable d'un coup
d'œil en revue de code.

### Ce qui ne peut pas être un paramètre

Un paramètre remplace une **valeur**, jamais un morceau de syntaxe. Ne peuvent pas être paramétrés :

- les **identifiants** : nom de table, de colonne (`ORDER BY $1` trie par une constante, pas par la
  colonne nommée) ;
- les **mots-clés** : `ASC`/`DESC`, opérateurs ;
- la **structure** : le nombre de conditions d'un `WHERE`.

Pour ceux-là, la seule méthode sûre est la **liste blanche** : l'entrée de l'utilisateur sert de **clé**
dans une table écrite par le programmeur, et c'est la **valeur** de cette table qui entre dans le SQL.
Une clé inconnue est une erreur. Le texte de l'utilisateur ne touche jamais le SQL.

Pour un identifiant réellement dynamique (un outil d'administration qui manipule des tables dont le nom
est une donnée), pgx fournit `pgx.Identifier{"public", name}.Sanitize()`, qui met le nom entre guillemets
doubles et double les guillemets internes (`"bo""oks; DROP"`). C'est l'exception, pas la règle.

### Listes et motifs

- **Une liste de valeurs** (`IN`) : en PostgreSQL, un **seul** paramètre de type tableau,
  `WHERE author = ANY($1)` avec un `[]string`. Pas de `IN ($1, $2, …)` construit à la main, et pas de
  `strings.Join` de valeurs.
- **`LIKE` / `ILIKE`** : la valeur est bien un paramètre, mais `%` et `_` y restent des **jokers**. Une
  recherche de « 100% » sans précaution trouve aussi « 1000 recettes ». Il faut échapper `\`, `%` et `_`
  dans la valeur (la barre oblique inverse est le caractère d'échappement par défaut de `LIKE` en
  PostgreSQL). Ce n'est pas une faille d'injection (la requête ne change pas de structure), mais un bug
  fonctionnel, et une requête `LIKE '%_%_%_%…'` peut coûter cher au serveur.

### Préparer explicitement : `PrepareContext`

```go
stmt, err := db.PrepareContext(ctx, `SELECT name FROM l3_users WHERE name = $1`)
defer stmt.Close()
err = stmt.QueryRowContext(ctx, "alice").Scan(&name)
```

Un `*sql.Stmt` est analysé une fois et exécuté plusieurs fois. Il est sûr en concurrence, et
`database/sql` le prépare de nouveau, de façon transparente, sur chaque connexion du pool où il est
utilisé.

Mais **pgx garde déjà un cache** d'instructions préparées par connexion : une requête paramétrée
exécutée deux fois sur la même connexion n'est analysée qu'une fois. Mesuré (`b.Loop`, une connexion,
requête par clé ; chiffres bruités par le réseau local) :

| Façon d'exécuter | Temps par requête |
|---|---|
| `db.QueryRowContext` (cache de pgx, défaut) | ≈ 0,11 – 0,20 ms |
| `stmt.QueryRowContext` (préparée une fois) | ≈ 0,10 – 0,18 ms |
| `PrepareContext` + exécution + `Close` **à chaque appel** | ≈ 0,33 – 0,60 ms |

Avec pgx, la préparation explicite n'apporte presque rien ; la préparer **à chaque appel** coûte un
aller-retour de plus. Règle : ne pas préparer « pour la performance » sans mesure ; utiliser
`PrepareContext` pour une requête exécutée en boucle serrée (un import de masse, par exemple), à
l'intérieur d'une transaction (`tx.StmtContext`, leçon 4), ou avec un pilote qui n'a pas de cache.

Une contrainte d'infrastructure : derrière un *pooler* comme PgBouncer en mode « transaction », deux
requêtes successives peuvent arriver sur deux connexions serveur différentes, et les instructions
préparées côté serveur posent problème. pgx propose alors
`default_query_exec_mode=simple_protocol` dans le DSN (les valeurs restent des paramètres, envoyés sans
préparation nommée).

## Exemple

Une recherche avec filtres optionnels, tri choisi par le client et limite : la forme la plus courante de
requête dynamique dans une API (N7 L4).

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"os"
	"strconv"
	"strings"

	_ "github.com/jackc/pgx/v5/stdlib"
)

// Filter vient de la requête HTTP : TOUT y est hostile.
type Filter struct {
	TitleContains string
	Authors       []string
	SortBy        string // "title" ou "year"
	Desc          bool
	Limit         int
}

// sortColumns : la liste blanche. Une clé inconnue est refusée, jamais insérée dans le SQL.
var sortColumns = map[string]string{
	"title": "title",
	"year":  "year",
}

// escapeLike neutralise les jokers de LIKE : "100%" doit chercher le texte « 100% ».
func escapeLike(s string) string {
	return strings.NewReplacer(`\`, `\\`, `%`, `\%`, `_`, `\_`).Replace(s)
}

// buildSearch construit le SQL avec des paramètres numérotés : les valeurs ne touchent jamais le texte.
func buildSearch(f Filter) (string, []any, error) {
	var (
		where []string
		args  []any
	)
	arg := func(v any) string { // ajoute une valeur, rend son marqueur $n
		args = append(args, v)
		return "$" + strconv.Itoa(len(args))
	}
	if f.TitleContains != "" {
		where = append(where, "title ILIKE "+arg("%"+escapeLike(f.TitleContains)+"%"))
	}
	if len(f.Authors) > 0 {
		where = append(where, "author = ANY("+arg(f.Authors)+")") // une liste = UN paramètre
	}
	col, ok := sortColumns[f.SortBy]
	if !ok {
		return "", nil, fmt.Errorf("tri %q non autorisé", f.SortBy)
	}
	dir := "ASC"
	if f.Desc {
		dir = "DESC"
	}
	if f.Limit < 1 || f.Limit > 100 {
		f.Limit = 20
	}
	q := "SELECT title, author FROM l3_books"
	if len(where) > 0 {
		q += " WHERE " + strings.Join(where, " AND ")
	}
	q += " ORDER BY " + col + " " + dir + ", id LIMIT " + arg(f.Limit)
	return q, args, nil
}

func search(ctx context.Context, db *sql.DB, f Filter) ([]string, error) {
	q, args, err := buildSearch(f)
	if err != nil {
		return nil, err
	}
	fmt.Printf("  SQL  : %s\n  args : %v\n", q, args)
	rows, err := db.QueryContext(ctx, q, args...)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	var out []string
	for rows.Next() {
		var title, author string
		if err := rows.Scan(&title, &author); err != nil {
			return nil, err
		}
		out = append(out, title+" ("+author+")")
	}
	return out, rows.Err()
}

func main() {
	ctx := context.Background()
	db, err := sql.Open("pgx", os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()
	if _, err := db.ExecContext(ctx, `
		DROP TABLE IF EXISTS l3_books;
		CREATE TABLE l3_books (id int GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
			title text NOT NULL, author text NOT NULL, year int NOT NULL);
		INSERT INTO l3_books (title, author, year) VALUES
			('L''Étranger', 'Camus', 1942), ('La Peste', 'Camus', 1947),
			('Le Petit Prince', 'Saint-Exupéry', 1943), ('Vol de nuit', 'Saint-Exupéry', 1931),
			('100% Go', 'Anonyme', 2026), ('1000 recettes', 'Anonyme', 2001)`); err != nil {
		log.Fatal(err)
	}

	for _, f := range []Filter{
		{Authors: []string{"Camus", "Saint-Exupéry"}, SortBy: "year", Desc: true, Limit: 3},
		{TitleContains: "100%", SortBy: "title"},
		{TitleContains: "x' OR '1'='1", SortBy: "title"},
		{SortBy: "year; DROP TABLE l3_books"},
	} {
		fmt.Printf("filtre %+v\n", f)
		res, err := search(ctx, db, f)
		if err != nil {
			fmt.Println("  refusé :", err)
			continue
		}
		fmt.Printf("  → %d résultat(s) %q\n", len(res), res)
	}
}
```

Sortie :

```
filtre {TitleContains: Authors:[Camus Saint-Exupéry] SortBy:year Desc:true Limit:3}
  SQL  : SELECT title, author FROM l3_books WHERE author = ANY($1) ORDER BY year DESC, id LIMIT $2
  args : [[Camus Saint-Exupéry] 3]
  → 3 résultat(s) ["La Peste (Camus)" "Le Petit Prince (Saint-Exupéry)" "L'Étranger (Camus)"]
filtre {TitleContains:100% Authors:[] SortBy:title Desc:false Limit:0}
  SQL  : SELECT title, author FROM l3_books WHERE title ILIKE $1 ORDER BY title ASC, id LIMIT $2
  args : [%100\%% 20]
  → 1 résultat(s) ["100% Go (Anonyme)"]
filtre {TitleContains:x' OR '1'='1 Authors:[] SortBy:title Desc:false Limit:0}
  SQL  : SELECT title, author FROM l3_books WHERE title ILIKE $1 ORDER BY title ASC, id LIMIT $2
  args : [%x' OR '1'='1% 20]
  → 0 résultat(s) []
filtre {TitleContains: Authors:[] SortBy:year; DROP TABLE l3_books Desc:false Limit:0}
  refusé : tri "year; DROP TABLE l3_books" non autorisé
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `arg(v)` | Ajoute la valeur à `args` et rend **son** marqueur `$n` : la numérotation ne peut pas se décaler. |
| `where` construit avec des marqueurs seulement | Le texte SQL ne contient **que** des chaînes écrites par le programmeur. |
| `sortColumns[f.SortBy]` | Liste blanche : `"year; DROP TABLE l3_books"` n'est pas une clé, donc refusé. |
| `ASC`/`DESC` choisis par un booléen | Le client ne fournit jamais le mot-clé lui-même. |
| `ORDER BY …, id` | Un tri **total** : sans départage, deux livres de même année sortent dans un ordre arbitraire (pagination instable). |
| `author = ANY($1)` avec un `[]string` | Une liste de taille quelconque, **un seul** paramètre. |
| `escapeLike` | « 100% » ne trouve que « 100% Go », pas « 1000 recettes ». |
| `LIMIT $n` borné | Une valeur, donc un paramètre ; bornée pour qu'aucun client ne demande un million de lignes. |
| L'injection dans `TitleContains` | Cherchée littéralement : 0 résultat. |

## Erreurs fréquentes

1. **Concaténer une valeur** dans le SQL, « parce qu'elle est déjà validée ».
2. **`fmt.Sprintf("… WHERE id = %d", id)`** : même faute, avec une apparence de sécurité.
3. **Échapper les apostrophes à la main.**
4. **`ORDER BY $1`** en croyant trier par la colonne nommée.
5. **Insérer le nom de colonne fourni par le client** après un simple `strings.Contains` ou une regexp.
6. **`IN (` + `strings.Join(ids, ",")` + `)`**.
7. **Oublier d'échapper `%` et `_`** dans un motif `LIKE`.
8. **`PrepareContext` à chaque appel** : un aller-retour de plus, sans aucun gain.
9. **Un `*sql.Stmt` jamais fermé** : il reste préparé sur chaque connexion qui l'a utilisé.
10. **Journaliser la requête avec ses valeurs** : mots de passe et données personnelles dans les
    journaux.

## Bonnes pratiques Go

- Toute valeur = paramètre `$n`, sans exception ni « cas sûr ».
- Identifiants, mots-clés, directions : liste blanche (`map` de clé client → fragment SQL écrit en dur).
- Requêtes dynamiques : un petit constructeur qui numérote les paramètres lui-même.
- Listes : `= ANY($n)` avec une slice. Motifs : échapper `\`, `%`, `_`.
- Pas de `PrepareContext` sans mesure avec pgx ; `Close` systématique quand on en fait un.
- Un test de sécurité par point d'entrée dynamique : une valeur hostile, et la vérification qu'elle est
  traitée comme une donnée.

## Ce que je dois retenir

- L'injection vient **toujours** d'un SQL construit avec des valeurs ; les paramètres l'empêchent par
  construction (*Parse* puis *Bind*).
- Un paramètre remplace une **valeur**, jamais un identifiant ni un mot-clé : pour ceux-là, une liste
  blanche.
- `= ANY($1)` pour les listes ; échapper les jokers de `LIKE`.
- pgx cache déjà les instructions préparées ; `PrepareContext` explicite se justifie par une mesure.

---

➡️ [Exercices](exercices.md)
