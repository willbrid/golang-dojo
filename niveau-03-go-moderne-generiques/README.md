# Niveau 3 — Go moderne et génériques

**Objectif :** écrire du code générique quand il apporte quelque chose — et savoir s'en
abstenir quand il n'apporte rien —, puis utiliser la bibliothèque standard moderne
(`slices`, `maps`, `cmp`, itérateurs) qui en découle.

**Prérequis :** tout le niveau 2. En particulier les **interfaces** (leçon 4) : une contrainte
de type *est* une interface, et la moitié de ce niveau consiste à voir ce que ce mot recouvre
de plus qu'au niveau 2. Les **types fonction et closures** (niveau 1, leçon 11) sont
indispensables pour les leçons 4 et 5.

> **Ordre des leçons.** On commence par les paramètres de type avec les seules contraintes
> déjà connues (`any`, une interface à méthodes). Les contraintes à ensembles de types
> (`~int | ~float64`) viennent ensuite, parce qu'elles exigent de revoir ce qu'est une
> interface. Les méthodes génériques (Go 1.27) supposent les deux. Les paquets `slices`,
> `maps` et `cmp` sont lus comme des *exemples* de code générique bien conçu, et les
> itérateurs viennent en dernier parce qu'ils combinent génériques et closures.

## Leçons

| # | Leçon | Ce que je saurai faire |
|---|---|---|
| 1 | [Paramètres de type](lecon-01-parametres-de-type/) | Écrire une fonction et un type génériques, lire l'inférence, décider s'il faut un générique |
| 2 | [Contraintes de type](lecon-02-contraintes/) | Écrire ses contraintes, comprendre `~`, `comparable` et `cmp.Ordered` |
| 3 | [Méthodes génériques](lecon-03-methodes-generiques/) | Utiliser les méthodes à paramètres de type de Go 1.27, connaître leurs limites |
| 4 | [`slices`, `maps`, `cmp`](lecon-04-slices-maps-cmp/) | Remplacer les boucles écrites à la main par l'API standard, sans en abuser |
| 5 | [Itérateurs et `range` sur fonction](lecon-05-iterateurs/) | Écrire et consommer des `iter.Seq`/`iter.Seq2`, comprendre `yield` |
| 6 | [Code idiomatique](lecon-06-code-idiomatique/) | Relire du code Go comme un relecteur de projet open source |

## Évaluation de fin de niveau

Pas de projet dédié : l'évaluation consiste à **réécrire** une partie du projet 1
(`wordstat`) avec les outils du niveau — classement par `slices.SortFunc` et `cmp.Compare`,
itérateur sur les mots d'un `io.Reader` — puis à justifier chaque générique introduit.
Un générique qui ne se justifie pas en une phrase doit être retiré.

## Comment travailler une leçon

1. Lire `cours.md` en entier, sans écrire de code.
2. Retaper l'exemple **à la main** et le faire tourner.
3. Faire les exercices dans `mes-solutions/niveau-03/lecon-XX/`.
4. Comparer sa sortie à celle attendue : `go run . | cat -A` montre ce que l'œil ne voit pas.
5. Répondre au quiz **sans relire le cours**.
6. Ne consulter la branche `corrections` qu'après tout cela.
