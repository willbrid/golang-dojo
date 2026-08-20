# Leçon 8 — Exercices

Code dans `mes-solutions/niveau-02/lecon-08/`.

## Exercices faciles

**E1 — Multi-fichiers.**
Créer un package `mathx` réparti sur trois fichiers (`basic.go`, `stats.go`, `doc.go`), avec
des fonctions exportées et non exportées. Vérifier qu'une fonction de `basic.go` est utilisable
depuis `stats.go` **sans import**. Écrire le commentaire de package dans `doc.go`.

**E2 — Visibilité.**
Depuis `main`, tenter d'accéder à : un type exporté, un champ non exporté d'un type exporté,
une fonction non exportée, une méthode non exportée. Relever les quatre messages d'erreur et
les comparer.

**E3 — Cycle d'import.**
Créer volontairement deux packages qui s'importent mutuellement. Relever le message exact du
compilateur. Puis casser le cycle des **trois** façons décrites dans le cours et dire, pour ce
cas précis, laquelle est la meilleure.

**E4 — `internal/`.**
Créer `monprojet/internal/secret` et tenter de l'importer depuis un **second module** situé
ailleurs sur le disque. Relever le message. Puis vérifier qu'il s'importe bien depuis
`monprojet`. Où exactement se situe la frontière ? *(Tester avec un sous-répertoire à
différents niveaux.)*

**E5 — `init()` et import anonyme.**
Écrire un package avec deux `init()` dans deux fichiers, plus une variable de package
initialisée par un appel de fonction. Afficher un message dans chaque. **Prédire l'ordre**
d'exécution avant de lancer. Puis l'importer avec `_` depuis `main` sans utiliser aucun de ses
identifiants.

---

## Exercice intermédiaire — `taskman`

Découper une application en packages cohérents, sans cycle.

**Fonctionnalité :** un gestionnaire de tâches en mémoire — créer, lister, terminer,
supprimer, filtrer par étiquette et par statut, avec un affichage tabulaire et un export
texte.

**Contraintes :**
1. Au moins **quatre packages**, dont un `main` minimal (moins de 60 lignes).
2. Le cœur métier vit sous `internal/`. Justifier ce qui est sous `internal/` et ce qui ne l'est pas.
3. **Aucun cycle**, et l'expliquer : dessiner le graphe de dépendances en commentaire dans le README du projet.
4. Le package d'affichage ne doit **pas** importer le package métier. *(Rappel : interface côté consommateur.)*
5. Chaque package a un commentaire `// Package x …` expliquant sa raison d'exister **en une phrase**. Si la phrase contient « et », interroger le découpage.
6. Le nombre d'identifiants exportés doit être minimal : lister ce qui est exporté et **justifier chaque élément**.
7. Aucun package nommé `util`, `common`, `models` ou `helpers`.
8. Aucun `init()`.

*L'exercice n'est pas d'écrire la fonctionnalité — elle est triviale — mais de **défendre le
découpage**. Rédiger dans le README : pourquoi ces packages, quelles alternatives ont été
écartées, et ce qui se passerait si le projet triplait de taille.*

---

## Défi

**a) Refactorisation d'un cycle réel.**
Écrire trois packages `user`, `order` et `notification` où : une commande référence un
utilisateur, une notification concerne une commande **et** un utilisateur, et un utilisateur
doit pouvoir lister ses commandes. Un cycle apparaît naturellement.
Le résoudre de **deux** manières différentes, puis comparer les deux architectures sur :
nombre de packages, taille des interfaces, testabilité, et ce qui se passe quand on ajoute
une facturation.

**b) API publique minimale.**
Prendre le package `catalog` de l'exemple du cours et le repenser pour qu'il expose le moins
possible : quels types peuvent devenir non exportés ? Peut-on retourner une **interface** au
lieu d'une struct ? Faut-il le faire ?
*(Attention au piège : « accept interfaces, return structs » dit l'inverse. Dans quel cas
l'exception se justifie-t-elle ? Chercher comment `database/sql` s'y prend.)*

**c) Le coût de `internal/`.**
Une équipe met tout son code sous `internal/`. Six mois plus tard, une autre équipe veut
réutiliser un composant. Que se passe-t-il ? Quelles sont les options, et que coûte chacune ?
Écrire dix lignes.
Puis chercher comment de grands projets Go (Kubernetes, Docker, la bibliothèque standard
elle-même) gèrent cette frontière — et ce que le répertoire `pkg/` apporte réellement, ou pas.

---

## Quiz

1. Qu'est-ce qui délimite un package en Go ?
2. Deux fichiers du même répertoire doivent-ils s'importer mutuellement ?
3. Comment déclare-t-on qu'un identifiant est privé ?
4. L'encapsulation Go s'applique-t-elle au type ou au package ?
5. Que se passe-t-il si deux packages s'importent mutuellement ?
6. Citer trois façons de casser un cycle d'import.
7. Qui peut importer un package situé sous `internal/` ?
8. À quoi sert un import anonyme `_` ? Donner un cas réel.
9. Pourquoi l'import point est-il déconseillé ?
10. Quand `init()` est-il justifié ? Que faire à la place le reste du temps ?
11. Pourquoi `util` est-il un mauvais nom de package ?
