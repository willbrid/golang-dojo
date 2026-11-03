# Niveau 8 — Bases de données

**Objectif :** parler à PostgreSQL depuis Go **correctement** : un pool qu'on comprend, des requêtes
paramétrées, des transactions qui ne fuient pas, un schéma qui évolue sans coupure, et des tests contre
une vraie base.

**Prérequis :** niveaux 1 à 7. En particulier :
- les **erreurs** (N2 L6) et `errors.As` : les erreurs PostgreSQL portent un code qu'on inspecte ;
- les **interfaces côté consommateur** (N2 L4, N5 L3) : le *repository* en est une application directe ;
- `defer` (N2 L7) : chaque `Rows` se ferme, chaque transaction se termine ;
- la **concurrence** (niveau 6) et `context` (N6 L8, N7 L7) : un `*sql.DB` est un pool partagé par
  toutes les goroutines, et chaque requête reçoit le contexte de la requête HTTP qui l'a déclenchée ;
- `embed` (N4 L7) : les migrations SQL sont embarquées dans le binaire.

## Mise en place (une fois pour tout le niveau)

Chaque exemple s'exécute contre un vrai PostgreSQL. Le plus simple est Docker :

```sh
docker run -d --name dojo-pg \
  -e POSTGRES_USER=dojo -e POSTGRES_PASSWORD=dojo -e POSTGRES_DB=dojo \
  -p 127.0.0.1:5432:5432 postgres:17

export DATABASE_URL='postgres://dojo:dojo@127.0.0.1:5432/dojo?sslmode=disable'
```

Pour ouvrir une console SQL : `docker exec -it dojo-pg psql -U dojo`.

Le pilote utilisé est **pgx** (`github.com/jackc/pgx/v5`), le pilote PostgreSQL le plus utilisé en Go :

```sh
go mod init exemple
go get github.com/jackc/pgx/v5
```

> `sslmode=disable` est réservé à une base **locale**. En production, la connexion est chiffrée
> (`sslmode=verify-full`, N7 L11).

> **Ordre des leçons.** Le modèle de `database/sql` d'abord (1), puis la lecture des résultats (2), la
> sécurité des paramètres (3) et les transactions (4) ; le pool (5) se comprend mieux une fois qu'on a
> vu ce qui **retient** une connexion (lignes ouvertes, transactions). Les migrations (6) et le
> *repository* (7) organisent le tout ; la performance (8) vient en dernier, quand on sait mesurer ce
> qu'on fait.

## Leçons

| # | Leçon | Ce que je saurai faire |
|---|---|---|
| 1 | [`database/sql`](lecon-01-database-sql/) | Ouvrir un pool, comprendre pilote et DSN, exécuter une instruction, connaître pgx natif |
| 2 | [Requêtes et scan](lecon-02-requetes-scan/) | Lire une ligne, plusieurs lignes, les `NULL`, écrire un `Scanner`/`Valuer` |
| 3 | [Requêtes préparées](lecon-03-requetes-preparees/) | Éviter l'injection SQL, comprendre la préparation, construire une requête dynamique sûre |
| 4 | [Transactions](lecon-04-transactions/) | `BeginTx`, `Commit`/`Rollback` sans fuite, niveaux d'isolation, réessayer une sérialisation |
| 5 | [Pool de connexions](lecon-05-pool/) | Dimensionner le pool, diagnostiquer une saturation, trouver une fuite de lignes |
| 6 | [Migrations](lecon-06-migrations/) | Versionner un schéma, écrire un migrateur, migrer sans coupure |
| 7 | [Repository pattern](lecon-07-repository/) | Abstraire le stockage sans sur-abstraire, tester contre une vraie base |
| 8 | [Performance et ORM](lecon-08-performance-orm/) | Lire `EXPLAIN`, éviter le N+1, choisir entre SQL, générateur et ORM |

## Évaluation de fin de niveau

- **[Projet 4 — API avec PostgreSQL](../projets/projet-04-api-postgresql/)** : le stockage du projet 3
  remplacé par PostgreSQL, **sans toucher** aux handlers.
- **[Projet 6 — Service avec authentification](../projets/projet-06-service-auth/)** : un service
  d'identité complet (comptes, sessions, jetons, réinitialisation de mot de passe).

## Règle du niveau

**Toute requête reçoit un `context`, tout `Rows` est fermé, toute transaction se termine.** Une
requête sans contexte ne peut pas être annulée ; un `Rows` oublié ou une transaction ouverte retient
une connexion du pool, et le service finit par s'arrêter, sans erreur, faute de connexion libre.
