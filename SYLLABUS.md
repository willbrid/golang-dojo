# Plan détaillé de la formation

Version de Go ciblée : **1.27**. Chaque leçon suit le même format :
*Objectifs · Explication · Exemple · Explication du code · Erreurs fréquentes ·
Bonnes pratiques · Exercices faciles · Exercice intermédiaire · Défi · Quiz · À retenir.*

Légende : ✅ rédigée · 🔜 planifiée

---

## Principe d'ordonnancement

Le parcours respecte une règle stricte : **aucune leçon n'utilise un concept qui n'a pas
encore été vu**. Cela impose plusieurs choix qui s'écartent de l'ordre habituel des livres :

| Décision | Raison |
|---|---|
| Les **boucles** précèdent les collections, mais `range` est introduit **type par type** | `range` sur slice a besoin des slices, `range` sur map de l'ordre aléatoire : ce ne sont pas les mêmes pièges |
| **Slices et maps** avant les **chaînes** | `[]byte` et `[]rune` sont des slices ; la normalisation de texte utilise des maps |
| **Pointeurs** avant **structs** et **méthodes** | un récepteur pointeur suppose de savoir ce qu'est un pointeur |
| **Méthodes** avant **interfaces** | la satisfaction d'interface repose sur l'*ensemble de méthodes* |
| Les **erreurs** en deux temps : usage au niveau 1, mécanique au niveau 2 | on doit retourner `(T, error)` dès les premières fonctions, mais `error` **est** une interface — sa mécanique (`%w`, `Is`, `As`) suppose de les avoir vues |
| **Génériques** après les interfaces | une contrainte de type *est* une interface |
| **Tests** avant la concurrence | le détecteur de races s'utilise avec `go test -race` |
| **Réflexion** après JSON et les bases de données | elle n'a de sens qu'une fois vus les usages qu'elle sert |

## Correspondance avec les chapitres de *Pro Go*

| Chapitre | Où il est traité |
|---|---|
| `2-tools` | N1 L1 |
| `3-basicFeatures` | N1 L2, L8 |
| `4-operations` | N1 L3 |
| `5-flowcontrol` | N1 L4, L5 |
| `6-collections` | N1 L6, L7 |
| `7-functions` | N1 L9 |
| `8-functionTypes` | N1 L11 |
| `9-structs` | N2 L2 |
| `10-methodsAndInterfaces` | N2 L3, L4 |
| `11-packages` | N2 L8, L9 |
| `12-composition` | N2 L5 |
| `13-concurrency` | N6 L1 à L9 |
| `14-errorHandling` | N1 L10 (usage), N2 L6, L7 (mécanique) |
| `15-stringsandregexp` | N4 L2 |
| `16-usingstrings` | N4 L1 |
| `17-mathandsorting` | N4 L3 |
| `18-datesandtimes` | N4 L4 |
| `19-readersandwriters` | N4 L5 |
| `20-readersandwritersjson` | N4 L6 |
| `21-files` | N4 L7 |
| `22-htmltext` | N4 L8 |
| `23-httpserver` | N7 L1 à L5, L9 |
| `24-httpclient` | N7 L6, L7 |
| `25-data` | N8 |
| `26/27/28-reflection` | N9 |
| `29-coordination` | N6 L5 à L7, L11 |
| `30-tests` | N5 |

Les niveaux 10 à 13 (architecture, performance, production, expert) vont au-delà du livre.

---

## Niveau 1 — Fondamentaux du langage

