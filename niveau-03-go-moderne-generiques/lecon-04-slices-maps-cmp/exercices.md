# Leçon 4 — Exercices

Code dans `mes-solutions/niveau-03/lecon-04/`. **Interdit** dans cette leçon : écrire une
boucle `for` là où une fonction de `slices` ou `maps` fait le travail — sauf si tu justifies
par écrit que la boucle est plus claire.

## Exercices faciles

**E1 — Ce que voit l'ancien slice.**
Exécuter, puis expliquer **chaque** valeur affichée :

```go
s := []int{10, 20, 30, 40, 50}
a := s[:2]
t := slices.Delete(s, 1, 3)
fmt.Println(t, s, a)
u := slices.Insert(t, 1, 99)
fmt.Println(u, s, len(u), cap(u))
v := slices.Insert(u, 0, 1, 2, 3)
fmt.Println(v, s)
```

À quel moment `v` cesse-t-il de partager le tableau de `s` ? Comment le prouver avec `cap` ?

**E2 — Remplacer `sort`.**
Réécrire avec `slices` et `cmp` :

```go
sort.Slice(people, func(i, j int) bool { return people[i].Age < people[j].Age })
sort.Strings(names)
i := sort.SearchInts(sorted, 42)
```

Puis expliquer en une phrase la différence de **signature** entre la fonction passée à
`sort.Slice` et celle passée à `slices.SortFunc`.

**E3 — Le comparateur qui déborde.**
Trier `[]int{math.MinInt, 1, -1, math.MaxInt}` avec `func(a, b int) int { return a - b }`,
puis avec `cmp.Compare`. Comparer les deux sorties. **Quelle comparaison précise** a donné un
signe faux ? Calculer à la main sa valeur.

**E4 — Dédoublonner.**
Écrire `Unique[S ~[]E, E cmp.Ordered](s S) S` qui rend les valeurs distinctes, triées, **sans
modifier** `s`. Deux lignes suffisent. Vérifier que `type Tags []string` ressort bien en `Tags`
(`%T`).

**E5 — `maps.Clone` est superficiel.**
Créer `m := map[string][]int{"a": {1, 2}}`, la cloner en `c`, puis faire `c["a"][0] = 99` et
`c["b"] = []int{3}`. Qu'est-ce qui a changé dans `m` ? Écrire `DeepClone` qui rend une copie
réellement indépendante.

---

## Exercice intermédiaire — Classement d'un tournoi

Données : une liste de matchs `Match{Home, Away string; HomeGoals, AwayGoals int}`.

Produire le classement :

```
#   équipe       J  G  N  P   bp  bc  diff  pts
1   Lyon         4  3  1  0   9   3   +6    10
2   Nantes       4  3  1  0   7   2   +5    10
…
```

**Règles :** victoire 3 points, nul 1, défaite 0. Départage, dans l'ordre : points,
différence de buts, buts marqués, **puis ordre alphabétique** du nom.

**Contraintes :**
1. Agréger dans une `map[string]*Stats`, puis extraire un slice et le trier avec **un seul**
   `slices.SortFunc` utilisant `cmp.Or`.
2. Pour extraire les valeurs de la map sans encore connaître les itérateurs : une boucle
   `range` est autorisée ici. *(La leçon 5 la remplacera par `slices.Collect(maps.Values(m))`.)*
3. Le classement doit être **déterministe** : lancer le programme 20 fois
   (`for i in $(seq 20); do go run . ; done | sort | uniq -c`) et vérifier qu'une seule sortie
   apparaît.
4. Une équipe qui joue contre elle-même est une **erreur** de données, signalée avec le
   numéro du match.
5. La diff s'affiche avec son signe (`+6`, `0`, `-3`). Quel verbe de `fmt` le fait
   directement ? *(Niveau 1, leçon 3.)*

Données de test (à mettre dans le `main`) : au moins 6 équipes et 10 matchs, avec **au
moins deux égalités parfaites** sur les trois premiers critères — sinon le dernier critère
n'est jamais testé.

---

## Défi — `GroupBy` et `TopN`

```go
func GroupBy[S ~[]E, E any, K comparable](s S, key func(E) K) map[K]S
func TopN[S ~[]E, E any](s S, n int, cmp func(a, b E) int) S
```

1. `GroupBy` : chaque groupe doit garder l'**ordre d'origine** des éléments.
2. `TopN` rend les `n` plus grands selon `cmp`, triés décroissants, **sans modifier** `s`.
   Si `n > len(s)`, rendre tout. Si `n <= 0`, rendre un slice vide non nil.
3. Première version : cloner et trier. Complexité ? Seconde version, pour `n` petit devant
   `len(s)` : maintenir les `n` meilleurs au fil d'un seul parcours avec `slices.Insert` et
   `slices.BinarySearchFunc`. Complexité ?
4. Sur un slice d'un million d'entiers aléatoires (`math/rand/v2`, `rand.IntN`) et `n = 10`,
   mesurer les deux versions avec `time.Now()` / `time.Since`. *(Mesure grossière — la vraie
   méthode viendra avec les benchmarks du niveau 5.)*

---

## Quiz

1. Pourquoi les fonctions de `slices` sont-elles déclarées `[S ~[]E, E …]` plutôt que `s []E` ?
2. Que deviennent les cases libérées par `slices.Delete` depuis Go 1.22, et pourquoi ?
3. Que doit rendre une fonction de comparaison pour `SortFunc` ?
4. Pourquoi ne pas écrire `return a - b` ?
5. Comment trier sur trois critères sans `if` ?
6. Différence entre `SortFunc` et `SortStableFunc` ? Quand la stabilité est-elle inutile ?
7. Que fait `Compact` sur `[1 2 1]` ?
8. Que rend `BinarySearch` quand l'élément est absent ?
9. Pourquoi `"Zoé" < "abricot"` est-il vrai ?
10. `maps.Clone` : copie profonde ou superficielle ? Que rend `maps.Clone(nil)` ?
