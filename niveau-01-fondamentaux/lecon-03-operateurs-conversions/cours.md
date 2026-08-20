# Leçon 3 — Opérateurs, conversions et formatage

## Objectifs

1. Connaître tous les opérateurs Go et leurs pièges (division entière, modulo négatif, débordement).
2. Manier les opérateurs binaires, utiles bien au-delà de la « programmation système ».
3. Convertir entre types **numériques** (conversion) et entre **texte et nombres** (`strconv`).
4. Maîtriser les verbes de `fmt` — l'outil de diagnostic quotidien.

## Explication

### Opérateurs arithmétiques

```go
a + b    a - b    a * b    a / b    a % b
a++      a--                        // instructions, PAS des expressions
```

Trois différences avec les langages habituels :

**1. `++` et `--` sont des instructions, pas des expressions.**
```go
i++            // OK
x := i++       // ERREUR de compilation
arr[i++] = 1   // ERREUR
```
Pas de `++i` non plus. Go supprime volontairement une source classique de code illisible.

**2. La division entre entiers est une division entière.**
```go
7 / 2        // 3, pas 3.5
7.0 / 2.0    // 3.5
float64(7) / 2   // 3.5
```
C'est **le** piège numéro un du calcul de moyennes et de pourcentages. Convertir *avant*
de diviser, jamais après : `float64(a/b)` calcule d'abord `a/b` en entier.

**3. `%` prend le signe du dividende.**
```go
 7 % 3   //  1
-7 % 3   // -1     (Python donnerait 2)
 7 % -3  //  1
```
Conséquence pratique : `x % n` peut être négatif. Pour un modulo « mathématique » toujours
positif : `((x % n) + n) % n`.

Le débordement, rappel de la leçon 2 : sur des **variables**, il est silencieux et
enroulant. `var x int8 = 127; x++` donne `-128`. Seules les constantes sont vérifiées à la
compilation.

Division par zéro : `1/0` entre entiers **panique** (`integer divide by zero`) ; entre
flottants, elle donne `+Inf` conformément à la norme IEEE 754, sans panique.

### Opérateurs de comparaison et logiques

```go
==   !=   <   <=   >   >=
&&   ||   !
```

- Les opérandes doivent être de **types identiques** : `intVal == int64Val` ne compile pas.
- `&&` et `||` sont à **court-circuit** : `if p != nil && p.Name != ""` est sûr, le second
  test n'est évalué que si le premier est vrai. C'est l'idiome de garde standard.
- Il n'existe **pas** de conversion implicite vers `bool` : `if x` avec `x` entier ne
  compile pas. Adieu les `if (x = 5)` accidentels du C.
- Comparables avec `==` : numériques, `string`, `bool`, pointeurs, channels, interfaces, et
  les structs/tableaux dont **tous** les champs le sont. Pas les slices, maps ni fonctions
  (sauf contre `nil`).

### Opérateurs binaires

```go
a & b     // ET     : bits présents dans les deux
a | b     // OU     : bits présents dans l'un ou l'autre
a ^ b     // XOR    : bits présents dans un seul
a &^ b    // AND NOT (« bit clear ») : bits de a absents de b — spécifique à Go
a << n    // décalage à gauche  : multiplie par 2^n
a >> n    // décalage à droite  : divise par 2^n
^a        // NOT unaire (Go n'utilise pas ~)
```

`&^` n'existe pas en C ni en Java : `a &^ b` équivaut à `a & (^b)`. Il sert à **retirer**
des drapeaux.

Ce n'est pas réservé aux systèmes embarqués. Les drapeaux binaires sont partout en Go :

```go
type Permission uint8

const (
	Read    Permission = 1 << iota // 0b001
	Write                          // 0b010
	Execute                        // 0b100
)

p := Read | Write            // combiner
p |= Execute                 // ajouter
p &^= Write                  // retirer
has := p&Read != 0           // tester
fmt.Printf("%03b\n", p)      // 101
```

