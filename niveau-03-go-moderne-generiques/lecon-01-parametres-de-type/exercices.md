# Leçon 1 — Exercices

Code dans `mes-solutions/niveau-03/lecon-01/`. Avant de rendre : `gofmt -l .`, `go vet ./...`,
et **comparer sa sortie à celle attendue** avec `go run . | cat -A`.

## Exercices faciles

**E1 — De trois fonctions à une.**
Écrire `IndexInt([]int, int) int`, `IndexString([]string, string) int` et
`IndexFloat([]float64, float64) int`, qui rendent l'indice de la première occurrence ou `-1`.
Puis les remplacer par **une** fonction générique `Index`. Quelle contrainte faut-il, et
pourquoi `any` ne suffit-il pas ? Recopier **mot pour mot** le message du compilateur obtenu
avec `any`.

**E2 — Ce que l'inférence sait faire.**
Pour chacun des appels suivants, prédire **par écrit** s'il compile, et si oui la valeur de
`T`, **puis** vérifier :

```go
func First[T any](xs []T) T { return xs[0] }
func Pair[T any](a, b T) []T { return []T{a, b} }
func Zero[T any]() T { var z T; return z }

First([]string{"a"})
Pair(1, 2)
Pair(1, 2.5)
Pair(1, "a")
Pair[int64](1, 2)
Zero()
Zero[bool]()
```

Pour chaque échec, relever **la valeur ou le type** que nomme le compilateur.

**E3 — La zéro-valeur.**
Écrire `Last[T any](xs []T) (T, bool)` qui rend le dernier élément, ou la zéro-valeur et
`false` si le slice est vide. Essayer d'abord d'écrire `return nil, false` : quel message
obtient-on ? Corriger.

**E4 — Deux types distincts.**
Avec la `Stack[T]` du cours : déclarer `a := Stack[int]{}` et `b := Stack[int64]{}`.
Essayer `a = b`. Puis essayer de mettre `a` et `b` dans un même slice. Qu'est-ce qui le
permettrait ? *(Une réponse qui fonctionne existe ; une autre qui fonctionne et reste typée
aussi. Chercher les deux.)*

**E5 — Le procès d'un générique.**
Pour chacune de ces signatures, dire si le générique est **justifié** ou **inutile**, et
réécrire les inutiles :

```go
func Print[T fmt.Stringer](v T)
func Keys[K comparable, V any](m map[K]V) []K
func Close[T io.Closer](c T) error
func Reverse[T any](xs []T)
func Describe[T any](v T) string   // corps : return fmt.Sprintf("%v", v)
func Clamp[T int | float64](v, lo, hi T) T
```

---

## Exercice intermédiaire — `Queue[T]`, une file bornée

Implémenter une **file FIFO à capacité fixe** sur un tampon circulaire.

```go
type Queue[T any] struct { /* à concevoir */ }

func NewQueue[T any](capacity int) (*Queue[T], error)
func (q *Queue[T]) Push(v T) error      // erreur si pleine
func (q *Queue[T]) Pop() (T, bool)      // false si vide
func (q *Queue[T]) Peek() (T, bool)
func (q *Queue[T]) Len() int
func (q *Queue[T]) Cap() int
```

**Contraintes :**
1. Le stockage est un slice alloué **une seule fois** dans `NewQueue`. Aucun `append`.
2. `Push` et `Pop` sont en temps constant : pas de décalage des éléments.
3. `NewQueue` refuse une capacité ≤ 0 avec un message qui cite la valeur reçue.
4. L'erreur « file pleine » est une **sentinelle** exportée (niveau 2, leçon 6) pour que
   l'appelant puisse la tester avec `errors.Is`.
5. Après un `Pop`, la case libérée doit être remise à la zéro-valeur. **Pourquoi ?** Pense à
   une `Queue[*Image]` où chaque image fait 50 Mo. *(La réponse touche au ramasse-miettes,
   vu en détail au niveau 11 — l'intuition suffit ici.)*
6. Écrire un `main` qui remplit une `Queue[string]` de capacité 3, en retire 2, en rajoute
   3, et affiche l'état après chaque opération. Sortie attendue de la dernière ligne :
   `len=3 cap=3`. Le dernier `Push` doit échouer.

**Question à trancher par écrit :** `NewQueue` rend `*Queue[T]`. Pourrait-on rendre une
`Queue[T]` par valeur ? Que se passerait-il si l'appelant la copiait, puis faisait un `Push`
sur chaque copie ? *(Relire le tableau [Copié ou partagé ?](../../ressources/copie-ou-partage.md).)*

---

## Défi — `Set[T]`

Écrire un type ensemble générique, sur une map :

```go
type Set[T comparable] struct { m map[T]struct{} }

func NewSet[T comparable](items ...T) Set[T]
func (s Set[T]) Add(v T)
func (s Set[T]) Has(v T) bool
func (s Set[T]) Len() int
func Union[T comparable](a, b Set[T]) Set[T]
func Intersect[T comparable](a, b Set[T]) Set[T]
```

1. Pourquoi `Add` peut-il avoir un récepteur **valeur** et modifier quand même l'ensemble ?
   Que se passe-t-il avec `var s Set[int]; s.Add(1)` ? Comment rendre la zéro-valeur utilisable ?
2. Pourquoi `Union` est-elle une **fonction** et pas une méthode `a.Union(b)` ? *(Elle
   pourrait être une méthode — chercher l'argument de symétrie, pas une limite du langage.)*
3. `Intersect` doit parcourir le **plus petit** des deux ensembles. Pourquoi ?
4. Afficher un `Set[string]` avec `fmt.Println` : la sortie est-elle stable d'une exécution à
   l'autre ? Ajouter une méthode `String()` qui la rend stable. *(On n'a pas encore vu le
   paquet `slices` : un tri à la main ou `sort.Strings` suffit.)*

---

## Quiz

1. Où se déclarent les paramètres de type d'une fonction ?
2. Pourquoi faut-il une contrainte pour écrire `a == b` sur deux `T` ?
3. Qu'appelle-t-on « instancier » une fonction générique ?
4. Dans quels cas l'inférence de type échoue-t-elle ?
5. Comment obtient-on la zéro-valeur d'un `T` ?
6. `Stack[int]` et `Stack[int64]` : quel lien entre ces deux types ?
7. Quelle est la différence entre `func F(x any)` et `func F[T any](x T)` ?
8. Un générique est-il plus rapide qu'une interface ? Comment le compilateur Go les compile-t-il ?
9. Quel est le signal qu'un paramètre de type est inutile ?
10. Citer deux situations où le générique est justifié, et deux où il ne l'est pas.
