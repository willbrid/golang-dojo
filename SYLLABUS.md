# Plan détaillé de la formation

Version de Go ciblée : **1.27**. Chaque leçon suit le même format :
*Objectifs · Explication · Exemple · Explication du code · Erreurs fréquentes ·
Bonnes pratiques · Exercices faciles · Exercice intermédiaire · Défi · Quiz · À retenir.*

Légende : ✅ rédigée · 🔜 planifiée

---

## Niveau 1 — Fondamentaux

Objectif : écrire seul un programme Go correct, sans copier-coller.

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | [Environnement et outils](niveau-01-fondamentaux/lecon-01-environnement/) | modules, packages, `go run/build/fmt/vet`, `go.mod`, binaire autonome | ✅ |
| 2 | [Variables, constantes, types](niveau-01-fondamentaux/lecon-02-variables-types/) | `var` vs `:=`, zéro-valeur, entiers, flottants, `const`, `iota`, conversions | ✅ |
| 3 | [Strings, runes, bytes](niveau-01-fondamentaux/lecon-03-strings-runes/) | UTF-8, `byte` vs `rune`, immutabilité, `strings`, `strings.Builder`, formatage | ✅ |
| 4 | [Conditions et boucles](niveau-01-fondamentaux/lecon-04-conditions-boucles/) | `if` avec instruction, `switch`, `for` (4 formes), `range`, `break`/`continue`, labels | ✅ |
| 5 | [Fonctions](niveau-01-fondamentaux/lecon-05-fonctions/) | retours multiples et nommés, variadiques, portée, closures, fonctions valeurs | ✅ |
| 6 | [Tableaux et slices](niveau-01-fondamentaux/lecon-06-tableaux-slices/) | tableau vs slice, `len`/`cap`, `append`, partage de tableau sous-jacent, `copy` | ✅ |
| 7 | [Maps](niveau-01-fondamentaux/lecon-07-maps/) | déclaration, idiome `v, ok`, suppression, itération non déterministe, sets | ✅ |
| 8 | [Erreurs simples](niveau-01-fondamentaux/lecon-08-erreurs-simples/) | `error` est une valeur, `errors.New`, `fmt.Errorf`, retour `(T, error)` | ✅ |
| 9 | [Pointeurs](niveau-01-fondamentaux/lecon-09-pointeurs/) | `&`/`*`, `nil`, passage par valeur, `new`, quand un pointeur est justifié | ✅ |
| 10 | [Structs et méthodes](niveau-01-fondamentaux/lecon-10-structs-methodes/) | littéraux, champs, récepteur valeur vs pointeur, `String()`, comparabilité | ✅ |
| 11 | [Interfaces et composition](niveau-01-fondamentaux/lecon-11-interfaces-composition/) | satisfaction implicite, petites interfaces, embedding, interface `nil` | ✅ |

➡️ **[Projet 1 — CLI simple](projets/projet-01-cli-simple/)**

---

## Niveau 2 — Go idiomatique

Objectif : écrire du code qu'un relecteur Go expérimenté accepterait sans réserve.

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | Packages et visibilité | découpage, cycles d'import, `internal/`, nommage | 🔜 |
| 2 | Modules et `go.mod` | versionnement sémantique, `go get`, `go mod tidy`, MVS, `replace` | 🔜 |
| 3 | Organisation d'un projet | `cmd/`, `internal/`, `pkg/` — et pourquoi `pkg/` est discutable | 🔜 |
| 4 | Erreurs idiomatiques | l'erreur est une valeur, sentinelles, erreurs typées | 🔜 |
| 5 | Enveloppement d'erreurs | `fmt.Errorf` + `%w`, `errors.Is`, `errors.As`, `errors.Join` | 🔜 |
| 6 | `defer` en profondeur | ordre LIFO, évaluation des arguments, `defer` et retours nommés | 🔜 |
| 7 | `panic` / `recover` | pourquoi presque jamais, les rares cas légitimes | 🔜 |
| 8 | Interfaces avancées | assertions, `switch` de type, interface nil ≠ pointeur nil, `any` | 🔜 |
| 9 | Génériques | paramètres de type, contraintes, inférence, **méthodes génériques (1.27)** | 🔜 |
| 10 | Stdlib moderne | `slices`, `maps`, `cmp`, itérateurs et `range` sur fonction | 🔜 |
| 11 | Code idiomatique | conception d'API, nommage, `Effective Go`, revue de code | 🔜 |

➡️ **[Projet 2 — Gestionnaire de tâches CLI](projets/projet-02-gestionnaire-taches/)**

---

