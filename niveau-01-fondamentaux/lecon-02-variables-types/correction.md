# Leçon 2 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Zéro-valeurs

```go
var i int; var f float64; var b bool; var s string
var sl []int; var m map[string]int; var p *int

fmt.Printf("%v %T\n", i, i)   // 0 int
fmt.Printf("%v %T\n", f, f)   // 0 float64
fmt.Printf("%v %T\n", b, b)   // false bool
fmt.Printf("%q %T\n", s, s)   // "" string
fmt.Printf("%v %T\n", sl, sl) // [] []int      ← affiché [], mais sl == nil est VRAI
fmt.Printf("%v %T\n", m, m)   // map[] map[string]int
fmt.Printf("%v %T\n", p, p)   // <nil> *int
```

Le piège : `%v` affiche `[]` pour un slice nil et `map[]` pour une map nil. L'affichage ne
permet **pas** de distinguer nil de vide. Seul `sl == nil` le dit.

## E2 — Conversions

### Première surprise : aucune des sept lignes ne compile

Recopiées telles quelles dans un `main`, les sept expressions produisent sept erreurs. Mais
elles ne disent pas toutes la même chose, et c'est là qu'est l'exercice — il y a **deux
couches d'erreurs empilées**.

```
./main.go:4:2: int8(127) + 1 (constant 128 of type int8) overflows int8
./main.go:5:7: constant 300 overflows int8
./main.go:6:6: cannot convert 3.99 (untyped float constant) to type int
./main.go:7:2: float64(7) / 2 (constant 3.5 of type float64) is not used
./main.go:8:2: 7 / 2 (untyped int constant 3) is not used
./main.go:9:2: 7 % -2 (untyped int constant 1) is not used
./main.go:10:2: uint8(255) + 1 (constant 256 of type uint8) overflows uint8
```

