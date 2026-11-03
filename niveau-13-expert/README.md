# Niveau 13 — Expert

**Objectif :** résoudre des problèmes de niveau senior sans guidage pas à pas. On conçoit à partir
d'invariants, on diagnostique des pannes qui ne se reproduisent pas, on raisonne sur des systèmes
distribués, et on prend, puis on documente, des décisions dont on assumera les conséquences pendant des
années.

**Prérequis :** tous les niveaux précédents.

> **Ce niveau est différent.** Les leçons gardent leur format, mais les exercices sont plus ouverts : il y a
> souvent plusieurs bonnes réponses. Ce qui est évalué, c'est la justification, chiffrée quand c'est
> possible. Les corrigés donnent **une** solution argumentée, pas **la** solution.

> **Ordre des leçons.** On commence par concevoir des composants concurrents corrects (1), puis rapides (2).
> On passe à plusieurs machines (3). Viennent ensuite les outils et les méthodes du diagnostic difficile
> (4, 5). Enfin, ce qui dure plus longtemps que le code : les API (6), la tenue en charge (7) et les
> décisions d'architecture (8).

## Leçons

| # | Module | Ce que je saurai faire |
|---|---|---|
| 1 | [Systèmes concurrents](lecon-01-systemes-concurrents/) | Concevoir un composant concurrent à partir de ses invariants, le justifier par une preuve informelle, et le tester pour de vrai |
| 2 | [Serveurs haute performance](lecon-02-serveurs-performants/) | Tenir un budget d'allocation, éviter les copies, comprendre ce que fait le *runtime* sous un serveur réseau |
| 3 | [Systèmes distribués](lecon-03-systemes-distribues/) | Raisonner sur la cohérence, les horloges, les partitions et le consensus |
| 4 | [Debugging complexe](lecon-04-debugging/) | Utiliser `delve`, le détecteur de races, les vidages mémoire et les traces d'exécution |
| 5 | [Bugs mémoire et races difficiles](lecon-05-bugs-difficiles/) | Reproduire, réduire et corriger un bug qui ne se montre qu'une fois sur mille |
| 6 | [Conception d'API durable](lecon-06-api-durable/) | Faire évoluer une API sans casser ses utilisateurs : compatibilité, dépréciation, versions majeures |
| 7 | [Résilience et scalabilité](lecon-07-resilience-scalabilite/) | Dégrader proprement, planifier la capacité, tester la charge et les pannes |
| 8 | [Décisions d'architecture](lecon-08-decisions-architecture/) | Écrire un ADR, arbitrer entre options, faire une revue de code senior |

➡️ **[Projet final](../projets/projet-final/)**

## Règle du niveau

**Une affirmation sans preuve est une opinion.** La preuve peut être un test, une mesure, un argument écrit
qui tient en une page, ou un lien vers la spécification. « Ça devrait marcher » n'en est pas une.
