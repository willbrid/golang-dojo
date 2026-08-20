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

| Expression | Résultat |
|---|---|
| `int8(127) + 1` | **ne compile pas** : `127 + 1` est évalué en constante et déborde `int8`. Avec une variable, ça donnerait `-128` silencieusement. |
| `int8(300)` | **ne compile pas** : `constant 300 overflows int8`. Go vérifie les constantes. Avec une variable `int` valant 300, `int8(x)` donne `44` sans alerte. |
| `int(3.99)` | `3` — troncature **vers zéro**, pas d'arrondi. `int(-3.99)` donne `-3`. |
| `float64(7) / 2` | `3.5` |
| `7 / 2` | `3` — division entière |
| `7 % -2` | `1` — en Go le reste a le signe du **dividende** (comme en C, contrairement à Python qui donne `-1`) |
| `uint8(255) + 1` | ne compile pas en constante ; sur variable : `0` (repli silencieux) |

Leçon centrale : **Go vérifie les constantes à la compilation, jamais les variables à
l'exécution.** Le dépassement sur variable est silencieux.

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

func (p Planet) String() string {
	names := [...]string{"", "Mercure", "Vénus", "Terre", "Mars",
		"Jupiter", "Saturne", "Uranus", "Neptune"}
	if p < 1 || int(p) >= len(names) {
		return fmt.Sprintf("Planet(%d)", int(p))
	}
	return names[p]
}
```

`iota + 1` est plus lisible que `_ = iota` suivi de `Mercury`. Le `default` de `String()`
est indispensable : `Planet(99)` est une valeur légale que le compilateur n'empêche pas.

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

// String choisit la plus grande unité pertinente.
func (b ByteSize) String() string {
	switch {
	case b >= EiB:
		return fmt.Sprintf("%.2f EiB", b.in(EiB))
	case b >= PiB:
		return fmt.Sprintf("%.2f PiB", b.in(PiB))
	case b >= TiB:
		return fmt.Sprintf("%.2f TiB", b.in(TiB))
	case b >= GiB:
		return fmt.Sprintf("%.2f GiB", b.in(GiB))
	case b >= MiB:
		return fmt.Sprintf("%.2f MiB", b.in(MiB))
	case b >= KiB:
		return fmt.Sprintf("%.2f KiB", b.in(KiB))
	default:
		return fmt.Sprintf("%d B", int64(b))
	}
}

// in exprime b dans l'unité donnée, en flottant.
// La conversion en float64 n'a lieu QU'ICI, au dernier moment : la valeur
// exacte reste en int64 tout le reste du temps.
func (b ByteSize) in(unit ByteSize) float64 {
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
	fmt.Printf("%d B = %v\n", n, ByteSize(n))
	fmt.Printf("max int64 = %v\n", ByteSize(math.MaxInt64))
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
