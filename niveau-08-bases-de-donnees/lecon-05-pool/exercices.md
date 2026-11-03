# Leçon 5 — Exercices

Code dans `mes-solutions/niveau-08/lecon-05/`.

## Exercices faciles

**E1 — Le coût de la réserve.**
Reproduire la mesure du cours : 50 rafales de 20 requêtes `SELECT 1`, `MaxOpenConns(20)`, avec
`MaxIdleConns` à 2, 5, 10 et 20. Tracer durée et `MaxIdleClosed`. Pendant la version à 2, observer
`pg_stat_activity` en boucle (`\watch 0.2` dans `psql`) : que voit-on ?

**E2 — Les quatre fuites.**
Écrire quatre fonctions, chacune avec une fuite différente : `Rows` lu partiellement, `QueryRow` sans
`Scan`, transaction sans fin, `db.Conn` sans `Close`. Pour chacune, avec `MaxOpenConns(3)` et
`context.Background()`, combien d'appels avant que le pool soit bloqué ? Que montre `pg_stat_activity`
(`state`) pour chaque type de fuite ?

**E3 — Le filet de l'annulation.**
Reprendre la fonction qui fuit dans un **handler HTTP** (N7 L1) qui utilise `r.Context()`, avec
`MaxOpenConns(2)`. Envoyer 100 requêtes séquentielles avec `httptest` : combien échouent ? Puis 100
requêtes **concurrentes** : combien échouent maintenant, et pourquoi la différence ?

**E4 — Trop de connexions.**
Sans `SetMaxOpenConns`, lancer 300 goroutines qui exécutent `SELECT pg_sleep(0.5)`. Compter les erreurs
et lire leur message. Recommencer avec `SetMaxOpenConns(20)` : combien d'erreurs, et combien de temps
dure l'ensemble ? Quelle version préférer si chaque goroutine représente une requête HTTP avec un délai
de 2 s ?

**E5 — Dimensionner sur le papier.**
Un service tourne sur 6 instances ; PostgreSQL a `max_connections = 200` ; on réserve 20 connexions
pour l'administration et les migrations. Chaque requête HTTP fait en moyenne 3 requêtes SQL de 2 ms ;
le service reçoit au plus 1 500 requêtes HTTP/s au total. Combien de connexions sont occupées en
moyenne ? Quelle valeur de `MaxOpenConns` choisir, et avec quelle marge ? Que se passe-t-il le jour où
l'on passe à 12 instances sans rien changer d'autre ?

---

## Exercice intermédiaire — Un test qui attrape les fuites

1. Écrire un *helper* de test `func CheckNoLeak(t *testing.T, db *sql.DB)` qui enregistre, avec
   `t.Cleanup`, une vérification de fin de test : après une attente brève (les fermetures peuvent être
   asynchrones), `db.Stats().InUse` doit valoir 0. En cas d'échec, le message liste aussi les requêtes
   des connexions `idle in transaction` ou `active` de **ce** programme (`pg_stat_activity` filtré par
   `application_name`, que l'on fixe dans le DSN).
2. L'appliquer aux quatre fonctions d'E2 : les quatre tests doivent **échouer**, avec un message qui
   aide à trouver la fuite.
3. Pourquoi faut-il que la vérification se fasse sur un pool **propre au test**, et pas sur un pool
   partagé entre tests parallèles (`t.Parallel`) ?

---

## Défi — Un tableau de bord du pool

1. Écrire un `PoolMonitor` qui, toutes les secondes (`time.Ticker`, arrêt par `context`), lit
   `db.Stats()` et calcule sur la dernière seconde : requêtes ayant attendu, attente moyenne, taux
   d'occupation (`InUse / MaxOpenConnections`).
2. L'exposer en JSON sur `GET /debug/pool` (N7 L1).
3. Écrire un générateur de charge qui fait varier le débit (rampe de 10 à 2 000 requêtes/s, requêtes
   `pg_sleep(0.005)`), et trouver expérimentalement le point de saturation pour `MaxOpenConns` = 5,
   10, 20. La relation *débit × durée* du cours prédit-elle ces points ?
4. Ajouter une alerte (une ligne de journal) quand l'attente moyenne dépasse 10 ms pendant 3 secondes
   consécutives. Pourquoi « pendant 3 secondes » et pas « dès que » ?

---

## Quiz

1. Que fait le pool quand toutes les connexions sont utilisées et que le plafond est atteint ?
2. Quelles sont les valeurs par défaut de `MaxOpenConns` et `MaxIdleConns`, et pourquoi posent-elles
   problème ?
3. Pourquoi régler `MaxIdleConns` égal à `MaxOpenConns` ?
4. À quoi sert `ConnMaxLifetime` ?
5. Comment calculer le nombre total de connexions qu'un service peut ouvrir ?
6. Que signifient `WaitCount` et `WaitDuration` qui augmentent ?
7. Comment distinguer une fuite d'une base saturée ?
8. Quelles quatre erreurs de code retiennent une connexion ?
9. Pourquoi une fuite de `Rows` dans un handler HTTP est-elle moins grave que dans une tâche de fond ?
10. Pourquoi un pool plus petit que la demande n'est-il pas forcément un problème ?
