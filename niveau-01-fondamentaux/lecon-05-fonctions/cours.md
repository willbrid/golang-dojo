# Leçon 5 — Fonctions

## Objectifs

1. Écrire des fonctions à retours multiples et savoir pourquoi Go a fait ce choix.
2. Utiliser les paramètres variadiques et comprendre ce qu'ils coûtent.
3. Maîtriser la **portée lexicale** et les **closures**, y compris leur piège principal.
4. Traiter les fonctions comme des valeurs : les passer, les retourner, les stocker.

## Explication

### Signature : le nom avant le type

```go
func add(a int, b int) int          { return a + b }
func add(a, b int) int              { return a + b }   // types groupés
func divmod(a, b int) (int, int)    { return a / b, a % b }
func noReturn(msg string)           { fmt.Println(msg) }
```

Une fonction Go est **toujours** appelée par valeur : ses arguments sont copiés. Copier un
`int` est gratuit ; copier une struct de 200 octets ne l'est pas, d'où les pointeurs
(leçon 9). Copier un slice, une map ou une string copie leur **descripteur** (quelques
mots machine), jamais leur contenu.

### Retours multiples : le remplacement des exceptions

```go
func divide(a, b float64) (float64, error) {
	if b == 0 {
		return 0, errors.New("division par zéro")
	}
	return a / b, nil
}

result, err := divide(10, 2)
if err != nil {
	return err
}
```

C'est **la** décision de conception structurante de Go. Pas d'exceptions : une fonction
qui peut échouer retourne son erreur comme dernière valeur, et l'appelant la traite
immédiatement. Conséquences :

- On voit *dans la signature* si une fonction peut échouer.
- On voit *à l'appel* que l'erreur est traitée — ou volontairement ignorée avec `_`.
- Il n'existe aucun chemin d'exécution invisible ; pas de saut de pile surprise.
- En échange : c'est **verbeux**. Trois lignes de `if err != nil` pour un appel. Les
  détracteurs de Go n'ont pas tort sur ce point ; c'est un compromis assumé,
  lisibilité contre concision. La leçon 8 approfondira.

Convention absolue : **`error` est toujours le dernier résultat**. L'idiome `(T, bool)`
(comme `v, ok := m[k]`) sert quand l'absence n'est pas une erreur.

### Résultats nommés : utiles, mais à manier avec prudence

```go
func split(sum int) (x, y int) {
	x = sum * 4 / 9
	y = sum - x
	return          // « naked return » : retourne x et y
}
```

Les résultats nommés servent surtout à **documenter** une signature ambiguë —
`func find() (index int, found bool)` se lit mieux que `(int, bool)`. Leur second usage,
plus technique, est de permettre à un `defer` de modifier la valeur retournée (niveau 2).

En revanche le `return` nu est déconseillé au-delà de quelques lignes : il oblige le
lecteur à remonter chercher ce qui est retourné. Nommer les résultats, oui ; retourner
à vide dans une fonction de 40 lignes, non.

### Fonctions variadiques

```go
func sum(nums ...int) int {
	total := 0
	for _, n := range nums {   // nums est un []int
		total += n
	}
	return total
}

sum()                    // 0
sum(1, 2, 3)             // 6
xs := []int{1, 2, 3}
sum(xs...)               // 6 — « étalement » d'un slice existant
```

Points techniques :
- Le paramètre variadique doit être **le dernier**.
- À l'intérieur, c'est un slice ordinaire. `sum()` reçoit un slice `nil` (longueur 0), ce
  qui fonctionne parfaitement avec `range`.
- `sum(xs...)` **ne copie pas** `xs` : la fonction reçoit un slice qui partage le même
  tableau sous-jacent. Si elle le modifie, l'appelant le voit. Piège réel (leçon 6).
- `fmt.Println(a ...any)` est l'exemple canonique.

### Portée lexicale

```go
var global = "package"          // visible dans tout le package

func f() {
	x := "fonction"
	if true {
		x := "bloc"             // NOUVELLE variable qui masque la précédente
		fmt.Println(x)          // "bloc"
	}
	fmt.Println(x)              // "fonction"
}
```

Chaque paire d'accolades ouvre une portée. Une variable est visible de sa déclaration
jusqu'à la fin de son bloc. Le **masquage** (*shadowing*) est légal et parfois utile, mais
c'est une source de bugs classiques avec `err` :

```go
data, err := load()
if err != nil { return err }

if cond {
	data, err := transform(data)   // ← masque LES DEUX ; le résultat est perdu
	_ = data
	_ = err
}
```

Le linter `shadow` (via `golangci-lint`) le détecte ; `go vet` seul, non.

### Les fonctions sont des valeurs

```go
var op func(int, int) int      // zéro-valeur : nil
op = add
fmt.Println(op(2, 3))          // 5

// Fonction anonyme, appelée immédiatement
func() { fmt.Println("hop") }()

// Passée en argument — très fréquent dans la stdlib
slices.SortFunc(people, func(a, b Person) int {
	return cmp.Compare(a.Age, b.Age)
})
```

Appeler une variable de fonction `nil` provoque une panique — vérifier avant si la valeur
peut légitimement être absente.

### Closures : capture par référence

Une fonction anonyme **capture les variables de son environnement**, par référence et non
par copie.