## Niveau 3 — Tests et qualité

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | Le paquet `testing` | premier test, `t.Run`, `t.Helper`, `t.Cleanup`, échec vs fatal | 🔜 |
| 2 | Table-driven tests | le patron canonique Go, sous-tests, cas limites | 🔜 |
| 3 | Tester les dépendances | interfaces, fakes, mocks — et pourquoi Go préfère les fakes | 🔜 |
| 4 | Couverture et golden files | `-cover`, ce que la couverture ne dit pas, fichiers de référence | 🔜 |
| 5 | Exemples exécutables | `func Example…`, documentation testée | 🔜 |
| 6 | Benchmarks | `testing.B`, `b.Loop` , mesures fiables, `benchstat` | 🔜 |
| 7 | Fuzzing | `func Fuzz…`, corpus, bugs trouvés par la machine | 🔜 |
| 8 | Outillage qualité | `go vet`, `staticcheck`, `golangci-lint`, `govulncheck` | 🔜 |

➡️ **[Projet 3 — API REST avec tests](projets/projet-03-api-rest/)**

---

## Niveau 4 — Concurrence

Le cœur de Go. Beaucoup d'exercices, volontairement.

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | Le modèle de concurrence | goroutines, ordonnanceur M:N, `GOMAXPROCS`, concurrence ≠ parallélisme | 🔜 |
| 2 | Channels non bufferisés | le rendez-vous, direction des channels | 🔜 |
| 3 | Channels bufferisés | capacité, fermeture, `range`, qui ferme un channel | 🔜 |
| 4 | `select` | multiplexage, `default`, `time.After`, boucles de service | 🔜 |
| 5 | `sync.WaitGroup` | attendre des goroutines, `WaitGroup.Go` , pièges classiques | 🔜 |
| 6 | `Mutex` et `RWMutex` | protéger un état, granularité, `sync.Map` | 🔜 |
| 7 | `atomic`, `Once`, `Pool` | opérations atomiques, initialisation unique, réutilisation | 🔜 |
| 8 | `context` | annulation, `WithTimeout`, propagation, valeurs (avec parcimonie) | 🔜 |
| 9 | Patrons | worker pool, pipeline, fan-in / fan-out, `errgroup` | 🔜 |
| 10 | Pathologies | data races, deadlocks, livelocks, famine, fuites de goroutines, `-race`, profil `goroutineleak` (1.27) | 🔜 |
| 11 | Backpressure | dimensionner les files, rejeter proprement, charge et latence | 🔜 |

➡️ **[Projet 5 — Serveur concurrent](projets/projet-05-serveur-concurrent/)**

---

## Niveau 5 — Réseau et backend

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | `net/http` — le serveur | `Handler`, `HandlerFunc`, `ServeMux` et son routage à motifs | 🔜 |
| 2 | Requêtes et réponses | statuts, en-têtes, corps, streaming, `http.Error` | 🔜 |
| 3 | JSON | `encoding/json`, **`encoding/json/v2` (1.27)**, `jsontext`, tags, validation | 🔜 |
| 4 | Middleware | composition, chaînage, journalisation, `recover`, requête ID | 🔜 |
| 5 | Conception d'API REST | ressources, verbes, statuts, pagination, versionnement | 🔜 |
| 6 | Client HTTP | `http.Client`, `Transport`, pools, timeouts, retries | 🔜 |
| 7 | `context` de bout en bout | annulation client → serveur → base, arrêt gracieux | 🔜 |
| 8 | Authentification | sessions, JWT, clés d'API, OAuth2, hachage de mots de passe | 🔜 |
| 9 | Erreurs HTTP | modèle d'erreur, mapping domaine → statut, ne pas fuiter d'information | 🔜 |
| 10 | TCP et UDP | `net.Listen`, protocoles binaires, framing, DNS | 🔜 |
| 11 | TLS et WebSockets | certificats, mTLS, mise à niveau de connexion | 🔜 |
| 12 | RPC et gRPC | protobuf, streaming, intercepteurs, quand gRPC plutôt que REST | 🔜 |

➡️ **[Projet 6 — Service backend avec authentification](projets/projet-06-service-auth/)**

---

## Niveau 6 — Bases de données

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | `database/sql` | modèle, drivers, `pgx` avec PostgreSQL | 🔜 |
| 2 | Requêtes et scan | `QueryRow`, `Query`, `NULL`, types personnalisés | 🔜 |
| 3 | Requêtes préparées | injection SQL, paramètres, réutilisation | 🔜 |
| 4 | Transactions | `Begin`/`Commit`/`Rollback`, niveaux d'isolation, `defer` et rollback | 🔜 |
| 5 | Pool de connexions | `SetMaxOpenConns`, saturation, fuites de lignes | 🔜 |
| 6 | Migrations | outillage, migrations réversibles, déploiement sans coupure | 🔜 |
| 7 | Repository pattern | abstraire sans sur-abstraire, tests avec `testcontainers` | 🔜 |
| 8 | Performance et ORM | index, N+1, `EXPLAIN`, avantages et coûts d'un ORM | 🔜 |

