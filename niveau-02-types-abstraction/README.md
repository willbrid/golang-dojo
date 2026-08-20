# Niveau 2 — Types, méthodes et abstraction

**Objectif :** concevoir ses propres types, les doter d'un comportement, les rendre
interchangeables par des interfaces, et organiser le tout en packages et en modules.

**Prérequis :** tout le niveau 1. En particulier les slices, les maps et les fonctions.

> **Ordre des leçons.** Les pointeurs viennent en premier parce que les récepteurs de méthode
> en dépendent. Les interfaces viennent après les méthodes parce qu'elles reposent sur la
> notion d'**ensemble de méthodes**. Les erreurs avancées viennent après les interfaces parce
> qu'`error` **est** une interface — c'est le défaut d'ordre le plus fréquent des cours de Go.

## Leçons

| # | Leçon | Ce que je saurai faire |
|---|---|---|
| 1 | [Pointeurs](lecon-01-pointeurs/) | Savoir *quand* un pointeur est justifié, éviter les paniques `nil` |
| 2 | [Structs](lecon-02-structs/) | Modéliser des données, comprendre la copie superficielle et la comparabilité |
| 3 | [Méthodes](lecon-03-methodes/) | Choisir récepteur valeur ou pointeur, raisonner sur l'ensemble de méthodes |
| 4 | [Interfaces](lecon-04-interfaces/) | Concevoir de petites interfaces côté consommateur, éviter le piège de l'interface nil |
| 5 | [Composition et embedding](lecon-05-composition/) | Composer sans héritage, écrire des décorateurs |
| 6 | [Erreurs avancées](lecon-06-erreurs-avancees/) | `%w`, `errors.Is`/`As`/`Join`, types d'erreur |
| 7 | [`defer`, `panic`, `recover`](lecon-07-defer-panic-recover/) | Libérer des ressources de façon sûre, savoir où `recover` est légitime |
| 8 | [Packages et visibilité](lecon-08-packages-visibilite/) | Découper sans cycle, utiliser `internal/`, nommer correctement |
| 9 | [Modules et `go.mod`](lecon-09-modules-gomod/) | Gérer les dépendances, comprendre MVS et le semver de Go |
| 10 | [Organisation d'un projet](lecon-10-organisation-projet/) | Structurer un projet réel sans sur-structurer |

## Correspondance avec les chapitres de *Pro Go*

`9-structs` → leçon 2 · `10-methodsAndInterfaces` → leçons 3 et 4 · `11-packages` → leçons 8
et 9 · `12-composition` → leçon 5 · `14-errorHandling` → leçons 6 et 7 (la première approche
des erreurs est au niveau 1, leçon 10).

## Évaluation de fin de niveau

Évaluation cumulative des niveaux 1 et 2, puis le
**[projet 1 — `wordstat`](../projets/projet-01-cli-simple/)**.