On retrouve ce motif dans `os.OpenFile(path, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0644)`,
que l'on utilisera au niveau 4.

Attention : le décalage sur un entier **signé** conserve le bit de signe
(`-8 >> 1 == -4`), alors que sur un non signé il introduit des zéros. Décaler de plus que
la largeur du type donne 0 (ou -1 pour un négatif), sans erreur.

### Opérateurs d'affectation

```go
x += 1    x -= 1    x *= 2    x /= 2    x %= 3
x &= m    x |= m    x ^= m    x &^= m   x <<= 1   x >>= 1
```

Rappel : `+` sur des `string` les concatène — mais jamais dans une boucle (leçon 8).

### Précédence

Go n'a que **cinq** niveaux, contre quinze en C :

```
5 :  *  /  %  <<  >>  &  &^
4 :  +  -  |  ^
3 :  ==  !=  <  <=  >  >=
2 :  &&
1 :  ||
```

À noter : `&` a la même précédence que `*`, et `|` que `+`. Cela réserve une surprise :

```go
a & b == 0     // se lit  a & (b == 0)  → ne compile pas
(a & b) == 0   // ce qu'on voulait
```

Conseil : parenthéser dès qu'une expression mêle binaire et comparaison. `gofmt` conserve
les parenthèses ; il ne les considère pas comme du bruit.

### Conversions numériques

Rappel de la leçon 2 : **aucune conversion implicite**, jamais, même entre `int` et `int64`.

```go
var i int = 300
var b byte = byte(i)      // 44 — tronqué SILENCIEUSEMENT
var f float64 = float64(i)
var j int = int(3.99)     // 3 — troncature vers zéro, pas d'arrondi
```

Pour arrondir : `math.Round`, `math.Floor`, `math.Ceil` — puis convertir.

Une conversion qui déborde ne panique pas. C'est l'un des rares endroits où Go laisse
passer une erreur silencieuse, et c'est un choix de performance assumé. À valider soi-même
quand la source n'est pas maîtrisée :

```go
if i < 0 || i > math.MaxUint8 {
	return fmt.Errorf("valeur %d hors des bornes d'un byte", i)
}
```

### Texte ↔ nombres : `strconv`

**`strconv` n'est pas une conversion de type.** `int` et `string` sont des types sans
rapport ; passer de l'un à l'autre est une **analyse** ou un **formatage**, qui peut échouer.

```go
n, err := strconv.Atoi("42")                 // string → int
n, err := strconv.ParseInt("ff", 16, 64)     // base 16, résultat sur 64 bits
f, err := strconv.ParseFloat("3.14", 64)
b, err := strconv.ParseBool("true")          // accepte 1, t, T, TRUE, true…

s := strconv.Itoa(42)                        // int → string : "42"
s := strconv.FormatInt(255, 16)              // "ff"
s := strconv.FormatFloat(3.14159, 'f', 2, 64) // "3.14"
s := strconv.Quote("bon\tjour")              // "\"bon\\tjour\""
```

Le piège capital, déjà signalé en leçon 2 :

```go
string(65)          // "A"  — interprète 65 comme un POINT DE CODE
strconv.Itoa(65)    // "65" — ce qu'on voulait presque toujours
```
`go vet` signale ce cas précis. Écouter l'avertissement.

### `fmt` : les verbes à connaître

| Verbe | Effet | Exemple |
|---|---|---|
| `%v` | valeur, format par défaut | `{1 Alice}` |
| `%+v` | idem, **avec les noms de champs** | `{ID:1 Name:Alice}` |
| `%#v` | syntaxe Go complète | `main.User{ID:1, Name:"Alice"}` |
| `%T` | **type** de la valeur | `main.User` |
| `%d` | entier décimal | `42` |
| `%b` `%o` `%x` `%X` | binaire, octal, hexa | `101010` `52` `2a` `2A` |
| `%f` `%.2f` | flottant, 2 décimales | `3.141593` `3.14` |
| `%e` `%g` | notation scientifique / la plus courte | `3.14e+00` |
| `%s` | chaîne | `bonjour` |
| `%q` | chaîne **entre guillemets, échappée** | `"bon\tjour"` |
| `%c` | caractère d'un point de code | `A` |
| `%U` | notation Unicode | `U+0041` |
| `%p` | adresse d'un pointeur | `0xc000012345` |
| `%%` | un pourcent littéral | `%` |
| `%t` | booléen | `true` |
| `%w` | enveloppe une erreur (`fmt.Errorf` seulement) | — |