➡️ **[Projet 4 — API avec PostgreSQL](projets/projet-04-api-postgresql/)**

---

## Niveau 7 — Architecture

Avec, à chaque leçon, la question honnête : **quand ne PAS faire ça**.

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | Séparation des responsabilités | couches, dépendances dirigées vers le domaine | 🔜 |
| 2 | handler / service / repository | le découpage par défaut d'un backend Go | 🔜 |
| 3 | Injection de dépendances | sans framework, par constructeur ; `wire` en option | 🔜 |
| 4 | Interfaces bien conçues | définies côté consommateur, petites, découvertes tardivement | 🔜 |
| 5 | Hexagonale et clean | ports/adaptateurs, coût réel, sur-ingénierie | 🔜 |
| 6 | Domain-Driven Design | langage ubiquitaire, agrégats, contextes bornés | 🔜 |
| 7 | Monolithe modulaire vs microservices | le monolithe modulaire comme défaut raisonnable | 🔜 |
| 8 | Événementiel et messaging | queues, pub/sub, outbox, ordre et livraison | 🔜 |
| 9 | Résilience | idempotence, retries avec jitter, circuit breakers, timeouts | 🔜 |
| 10 | Observabilité | logs / métriques / traces comme décision d'architecture | 🔜 |

➡️ **[Projet 7 — Système avec workers et queues](projets/projet-07-workers-queues/)**

---

## Niveau 8 — Performance

Règle du niveau : **mesurer avant d'optimiser**, toujours.

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | Mesurer | benchmarks fiables, bruit, `benchstat`, micro vs macro | 🔜 |
| 2 | Pile, tas, escape analysis | `-gcflags=-m`, ce qui s'échappe et pourquoi | 🔜 |
| 3 | Le ramasse-miettes | tri-color, `GOGC`, `GOMEMLIMIT`, pauses | 🔜 |
| 4 | pprof — CPU | collecte, flamegraphs, lecture d'un profil | 🔜 |
| 5 | pprof — mémoire, block, mutex | allocations, contention, profil `goroutineleak` (1.27) | 🔜 |
| 6 | Slices et maps | préallocation, coût réel, alternatives | 🔜 |
| 7 | Latence et débit | percentiles, loi de Little, files d'attente | 🔜 |
| 8 | Optimisations qui valent le coup | pooling, réduction d'allocations, ce qu'il ne faut pas faire | 🔜 |

---

## Niveau 9 — Outils et production

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | Git et projets Go | branches, tags de version de module, conventions de commit | 🔜 |
| 2 | Build de production | cross-compilation, `-ldflags`, builds reproductibles | 🔜 |
| 3 | Docker | multi-stage, images distroless, binaire statique | 🔜 |
| 4 | CI/CD | GitHub Actions : test, lint, race, build, release | 🔜 |
| 5 | Configuration | variables d'environnement, fichiers, précédence, validation au démarrage | 🔜 |
| 6 | Logs structurés | `log/slog`, niveaux, contexte, ce qu'on ne journalise jamais | 🔜 |
| 7 | Métriques et traces | Prometheus, OpenTelemetry, corrélation | 🔜 |
| 8 | Sécurité et secrets | `govulncheck`, gestion de secrets, surface d'attaque | 🔜 |
| 9 | Production | déploiement, santé, monitoring, debugging à chaud | 🔜 |

➡️ **[Projet 8 — Microservice Go professionnel](projets/projet-08-microservice/)**

---

## Niveau 10 — Expert

Problèmes de niveau senior/expert, sans guidage pas à pas.

| # | Module | Contenu | État |
|---|---|---|---|
| 1 | Systèmes concurrents | conception, invariants, preuves informelles | 🔜 |
| 2 | Serveurs haute performance | budget d'allocation, zero-copy, epoll et le runtime | 🔜 |
| 3 | Systèmes distribués | cohérence, horloges, consensus, partitions | 🔜 |
| 4 | Debugging complexe | `delve`, détecteur de races, core dumps, traces d'exécution | 🔜 |
| 5 | Bugs mémoire et races difficiles | reproduction, réduction, correction | 🔜 |
| 6 | Conception d'API durable | compatibilité, dépréciation, versions majeures | 🔜 |
| 7 | Résilience et scalabilité | dégradation gracieuse, capacité, tests de charge | 🔜 |
| 8 | Décisions d'architecture | ADR, arbitrages, revue de code senior | 🔜 |

➡️ **[Projet final](projets/projet-final/)**
