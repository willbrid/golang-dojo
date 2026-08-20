# Niveau 1 — Fondamentaux du langage

**Objectif :** écrire seul un programme Go correct et lisible, et savoir chercher une réponse
dans la documentation officielle plutôt que de deviner.

**Prérequis :** aucun en Go. Chaque notion non spécifique à Go (pile/tas, pointeur,
valeur/référence, UTF-8) est réexpliquée au passage.

> **Ordre des leçons.** Il est strictement dépendant : aucune leçon n'utilise un concept non
> encore vu. C'est pourquoi les boucles précèdent les collections mais `range` n'est introduit
> qu'au fil des types parcourus, et pourquoi les structs, méthodes et interfaces sont au
> niveau 2 — elles reposent sur les pointeurs.

## Leçons

| # | Leçon | Ce que je saurai faire |
|---|---|---|
| 1 | [Environnement et outils](lecon-01-environnement/) | Créer un module, compiler, formater, analyser, lire une erreur du compilateur |
| 2 | [Variables, constantes, types](lecon-02-variables-types/) | Choisir le bon type, comprendre la zéro-valeur, manier `const` et `iota` |
| 3 | [Opérateurs, conversions, formatage](lecon-03-operateurs-conversions/) | Éviter les pièges de la division entière et du débordement, manier `fmt` et `strconv` |
| 4 | [Conditions et branchements](lecon-04-conditions/) | *Early return*, `switch` sans expression, code sans imbrication |
| 5 | [Boucles](lecon-05-boucles/) | Les quatre formes de `for`, `range` sur entier, labels |
| 6 | [Tableaux et slices](lecon-06-tableaux-slices/) | `len`/`cap`/`append`, partage du tableau sous-jacent, `range` sur slice |
| 7 | [Maps](lecon-07-maps/) | L'idiome `v, ok`, l'itération non déterministe, les ensembles |
| 8 | [Strings, runes et bytes](lecon-08-strings-runes/) | Manipuler de l'UTF-8 sans le corrompre, construire des chaînes efficacement |
| 9 | [Fonctions](lecon-09-fonctions/) | Retours multiples, variadiques, portée, masquage |
| 10 | [Erreurs — première approche](lecon-10-erreurs-premiere-approche/) | Retourner et traiter `(T, error)`, rédiger de bons messages |
| 11 | [Types fonction et closures](lecon-11-types-fonction-closures/) | Fonctions d'ordre supérieur, closures, décorateurs, options fonctionnelles |

## Correspondance avec les chapitres de *Pro Go*

`2-tools` → leçon 1 · `3-basicFeatures` → leçons 2 et 8 · `4-operations` → leçon 3 ·
`5-flowcontrol` → leçons 4 et 5 · `6-collections` → leçons 6 et 7 · `7-functions` → leçon 9 ·
`8-functionTypes` → leçon 11.

## Évaluation de fin de niveau

Un exercice de synthèse noté sur les quatre axes (*fonctionne / correct / idiomatique /
production*), suivi du niveau 2. Le **projet 1** vient après le niveau 2, car il exige un
découpage en plusieurs fichiers et une gestion d'erreurs structurée.

## Comment travailler une leçon

1. Lire `cours.md` en entier, sans écrire de code.
2. Retaper l'exemple **à la main** et le faire tourner.
3. Faire les exercices dans `mes-solutions/niveau-01/lecon-XX/`.
4. Envoyer le code, même s'il ne compile pas.
5. Répondre au quiz **sans relire le cours**.
6. Ne consulter la branche `corrections` qu'après tout cela.