Modificateurs de largeur : `%8d` (aligné à droite), `%-8s` (à gauche), `%08.2f` (complété
de zéros), `%*d` (largeur passée en argument).

Les trois familles de fonctions :

```go
fmt.Printf("x = %d\n", x)              // écrit sur la sortie standard
s := fmt.Sprintf("x = %d", x)          // RETOURNE la chaîne
fmt.Fprintf(os.Stderr, "x = %d\n", x)  // écrit vers une destination au choix
```

`Fprintf` est la plus importante des trois pour du code de production : elle permet
d'écrire vers `os.Stderr`, un fichier, une réponse HTTP ou un buffer de test.

**`%+v` et `%T` sont les deux outils de débogage les plus rentables du langage.** Devant un
comportement incompréhensible, afficher le type et la structure complète résout la moitié
des cas. `%+v` et `%#v` ne montrent leur pleine utilité que sur les structs (niveau 2), mais
`%T` est utile dès aujourd'hui.

## Exemple

Aucune collection ici : les slices et les maps arrivent aux leçons 6 et 7, les méthodes au
niveau 2.

```go
package main

import (
	"fmt"
	"math"
	"os"
	"strconv"
)

type Flags uint8

const (
	Verbose Flags = 1 << iota // 0b001
	DryRun                    // 0b010
	Force                     // 0b100
)

// flagNames décrit les drapeaux actifs. Fonction, pas méthode : les méthodes
// sont au programme du niveau 2.
func flagNames(f Flags) string {
	out := ""
	if f&Verbose != 0 { // parenthèses inutiles ici, mais & lie plus fort que !=
		out += "verbose"
	}
	if f&DryRun != 0 {
		if out != "" {
			out += ","
		}
		out += "dry-run"
	}
	if f&Force != 0 {
		if out != "" {
			out += ","
		}
		out += "force"
	}
	if out == "" {
		return "aucun"
	}
	return out
}

// safeByte convertit un int en byte en refusant les valeurs hors bornes,
// au lieu de tronquer silencieusement.
func safeByte(n int) (byte, error) {
	if n < 0 || n > math.MaxUint8 {
		return 0, fmt.Errorf("%d hors des bornes [0,255]", n)
	}
	return byte(n), nil
}

func main() {
	f := Verbose | Force
	fmt.Printf("drapeaux : %s (%03b, valeur %d)\n", flagNames(f), uint8(f), f)
	f &^= Force // retirer Force
	fmt.Printf("après retrait : %s (%03b)\n", flagNames(f), uint8(f))

	// Division entière : le piège classique
	reussis, total := 7, 9
	fmt.Printf("faux  : %.2f%%\n", float64(reussis/total)*100) // 0.00 %
	fmt.Printf("juste : %.2f%%\n", float64(reussis)/float64(total)*100)

	// Modulo négatif
	x := -7
	fmt.Printf("%d %% 3 = %d, modulo positif = %d\n", x, x%3, ((x%3)+3)%3)

	// Texte → nombre : base 0, la base est déduite du préfixe
	for _, s := range [3]string{"42", "0x1f", "abc"} { // un TABLEAU, vu en leçon 2
		n, err := strconv.ParseInt(s, 0, 64)
		if err != nil {
			fmt.Fprintf(os.Stderr, "%q : %v\n", s, err)
			continue
		}
		fmt.Printf("%-6q → %d (hexa %#x, binaire %b)\n", s, n, n, n)
	}

	// Conversion sûre
	if b, err := safeByte(300); err != nil {
		fmt.Fprintln(os.Stderr, "conversion refusée :", err)
	} else {
		fmt.Println(b)
	}

	// Les verbes de diagnostic sur une valeur simple
	v := 3.14159
	fmt.Printf("%v | %.2f | %e | %T\n", v, v, v, v)
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `f&Verbose != 0` | `&` a la précédence de `*`, donc plus forte que `!=` : l'expression est correcte sans parenthèses. Les mettre reste préférable dès que l'expression se complique. |
| `f &^= Force` | Retire un drapeau. `f &= ^Force` marcherait aussi ; `&^` est la forme idiomatique Go. |
| `float64(reussis/total)` | La division entière a **déjà eu lieu** : convertir après ne récupère rien. Erreur extrêmement fréquente. |
| `((x%3)+3)%3` | Rend le modulo toujours positif. Nécessaire pour indexer un tableau circulaire. |
| `strconv.ParseInt(s, 0, 64)` | Base **0** : la base est déduite du préfixe (`0x`, `0b`, `0o`, sinon décimal). Très pratique pour de la configuration. |
| `%#x` | Affiche avec le préfixe `0x`. Le drapeau `#` demande la « forme alternative ». |
| `for _, s := range [3]string{…}` | Un **tableau** de taille fixe, connu depuis la leçon 2. Le slice équivalent arrive à la leçon 6. |
| `%v` / `%.2f` / `%e` | Trois représentations du même flottant. `%+v` et `%#v` prendront tout leur sens sur les structs, au niveau 2. |

