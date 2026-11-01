# Leçon 2 — Exercices

Code dans `mes-solutions/niveau-03/lecon-02/`.

## Exercices faciles

**E1 — Le `~` manquant.**
Écrire `type Float interface{ float32 | float64 }` et `func Avg[T Float](xs []T) T`.
L'appeler avec `[]float64` puis avec `[]Celsius` (`type Celsius float64`). Recopier le
message d'erreur. **Quel type le compilateur nomme-t-il, et dans quel ensemble dit-il qu'il
manque ?** Corriger.

**E2 — Prédire, puis mesurer.**
Pour chaque fonction, dire si elle compile, et sinon quelle opération est refusée :

```go
func A[T ~int | ~float64](x T) T  { return x % 2 }
func B[T ~int | ~uint](x T) T     { return x % 2 }
func C[T ~string | ~[]byte](x T) int { return len(x) }
func D[T ~int | ~string](x T) int { return len(x) }
func E[T ~int | ~string](x T) T   { return x + x }
func F[T comparable](x T) bool    { return x < x }
```

`E` est la question piège : réfléchir à ce que `+` veut dire pour chaque type de l'ensemble.

**E3 — Contrainte, pas type.**
Essayer de déclarer `var xs []cmp.Ordered`. Recopier le message. Expliquer en deux phrases
pourquoi le langage l'interdit (penser à ce que vaudrait `xs[0] < xs[1]`).

**E4 — `comparable` qui panique.**
Écrire `func Count[T comparable](xs []T, v T) int`. L'appeler avec un `[]any` qui contient
`[]int{1}`. Ça compile ? Ça s'exécute ? **Où** le programme s'arrête-t-il, et quel est le
message exact ?

**E5 — NaN.**
Écrire deux versions de `Max` sur un `[]T` : l'une avec `>`, l'autre avec `cmp.Compare`.
Les appeler sur `[]float64{1, math.NaN(), 3}` puis sur `[]float64{math.NaN(), 1, 3}`.
Remplir le tableau :

| Entrée | `>` | `cmp.Compare` | `slices.Max` *(à essayer)* |
|---|---|---|---|
| `{1, NaN, 3}` | | | |
| `{NaN, 1, 3}` | | | |

Laquelle des trois est **déterministe** ? Laquelle « propage » NaN ?

---

## Exercice intermédiaire — `stats`, un petit paquet de statistiques

Créer un paquet `stats` (dans son propre répertoire, importé par un `main`) avec :

```go
func Sum[T Number](xs []T) T
func Mean[T Number](xs []T) (float64, error)
func Median[T Number](xs []T) (float64, error)
func MinMax[T cmp.Ordered](xs []T) (lo, hi T, err error)
func Histogram[T cmp.Ordered](xs []T) map[T]int
```

**Contraintes :**
1. `Number` est définie dans le paquet. Doit-elle être exportée ? **Trancher** en une phrase :
   un utilisateur qui veut écrire sa propre fonction `Variance` sur les mêmes types en a-t-il
   besoin ?
2. Une liste vide produit une **erreur sentinelle** `ErrEmpty`, testable par `errors.Is`.
3. `Median` ne doit **pas** modifier le slice de l'appelant. *(Rappel niveau 1, leçon 6 : un
   slice passé à une fonction partage son tableau.)*
4. `Sum` sur un `[]int8` de valeurs `100, 100` : que vaut le résultat ? Documenter ce
   comportement dans le commentaire de `Sum` — l'appelant doit savoir que le débordement est
   silencieux.
5. `Mean` doit fonctionner sur `[]time.Duration`.
6. Le `main` montre chaque fonction sur des `int`, des `Celsius` et des `string` quand c'est
   possible. Pour les fonctions qui refusent les `string`, laisser l'appel en commentaire
   avec le message du compilateur.

---

## Défi — Une contrainte qui exige une méthode

On veut trier des valeurs de n'importe quel type qui sait se **comparer** à un autre de son
type :

```go
type Comparer[T any] interface {
	Compare(other T) int
}

func SortSlice[T Comparer[T]](xs []T)
```

1. Pourquoi la contrainte s'écrit-elle `Comparer[T]` et non `Comparer` ? *(Une contrainte peut
   elle-même être générique et **se référer au paramètre qu'elle contraint**.)*
2. Implémenter `SortSlice` avec un tri par insertion écrit à la main.
3. Définir `type Version struct{ Major, Minor, Patch int }` avec une méthode `Compare` et
   trier `{1 2 0} {1 10 0} {1 9 3} {0 9 9}`. Sortie attendue : `[0.9.9 1.2.0 1.9.3 1.10.0]`
   (donc avec une méthode `String`).
4. Pourquoi un tri par `String()` donnerait-il un mauvais résultat ici ?
5. Est-ce que `int` satisfait `Comparer[int]` ? Qu'en conclure sur la **frontière** entre
   contrainte par méthode et contrainte par ensemble de types ?

---

## Quiz

1. Dans le vocabulaire de Go 1.18+, que *définit* une interface ?
2. Que signifie `~float64` ? Pourquoi `~Celsius` est-il refusé ?
3. Quelle est la règle qui décide si une opération est permise sur un `T` ?
4. Pourquoi ne peut-on pas déclarer `var n cmp.Ordered` ?
5. Que garantit exactement `comparable` ? Que ne garantit-il pas ?
6. Que contient `cmp.Ordered` ? Les chaînes en font-elles partie ?
7. Pourquoi `cmp.Compare` est-il préférable à `<` dans du code générique ?
8. Comment exprimer « un entier qui a une méthode `String()` » ?
9. Peut-on accéder à un champ de struct à travers une contrainte ?
10. Que signifie une ligne vide entre deux éléments d'une interface de contrainte : union ou intersection ?