Objectif : écrire seul un programme Go correct, sans copier-coller.

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | [Environnement et outils](niveau-01-fondamentaux/lecon-01-environnement/) | modules, packages, `go run/build/fmt/vet`, `go.mod`, binaire autonome | ✅ |
| 2 | [Variables, constantes, types](niveau-01-fondamentaux/lecon-02-variables-types/) | `var` vs `:=`, zéro-valeur, entiers, flottants, `const`, `iota`, constantes non typées | ✅ |
| 3 | [Opérateurs, conversions, formatage](niveau-01-fondamentaux/lecon-03-operateurs-conversions/) | division entière, modulo signé, opérateurs binaires, `&^`, précédence, `strconv`, verbes de `fmt` | ✅ |
| 4 | [Conditions et branchements](niveau-01-fondamentaux/lecon-04-conditions/) | `if` avec instruction, *early return*, `switch` (valeurs multiples, sans expression, `fallthrough`) | ✅ |
| 5 | [Boucles](niveau-01-fondamentaux/lecon-05-boucles/) | les 4 formes de `for`, `range` sur entier (1.22), `break`/`continue`, labels | ✅ |
| 6 | [Tableaux et slices](niveau-01-fondamentaux/lecon-06-tableaux-slices/) | tableau vs slice, `len`/`cap`, `append`, aliasing, `s[a:b:c]`, `copy`, paquet `slices` | ✅ |
| 7 | [Maps](niveau-01-fondamentaux/lecon-07-maps/) | `v, ok`, map nil, itération non déterministe, clés comparables, ensembles | ✅ |
| 8 | [Strings, runes et bytes](niveau-01-fondamentaux/lecon-08-strings-runes/) | UTF-8, `byte` vs `rune`, immutabilité, `strings.Builder`, `Cut`/`CutLast` (1.27) | ✅ |
| 9 | [Fonctions](niveau-01-fondamentaux/lecon-09-fonctions/) | passage par valeur, retours multiples, résultats nommés, variadiques, portée, masquage | ✅ |
| 10 | [Erreurs — première approche](niveau-01-fondamentaux/lecon-10-erreurs-premiere-approche/) | `error` comme valeur, `errors.New`, `fmt.Errorf`, conventions de message, `_ =` | ✅ |
| 11 | [Types fonction et closures](niveau-01-fondamentaux/lecon-11-types-fonction-closures/) | fonctions valeurs, types nommés, closures, ordre supérieur, décorateur, options fonctionnelles | ✅ |

---

## Niveau 2 — Types, méthodes et abstraction

