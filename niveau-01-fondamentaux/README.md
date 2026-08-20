# Niveau 1 — Fondamentaux

**Objectif du niveau :** écrire seul un programme Go correct et lisible, comprendre le
système de types, et savoir chercher une réponse dans la documentation officielle plutôt
que de deviner.

**Prérequis :** aucun en Go. Des bases générales de programmation aident, mais chaque
notion non spécifique à Go (pile/tas, pointeur, valeur/référence) est réexpliquée au passage.

## Leçons

| # | Leçon | Ce que je saurai faire à la fin |
|---|---|---|
| 1 | [Environnement et outils](lecon-01-environnement/) | Créer un module, compiler, formater, analyser, lire une erreur du compilateur |
| 2 | [Variables, constantes, types](lecon-02-variables-types/) | Choisir le bon type, comprendre la zéro-valeur, manier `const` et `iota` |
| 3 | [Strings, runes, bytes](lecon-03-strings-runes/) | Manipuler du texte UTF-8 sans le casser, construire des chaînes efficacement |
| 4 | [Conditions et boucles](lecon-04-conditions-boucles/) | Utiliser les 4 formes de `for`, `switch` sans `break`, `range` correctement |
| 5 | [Fonctions](lecon-05-fonctions/) | Retours multiples, variadiques, closures, portée lexicale |
| 6 | [Tableaux et slices](lecon-06-tableaux-slices/) | Comprendre `len`/`cap`/`append` et le partage de tableau sous-jacent |
| 7 | [Maps](lecon-07-maps/) | L'idiome `v, ok`, l'itération non déterministe, implémenter un ensemble |
| 8 | [Erreurs simples](lecon-08-erreurs-simples/) | Retourner et traiter `(T, error)` correctement |
| 9 | [Pointeurs](lecon-09-pointeurs/) | Savoir *quand* un pointeur est justifié, éviter les `nil` panics |
| 10 | [Structs et méthodes](lecon-10-structs-methodes/) | Modéliser des données, choisir récepteur valeur ou pointeur |
| 11 | [Interfaces et composition](lecon-11-interfaces-composition/) | Concevoir de petites interfaces, préférer la composition à l'héritage |

## Évaluation de fin de niveau

Un exercice de synthèse noté sur les quatre axes (*fonctionne / correct / idiomatique /
production*) et le **[Projet 1 — CLI simple](../projets/projet-01-cli-simple/)**.

## Comment travailler une leçon

1. Lire `cours.md` en entier, sans écrire de code.
2. Retaper l'exemple **à la main** (pas de copier-coller) et le faire tourner.
3. Faire les exercices de `exercices.md` dans `mes-solutions/niveau-01/lecon-XX/`.
4. Envoyer le code, même s'il ne compile pas.
5. Répondre au quiz **sans relire le cours**.
6. Ne consulter la branche `corrections` qu'après tout ça.
