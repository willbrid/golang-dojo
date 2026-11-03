# Leçon 8 — Exercices

Code dans `mes-solutions/niveau-08/lecon-08/`. Créer d'abord la table d'un million de livres du cours :

```sql
CREATE TABLE l8_books (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, title text NOT NULL,
                       author text NOT NULL, created_at timestamptz NOT NULL);
INSERT INTO l8_books (title, author, created_at)
SELECT 'titre ' || g, 'auteur ' || (g % 10000), timestamptz '2020-01-01' + g * interval '1 minute'
FROM generate_series(1, 1000000) g;
ANALYZE l8_books;
```

## Exercices faciles

**E1 — Le mauvais index.**
Avec seulement `CREATE INDEX ON l8_books (created_at)`, lancer `EXPLAIN (ANALYZE, BUFFERS)` sur la
requête du cours (un auteur, 20 plus récents). Quel plan PostgreSQL choisit-il ? Combien de lignes
parcourt-il ? Essayer ensuite avec `'auteur 9999'` puis avec un auteur **inexistant** : que devient ce
plan, et pourquoi est-il catastrophique dans le dernier cas ?

**E2 — L'ordre des colonnes.**
Comparer `(author, created_at)` et `(created_at, author)` pour la requête du cours. Puis pour
`WHERE created_at > now() - interval '1 day'`. Énoncer la règle.

**E3 — Un test qui compte.**
Avec le `counter` de l'exemple (ou un *middleware* qui compte par requête HTTP), écrire un test qui
échoue si le chargement d'une page de 50 livres fait plus de 2 requêtes. Le faire échouer avec
`loadNPlusOne`, passer avec `loadBatched`.

**E4 — `OFFSET` contre curseur.**
Paginer les livres d'un auteur très prolifique (ajouter 200 000 livres à `'auteur X'`), 20 par page, par
`ORDER BY created_at DESC, id DESC` : (a) avec `LIMIT 20 OFFSET n`, (b) avec un curseur
`WHERE (created_at, id) < ($1, $2)`. Mesurer pages 1, 100 et 10 000. Expliquer la différence avec
`EXPLAIN`. Quel index faut-il pour (b) ?

**E5 — Le coût du `fsync`.**
Reproduire les 10 000 `INSERT` hors transaction, puis recommencer après `SET synchronous_commit = off`
dans la session. Que gagne-t-on, et que **risque**-t-on exactement en cas de panne du serveur ? Pour
quel type de données serait-ce acceptable ?

---

## Exercice intermédiaire — Un import en masse

On reçoit des fichiers CSV de catalogue (`isbn,title,author,year`) de 100 000 lignes, à importer
plusieurs fois par jour dans une table `catalog(isbn PRIMARY KEY, …)`.

1. Écrire l'import avec `CopyFrom` (pgx natif) dans une table **temporaire**, puis un
   `INSERT INTO catalog SELECT … FROM tmp ON CONFLICT (isbn) DO UPDATE SET …` : l'import est
   **rejouable** (un même fichier importé deux fois donne le même résultat).
2. Le tout dans une transaction : un fichier invalide ne laisse **aucune** trace.
3. Une ligne invalide (année non numérique, ISBN de mauvaise longueur) : la rejeter **avant** `COPY`
   avec son numéro de ligne, et choisir : tout refuser, ou importer le reste et rendre un rapport.
4. Mesurer la durée pour 100 000 lignes, et comparer avec `unnest` par lots de 5 000.
5. Pourquoi une table temporaire ici, alors que la leçon 1 mettait en garde contre elles ?

---

## Défi — Les trois derniers avis de chaque livre

Pour une page de 20 livres, rendre chaque livre avec ses **3 derniers avis** (une table de 2 millions
d'avis, 100 par livre en moyenne).

1. Trois versions : (a) chargement par lot `= ANY($1)` de **tous** les avis puis filtrage en Go ;
   (b) `ROW_NUMBER() OVER (PARTITION BY book_id ORDER BY created_at DESC)` filtré à `<= 3` ; (c) une
   jointure `LATERAL (SELECT … ORDER BY created_at DESC LIMIT 3)`.
2. Pour chacune : nombre de requêtes, lignes transférées, `EXPLAIN (ANALYZE, BUFFERS)`, durée.
3. Quel index rend (c) très rapide ? Et (b) en profite-t-il autant ?
4. Écrire la même chose avec GORM (`Preload` avec une fonction de personnalisation). Quel SQL produit-il ?
   Peut-il exprimer « 3 par livre » ?

---

## Quiz

1. Qu'ajoutent `ANALYZE` et `BUFFERS` à `EXPLAIN` ?
2. Que signifie *Seq Scan* avec un grand *Rows Removed by Filter* ?
3. Dans quel ordre mettre les colonnes d'un index composite ?
4. Pourquoi ne pas indexer toutes les colonnes ?
5. Qu'est-ce que le problème N+1, et pourquoi coûte-t-il plus cher en production qu'en local ?
6. Deux façons de le corriger ?
7. Pourquoi 10 000 `INSERT` hors transaction sont-ils si lents ?
8. Qu'est-ce que `COPY`, et comment l'utiliser depuis Go ?
9. Qu'est-ce qu'un générateur comme `sqlc` apporte par rapport au SQL écrit à la main ?
10. Un ORM empêche-t-il le N+1 ?
