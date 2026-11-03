# Leçon 6 — Exercices

Code dans `mes-solutions/niveau-11/lecon-06/`, en partant du paquet `mem` de l'exemple.

## Exercices faciles

**E1 — La croissance d'`append`.**
Afficher `cap(s)` après chaque `append` qui réalloue, pour n allant jusqu'à 10 000. Quel facteur de
croissance observe-t-on pour les petites slices ? pour les grandes ? Combien de réallocations pour
1 000 000 d'éléments (le benchmark en compte 38) ?

**E2 — Slice ou map, avec d'autres clés.**
Reprendre `BenchmarkLookup` avec des clés `int` au lieu de `string`, et deux distributions : élément cherché
en **première** position, et en **dernière**. Jusqu'à quelle taille la recherche linéaire gagne-t-elle
maintenant ? L'hypothèse du cours (« avec des clés entières, la slice gagnerait sans doute ») tient-elle ?

**E3 — Les pointeurs dans l'ordre.**
Dans `Pointers`, supprimer le mélange : les livres sont alloués à la suite et les pointeurs restent dans
l'ordre d'allocation. Mesurer `SumPointers`. Que reste-t-il de l'écart avec `SumValues`, et pourquoi ?

**E4 — La taille de la matrice.**
Mesurer `SumRows` et `SumCols` pour N = 64, 256, 1 024, 2 048, 4 096. Jusqu'à quelle taille l'écart est-il
faible ? Le relier à la taille des caches de ta machine (`lscpu` sous Linux).

**E5 — Compteurs locaux.**
Écrire `CountLocal` : chaque goroutine incrémente une variable **locale** et n'écrit le total dans le
tableau partagé qu'à la fin. Comparer avec `CountShared` et `CountPadded`.

---

## Exercice intermédiaire — Un index en mémoire

Un service garde en mémoire 1 000 000 de livres et répond à « les livres d'un auteur, triés par année »
(10 000 auteurs).

1. Version A : `map[string][]*Book` (auteur → pointeurs vers les livres).
2. Version B : tous les livres dans une `[]Book` triée par (auteur, année), et une
   `map[string]struct{ start, end int32 }` qui donne la tranche de chaque auteur.
3. Mesurer : mémoire totale (`runtime.ReadMemStats` après un `runtime.GC()`), temps de construction, temps
   d'une requête, temps de parcours de **tous** les livres, et part du GC pendant une charge d'allocations
   parallèle (leçon 3).
4. Quelle version choisir pour un index en lecture seule reconstruit chaque nuit ? Et si les livres changent
   souvent ?

---

## Défi — Le faux partage dans un vrai programme

1. Écrire un compteur de requêtes par route pour un serveur HTTP : `[16]atomic.Int64` indexé par route.
   Mesurer sous charge (8 goroutines qui incrémentent des routes différentes) avec et sans rembourrage.
2. Comparer avec un compteur **par processeur** (une case par goroutine de travail, sommées à la lecture) :
   comment savoir sur quel « processeur » on tourne sans `runtime` interne ? (Indice : on ne peut pas ;
   concevoir autrement, avec un compteur par worker.)
3. Lire la documentation de `sync.Pool` et d'`expvar` : leurs implémentations évitent-elles le faux
   partage ? Comment le vérifier par une mesure ?

---

## Quiz

1. Pourquoi l'accès séquentiel à la mémoire est-il beaucoup plus rapide que l'accès aléatoire ?
2. Que coûte un `append` sans préallocation ?
3. Pourquoi une `[]Book` est-elle plus rapide à parcourir qu'une `[]*Book` ?
4. Quelle règle suivre pour l'ordre des boucles sur un tableau à deux dimensions ?
5. Une petite slice est-elle toujours plus rapide qu'une map ?
6. Pourquoi l'ordre des champs change-t-il la taille d'une struct ?
7. Qu'est-ce que le faux partage, et pourquoi le détecteur de *data races* ne le signale-t-il pas ?
8. Deux remèdes au faux partage ?
9. Qu'ont changé les maps de Go 1.24 ?
10. Quand ces optimisations ne valent-elles pas la peine ?