**Couche 1 — `is not used`** (lignes 4 à 6 de l'énoncé). Ces trois expressions sont
parfaitement légales. Go refuse simplement une expression **utilisée comme instruction** :
en Go, seuls un appel de fonction, une affectation, un `<-` et quelques autres formes sont
des instructions valides. `3 + 4` tout seul sur une ligne n'est pas un programme, c'est un
oubli. Ces erreurs ne parlent donc pas de conversion du tout — le message donne d'ailleurs
la valeur calculée (`3.5`, `3`, `1`), preuve que le compilateur a fait le calcul sans
broncher. Les mettre dans un `fmt.Println` suffit à les faire compiler.

**Couche 2 — `overflows` / `cannot convert`** (lignes 1, 2, 3 et 7). Celles-là sont de
vraies erreurs de typage, et elles ont toutes la même cause : **ce sont des constantes**.

### Le point central : constante ou variable ?

Une constante est évaluée par le compilateur, qui vérifie qu'elle tient dans son type et
refuse sinon. Une variable n'est pas connue à la compilation : la même opération passe, et
déborde **en silence** à l'exécution. Toute la question de l'exercice — « le compilateur
refuse-t-il, ou accepte-t-il en produisant un résultat surprenant ? » — est là : c'est
*ou bien l'un, ou bien l'autre*, selon la forme, jamais les deux.

| Expression | Écrite telle quelle (constante) | Avec une variable |
|---|---|---|
| `int8(127) + 1` | **refusé** : `constant 128 of type int8 overflows int8` | `var a int8 = 127` puis `a + 1` → **-128** |
| `int8(300)` | **refusé** : `constant 300 overflows int8` | `var n = 300` puis `int8(n)` → **44** |
| `int(3.99)` | **refusé** : `cannot convert 3.99 (untyped float constant) to type int` | `var f = 3.99` puis `int(f)` → **3** |
| `float64(7) / 2` | légal, vaut **3.5** (seule l'instruction nue est refusée) | idem |
| `7 / 2` | légal, vaut **3** | idem |
| `7 % -2` | légal, vaut **1** | idem |
| `uint8(255) + 1` | **refusé** : `constant 256 of type uint8 overflows uint8` | `var u uint8 = 255` puis `u + 1` → **0** |

### Les quatre pièges, un par un

**`int8(127) + 1` et `uint8(255) + 1` — ce n'est pas la conversion qui échoue.**
`127` est exactement `math.MaxInt8`, et `255` exactement `math.MaxUint8` : les deux tiennent
parfaitement dans leur type, la conversion est irréprochable. C'est l'**addition qui suit**
qui déborde. Le message le dit mot pour mot : `constant 128 of type int8` — la valeur
fautive est 128, pas 127. Lire la valeur citée par le compilateur, pas celle qu'on a écrite.

**`int8(300)` — là, c'est bien la conversion.** `300` ne tient pas dans `int8` (qui va de
-128 à 127). Noter que le message diffère : `constant 300 overflows int8`, sans mention de
type sur la constante, parce que `300` est encore non typée au moment du refus.

**`int(3.99)` — ce n'est pas « parce que c'est une constante non typée ».** Les constantes
non typées se convertissent très bien : `int(4.0)` compile et vaut `4`. La règle est que la
conversion d'une constante doit être **exacte** : `3.99` n'a aucune représentation entière,
donc Go refuse plutôt que de choisir à votre place. Sur une variable il tronque **vers
zéro** — `int(3.99)` donne `3`, et `int(-3.99)` donne `-3`, pas `-4`.

**`7 % -2` — le signe suit le dividende.** Le résultat est `1`, pas `-1`. En Go comme en C,
`a % b` a le signe de `a` ; Python, lui, donne `-1`. Piège classique quand on implémente un
modulo cyclique : `(i - 1) % n` sur un index à 0 donne un indice **négatif** en Go.

### La vérification à lancer

```go
var a int8 = 127
var n = 300
var f, g = 3.99, -3.99
var u uint8 = 255

fmt.Println(a+1)                 // -128
fmt.Println(int8(n))             // 44
fmt.Println(int(f), int(g))      // 3 -3
fmt.Println(u+1)                 // 0
fmt.Println(7%-2, -7%2)          // 1 -1
fmt.Println(float64(7)/2, 7/2)   // 3.5 3
```

Leçon centrale : **Go vérifie les constantes à la compilation, et uniquement elles.** Dès
qu'une valeur transite par une variable, plus aucun filet — le débordement est silencieux
et c'est au développeur de le prévoir. C'est exactement ce que le défi `SafeAdd` / `SafeMul`
de cette leçon demande de construire à la main.

## E3 — `iota`

```go
type Planet int

const (
	Mercury Planet = iota + 1 // décale toute l'énumération de 1
	Venus
	Earth
	Mars
	Jupiter
	Saturn
	Uranus
	Neptune
)

func planetName(p Planet) string {
	names := [...]string{"", "Mercure", "Vénus", "Terre", "Mars",
		"Jupiter", "Saturne", "Uranus", "Neptune"}
	if p < 1 || int(p) >= len(names) {
		return fmt.Sprintf("Planet(%d)", int(p)) // Planet(99)
	}
	return names[p]
}
```

`iota + 1` est plus lisible que `_ = iota` suivi de `Mercury`. Le test de bornes est
indispensable : `Planet(99)` est une valeur parfaitement légale que le compilateur n'empêche
pas — sans lui, l'accès `names[99]` **panique**. Une énumération Go ne se valide jamais toute
seule.

`[...]string{…}` est un **tableau** de taille déduite, pas un slice : c'est ce qui est
disponible à ce stade de la formation.

Au niveau 2, `planetName` deviendra une méthode `String()` et `fmt.Println(p)` l'appellera
automatiquement — c'est la seule différence, mais elle change l'ergonomie du type.

## E4 — Constantes non typées

`const big = 1 << 62` est une **constante non typée** : elle n'a pas encore de type et est
manipulée en précision arbitraire par le compilateur. Affectée à un `float64`, elle est
convertie à ce moment-là, sans erreur.

`var n = 1 << 62` force le compilateur à choisir un type : le type par défaut d'une
constante entière, `int`. `n` est donc une **variable** de type `int`, et Go n'autorise
aucune conversion implicite entre `int` et `float64` : il faut écrire `float64(n)`.

En un mot : une constante s'adapte à son contexte, une variable a un type figé.

## E5 — Tailles

```go
fmt.Println(unsafe.Sizeof(int(0)))     // 8
fmt.Println(unsafe.Sizeof(int32(0)))   // 4
fmt.Println(unsafe.Sizeof(true))       // 1
fmt.Println(unsafe.Sizeof("bonjour"))  // 16  ← pas 7 !
fmt.Println(math.MaxInt64, math.MinInt64)
```

`unsafe.Sizeof("bonjour")` vaut **16** parce qu'une `string` est un **descripteur** de deux
mots machine : un pointeur (8 octets) vers les octets, et une longueur (8 octets). Les
octets du texte vivent ailleurs. `Sizeof` mesure la taille de la valeur elle-même, jamais
ce qu'elle référence. Même logique pour un slice (24 octets : pointeur, len, cap).

## Exercice intermédiaire — `unitconv`

```go
package main

import (
	"fmt"
	"math"
	"os"
	"strconv"
)

type ByteSize int64

const (
	B  ByteSize = 1 << (10 * iota) // 1
	KiB                            // 1024
	MiB
	GiB
	TiB
	PiB
	EiB // 1 << 60 : tient dans int64 (max ≈ 9,2 EiB)
)

// format choisit la plus grande unité pertinente.
// Au niveau 2, ce sera une méthode String().
func format(b ByteSize) string {
	switch {
	case b >= EiB:
		return fmt.Sprintf("%.2f EiB", in(b, EiB))
	case b >= PiB:
		return fmt.Sprintf("%.2f PiB", in(b, PiB))
	case b >= TiB:
		return fmt.Sprintf("%.2f TiB", in(b, TiB))
	case b >= GiB:
		return fmt.Sprintf("%.2f GiB", in(b, GiB))
	case b >= MiB:
		return fmt.Sprintf("%.2f MiB", in(b, MiB))
	case b >= KiB:
		return fmt.Sprintf("%.2f KiB", in(b, KiB))
	default:
		return fmt.Sprintf("%d B", int64(b))
	}
}

// in exprime b dans l'unité donnée, en flottant.
// La conversion en float64 n'a lieu QU'ICI, au dernier moment : la valeur
// exacte reste en int64 tout le reste du temps.
func in(b, unit ByteSize) float64 {
	return float64(b) / float64(unit)
}

func main() {
	if len(os.Args) != 2 {
		fmt.Fprintln(os.Stderr, "usage: unitconv <octets>")
		os.Exit(1)
	}
	n, err := strconv.ParseInt(os.Args[1], 10, 64)
	if err != nil {
		fmt.Fprintf(os.Stderr, "valeur invalide %q : %v\n", os.Args[1], err)
		os.Exit(1)
	}
	if n < 0 {
		fmt.Fprintf(os.Stderr, "valeur négative : %d\n", n)
		os.Exit(1)
	}
	fmt.Printf("%d B = %s\n", n, format(ByteSize(n)))
	fmt.Printf("max int64 = %s\n", format(ByteSize(math.MaxInt64)))
}
```

**La question de la précision.** Un `float64` a 53 bits de mantisse : au-delà de 2⁵³
(≈ 9·10¹⁵), il ne peut plus représenter tous les entiers. `math.MaxInt64` ≈ 9,2·10¹⁸ est
donc **déjà inexact** en `float64`. La parade est de garder la valeur en `int64` jusqu'au
tout dernier instant, et de ne convertir que pour l'affichage à deux décimales — où la
perte est invisible. C'est un raisonnement de niveau 8 appliqué au niveau 1.

## Défi — `SafeAdd` / `SafeMul`

```go
func SafeAdd(a, b int) (int, bool) {
	if b > 0 && a > math.MaxInt-b {
		return 0, false // débordement positif
	}
	if b < 0 && a < math.MinInt-b {
		return 0, false // débordement négatif
	}
	return a + b, true
}
```

L'astuce : ne **jamais** calculer `a+b` avant de savoir si c'est sûr. On réécrit
`a + b > MaxInt` en `a > MaxInt - b`, qui ne déborde pas puisque `b > 0`.

```go
func SafeMul(a, b int) (int, bool) {
	if a == 0 || b == 0 {
		return 0, true
	}
	// Le cas pathologique : -MinInt n'est pas représentable.
	if a == -1 && b == math.MinInt {
		return 0, false
	}
	if b == -1 && a == math.MinInt {
		return 0, false
	}
	p := a * b
	if p/b != a { // la division inverse détecte le débordement
		return 0, false
	}
	return p, true
}
```

Le cas `MinInt` est celui qui piège : en complément à deux, `MinInt = -9223372036854775808`
mais `MaxInt = 9223372036854775807`. L'opposé de `MinInt` **n'existe pas** dans le type.
D'où `-1 * MinInt` qui déborde, et pourquoi `abs(MinInt)` est négatif.

Note pour plus tard : `math/bits.Add64` et `Mul64` font ça au niveau matériel, sans division.

## Réponses du quiz

1. `""` pour une string ; `nil` pour une map ; `nil` pour un `*int`.
2. Go n'a **aucune conversion implicite**. Il faut `float64(someInt)`. C'est voulu : toute
   perte de précision est visible dans le code.
3. `-56`. Non, aucune panique : le dépassement d'une **variable** est silencieux (seules
   les constantes sont vérifiées à la compilation).
4. `const n = 100` est **non typée** : elle s'adapte au contexte (`int`, `float64`, `int8`…)
   et a une précision arbitraire. `const n int = 100` est figée en `int`.
5. Non : `byte` est un **alias** de `uint8`, c'est exactement le même type. Le nom exprime
   l'intention.
6. 64 bits sur les plateformes modernes, mais la spécification garantit seulement **au
   moins 32 bits**. Ne pas écrire de code qui dépend de 64.
7. Parce que `0.1` n'a pas de représentation binaire exacte : les additions accumulent des
   erreurs et `0.1+0.2 != 0.3`. Utiliser des entiers de centimes, ou une bibliothèque décimale.
8. `2` (il commence à 0). Il est **remis à 0** dans chaque nouveau bloc `const`.
