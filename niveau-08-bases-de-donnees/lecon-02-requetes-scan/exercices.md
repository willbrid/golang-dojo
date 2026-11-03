# Leçon 2 — Exercices

Code dans `mes-solutions/niveau-08/lecon-02/`.

## Exercices faciles

**E1 — Le `NULL` de production.**
Dans l'exemple, remplacer `Year sql.Null[int]` par `Year int`. Lancer : quel livre fait échouer le
programme, avec quel message ? Corriger de trois façons : `sql.Null[int]`, `*int`, et `COALESCE` dans le
SQL. Pour chacune, comment l'API JSON (N7 L4) afficherait-elle l'année inconnue ?

**E2 — Le résultat partiel.**
Écrire une fonction qui lit `SELECT 10 / (3 - n) FROM generate_series(1, 5) AS n` dans un `[]int`, d'abord
**sans** `rows.Err()`, puis avec. Qu'obtient l'appelant dans chaque cas ? Écrire le test qui aurait
attrapé l'oubli.

**E3 — Une destination de trop.**
Ajouter `author` à `bookColumns` sans modifier `scanBook`. Quel message obtient-on, et **quand** :
à la compilation, au démarrage, ou à la première requête ? Que dire d'un test qui n'appellerait jamais
`getBook` ?

**E4 — Les fuseaux.**
Insérer un livre, relire `created_at`, et l'afficher avec `%v` puis avec `.UTC()`. Lancer le programme
avec `TZ=America/New_York go run .` puis `TZ=Asia/Tokyo go run .`. Qu'est-ce qui change, et qu'est-ce
qui ne change pas ? Que donne `==` entre la valeur relue et la même valeur `.UTC()` ? Et `Equal` ?

**E5 — Un tableau PostgreSQL.**
Ajouter à la table une colonne `authors text[] NOT NULL DEFAULT '{}'`. L'écrire avec un `[]string` en
paramètre, et la relire : d'abord dans un `[]string` directement (message ?), puis avec
`pgtype.NewMap().SQLScanner`. Quand préférer `text[]` à `jsonb` ?

---

## Exercice intermédiaire — Des types qui se défendent

1. Écrire un type `ISBN` (une `string` de 13 chiffres avec un chiffre de contrôle valide) avec
   `ParseISBN(string) (ISBN, error)`. Le chiffre de contrôle : somme des 12 premiers chiffres pondérés
   alternativement par 1 et 3 ; le 13ᵉ vaut `(10 - somme % 10) % 10`.
2. Lui donner `Value()` : refuser d'écrire un `ISBN` invalide (un `ISBN("123")` construit sans
   `ParseISBN`), **avant** tout aller-retour réseau.
3. Lui donner `Scan(src any)` : refuser de lire une valeur invalide (une ligne insérée à la main dans
   `psql`). Le message d'erreur doit citer la colonne et la valeur.
4. Écrire un type `Money` (centimes en `int64`) stocké dans une colonne `numeric(12,2)` : `Value` rend
   `"12.34"`, `Scan` accepte `"12.34"`, `"12.3"`, `"12"` et refuse `"12.345"`. **Aucun `float64`**.
5. Table de tests pour `Scan` et `Value` **sans base** (ce sont des méthodes ordinaires), plus un test
   d'aller-retour contre PostgreSQL.

---

## Défi — Un scan générique

Écrire, avec les génériques (niveau 3) :

```go
func QueryAll[T any](ctx context.Context, db *sql.DB, scan func(rowScanner) (T, error),
	query string, args ...any) ([]T, error)

func QueryOne[T any](ctx context.Context, db *sql.DB, scan func(rowScanner) (T, error),
	query string, args ...any) (T, error)
```

1. `QueryAll` applique le patron complet (`Close`, `Err`) une fois pour toutes.
2. `QueryOne` rend `sql.ErrNoRows` si aucune ligne, et une erreur **si plus d'une ligne** (contrairement
   à `QueryRow`, qui ignore les suivantes en silence).
3. Une variante `QuerySeq` qui rend un `iter.Seq2[T, error]` (N3 L5) : quand la boucle de l'appelant
   fait `break`, `rows` doit être fermé. Le prouver par un test avec `db.SetMaxOpenConns(1)` : après un
   `break`, une seconde requête doit réussir sans attendre.
4. Comparer avec ce que pgx natif propose (`pgx.CollectRows`, `pgx.RowToStructByName`) : qu'apporte le
   mappage **par nom** de colonne, et quel est son coût ?

---

## Quiz

1. Où apparaît l'erreur d'un `QueryRowContext` ?
2. Que rend `Scan` quand la requête ne produit aucune ligne ?
3. Que fait `QueryRow` si la requête produit trois lignes ?
4. Pourquoi `defer rows.Close()` alors que la boucle complète ferme `rows` d'elle-même ?
5. Quand `rows.Next()` rend `false`, comment savoir si c'est la fin ou une erreur ?
6. Pourquoi éviter `SELECT *` ?
7. Trois façons de lire une colonne nullable ?
8. Quel type Go pour une colonne `numeric` qui contient des montants ?
9. Dans quel fuseau pgx rend-il un `timestamptz` ?
10. Quel récepteur pour `Scan` et pour `Value`, et pourquoi ?
