# Projet 4 — `bookshelf` sur PostgreSQL

**À réaliser après le niveau 8**, à partir de ton [projet 3](../projet-03-api-rest/). Durée indicative :
12 à 20 heures. Écrire le code dans `mes-solutions/projets/projet-04/` (copie du projet 3).

---

## Objectif pédagogique

Remplacer le stockage en mémoire du projet 3 par PostgreSQL **sans modifier** ni le domaine, ni les
handlers, ni leurs tests. C'est le test grandeur nature de l'architecture du projet 3 : si l'interface du
stockage était bien conçue, ce projet ne touche que le paquet `store`, `main` et la configuration. Tout le
niveau 8 sert ici : pool, requêtes et scan, transactions et isolation, migrations, suite de contrat,
index et `EXPLAIN`.

---

## Spécifications

L'API publique est **exactement** celle du projet 3 (mêmes routes, mêmes statuts, mêmes corps) : un client
du projet 3 ne doit voir aucune différence. Exigences nouvelles :

| # | Exigence |
|---|---|
| S1 | Implémentation PostgreSQL de **toutes** les interfaces de stockage du projet 3 ; l'implémentation mémoire est conservée. Le choix se fait dans `main` par configuration (`STORE=memory` ou `STORE=postgres`). |
| S2 | La **suite de contrat** du projet 3 (`storetest`) passe, **inchangée**, contre les deux implémentations. Si elle doit changer, c'est qu'elle était incomplète : documenter pourquoi. |
| S3 | Schéma créé par des **migrations** SQL numérotées, embarquées (`embed`), appliquées au démarrage par un migrateur (le tien, leçon 6, ou un outil établi), avec verrou consultatif et `lock_timeout`. |
| S4 | Contraintes **en base** pour les règles d'intégrité : unicité (ISBN par utilisateur, un avis par utilisateur et par livre, email), clés étrangères, `CHECK` (note de 1 à 5, année). Une violation remonte en erreur **du domaine** (409, 422), jamais en 500. |
| S5 | Pagination par curseur `(created_at, id)` servie par un index ; `EXPLAIN (ANALYZE, BUFFERS)` de la liste paginée joint au rendu (première page et page 500 sur 1 million de livres). |
| S6 | Recherche `q` dans les titres insensible à la casse **et aux accents** (« Etranger » trouve « L'Étranger ») ; justifier l'index choisi (`unaccent`, `pg_trgm`, colonne normalisée…), mesurer avec `EXPLAIN`. |
| S7 | Liste des avis d'une page de livres **sans N+1** ; un test vérifie le nombre de requêtes. |
| S8 | Suppression d'un livre et de ses avis : atomique, et cohérente avec la décision D3 du projet 3. |
| S9 | Pool configuré (`DB_MAX_CONNS`, `DB_MAX_IDLE_TIME`…) ; `db.Stats()` exposé sur `GET /debug/db` (protégé). `/readyz` vérifie la base avec un délai court. |
| S10 | Démarrage robuste : la base peut démarrer **après** le service (Docker Compose) ; le service réessaie avec attente croissante, sans jamais journaliser le mot de passe. |
| S11 | Toute requête SQL reçoit le contexte de la requête HTTP ; une requête annulée par le client libère sa connexion (vérifié par un test). |
| S12 | Un `docker-compose.yml` (service + PostgreSQL 17) et un `Makefile` (ou script) : `up`, `migrate`, `test`, `test-integration`. |

---

## Contraintes

1. Pilote pgx (`database/sql` + `pgx/stdlib`, ou `pgxpool` : choisir et justifier). Pas d'ORM.
2. Aucune modification des paquets `domain` et `httpapi` du projet 3, sauf correction de bug documentée.
3. Les tests d'intégration lisent `DATABASE_URL` et se **sautent** sans elle ; isolation par schéma (ou
   base modèle) ; parallèles ; aucun déchet après exécution (vérifié).
4. Toutes les erreurs PostgreSQL sont traduites dans `store` ; aucun autre paquet n'importe `pgconn`.
5. `go test -race ./...` passe, avec et sans base. `go vet`, `go fix -diff`, `golangci-lint` silencieux.
6. Un test vérifie qu'aucune connexion ne fuit (`InUse == 0` en fin de test, leçon 5).

---

## Difficultés glissées volontairement

- **D1.** Le doublon d'ISBN (D1 du projet 3) : en mémoire, tu l'avais réglé par un verrou Go. Que devient
  ce verrou avec deux **instances** du service devant la même base ? Quelle garantie la base offre-t-elle,
  et que reste-t-il à faire dans le code ?
