# Les projets

Neuf projets de difficulté croissante. Chacun suit le même déroulé :

1. **Spécifications** — ce que le programme doit faire.
2. **Contraintes** — ce qui est imposé, y compris les difficultés glissées exprès.
3. **Architecture suggérée** — une piste, pas une obligation : concevoir d'abord sa propre solution.
4. **Tests à écrire** — la liste minimale, à compléter.
5. **Revue de code** — verdict sur les quatre axes de qualité.

| Ordre | # | Projet | Après le niveau | Compétences visées |
|---|---|---|---|---|
| 1er | 1 | [CLI simple](projet-01-cli-simple/) | 2 | Syntaxe, slices, maps, structs, erreurs, découpage en fichiers |
| 2e | 2 | [Gestionnaire de tâches CLI](projet-02-gestionnaire-taches/) | 4 | Fichiers, JSON, temps, erreurs enveloppées, conception d'API |
| 3e | 5 | [Serveur concurrent](projet-05-serveur-concurrent/) | 6 | Goroutines, channels, `context`, arrêt gracieux |
| 4e | 3 | [API REST avec tests](projet-03-api-rest/) | 7 | `net/http`, JSON, table-driven tests, middleware |
| 5e | 4 | [API avec PostgreSQL](projet-04-api-postgresql/) | 8 | `database/sql`, transactions, migrations, repository |
| 6e | 6 | [Service avec authentification](projet-06-service-auth/) | 8 | Sessions/JWT, hachage, sécurité, erreurs HTTP |
| 7e | 7 | [Workers et queues](projet-07-workers-queues/) | 10 | Worker pool, idempotence, retries, backpressure |
| 8e | 8 | [Microservice professionnel](projet-08-microservice/) | 12 | Docker, CI/CD, `slog`, métriques, OpenTelemetry |
| 9e | — | [Projet final](projet-final/) | 13 | Démonstration d'un niveau senior/expert |

La **numérotation** est celle du programme initial ; la colonne **Ordre** donne la
chronologie réelle, qui suit les prérequis techniques.

**Le code des projets va dans `mes-solutions/projets/`** — ignoré par git, comme le reste
du code d'entraînement. Les spécifications, elles, restent versionnées ici.
