# Leçon 1 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — De trois fonctions à une

```go
func Index[T comparable](xs []T, v T) int {
	for i, x := range xs {
		if x == v {
			return i
		}
	}
	return -1
}
```

Avec `any`, le compilateur refuse :

```
invalid operation: x == v (incomparable types in type set)
```

À lire attentivement : il ne dit pas « `T` n'est pas comparable », il dit que **l'ensemble
de types** autorisé par la contrainte contient des types incomparables. `any` admet les
slices, les maps et les fonctions, pour lesquels `==` n'existe pas. Comme le corps est vérifié
une seule fois pour tous les `T` possibles, il suffit d'**un** type interdit pour que
l'opération soit refusée. Le mot « type set » annonce la leçon 2.

## E2 — Ce que l'inférence sait faire

| Appel | Compile ? | `T` | Pourquoi |
|---|---|---|---|
| `First([]string{"a"})` | oui | `string` | `[]T` unifié avec `[]string` |
| `Pair(1, 2)` | oui | `int` | deux constantes non typées entières → type par défaut `int` |
| `Pair(1, 2.5)` | **oui** | `float64` | deux constantes non typées : le compilateur prend la « plus large » des sortes (entier < flottant) |
| `Pair(1, "a")` | non | — | `mismatched types untyped int and untyped string (cannot infer T)` |
| `Pair[int64](1, 2)` | oui | `int64` | explicite ; les constantes sont converties |
| `Zero()` | non | — | `cannot infer T` : `T` n'apparaît dans aucun paramètre |
| `Zero[bool]()` | oui | `bool` | explicite |

Le cas `Pair(1, 2.5)` est celui qui surprend : c'est la même règle des constantes non typées
qu'au niveau 1 (leçon 2), appliquée à l'inférence. Avec des **variables** (`i := 1; f := 2.5;
Pair(i, f)`), l'appel échoue : `int` et `float64` sont incompatibles.

Le message de `Pair(1, "a")` nomme deux **sortes** de constantes (`untyped int`, `untyped
string`), pas deux types : c'est ce qu'il fallait relever.

## E3 — La zéro-valeur

```go
func Last[T any](xs []T) (T, bool) {
	if len(xs) == 0 {
		var zero T
		return zero, false
	}
	return xs[len(xs)-1], true
}
```

`return nil, false` donne :

```
cannot use nil as T value in return statement
```

`nil` n'est une valeur valide que pour les pointeurs, slices, maps, channels, fonctions et
interfaces. `T` peut être `int`. Comme toujours, une seule exception suffit à refuser.

Variante courante, équivalente : un **résultat nommé** `func Last[T any](xs []T) (v T, ok
bool)` puis `return` nu dans la branche vide. C'est un des rares cas où les résultats nommés
rendent le code plus clair.

## E4 — Deux types distincts

`a = b` échoue : `cannot use b (variable of struct type Stack[int64]) as Stack[int] value`.
Les deux instanciations sont **deux types sans lien**.

Pour les mettre dans un même slice :

1. **`[]any{a, b}`** — fonctionne, mais on a tout perdu : il faudra une assertion de type
   pour s'en servir.
2. **Une interface à méthodes** commune aux deux, typée par ce qu'elles savent faire en
   commun :

   ```go
   type sizer interface{ Len() int }
   all := []sizer{&a, &b}   // &a : Len a un récepteur pointeur
   ```

   C'est la réponse idiomatique : on ne garde que le comportement partagé. Noter le `&` —
   la règle de l'ensemble de méthodes (niveau 2, leçon 3) s'applique aux types génériques
   exactement comme aux autres.

## E5 — Le procès d'un générique

