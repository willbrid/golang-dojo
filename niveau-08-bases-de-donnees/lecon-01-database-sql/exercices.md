# Leçon 1 — Exercices

Code dans `mes-solutions/niveau-08/lecon-01/`. PostgreSQL lancé et `DATABASE_URL` définie (voir le
[README du niveau](../README.md)).

## Exercices faciles

**E1 — Ce que `Open` ne vérifie pas.**
Lancer l'exemple avec, successivement : un mot de passe faux, un port où rien n'écoute, un nom de
pilote faux (`"postgres"`), et **sans** `sslmode`. Pour chaque cas, noter **quelle ligne** échoue
(`Open` ou `Ping`) et le message. Pourquoi `database/sql` ne se connecte-t-il pas dans `Open` ?

**E2 — Le pool vu de l'intérieur.**
Après le `count`, lancer 20 goroutines (`WaitGroup.Go`, N6 L5) qui exécutent chacune
`SELECT pg_sleep(0.2)`, puis afficher `db.Stats()`. Combien de connexions sont ouvertes ? Relancer avec
`db.SetMaxOpenConns(5)` avant les goroutines : combien de temps dure l'ensemble, et que valent
`WaitCount` et `WaitDuration` ?

**E3 — Qui est connecté ?**
Pendant que les 20 goroutines dorment, interroger la base depuis `psql` :
`SELECT pid, state, query FROM pg_stat_activity WHERE datname = 'dojo';`. Retrouver les connexions du
programme. Que montre la colonne `state` pour une connexion inactive du pool ?

**E4 — La table temporaire perdue.**
Reproduire le piège du cours (table temporaire, `Query` laissé ouvert, `Exec` sur une autre connexion).
Corriger de deux façons : avec `db.Conn(ctx)`, puis en fermant le `rows` à temps. Laquelle est robuste
quand d'autres goroutines utilisent le pool en même temps ?

**E5 — `RowsAffected` comme information.**
Écrire `func Rename(ctx context.Context, db *sql.DB, id int64, title string) error` qui fait un
`UPDATE … WHERE id = $2` et rend une sentinelle `ErrNotFound` si **aucune** ligne n'a été touchée.
Tester avec un identifiant existant et un identifiant absent.

---

## Exercice intermédiaire — Démarrage robuste

Écrire `func OpenDB(ctx context.Context, dsn string, log io.Writer) (*sql.DB, error)` qui :

1. refuse un DSN vide, ou dont le schéma n'est pas `postgres://` ou `postgresql://` (`net/url`) ;
2. ouvre le pool et fait `PingContext` ;
3. si la base n'est pas encore prête (cas d'un démarrage simultané base + service dans Docker),
   **réessaie** : 5 tentatives au plus, avec une attente qui double (100 ms, 200 ms, 400 ms…), en
   respectant l'annulation de `ctx` pendant l'attente (`select`, N6 L4) ;
4. journalise chaque tentative **sans** jamais écrire le mot de passe (`url.URL.Redacted()`) ;
5. ferme le pool si elle abandonne (sinon, quoi fuit ?).

Tester : DSN invalide, base arrêtée (`docker stop dojo-pg` puis `docker start` pendant les tentatives),
contexte annulé pendant une attente.

---

## Défi — Une session réservée, proprement

Certaines opérations exigent une session : `SET LOCAL`, verrous consultatifs
(`pg_advisory_lock`), tables temporaires. Écrire :

```go
func WithConn(ctx context.Context, db *sql.DB, fn func(*sql.Conn) error) error
```

qui réserve une connexion, appelle `fn`, et rend **toujours** la connexion au pool, même si `fn`
panique. Puis l'utiliser pour un verrou consultatif : deux goroutines appellent
`SELECT pg_advisory_lock(42)` puis `pg_advisory_unlock(42)` ; mesurer que la seconde attend la première.

Question : si `fn` prend le verrou et **oublie** de le relâcher, la connexion retourne au pool avec le
verrou. Que se passe-t-il pour la prochaine goroutine qui obtient cette connexion ? Et pour celle qui
obtient une **autre** connexion ? Proposer une parade dans `WithConn`.

---

## Quiz

1. Quel est le rôle de `database/sql`, et celui du pilote ?
2. Pourquoi l'import du pilote commence-t-il par `_` ?
3. Que fait `sql.Open` exactement ? Et `PingContext` ?
4. Combien de `*sql.DB` un programme doit-il créer pour une base ?
5. Quelle différence entre `ExecContext`, `QueryRowContext` et `QueryContext` ?
6. Pourquoi préférer les méthodes `…Context` ?
7. Quels marqueurs de paramètre utilise PostgreSQL ?
8. Comment obtenir l'identifiant d'une ligne insérée en PostgreSQL ?
9. Pourquoi une table temporaire peut-elle « disparaître » entre deux appels ?
10. Que vaut `sslmode=disable` en production ?
