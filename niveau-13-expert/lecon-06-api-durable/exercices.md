# Module 6 — Exercices

Code dans `mes-solutions/niveau-13/module-06/`. Pour chaque changement, **prédire** d'abord s'il est
compatible, puis vérifier avec `apidiff` **et** avec un client compilé.

## Exercices faciles

**E1 — Prédire, puis vérifier.**
Classer chacun de ces changements d'un paquet v1 (compatible ou non), puis le vérifier avec `apidiff` et un
client :

1. une fonction qui renvoyait `*Config` renvoie désormais `Config` ;
2. une méthode ajoutée à une **struct** exportée ;
3. la valeur d'une constante exportée change (`MaxItems = 100` → `200`) ;
4. un champ **non exporté** de type `[]string` ajouté à une struct exportée **jusqu'ici comparable** ;
5. un type exporté remplacé par un alias vers un autre paquet (`type Point = geom.Point`) ;
6. un paramètre `int` devient `int64`.

Lequel `apidiff` ne voit-il pas, alors qu'il peut casser des utilisateurs ?

**E2 — Prévoir les paramètres futurs.**
`func NewClient(addr string) *Client` doit bientôt accepter un délai, un nombre de réessais et un journal.
Écrire deux conceptions extensibles, une struct `Options` et des options fonctionnelles
(`NewClient(addr, WithTimeout(…))`). Pour chacune : comment ajouter une option en v1.2 sans rien casser ? Quel
comportement pour une option à zéro ? Laquelle choisir, et pourquoi ?

**E3 — Migrer une constante et un type.**
`//go:fix inline` fonctionne-t-il sur une constante (`const Timeout = DefaultTimeout`) ? Sur un alias de
type ? L'essayer, et lire ce que `go fix -diff` propose.

**E4 — `apidiff` en CI.**
Écrire un script `api-check.sh` qui compare l'API du module courant à celle de la **dernière étiquette** Git.
Il extrait l'étiquette dans un répertoire temporaire (`git worktree add`), puis lance `apidiff`. Il échoue si
un changement incompatible apparaît sans changement de version majeure. Le tester sur un dépôt créé pour
l'occasion.

**E5 — Renommer un champ JSON sans casser.**
`total` doit devenir `amount_cents`. Écrire la transition en trois versions du serveur :
1. il écrit les deux champs ;
2. il ne lit plus que le nouveau, et annonce la dépréciation de l'ancien ;
3. il retire l'ancien.

Tester chaque version contre un ancien client et un nouveau client, avec un test en table.

---

## Exercice intermédiaire — Une version 2 de `geo`

Les dépréciations de v1.1 et une meilleure conception justifient une v2 :
- `Nearest` disparaît au profit de `NearestContext`, renommée `Nearest` ;
- `Store` prend un `ctx` dans ses méthodes ;
- les distances se calculent dans une unité choisie à la création d'un `Calculator`.

1. Créer le module `example.com/api/v2` (sous-répertoire `v2/`, avec son propre `go.mod`), et sa nouvelle API.
2. Réécrire la v1 **par-dessus** la v2 : les fonctions de v1 appellent celles de v2. Ajouter
   `//go:fix inline` là où c'est possible, pour que `go fix` fasse passer le code des utilisateurs de v1 à v2.
3. Un programme qui importe **à la fois** v1 et v2 compile-t-il ? Montrer un cas où c'est utile.
4. Écrire le guide de migration (une page) : ce que `go fix` fait seul, ce qu'il faut changer à la main.
5. Combien de temps maintenir la v1, et qu'y publier ?

---

## Défi — Des contrats dictés par les clients

Pour une API HTTP, on veut savoir **avant** de déployer si un changement du serveur casse un client existant.

1. Chaque client enregistre ses **attentes** dans un fichier : la requête qu'il envoie, et les champs de la
   réponse qu'il **utilise** réellement, avec leur type. Pas toute la réponse, seulement ce dont il dépend.
2. Écrire un outil `contract` qui rejoue ces attentes contre un serveur (un `httptest.Server`) et rapporte
   chaque champ manquant, renommé ou de type changé.
3. L'utiliser en CI du serveur : les contrats de trois clients fictifs, et trois changements du serveur (un
   ajout, un renommage, un changement de type). L'outil doit accepter le premier et refuser les deux autres,
   avec un message qui dit **quel client** casse et **pourquoi**.
4. Quelles différences avec un test de bout en bout ? Avec `apidiff` ? Qui maintient les contrats ?

---

## Quiz

1. Pourquoi ajouter un paramètre variadique est-il incompatible ?
2. Pourquoi ajouter une méthode à une interface publiée est-il incompatible, et que faire à la place ?
3. Ajouter un champ à une struct est-il compatible ? Pour qui ne l'est-ce pas ?
4. Qu'est-ce qu'une interface optionnelle ? Un exemple de la bibliothèque standard.
5. Comment déprécier une fonction en Go, et quels outils le voient ?
6. Que fait `//go:fix inline` ?
7. Comment publier une version majeure d'un module Go ?
8. Que voit un ancien client JSON quand le serveur renomme un champ ?
9. Pourquoi un client strict (`DisallowUnknownFields`) est-il un mauvais choix pour lire des réponses ?
10. Que vérifie `apidiff`, et que ne vérifie-t-il pas ?