- **D2.** Le curseur du projet 3 encodait peut-être une date en millisecondes. PostgreSQL stocke des
  microsecondes, Go des nanosecondes : deux livres peuvent-ils se retrouver sautés ou dupliqués d'une
  page à l'autre ? Écrire le test qui le prouve, puis le corriger.
- **D3.** `created_at` avec `DEFAULT now()` : deux livres insérés dans la même transaction ont la même
  date. Le tri reste-t-il total ? Et avec l'horloge injectée du projet 3 : qui fixe la date, Go ou la
  base ? Décider, documenter, rester cohérent avec l'implémentation mémoire.
- **D4.** Le `PATCH` partiel (D4 du projet 3) traduit en SQL : un `UPDATE` qui ne modifie que les champs
  présents, **sans** concaténer de noms de colonnes non contrôlés (leçon 3).
- **D5.** L'avis unique par utilisateur : deux requêtes simultanées. La contrainte `UNIQUE` suffit-elle ?
  Quelle erreur, traduite comment ?
- **D6.** La suppression d'un utilisateur : ses sessions, ses livres, les avis des **autres** sur ses
  livres. `ON DELETE CASCADE` partout, ou une décision par relation ? Et la suppression de 10 000 livres
  d'un coup tient-elle des verrous trop longtemps ?
- **D7.** Une migration ajoute la recherche sans accents (S6) sur une table d'un million de livres déjà
  en production : quelles instructions, dans quel ordre, avec quels verrous ? (Leçon 6.)
- **D8.** L'implémentation mémoire et la vraie divergent sur un détail (tri des chaînes accentuées,
  précision des dates, `NULL` contre chaîne vide). Le contrat l'attrape-t-il ? Sinon, l'enrichir.

---

## Architecture suggérée

**À ne lire qu'après avoir dessiné ta propre conception.**

```
projet-04/
├── cmd/bookshelf/main.go         STORE=memory|postgres, OpenDB avec réessais, Migrate, Run
├── migrations/
│   ├── 0001_users_sessions.sql
│   ├── 0002_books.sql
│   ├── 0003_reviews.sql
│   └── 0004_search.sql
├── migrations.go                 //go:embed migrations/*.sql
└── internal/
    ├── domain/                   inchangé
    ├── httpapi/                  inchangé
    ├── store/
    │   ├── store.go              interfaces (inchangées)
    │   ├── memory/               implémentation du projet 3
    │   ├── postgres/
    │   │   ├── db.go             DBTX, WithTx, traduction des erreurs (23505 → ErrConflict…)
    │   │   ├── books.go, reviews.go, users.go, sessions.go
    │   │   └── postgres_test.go  storetest.Run + tests propres à PostgreSQL
    │   └── storetest/            suite de contrat (inchangée ou enrichie, voir S2/D8)
    └── migrate/                  si tu écris le tien
```

L'idée directrice : `main` est le **seul** endroit qui sait qu'il existe deux implémentations.

---

## Tests à écrire

- Contrat : `storetest.Run` contre la mémoire et contre PostgreSQL (schéma par test, parallèle).
- Concurrence contre PostgreSQL : D1 et D5 avec 50 requêtes simultanées, sous `-race`.
- Pagination : 1 000 livres de même `created_at`, parcours complet par curseur, aucun doublon ni oubli (D2,
  D3).
- Migrations : base vide → dernière version ; relance sans effet ; migration modifiée refusée.
- Nombre de requêtes de la liste paginée avec avis (S7).
- Annulation : une requête HTTP annulée pendant un `pg_sleep` libère sa connexion (S11).
- Fuites : `InUse == 0` à la fin de chaque test d'intégration.
- Bout en bout : les tests `httptest` du projet 3, exécutés une fois avec `STORE=postgres`.

---

## Ce qui sera évalué en revue de code

| Axe | Question posée |
|---|---|
| **Ça fonctionne** | Les 12 spécifications, le contrat inchangé, l'API identique au projet 3. |
| **C'est correct** | Les 8 difficultés traitées ou documentées ; intégrité garantie par la base ; aucune mise à jour perdue ; aucune fuite de connexion. |
| **C'est idiomatique** | Domaine et handlers intacts, `store` seul à connaître SQL, `DBTX`, contextes partout, erreurs traduites à la frontière. |
| **Prêt pour la production** | Migrations sûres, pool réglé et observable, démarrage robuste, index justifiés par `EXPLAIN`, aucun secret dans les journaux. |

Rends ton code même incomplet : une conception discutée vaut mieux qu'un projet abandonné.
