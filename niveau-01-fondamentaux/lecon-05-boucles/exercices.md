# Leçon 5 — Exercices

Code dans `mes-solutions/niveau-01/lecon-05/`.

> Toujours pas de collections : tout se fait avec des entiers et des boucles.
> Les exercices sur `range` appliqué aux slices, maps et chaînes arrivent aux leçons 6 à 8.

## Exercices faciles

**E1 — FizzBuzz.**
Afficher 1 à 100, avec `Fizz` pour les multiples de 3, `Buzz` pour 5, `FizzBuzz` pour 15.
**Contraintes : `for i := range` (forme 1.22) et un `switch` sans expression** — aucune
cascade de `if/else`. Attention à l'ordre des cas.

**E2 — Les quatre formes.**
Écrire **quatre fois** le calcul de la somme des entiers de 1 à 100, une par forme de `for`.
Vérifier que les quatre donnent 5050. Laquelle est la plus lisible ici ? La plus adaptée ?

**E3 — Chiffres d'un nombre.**
Sans convertir en chaîne, écrire :
- `DigitCount(n int) int` — le nombre de chiffres ;
- `DigitSum(n int) int` — la somme des chiffres ;
- `Reverse(n int) int` — le nombre à l'envers (`1234` → `4321`).
Gérer les nombres négatifs et `0`. *(Indice : `%10` et `/10` sont les seuls outils
nécessaires.)*

**E4 — Labels.**
Chercher le plus petit entier `n > 1` tel que `n` soit divisible par tous les entiers de 2 à
10. Utiliser deux boucles imbriquées et un label pour sortir des deux.
Puis réécrire la même recherche **sans label**, en extrayant une fonction. Comparer et dire
laquelle est préférable.

**E5 — Le piège du non signé.**
Écrire une boucle décroissante avec `uint` qui ne se termine jamais, la lancer, l'interrompre
avec Ctrl-C, expliquer pourquoi, puis la corriger de **deux façons différentes**.

---

## Exercice intermédiaire — `numbers`

Un outil d'exploration numérique piloté par la ligne de commande.

```
$ go run . collatz 27
111 étapes, valeur maximale 9232

$ go run . primes 50
2 3 5 7 11 13 17 19 23 29 31 37 41 43 47
15 nombres premiers

$ go run . fib 10
0 1 1 2 3 5 8 13 21 34

$ go run . perfect 10000
6 28 496 8128
```

**Contraintes :**
1. Quatre sous-commandes : `collatz`, `primes`, `fib`, `perfect` (nombres parfaits : égaux à la somme de leurs diviseurs stricts). Une commande inconnue affiche l'aide sur `os.Stderr` et sort avec le code 1.
2. Chaque calcul vit dans sa propre fonction. Aucune ne stocke de collection : les résultats sont **affichés au fil de l'eau** par la fonction appelante. *(Contrainte artificielle, imposée parce que les slices arrivent à la leçon 6 — et instructive : elle force à réfléchir à qui affiche quoi.)*
3. `primes` teste la primalité par divisions d'essai, en s'arrêtant à la **racine carrée**. Mesurer l'écart avec une version qui va jusqu'à `n`.
4. `fib` doit fonctionner jusqu'à n = 90 sans déborder ; au-delà, il retourne une erreur explicite plutôt qu'un résultat faux.
5. `collatz` se protège d'une boucle qui ne se terminerait pas : un plafond d'itérations avec une erreur claire. *(La conjecture n'est pas démontrée — un programme de production ne fait pas confiance à une conjecture.)*
6. Aucune boucle ne dépasse quinze lignes.

*La contrainte 4 est le vrai piège : `fib(93)` déborde un `int64`. Comment le détecter
**avant** de produire un résultat faux ? Rappel : leçon 2, exercice `SafeAdd`.*

---

## Défi

**a) Motifs.**
Sans le paquet `strings`, uniquement avec des boucles imbriquées et `fmt.Print`, dessiner :
un triangle rectangle, un triangle isocèle centré, un losange, puis le triangle de Pascal
sur 10 lignes avec les colonnes alignées. Le dernier est le seul vrai défi : l'alignement
demande de connaître la largeur du plus grand nombre **avant** d'afficher.

**b) PGCD et fractions.**
Écrire `GCD(a, b int) int` par l'algorithme d'Euclide, en version **itérative** puis
**récursive**. Puis `Simplify(num, den int) (int, int, error)` qui réduit une fraction,
gère les signes (le dénominateur reste positif) et refuse un dénominateur nul.
Combien d'itérations l'algorithme prend-il au pire ? *(Chercher « nombres de Fibonacci
pire cas Euclide » — le résultat est élégant.)*

**c) Boucles et localité mémoire.**
Déclarer `var m [1000][1000]int` (un tableau, pas un slice : c'est déjà connu depuis la
leçon 2). Le parcourir de deux façons : `m[i][j]` avec `j` variant le plus vite, puis
`m[j][i]`. Chronométrer avec `time.Now()` et `time.Since`.
**Prédire lequel est le plus rapide et de quel facteur avant de mesurer.** Expliquer le
résultat.
*(Le rapport dépasse souvent 5×. Premier contact avec le raisonnement du niveau 11 : la
complexité algorithmique est identique, la performance ne l'est pas du tout.)*

---

## Quiz

1. Combien de mots-clés de boucle Go possède-t-il ? Lesquels ?
2. Comment écrit-on un `while` en Go ? Un `do…while` ?
3. Qu'affiche `for i := range 3 { fmt.Print(i) }` ? Depuis quelle version de Go ?
4. Que fait `break` dans une boucle imbriquée, sans label ?
5. À quoi sert un label devant une boucle ? Où `gofmt` le place-t-il ?
6. Qu'a changé Go 1.22 concernant la variable de boucle ?
7. Qu'est-ce qui pilote cette sémantique dans un module donné ?
8. Pourquoi `for i := uint(0); i < n; i--` ne se termine-t-il jamais ?
9. Le compilateur Go supprime-t-il une boucle sans effet ?
10. Sur quels types `range` fonctionne-t-il ? *(Citer ceux déjà vus et ceux qui arrivent.)*