| Signature | Verdict | Réécriture |
|---|---|---|
| `Print[T fmt.Stringer](v T)` | **inutile** : `T` n'apparaît qu'une fois, on n'appelle qu'une méthode | `func Print(v fmt.Stringer)` |
| `Keys[K comparable, V any](m map[K]V) []K` | **justifié** : le type de clé ressort dans le résultat | — (c'est `maps.Keys`, leçon 4, en version itérateur) |
| `Close[T io.Closer](c T) error` | **inutile** | `func Close(c io.Closer) error` |
| `Reverse[T any](xs []T)` | **justifié** : un `[]int` n'est pas un `[]any` | — (c'est `slices.Reverse`) |
| `Describe[T any](v T) string` | **inutile** : `fmt.Sprintf` prend déjà un `any` | `func Describe(v any) string` |
| `Clamp[T int \| float64](v, lo, hi T) T` | **justifié** : le résultat a le type des arguments, et `<` est nécessaire | — (contrainte améliorée à la leçon 2) |

`Reverse` est le cas qui fait hésiter : `T` n'apparaît qu'une fois… mais **à l'intérieur de
`[]T`**. La règle « une seule occurrence » vise un `T` **nu**. Un `[]T`, une `map[K]V`, un
`chan T` ont besoin du générique, parce qu'un `[]int` ne se convertit pas en `[]any` sans
recopie.

## Exercice intermédiaire — `Queue[T]`

```go
// ErrFull est retournée par Push quand la file a atteint sa capacité.
var ErrFull = errors.New("file pleine")

// Queue est une file FIFO de capacité fixe, sur un tampon circulaire.
type Queue[T any] struct {
	buf  []T
	head int // indice du prochain élément à sortir
	n    int // nombre d'éléments présents
}

func NewQueue[T any](capacity int) (*Queue[T], error) {
	if capacity <= 0 {
		return nil, fmt.Errorf("capacité invalide : %d", capacity)
	}
	return &Queue[T]{buf: make([]T, capacity)}, nil
}

func (q *Queue[T]) Push(v T) error {
	if q.n == len(q.buf) {
		return ErrFull
	}
	q.buf[(q.head+q.n)%len(q.buf)] = v
	q.n++
	return nil
}

func (q *Queue[T]) Pop() (T, bool) {
	var zero T
	if q.n == 0 {
		return zero, false
	}
	v := q.buf[q.head]
	q.buf[q.head] = zero // libère la référence pour le ramasse-miettes
	q.head = (q.head + 1) % len(q.buf)
	q.n--
	return v, true
}

func (q *Queue[T]) Peek() (T, bool) {
	if q.n == 0 {
		var zero T
		return zero, false
	}
	return q.buf[q.head], true
}

func (q *Queue[T]) Len() int { return q.n }
func (q *Queue[T]) Cap() int { return len(q.buf) }
```

Points de correction :

- **Deux indices, pas trois.** Garder `head` et `n` plutôt que `head` et `tail` évite
  l'ambiguïté classique « `head == tail` : vide ou pleine ? ». `tail` se calcule.
- **Contrainte 5 — remise à zéro.** Le tampon garde une référence vers chaque élément tant
  que la case n'est pas réécrite. Pour une `Queue[*Image]`, une image sortie de la file
  resterait donc **joignable** — le ramasse-miettes ne peut pas la libérer — jusqu'à ce que
  le tampon fasse le tour. C'est une fuite mémoire **sans fuite apparente**. `slices.Delete`
  fait exactement ce nettoyage depuis Go 1.22.
- `NewQueue` cite la valeur : `capacité invalide : 0`. Message en minuscules, sans point
  final (niveau 1, leçon 10).

**Question — rendre `Queue[T]` par valeur.** La struct contient un **slice** : la copier
duplique `head` et `n`, mais **partage** le tableau sous-jacent. Deux copies feraient chacune
leur `Push` dans la **même** case (même `head`, même `n`), la seconde écrasant la première,
et chacune croirait contenir un élément. C'est le deuxième piège du tableau « Copié ou
partagé ? » : un état mi-copié, mi-partagé. Rendre un pointeur, et ne jamais copier une
`Queue`, est la seule conception saine.

## Défi — `Set[T]`

