# Niveau 5 — Tests et qualité

**Objectif :** rendre un code testable, le tester sérieusement, mesurer ses performances, et
outiller sa qualité comme le fait un projet open source.

**Prérequis :** niveaux 1 à 4. Les **fakes** reposent sur les interfaces (N2 L4), les **golden
files** sur les fichiers (N4 L7), les **benchmarks** sur `time` (N4 L4), les exemples
exécutables sur les commentaires de documentation (N3 L6).

> **Pourquoi avant la concurrence ?** Le détecteur de *data races* — l'outil n°1 du niveau 6 —
> s'utilise avec `go test -race`. Et un code concurrent sans tests n'est pas débogable. Ce
> niveau est **obligatoire** et se traite en entier avant le niveau 6.

## Leçons

| # | Leçon | Ce que je saurai faire |
|---|---|---|
| 1 | [Le paquet `testing`](lecon-01-testing/) | Écrire, lancer et lire un test ; `Error` contre `Fatal`, `Helper`, `Cleanup`, `TempDir` |
| 2 | [Table-driven tests](lecon-02-table-driven/) | Écrire le patron canonique de Go, avec sous-tests nommés et cas limites |
| 3 | [Tester les dépendances](lecon-03-dependances/) | Isoler le temps, le disque, le réseau ; écrire des fakes plutôt que des mocks |
| 4 | [Couverture et golden files](lecon-04-couverture-golden/) | Mesurer la couverture et savoir ce qu'elle ne dit pas ; tester des sorties volumineuses |
| 5 | [Exemples exécutables](lecon-05-exemples/) | Écrire une documentation qui est aussi un test |
| 6 | [Benchmarks](lecon-06-benchmarks/) | Mesurer avec `b.Loop`, comparer avec `benchstat` |
| 7 | [Fuzzing](lecon-07-fuzzing/) | Laisser la machine trouver les bugs |
| 8 | [Outillage qualité](lecon-08-outillage/) | `go vet`, `staticcheck`, `golangci-lint`, `govulncheck`, `go fix` |

## Évaluation de fin de niveau

Écrire la suite de tests complète du **projet 2** (`todo`) : table-driven sur `ParseDue` et le
tri, fake d'horloge, golden files pour l'affichage, fuzzing du décodage du fichier, couverture
supérieure à 80 % **sur le paquet `task`** — et une phrase qui justifie chaque ligne non couverte.

## Comment travailler une leçon

1. Lire `cours.md` en entier, sans écrire de code.
2. Retaper l'exemple, lancer `go test -v` et **lire** la sortie.
3. **Casser volontairement** le code testé et vérifier que le test échoue — un test qui ne peut
   pas échouer ne teste rien.
4. Faire les exercices dans `mes-solutions/niveau-05/lecon-XX/`.
5. Quiz sans relire le cours ; corrections ensuite.
