# Leçon 6 — Exercices

Code dans `mes-solutions/niveau-08/lecon-06/`.

## Exercices faciles

**E1 — Sans verrou.**
Remplacer `pg_advisory_lock` par une instruction neutre et lancer l'exemple plusieurs fois. Quelles
erreurs obtient-on ? Expliquer pourquoi `CREATE TABLE IF NOT EXISTS` ne suffit pas à protéger deux
exécutions simultanées.

**E2 — Le tri.**
Nommer les fichiers `1_a.sql`, `2_b.sql`, …, `10_j.sql`. Dans quel ordre `fs.Glob` les rend-il ? Dans
quel ordre le migrateur les applique-t-il ? Et un outil qui trierait les noms de fichiers ?

**E3 — L'état d'une base.**
Écrire `func Status(ctx context.Context, db *sql.DB, ms []Migration) ([]string, error)` qui rend, pour
chaque migration, `appliquée le …`, `en attente` ou `MODIFIÉE`, et signale les versions présentes en
base mais **absentes** des fichiers (quelqu'un a déployé une branche qui n'existe plus). En faire une
sous-commande `status` d'un petit binaire `migrate`.

**E4 — La file d'attente des verrous.**
Reproduire la mesure du cours : une transaction longue qui lit la table, un `ALTER TABLE` lancé
pendant ce temps, puis un `SELECT` ordinaire. Observer `pg_locks` et `pg_stat_activity`
(`wait_event_type`) pendant le blocage. Ajouter `SET lock_timeout = '500ms'` à la migration et
recommencer.

**E5 — Mesurer avant de migrer.**
Sur une table d'un million de lignes, mesurer (`\timing` dans `psql`) : `ADD COLUMN` sans défaut, avec
défaut constant, avec défaut `now()` (volatile ou non ?), `ALTER COLUMN … TYPE text` depuis `varchar(50)`,
`ADD CONSTRAINT … CHECK (…)`, puis la même contrainte en deux temps (`NOT VALID` puis
`VALIDATE CONSTRAINT`). Classer ces opérations en « sans risque » et « dangereuses en production ».

---

## Exercice intermédiaire — Migrations sans transaction

`CREATE INDEX CONCURRENTLY` ne peut pas s'exécuter dans une transaction.

1. Ajouter au migrateur un marqueur en première ligne de fichier : `-- migrate:no-transaction`. Une
   telle migration s'exécute hors transaction, puis est enregistrée dans `schema_migrations`.
2. Problème : si l'index échoue au milieu, `CONCURRENTLY` laisse un index **invalide**
   (`pg_index.indisvalid = false`). Que se passe-t-il à la relance si la migration est
   `CREATE INDEX CONCURRENTLY books_title ON …` ? Rendre la migration rejouable sans risque.
3. Dans une migration `no-transaction`, exécuter les instructions **une par une**, séparées par une
   ligne `-- migrate:split`. Pourquoi ne pas envoyer tout le fichier en un seul `ExecContext` ?
   (Essayer `DROP INDEX CONCURRENTLY IF EXISTS x; CREATE INDEX CONCURRENTLY x …` en un seul appel.)
4. Ajouter `SET lock_timeout` automatiquement avant chaque migration (valeur configurable), et un
   réessai du migrateur entier (3 fois, attente croissante) sur `55P03`.
5. Tests : avec `fstest.MapFS`, une migration normale, une `no-transaction`, une qui échoue, une
   relance après échec.

---

## Défi — Renommer une colonne sans coupure

Point de départ : `books(id, name text NOT NULL)` et un service qui lit et écrit `name`. Objectif :
renommer en `title`, sans qu'une seule requête échoue.

1. Écrire les migrations et les **versions successives du code** (au moins quatre), comme décrit dans
   le cours.
2. Le rattrapage : copier `name` dans `title` par lots de 1 000 lignes (`WHERE id > $1 ORDER BY id
   LIMIT 1000`), chaque lot dans sa transaction, en mesurant la durée maximale de verrouillage d'un lot.
3. Simuler le déploiement : deux goroutines qui écrivent et lisent en continu, l'une avec la version
   N, l'autre avec la version N+1, pendant chaque migration. Compter les erreurs : il doit y en avoir
   zéro à chaque étape.
4. Alternative à discuter : une **vue** `books_v2` qui expose `name AS title`. Que permet-elle, et
   qu'est-ce qu'elle ne règle pas ?

---

## Quiz

1. Pourquoi ne jamais modifier une migration déjà appliquée ?
2. Que contient la table `schema_migrations` et à quoi sert chaque colonne ?
3. Pourquoi chaque migration est-elle exécutée dans une transaction, avec son enregistrement ?
4. Que protège le verrou consultatif ?
5. Pourquoi écrire le migrateur contre `fs.FS` ?
6. Pourquoi les migrations `down` sont-elles rarement utilisées en production ?
7. Quelles opérations réécrivent une table en PostgreSQL ?
8. Pourquoi un `ALTER TABLE` qui attend son verrou bloque-t-il l'application ?
9. Que garantit `lock_timeout` ?
10. Décrire les étapes d'un renommage de colonne sans coupure.
