# Leçon 3 — Exercices

Code dans `mes-solutions/niveau-08/lecon-03/`. **Uniquement sur ta base locale `dojo`** : ces exercices
détruisent des données.

## Exercices faciles

**E1 — Rejouer l'attaque.**
Créer la table `l3_users(name text, secret text)` avec deux lignes. Écrire la recherche par
concaténation, et reproduire les quatre lignes du tableau du cours (filtre contourné, `UNION`,
instructions empilées via `QueryContext` puis via `ExecContext`). Corriger avec un paramètre et
vérifier que les quatre saisies ne rendent plus rien et ne modifient plus rien.

**E2 — `Sprintf` n'est pas un paramètre.**
Écrire `fmt.Sprintf("SELECT name FROM l3_users WHERE id = %d", id)`. Est-ce injectable si `id` est un
`int` ? Et si le code évolue vers `%s` le jour où les identifiants deviennent des UUID ? Pourquoi la règle
« toute valeur est un paramètre » ne fait-elle pas d'exception pour les entiers ?

**E3 — `ORDER BY $1`.**
Exécuter `SELECT title FROM l3_books ORDER BY $1` avec `"year"` en paramètre. Quel ordre obtient-on, et
pourquoi aucune erreur ? Et avec `ORDER BY $1::int` et la valeur `2` ?

**E4 — Le joker oublié.**
Retirer `escapeLike` de l'exemple. Que rend la recherche « 100% » ? Et la recherche « _ » ? Ajouter
au tableau d'entrées un titre `a_b` et vérifier que la version avec échappement ne trouve que lui pour
« a_b ».

**E5 — Préparer, ou pas.**
Reproduire les mesures du cours avec `b.Loop` (N5 L6) : cache par défaut, `Stmt` préparé une fois,
`Prepare` à chaque appel. Ajouter le mode `default_query_exec_mode=simple_protocol` dans le DSN.
Recommencer avec `-count 6` et comparer avec `benchstat` si tu l'as installé. Les écarts mesurés sont-ils
plus grands que le bruit ?

---

## Exercice intermédiaire — Un constructeur de requêtes sûr

Écrire un petit paquet `sqlq` qui généralise `buildSearch` :

```go
b := sqlq.Select("id, title, author").From("books")
b.Where("author = ?", author)          // ? est remplacé par $n à la construction
b.Where("year >= ?", minYear)
b.WhereIn("tag", tags)                 // tag = ANY($n)
b.OrderBy(userSort, map[string]string{"title": "title", "year": "year"}) // liste blanche
b.Limit(limit, 100)                    // borné
q, args, err := b.Build()
```

1. `Where` n'accepte qu'un fragment avec **exactement** autant de `?` que de valeurs ; sinon, erreur à
   `Build`.
2. `OrderBy` avec une clé inconnue : erreur à `Build`, pas une panique.
3. Tests de table sur le SQL et les arguments produits (sans base), et un test contre PostgreSQL.
4. Un test de fuzzing (N5 L7) : pour toute chaîne `s` passée à `Where("title = ?", s)` ou à `OrderBy`,
   le SQL produit ne contient **jamais** `s` (sauf si `s` est une clé de la liste blanche).
5. Limite de conception : `Where("title = '" + s + "'")` reste possible. Peut-on l'empêcher en Go ?
   (Indice : un paramètre de type non exporté que seule une constante littérale peut satisfaire.)

---

## Défi — Audit d'un service existant

On te remet ce code d'un service réel (simplifié) :

```go
func (s *Store) Search(ctx context.Context, r *http.Request) ([]Item, error) {
	q := r.URL.Query()
	sql := "SELECT id, name FROM items WHERE owner = $1"
	args := []any{userID(r)}
	if name := q.Get("name"); name != "" {
		sql += " AND name LIKE '%" + strings.ReplaceAll(name, "'", "''") + "%'"
	}
	if cat := q["cat"]; len(cat) > 0 {
		sql += " AND category IN ('" + strings.Join(cat, "','") + "')"
	}
	if sort := q.Get("sort"); sort != "" {
		sql += " ORDER BY " + regexp.MustCompile(`[^a-z_]`).ReplaceAllString(sort, "")
	}
	sql += fmt.Sprintf(" LIMIT %s", q.Get("limit"))
	rows, err := s.db.QueryContext(ctx, sql, args...)
	// …
}
```

1. Trouver **toutes** les failles et bugs (il y en a au moins cinq), et pour chacune écrire la requête
   HTTP qui l'exploite ou la déclenche.
2. L'attaquant peut-il lire les objets d'**un autre** propriétaire malgré `owner = $1` ? Comment ?
3. Réécrire la fonction avec les techniques de la leçon, et un test par faille.

---

## Quiz

1. D'où vient toujours une injection SQL ?
2. Pourquoi une valeur passée en paramètre ne peut-elle pas modifier la requête ?
3. Pourquoi échapper les apostrophes soi-même est-il une mauvaise idée ?
4. Que peut-on et ne peut-on pas passer en paramètre ?
5. Comment laisser le client choisir la colonne de tri sans risque ?
6. Comment passer une liste de valeurs à un `IN` en PostgreSQL ?
7. Pourquoi faut-il échapper `%` et `_` dans un paramètre de `LIKE` ?
8. Que fait pgx par défaut avec les requêtes paramétrées ?
9. Quand `PrepareContext` se justifie-t-il ?
10. Pourquoi ajouter `id` à la fin d'un `ORDER BY` ?
