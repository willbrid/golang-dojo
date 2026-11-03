# Leçon 4 — Exercices

Code dans `mes-solutions/niveau-10/lecon-04/`.

## Exercices faciles

**E1 — Casser la conformité.**
Renommer `Store.Overdue` en `Store.Late`. Où le compilateur signale-t-il l'erreur, avec la ligne
`var _ overdueLister = (*Store)(nil)` ? Et sans elle ? Dans quel cas la ligne est-elle vraiment utile ?

**E2 — Le faux partiel.**
Écrire un test de `Report` avec `partialFake`. Lire le message d'échec : indique-t-il la cause ? Corriger en
passant à `Report` un faux qui n'implémente que `loanCounter`.

**E3 — `io` comme modèle.**
Écrire `func CountLines(r io.Reader) (int, error)`. L'appeler avec un fichier, une chaîne
(`strings.NewReader`), le corps d'une réponse HTTP (`httptest`), et un `io.LimitReader`. Combien de types
différents ta fonction accepte-t-elle, sans le savoir ?

**E4 — Rendre une interface.**
Changer `NewStore` pour qu'il rende une interface `StoreInterface` avec ses trois méthodes. Ajouter ensuite
une méthode `Remove` à `Store`. Que doit faire un appelant qui a besoin de `Remove` ? Que se passe-t-il
pour une implémentation tierce de `StoreInterface` si l'on ajoute `Remove` à l'interface ?

**E5 — Pollution.**
On te donne ce code : `type UserService interface { … 14 méthodes … }` déclaré dans `users/service.go`,
une seule implémentation `userService` non exportée, et un *mock* généré de 400 lignes utilisé par trois
tests qui n'appellent chacun qu'une ou deux méthodes. Proposer, en cinq lignes, une refactorisation, et
dire ce que deviennent les tests.

---

## Exercice intermédiaire — Découvrir les interfaces du service de prêts

Reprendre le service des leçons 1 et 2 et lui ajouter deux consommateurs :

1. une tâche de nuit `SendReminders` qui envoie un rappel par adhérent en retard ;
2. un rapport mensuel `MonthlyReport` (nombre d'emprunts et de retours par jour).

Contraintes : chaque consommateur déclare son interface (une ou deux méthodes) ; le stockage en mémoire
et le stockage PostgreSQL les satisfont sans modification de leur part ; chaque test utilise un faux de
moins de dix lignes. Vérifier avec `go list` que ni la tâche de nuit ni le rapport n'importent le paquet
du stockage. Combien d'interfaces as-tu écrites, et combien de méthodes au total ? Comparer avec une
unique interface `LoanStore` qui couvrirait tous les besoins.

---

## Défi — Décorer par composition

Avec la petite interface `overdueLister` :

1. Écrire un **cache** `cachedOverdue` qui satisfait `overdueLister`, enveloppe un autre `overdueLister`,
   et garde le résultat 1 minute (horloge injectée). Le service ne change pas.
2. Écrire un **compteur** `countingOverdue` qui compte les appels et les erreurs (sûr en concurrence).
3. Les **empiler** : compteur → cache → stockage. Dans quel ordre les empiler pour compter les appels au
   **stockage**, et dans quel ordre pour compter les appels **reçus** ?
4. Tester chaque décorateur seul, avec un faux en une ligne.
5. Pourquoi ce patron (décorateur) est-il si simple en Go avec des interfaces d'une méthode, et si lourd
   avec une interface de quinze ?

---

## Quiz

1. Qui déclare l'interface en Go, et pourquoi est-ce possible ?
2. Quelle taille pour une interface, et pourquoi ?
3. Que signifie « accepter des interfaces, rendre des structs » ?
4. Quand est-il légitime de rendre une interface ?
5. Quand extraire une interface ?
6. Quels sont les signes de la pollution par les interfaces ?
7. Qu'est-ce qu'un adaptateur fonction, et quel exemple la bibliothèque standard en donne-t-elle ?
8. Pourquoi le faux partiel par embarquement est-il dangereux ?
9. À quoi sert `var _ I = (*T)(nil)` ?
10. Quand une simple fonction remplace-t-elle une interface ?