Objectif : concevoir ses propres types et organiser le code.

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | [Pointeurs](niveau-02-types-abstraction/lecon-01-pointeurs/) | `&`/`*`, `nil`, passage par valeur, escape analysis, quand un pointeur est justifié | ✅ |
| 2 | [Structs](niveau-02-types-abstraction/lecon-02-structs/) | littéraux, zéro-valeur, copie superficielle, comparabilité, imbrication, balises, alignement | ✅ |
| 3 | [Méthodes](niveau-02-types-abstraction/lecon-03-methodes/) | récepteur valeur/pointeur, ensemble de méthodes, adressabilité, `String()`, constructeurs | ✅ |
| 4 | [Interfaces](niveau-02-types-abstraction/lecon-04-interfaces/) | satisfaction implicite, petites interfaces, côté consommateur, assertions, `switch` de type, piège du nil | ✅ |
| 5 | [Composition et embedding](niveau-02-types-abstraction/lecon-05-composition/) | promotion, absence de dispatch virtuel, composition d'interfaces, décorateur, faux partiel | ✅ |
| 6 | [Erreurs avancées](niveau-02-types-abstraction/lecon-06-erreurs-avancees/) | `%w`, `errors.Is`/`As`/`Join`, types d'erreur, sentinelles, frontière d'API | ✅ |
| 7 | [`defer`, `panic`, `recover`](niveau-02-types-abstraction/lecon-07-defer-panic-recover/) | LIFO, évaluation des arguments, `defer` en boucle, résultats nommés, `Must`, frontières | ✅ |
| 8 | [Packages et visibilité](niveau-02-types-abstraction/lecon-08-packages-visibilite/) | découpage, cycles d'import, `internal/`, nommage, `init()`, imports nommés/anonymes | ✅ |
| 9 | [Modules et `go.mod`](niveau-02-types-abstraction/lecon-09-modules-gomod/) | semver, `/v2`, MVS, `go.sum`, proxy, `go mod tidy`, `go.work`, publication | ✅ |
| 10 | [Organisation d'un projet](niveau-02-types-abstraction/lecon-10-organisation-projet/) | quatre stades, `cmd/`/`internal/`/`pkg/`, domaine vs couche, `main` + `run() error` | ✅ |

➡️ **[Projet 1 — CLI simple](projets/projet-01-cli-simple/)**

---

## Niveau 3 — Go moderne et génériques

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | Paramètres de type | syntaxe, inférence, quand les génériques aident et quand ils nuisent | 🔜 |
| 2 | Contraintes de type | `any`, `comparable`, `cmp.Ordered`, unions de types, contraintes personnalisées | 🔜 |
| 3 | Méthodes génériques | nouveauté **Go 1.27**, inférence étendue, limites | 🔜 |
| 4 | `slices`, `maps`, `cmp` | l'API générique moderne, ce qu'elle remplace | 🔜 |
| 5 | Itérateurs et `range` sur fonction | `iter.Seq`, `iter.Seq2`, `slices.Collect`, `maps.Keys` (Go 1.23+) | 🔜 |
| 6 | Code idiomatique | conception d'API, `Effective Go`, `Code Review Comments`, revue | 🔜 |

---

## Niveau 4 — Bibliothèque standard essentielle

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | `strings` en profondeur | l'API complète, `Builder`, `Replacer`, performances | 🔜 |
| 2 | Expressions régulières | `regexp`, RE2 et l'absence de rétro-références, `MustCompile`, groupes nommés, coût | 🔜 |
| 3 | Math, tri et comparaison | `math`, `math/rand/v2`, `sort` vs `slices.SortFunc`, tri stable, `cmp` | 🔜 |
| 4 | Dates, heures et durées | `time.Time`, `Duration`, fuseaux, formatage par référence, monotonic clock, pièges | 🔜 |
| 5 | Readers et Writers | `io.Reader`/`Writer`, `bufio`, `io.Copy`, `TeeReader`, `MultiWriter`, composition | 🔜 |
| 6 | JSON | `encoding/json`, **`encoding/json/v2` et `jsontext` (1.27)**, balises, types personnalisés | 🔜 |
| 7 | Fichiers et systèmes de fichiers | `os`, `path/filepath`, `io/fs`, `os.Root` , fichiers temporaires, `//go:embed` | 🔜 |
| 8 | Gabarits | `text/template`, `html/template` et l'échappement contextuel, injection | 🔜 |

➡️ **[Projet 2 — Gestionnaire de tâches CLI](projets/projet-02-gestionnaire-taches/)**

---

## Niveau 5 — Tests et qualité

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | Le paquet `testing` | premier test, `t.Run`, `t.Helper`, `t.Cleanup`, `t.TempDir`, `Error` vs `Fatal` | 🔜 |
| 2 | Table-driven tests | le patron canonique Go, sous-tests, cas limites | 🔜 |
| 3 | Tester les dépendances | interfaces, fakes, mocks — et pourquoi Go préfère les fakes | 🔜 |
| 4 | Couverture et golden files | `-cover`, ce que la couverture ne dit pas, fichiers de référence | 🔜 |
| 5 | Exemples exécutables | `func Example…`, documentation testée, `go doc -ex` (1.27) | 🔜 |
| 6 | Benchmarks | `testing.B`, `b.Loop` (1.24+), mesures fiables, `benchstat` | 🔜 |
| 7 | Fuzzing | `func Fuzz…`, corpus, bugs trouvés par la machine | 🔜 |
| 8 | Outillage qualité | `go vet`, `staticcheck`, `golangci-lint`, `govulncheck`, `go fix` (1.27) | 🔜 |

---

## Niveau 6 — Concurrence et coordination

Le cœur de Go. Beaucoup d'exercices, volontairement.

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | Le modèle de concurrence | goroutines, ordonnanceur M:N, `GOMAXPROCS`, concurrence ≠ parallélisme | 🔜 |
| 2 | Channels non bufferisés | le rendez-vous, direction des channels | 🔜 |
| 3 | Channels bufferisés | capacité, fermeture, `range`, qui ferme | 🔜 |
| 4 | `select` | multiplexage, `default`, `time.After`, boucles de service | 🔜 |
| 5 | `sync.WaitGroup` | attendre des goroutines, `WaitGroup.Go` (1.25+), pièges classiques | 🔜 |
| 6 | `Mutex` et `RWMutex` | protéger un état, granularité, `sync.Map` | 🔜 |
| 7 | `atomic`, `Once`, `Pool` | opérations atomiques, `OnceValue`, réutilisation | 🔜 |
| 8 | `context` | annulation, `WithTimeout`, propagation, valeurs avec parcimonie | 🔜 |
| 9 | Patrons | worker pool, pipeline, fan-in/fan-out, `errgroup` | 🔜 |
| 10 | Pathologies | data races, deadlocks, livelocks, famine, fuites de goroutines, `-race`, profil `goroutineleak` (1.27) | 🔜 |
| 11 | Backpressure | dimensionner les files, rejeter proprement, charge et latence | 🔜 |
| 12 | Panique et goroutines | pourquoi `recover` ne protège que sa goroutine, supervision | 🔜 |

➡️ **[Projet 5 — Serveur concurrent](projets/projet-05-serveur-concurrent/)**

---

## Niveau 7 — Réseau, HTTP et services

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | `net/http` — le serveur | `Handler`, `HandlerFunc`, `ServeMux` et son routage à motifs (1.22+) | 🔜 |
| 2 | Requêtes et réponses | statuts, en-têtes, corps, streaming, `http.Error` | 🔜 |
| 3 | Middleware | composition, journalisation, `recover`, identifiant de requête | 🔜 |
| 4 | Conception d'API REST | ressources, verbes, statuts, pagination, versionnement | 🔜 |
| 5 | Erreurs HTTP | modèle d'erreur, mapping domaine → statut, ne rien fuiter | 🔜 |
| 6 | Client HTTP | `http.Client`, `Transport`, pools, timeouts, retries, drainage (1.27) | 🔜 |
| 7 | `context` de bout en bout | annulation client → serveur → base, arrêt gracieux | 🔜 |
| 8 | Authentification | sessions, JWT, clés d'API, OAuth2, hachage de mots de passe | 🔜 |
| 9 | Serveur en production | timeouts, limites de taille, `MaxHeaderValueCount` (1.27), reverse proxy | 🔜 |
| 10 | TCP et UDP | `net.Listen`, protocoles binaires, framing, DNS | 🔜 |
| 11 | TLS et WebSockets | certificats, mTLS, post-quantique (1.27), mise à niveau de connexion | 🔜 |
| 12 | RPC et gRPC | protobuf, streaming, intercepteurs, quand gRPC plutôt que REST | 🔜 |

➡️ **[Projet 3 — API REST avec tests](projets/projet-03-api-rest/)**

---

## Niveau 8 — Bases de données

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | `database/sql` | modèle, drivers, `pgx` avec PostgreSQL | 🔜 |
| 2 | Requêtes et scan | `QueryRow`, `Query`, `NULL`, types personnalisés, `Scanner`/`Valuer` | 🔜 |
| 3 | Requêtes préparées | injection SQL, paramètres, réutilisation | 🔜 |
| 4 | Transactions | `Begin`/`Commit`/`Rollback`, isolation, `defer` et rollback | 🔜 |
| 5 | Pool de connexions | `SetMaxOpenConns`, saturation, fuites de lignes | 🔜 |
| 6 | Migrations | outillage, réversibilité, déploiement sans coupure | 🔜 |
| 7 | Repository pattern | abstraire sans sur-abstraire, tests avec `testcontainers` | 🔜 |
| 8 | Performance et ORM | index, N+1, `EXPLAIN`, avantages et coûts d'un ORM | 🔜 |

➡️ **[Projet 4 — API avec PostgreSQL](projets/projet-04-api-postgresql/)** ·
**[Projet 6 — Service avec authentification](projets/projet-06-service-auth/)**

---

## Niveau 9 — Réflexion et métaprogrammation

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | Bases de `reflect` | `Type` et `Value`, `Kind`, les trois lois de la réflexion | 🔜 |
| 2 | Structs, champs et balises | parcourir une struct, lire les balises, modifier une valeur (`CanSet`) | 🔜 |
| 3 | Fonctions, méthodes et types dynamiques | `Call`, création dynamique, `reflect.DeepEqual` et ses surprises | 🔜 |
| 4 | Coût et alternatives | performance réelle, génération de code, génériques — quand ne pas réfléchir | 🔜 |

---

## Niveau 10 — Architecture

Avec, à chaque leçon, la question honnête : **quand ne PAS faire ça**.

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | Séparation des responsabilités | couches, dépendances dirigées vers le domaine | 🔜 |
| 2 | handler / service / repository | le découpage par défaut d'un backend Go | 🔜 |
| 3 | Injection de dépendances | sans framework, par constructeur ; `wire` en option | 🔜 |
| 4 | Interfaces bien conçues | côté consommateur, petites, découvertes tardivement | 🔜 |
| 5 | Hexagonale et clean | ports/adaptateurs, coût réel, sur-ingénierie | 🔜 |
| 6 | Domain-Driven Design | langage ubiquitaire, agrégats, contextes bornés | 🔜 |
| 7 | Monolithe modulaire vs microservices | le monolithe modulaire comme défaut raisonnable | 🔜 |
| 8 | Événementiel et messaging | queues, pub/sub, outbox, ordre et livraison | 🔜 |
| 9 | Résilience | idempotence, retries avec jitter, circuit breakers, timeouts | 🔜 |
| 10 | Observabilité | logs / métriques / traces comme décision d'architecture | 🔜 |

➡️ **[Projet 7 — Workers et queues](projets/projet-07-workers-queues/)**

---

## Niveau 11 — Performance

Règle du niveau : **mesurer avant d'optimiser**, toujours.

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | Mesurer | benchmarks fiables, bruit, `benchstat`, micro vs macro | 🔜 |
| 2 | Pile, tas, escape analysis | `-gcflags=-m`, ce qui s'échappe et pourquoi | 🔜 |
| 3 | Le ramasse-miettes | tri-color, `GOGC`, `GOMEMLIMIT`, pauses, allocation rapide (1.27) | 🔜 |
| 4 | pprof — CPU | collecte, flamegraphs, lecture d'un profil | 🔜 |
| 5 | pprof — mémoire, block, mutex | allocations, contention, profil `goroutineleak` (1.27) | 🔜 |
| 6 | Slices, maps et cache | préallocation, localité, coût réel | 🔜 |
| 7 | Latence et débit | percentiles, loi de Little, files d'attente | 🔜 |
| 8 | Optimisations qui valent le coup | pooling, réduction d'allocations, ce qu'il ne faut pas faire | 🔜 |

---

## Niveau 12 — Outils et production

| # | Leçon | Contenu | État |
|---|---|---|---|
| 1 | Git et projets Go | branches, tags de version, conventions de commit | 🔜 |
| 2 | Build de production | cross-compilation, `-ldflags`, builds reproductibles | 🔜 |
| 3 | Docker | multi-stage, distroless, binaire statique | 🔜 |
| 4 | CI/CD | GitHub Actions : test, lint, race, build, release | 🔜 |
| 5 | Configuration | variables d'environnement, précédence, validation au démarrage | 🔜 |
| 6 | Logs structurés | `log/slog`, niveaux, contexte, ce qu'on ne journalise jamais | 🔜 |
| 7 | Métriques et traces | Prometheus, OpenTelemetry, corrélation | 🔜 |
| 8 | Sécurité et secrets | `govulncheck`, gestion de secrets, surface d'attaque | 🔜 |
| 9 | Production | déploiement, santé, monitoring, debugging à chaud | 🔜 |

➡️ **[Projet 8 — Microservice Go professionnel](projets/projet-08-microservice/)**

---

## Niveau 13 — Expert

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

---

## Ordre chronologique des projets

| Après le niveau | Projet |
|---|---|
| 2 | [Projet 1 — CLI simple](projets/projet-01-cli-simple/) |
| 4 | [Projet 2 — Gestionnaire de tâches](projets/projet-02-gestionnaire-taches/) |
| 6 | [Projet 5 — Serveur concurrent](projets/projet-05-serveur-concurrent/) |
| 7 | [Projet 3 — API REST avec tests](projets/projet-03-api-rest/) |
| 8 | [Projet 4 — API PostgreSQL](projets/projet-04-api-postgresql/) · [Projet 6 — Service avec authentification](projets/projet-06-service-auth/) |
| 10 | [Projet 7 — Workers et queues](projets/projet-07-workers-queues/) |
| 12 | [Projet 8 — Microservice](projets/projet-08-microservice/) |
| 13 | [Projet final](projets/projet-final/) |

La numérotation des projets est celle du programme initial ; l'ordre de réalisation est
celui du tableau ci-dessus.
