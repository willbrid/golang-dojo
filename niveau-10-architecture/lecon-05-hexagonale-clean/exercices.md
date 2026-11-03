# Leçon 5 — Exercices

Code dans `mes-solutions/niveau-10/lecon-05/`, en partant du module de l'exemple.

## Exercices faciles

**E1 — Rompre la règle.**
Dans `core`, ajouter une fonction qui formate un message en utilisant une constante définie dans
`adapters/console`. Le programme compile-t-il ? Que donne `go list` ? Puis, en plus, faire importer `core`
par `console` (pour utiliser `core.MaxLoans`). Que se passe-t-il ? Lequel des deux imports respecte la règle
de dépendance, et quel outil t'avertit de la violation de l'autre ?

**E2 — Un adaptateur de plus.**
Écrire un adaptateur de sortie `file.Notifier` qui ajoute chaque notification à la fin d'un fichier
(`os.OpenFile` avec `O_APPEND`). Quelles lignes du cœur changent ? Et de `cmd/app` ?

**E3 — Sans port d'entrée.**
Supprimer l'interface `Lending` ; `New` rend `*Service` (exporté) et les adaptateurs d'entrée en
dépendent. Que perd-on ? (Indice : `TestStatuses`.) Comment tester l'adaptateur HTTP maintenant ? Est-ce
grave ?

**E4 — Compter.**
Réécrire le même comportement en **un seul paquet** (`lending`) avec quatre fichiers (domaine, service,
http, cli) et aucune interface sauf `Members` et `Notifier`. Compter fichiers, paquets, lignes, et
interfaces, et comparer avec l'exemple. Quels tests deviennent impossibles ou plus difficiles ?

**E5 — Quel découpage ?**
Pour chacun, dire si une architecture hexagonale complète, partielle (domaine + ports de sortie) ou aucune
est justifiée : (a) un service d'authentification qui doit fonctionner en HTTP, gRPC et comme bibliothèque
embarquée ; (b) un tableau de bord interne qui lit trois tables et affiche des graphiques ; (c) un moteur de
calcul de primes d'assurance utilisé par une API et par des traitements par lots nocturnes ; (d) un
service de redimensionnement d'images.

---

## Exercice intermédiaire — Une troisième porte d'entrée : les messages

Les emprunts peuvent aussi arriver par une file de messages (simulée par un `chan []byte` de JSON).

1. Écrire l'adaptateur d'entrée `queuein.Consume(ctx, l core.Lending, msgs <-chan []byte, out io.Writer)`
   qui décode chaque message, appelle le cœur, et écrit le résultat. Il s'arrête quand `ctx` est annulé ou
   le canal fermé.
2. Un message illisible ne doit pas arrêter le consommateur : il est journalisé et ignoré (« message
   empoisonné », leçon 8).
3. Tester l'adaptateur avec un faux de `Lending` (sans le cœur), puis assembler les trois entrées sur le
   même cœur dans un test d'intégration.
4. Quelles lignes du cœur as-tu modifiées ? Qu'en conclure sur le bénéfice du port d'entrée **ici** ?

---

## Défi — Mesurer la cérémonie

Prendre un service réel que tu connais (ou le projet 3/4), et :

1. Compter les conversions de types par cas d'usage (DTO → domaine → lignes → domaine → DTO) et les
   interfaces qui n'ont qu'une implémentation **et** aucun faux dans les tests.
2. Pour chaque interface sans second usage, la supprimer et mesurer l'effet (lignes, lisibilité, tests).
3. Pour chaque conversion entre types identiques, se demander si un seul type suffirait, et à quelle
   condition il faudrait les séparer de nouveau (leçon 2, projet 3 D8).
4. Rédiger en dix lignes une règle d'équipe : « quand ajoute-t-on un port, un adaptateur, un type de
   frontière ? ».

---

## Quiz

1. Qu'est-ce qu'un port d'entrée et un port de sortie ?
2. Qu'est-ce qu'un adaptateur ? Donner un exemple de chaque côté.
3. Quelle est la règle de dépendance ?
4. Qu'ont en commun les architectures hexagonale, en oignon et clean ?
5. Pourquoi, en Go, un adaptateur de sortie peut-il ne pas importer le cœur ?
6. À quoi sert un port d'entrée, et quand peut-on s'en passer ?
7. Citer trois coûts d'une architecture hexagonale complète.
8. Quand ses bénéfices se matérialisent-ils ?
9. Qu'est-ce que l'usage « partiel » recommandé en Go ?
10. Comment vérifier que la règle de dépendance est respectée ?