## Erreurs fréquentes

1. **Division entière involontaire** : `a/b*100` sur des entiers. Convertir **avant**.
2. **`float64(a/b)`** au lieu de `float64(a)/float64(b)`.
3. **Croire que `%` est toujours positif** : `-7%3` vaut `-1`.
4. **`x := i++`** : `++` n'est pas une expression en Go.
5. **Oublier les parenthèses** dans `a & b == 0`.
6. **`string(n)`** au lieu de `strconv.Itoa(n)`.
7. **Ignorer l'erreur de `strconv`** : `n, _ := strconv.Atoi(s)` transforme une saisie invalide en `0` silencieux.
8. **Conversion qui tronque** sans validation : `byte(n)` sur une valeur venue de l'extérieur.
9. **`%v` sur une erreur qu'on relaie** au lieu de `%w` (leçon 10).
10. **Comparer des flottants avec `==`**.

## Bonnes pratiques Go

- Convertir vers le type le plus large **avant** de calculer, pas après.
- Valider explicitement toute conversion rétrécissante sur une donnée non maîtrisée.
- Parenthéser les expressions mêlant opérateurs binaires et comparaisons.
- Toujours traiter l'erreur de `strconv` : c'est une frontière avec le monde extérieur.
- `%q` plutôt que `%s` pour déboguer une chaîne : les espaces et tabulations deviennent visibles.
- `%+v` et `%T` en premier réflexe de diagnostic.
- `fmt.Fprintf(os.Stderr, …)` pour les messages d'erreur, jamais `fmt.Printf`.
- Les drapeaux binaires méritent un type nommé et une méthode `String()`.

## Ce que je dois retenir

- `/` entre entiers est une **division entière** ; `%` prend le signe du **dividende**.
- `++` et `--` sont des **instructions**, jamais des expressions.
- `&^` (*bit clear*) est spécifique à Go et sert à retirer des drapeaux.
- `&` a la précédence de `*`, `|` celle de `+` : parenthéser près des comparaisons.
- Une conversion numérique qui déborde **tronque en silence**.
- `strconv` ≠ conversion de type : c'est une analyse qui peut **échouer**.
- `%v`, `%+v`, `%#v`, `%T`, `%q` : les cinq verbes du quotidien.

➡️ [Exercices](exercices.md)
