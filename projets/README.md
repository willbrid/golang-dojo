# Les projets

Neuf projets de difficulté croissante. Chacun suit le même déroulé :

1. **Spécifications** — ce que le programme doit faire.
2. **Contraintes** — ce qui est imposé, y compris les difficultés glissées exprès.
3. **Architecture suggérée** — une piste, pas une obligation : je conçois d'abord ma solution.
4. **Tests à écrire** — la liste minimale, à compléter.
5. **Revue de code** — verdict sur les quatre axes de qualité.

| # | Projet | Après le niveau | Compétences visées |
|---|---|---|---|
| 1 | [CLI simple](projet-01-cli-simple/) | 1 | Syntaxe, slices, maps, structs, erreurs, séparation calcul/E-S |
| 2 | [Gestionnaire de tâches CLI](projet-02-gestionnaire-taches/) | 2 | Packages, persistance, erreurs enveloppées, conception d'API |
| 3 | [API REST avec tests](projet-03-api-rest/) | 3 et 5 | `net/http`, JSON, table-driven tests, middleware |
| 4 | [API avec PostgreSQL](projet-04-api-postgresql/) | 6 | `database/sql`, transactions, migrations, repository |
| 5 | [Serveur concurrent](projet-05-serveur-concurrent/) | 4 | Goroutines, channels, `context`, arrêt gracieux |
| 6 | [Service avec authentification](projet-06-service-auth/) | 5 | Sessions/JWT, hachage, sécurité, gestion d'erreurs HTTP |
| 7 | [Workers et queues](projet-07-workers-queues/) | 7 | Worker pool, idempotence, retries, backpressure |
| 8 | [Microservice professionnel](projet-08-microservice/) | 9 | Docker, CI/CD, `slog`, métriques, OpenTelemetry |
| — | [Projet final](projet-final/) | 10 | Démonstration d'un niveau senior/expert |

**Mon code de projet va dans `mes-solutions/projets/`** — ignoré par git, comme le reste
de mon code d'apprentissage. Les spécifications, elles, restent versionnées ici.