```go
func counter() func() int {
	count := 0                  // survit au retour de counter !
	return func() int {
		count++                 // capture count par référence
		return count
	}
}

c := counter()
fmt.Println(c(), c(), c())      // 1 2 3
c2 := counter()
fmt.Println(c2())               // 1 — chaque closure a son propre count
```

`count` est déclaré dans `counter` mais utilisé après son retour : le compilateur détecte
qu'il **s'échappe** vers le tas (*escape analysis*, niveau 8). C'est automatique et sûr —
contrairement au C où retourner un pointeur vers une variable locale est un bug.

Le piège de la capture est celui de la variable de boucle, corrigé en Go 1.22 (leçon 4).
Mais il reste actif pour une variable déclarée **hors** de la boucle :

```go
var fns []func()
x := 0
for i := range 3 {
	x = i
	fns = append(fns, func() { fmt.Print(x) })  // capture LA MÊME variable x
}
for _, f := range fns { f() }                   // affiche 222, pas 012
```

## Exemple

```go
package main

import (
	"fmt"
	"strings"
)

// Validator vérifie une valeur et retourne une erreur descriptive, ou nil.
type Validator func(string) error

// minLength retourne un Validator paramétré : une closure sur n.
func minLength(n int) Validator {
	return func(s string) error {
		if len([]rune(s)) < n {
			return fmt.Errorf("longueur %d < minimum %d", len([]rune(s)), n)
		}
		return nil
	}
}

// noSpaces est directement un Validator.
func noSpaces(s string) error {
	if strings.ContainsAny(s, " \t\n") {
		return fmt.Errorf("contient un espace")
	}
	return nil
}

// validate applique tous les validateurs et retourne la première erreur.
// Le paramètre variadique permet d'en passer autant qu'on veut.
func validate(s string, checks ...Validator) error {
	for i, check := range checks {
		if err := check(s); err != nil {
			return fmt.Errorf("règle %d : %w", i+1, err)
		}
	}
	return nil
}

func main() {
	rules := []Validator{minLength(3), noSpaces}

	for _, input := range []string{"go", "hello world", "golang"} {
		if err := validate(input, rules...); err != nil {
			fmt.Printf("%-12q refusé : %v\n", input, err)
			continue
		}
		fmt.Printf("%-12q accepté\n", input)
	}
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `type Validator func(string) error` | Un **type fonction** nommé. Il documente l'intention et évite de répéter la signature. Très courant en Go (`http.HandlerFunc` en est l'exemple le plus connu). |
| `minLength(n int) Validator` | Une fonction qui **retourne** une fonction : le patron « fonction paramétrée ». `n` est capturé par la closure. |
| `noSpaces` | Une fonction ordinaire satisfait le type `Validator` : aucune déclaration explicite nécessaire. |
| `checks ...Validator` | Variadique de fonctions. Appelé avec `rules...` pour étaler un slice existant. |
| `%w` dans `fmt.Errorf` | **Enveloppe** l'erreur d'origine pour la retrouver plus tard avec `errors.Is`/`errors.As` (leçons 8 et niveau 2). |
| `if err := check(s); err != nil` | Portée minimale : `err` n'existe que dans le `if`. |

## Erreurs fréquentes

1. **Ignorer une erreur retournée** : `doSomething()` sans regarder le résultat. Si c'est délibéré, l'écrire : `_ = doSomething()`.
2. **`error` ailleurs qu'en dernière position.** Convention violée = code non idiomatique.
3. **`return` nu dans une longue fonction** : le lecteur ne sait plus ce qui est retourné.
4. **Masquer `err` avec `:=` dans un bloc imbriqué** : l'erreur est perdue silencieusement.
5. **Croire que `f(xs...)` copie le slice.** Il partage le tableau sous-jacent.
6. **Appeler une variable de fonction `nil`** → panique.
7. **Capturer une variable déclarée hors de la boucle** dans une closure : toutes les closures voient la même.
8. **Fonctions trop longues.** Go n'impose pas de limite, mais une fonction de plus de ~50 lignes cache presque toujours deux responsabilités.

## Bonnes pratiques Go

- Une fonction, une responsabilité. Si le nom contient « And », c'est probablement deux fonctions.
- `error` en dernier, toujours ; traiter l'erreur immédiatement après l'appel.
- Nommer les résultats seulement quand ça clarifie (`(n int, err error)`), pas par réflexe.
- Préférer des paramètres explicites à un état global. Une fonction qui ne dépend que de
  ses paramètres est testable et sûre en concurrence.
- Variadique quand le nombre d'arguments est vraiment variable — pas pour simuler des
  arguments optionnels. Pour ça, le patron « options fonctionnelles » (niveau 2) est
  l'idiome Go.
- Les fonctions courtes n'ont pas besoin de commentaire si leur nom est bon ; les fonctions
  exportées, si — en commençant par leur nom.

## Ce que je dois retenir

- Tout est passé **par valeur** ; slices, maps et strings copient un descripteur, pas le contenu.
- Les **retours multiples** remplacent les exceptions ; `error` est toujours le dernier.
- Le variadique est un slice à l'intérieur ; `f(xs...)` **partage** le tableau sous-jacent.
- Les fonctions sont des **valeurs** : paramètres, résultats, champs de struct.
- Une **closure capture par référence** ; les variables capturées s'échappent vers le tas.
- Le masquage de `err` par `:=` dans un bloc imbriqué est un bug fréquent et silencieux.

➡️ [Exercices](exercices.md)