```go
type Set[T comparable] struct{ m map[T]struct{} }

func NewSet[T comparable](items ...T) Set[T] {
	s := Set[T]{m: make(map[T]struct{}, len(items))}
	for _, v := range items {
		s.m[v] = struct{}{}
	}
	return s
}

func (s Set[T]) Add(v T)        { s.m[v] = struct{}{} }
func (s Set[T]) Has(v T) bool   { _, ok := s.m[v]; return ok }
func (s Set[T]) Len() int       { return len(s.m) }

func Union[T comparable](a, b Set[T]) Set[T] {
	out := NewSet[T]()
	for v := range a.m {
		out.Add(v)
	}
	for v := range b.m {
		out.Add(v)
	}
	return out
}

func Intersect[T comparable](a, b Set[T]) Set[T] {
	if a.Len() > b.Len() {
		a, b = b, a
	}
	out := NewSet[T]()
	for v := range a.m {
		if b.Has(v) {
			out.Add(v)
		}
	}
	return out
}

func (s Set[T]) String() string {
	parts := make([]string, 0, len(s.m))
	for v := range s.m {
		parts = append(parts, fmt.Sprint(v))
	}
	sort.Strings(parts)
	return "{" + strings.Join(parts, " ") + "}"
}
```

1. **Récepteur valeur.** Copier un `Set` copie le **pointeur** vers la table de la map : la
   copie et l'original partagent la même table. `Add` modifie donc bien l'ensemble. Mais
   `var s Set[int]; s.Add(1)` **panique** : `assignment to entry in nil map`. Deux réponses :
   soit documenter que seul `NewSet` produit un `Set` valide, soit passer `Add` en récepteur
   pointeur et initialiser la map paresseusement (`if s.m == nil { s.m = make(…) }`) — ce qui
   impose alors des récepteurs pointeurs **partout** par cohérence. La seconde rend la
   zéro-valeur utilisable, ce qui est le style préféré en Go.
2. **`Union` en fonction.** Elle est **symétrique** : `Union(a, b)` et `Union(b, a)` sont
   le même ensemble, aucun des deux n'a de rôle privilégié. `a.Union(b)` suggère que `a`
   est modifié ou qu'il est « l'objet principal ». C'est un choix de lisibilité.
3. **Parcourir le plus petit.** Le coût est proportionnel à la taille de l'ensemble parcouru,
   les recherches dans l'autre étant en temps constant. Intersecter 10 éléments avec 1 million
   coûte 10 recherches dans un sens, 1 million dans l'autre.
4. `fmt.Println` d'un `Set` sans `String()` affiche la map, et `fmt` **trie** les clés de
   map à l'affichage depuis Go 1.12 : la sortie est en fait stable. Mais c'est une propriété
   de `fmt`, pas de la map ; le `String()` explicite rend le contrat visible et contrôle le
   format. Le tri sur `fmt.Sprint(v)` trie le texte, pas les valeurs : `{10 2 3}` pour des
   entiers. La leçon 4 donnera un vrai tri générique avec `cmp.Ordered`.

## Quiz

1. Entre crochets, **avant** la liste des paramètres ordinaires : `func F[T any](x T)`.
2. Parce que le corps est vérifié **une fois** pour tous les types permis ; si la contrainte
   admet un seul type sans `==`, l'opération est refusée.
3. Fournir les arguments de type (explicitement ou par inférence) pour obtenir une fonction
   ou un type ordinaire, non générique.
4. Quand `T` n'apparaît que dans le résultat (ou nulle part dans les paramètres), et quand des
   arguments imposent des types contradictoires.
5. `var zero T`.
6. Aucun : deux types distincts, non affectables l'un à l'autre.
7. Dans le premier, `x` est une **interface** (type perdu, assertion nécessaire) ; dans le
   second, `x` a le type **concret** de l'appel, connu du compilateur.
8. Non. Go compile une version par **forme mémoire** (*GC shape*) et passe un dictionnaire
   pour le reste ; les performances sont du même ordre qu'une interface. On mesure, on ne
   suppose pas.
9. Un `T` **nu** qui n'apparaît qu'une seule fois dans la signature.
10. Justifié : fonctions sur `[]T`/`map[K]V`, structures de données, résultat du même type que
    l'argument. Injustifié : seulement des appels de méthode, seulement deux types concrets,
    « au cas où ».
