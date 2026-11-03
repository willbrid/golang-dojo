# Niveau 9 — Réflexion et métaprogrammation

**Objectif :** comprendre ce que font réellement `encoding/json`, les ORM et les chargeurs de
configuration — en écrire de petits soi-même — et savoir quand **ne pas** utiliser la réflexion.

**Prérequis :** niveaux 1 à 8. En particulier :
- les **interfaces** et l'assertion de type (N2 L4) : la réflexion part toujours d'une valeur `any` ;
- les **pointeurs** (N2 L1) : modifier une valeur par réflexion exige un pointeur ;
- les **génériques** (niveau 3) : l'alternative à la réflexion la plus souvent oubliée ;
- les **balises de struct** de JSON (N4 L6) et le *mapping* par nom de GORM (N8 L8) : ce que ce niveau
  explique de l'intérieur ;
- les **benchmarks** (N5 L6) : la dernière leçon mesure le coût réel.

> **Ordre des leçons.** Le modèle (`Type`, `Value`, `Kind`, les trois lois) d'abord ; puis les structs et
> leurs balises, l'usage le plus courant ; puis les fonctions, méthodes et types créés à l'exécution, et
> les surprises de `reflect.DeepEqual` ; enfin le coût mesuré et les alternatives (génériques, génération
> de code).

## Leçons

| # | Leçon | Ce que je saurai faire |
|---|---|---|
| 1 | [Bases de `reflect`](lecon-01-bases-reflect/) | Distinguer `Type`, `Value` et `Kind`, parcourir une valeur inconnue, appliquer les trois lois |
| 2 | [Structs, champs et balises](lecon-02-structs-balises/) | Parcourir les champs, lire les balises, remplir une struct (chargeur de configuration) |
| 3 | [Fonctions, méthodes et types dynamiques](lecon-03-fonctions-types-dynamiques/) | Appeler une méthode par son nom, créer des valeurs à l'exécution, se méfier de `DeepEqual` |
| 4 | [Coût et alternatives](lecon-04-cout-alternatives/) | Mesurer le coût de la réflexion, la remplacer par des génériques ou du code généré |

## Règle du niveau

**La réflexion est un dernier recours.** Avant d'en écrire, se demander si une interface, un générique ou
quelques lignes de code explicite ne suffisent pas. Le code réflexif n'est vérifié qu'à l'exécution : ses
erreurs sont des paniques, pas des erreurs de compilation.
