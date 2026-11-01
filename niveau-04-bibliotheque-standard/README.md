# Niveau 4 — Bibliothèque standard essentielle

**Objectif :** maîtriser les paquets qu'on utilise tous les jours — texte, expressions
régulières, nombres, temps, flux d'octets, JSON, fichiers, gabarits — et savoir lire la
documentation de n'importe quel autre.

**Prérequis :** niveaux 1 à 3. En particulier : les interfaces (`io.Reader` **est** une
interface, N2 L4), les erreurs enveloppées (N2 L6), `slices` et `cmp` (N3 L4), les itérateurs
(N3 L5).

> **Ordre des leçons.** `strings` d'abord, parce que tout le reste manipule du texte. Les
> expressions régulières ensuite, parce qu'elles s'évaluent face à `strings`. Les
> **Readers/Writers** (leçon 5) précèdent JSON, fichiers et gabarits : ces trois paquets sont
> construits sur `io.Reader` et `io.Writer`, et se comprennent mal sans eux.

## Leçons

| # | Leçon | Ce que je saurai faire |
|---|---|---|
| 1 | [`strings` en profondeur](lecon-01-strings/) | Choisir la bonne fonction, construire et remplacer efficacement |
| 2 | [Expressions régulières](lecon-02-regexp/) | Utiliser `regexp` à bon escient, connaître les limites de RE2 |
| 3 | [Math, aléatoire, tri](lecon-03-math-tri/) | Éviter les pièges des flottants, tirer des nombres correctement, trier |
| 4 | [Dates, heures et durées](lecon-04-temps/) | Formater, parser, comparer des instants sans bug de fuseau |
| 5 | [Readers et Writers](lecon-05-readers-writers/) | Composer des flux d'octets, bufferiser, copier |
| 6 | [JSON](lecon-06-json/) | Encoder et décoder, avec `encoding/json` et `encoding/json/v2` (1.27) |
| 7 | [Fichiers et systèmes de fichiers](lecon-07-fichiers/) | Lire, écrire de façon sûre, parcourir, embarquer des fichiers |
| 8 | [Gabarits](lecon-08-gabarits/) | Générer du texte et du HTML sans injection |

## Évaluation de fin de niveau

Le **[projet 2 — gestionnaire de tâches](../projets/projet-02-gestionnaire-taches/)** mobilise
les leçons 4 à 7 : dates, fichiers, JSON, écriture atomique.

## Comment travailler une leçon

1. Lire `cours.md` en entier, sans écrire de code.
2. Retaper l'exemple **à la main** et le faire tourner.
3. Faire les exercices dans `mes-solutions/niveau-04/lecon-XX/`.
4. Comparer sa sortie à celle attendue : `go run . | cat -A`.
5. Pour toute fonction inconnue : `go doc paquet.Fonction` **avant** de chercher ailleurs.
6. Répondre au quiz sans relire le cours ; consulter `corrections` ensuite.
