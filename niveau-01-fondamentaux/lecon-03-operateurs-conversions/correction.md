# Leçon 3 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Prédire

| Expression | Résultat | Pourquoi |
|---|---|---|
| `7 / 2` | `3` | division entière |
| `-7 / 2` | `-3` | troncature **vers zéro**, pas vers moins l'infini |
| `7 % 3` | `1` | |
| `-7 % 3` | `-1` | le reste prend le signe du **dividende** |
| `7 % -3` | `1` | idem : le signe du diviseur n'intervient pas |
| `1 << 10` | `1024` | |
| `-8 >> 1` | `-4` | décalage **arithmétique** : le bit de signe est conservé |
| `^0` | `-1` | NOT unaire : tous les bits à 1 = −1 en complément à deux |
| `5 &^ 3` | `4` | `0b101 &^ 0b011` = bits de 5 absents de 3 = `0b100` |
| `5 ^ 3` | `6` | XOR : `0b101 ^ 0b011 = 0b110` |
| `float64(7/2)` | `3` | la division entière a **déjà eu lieu** |
| `float64(7)/2` | `3.5` | `2` est une constante non typée, adoptée en `float64` |
| `int(-3.7)` | `-3` | troncature vers zéro |
| `int(3.7)` | `3` | idem |

Le couple `-7 / 2` et `-7 % 3` mérite d'être retenu : Go suit la convention du C
(troncature vers zéro), Python celle du plancher. Le même code donne des résultats
différents dans les deux langages sur les négatifs.

## E2 — Pourcentage

```go
func successRate(ok, total int) (float64, error) {
	if total <= 0 {
		return 0, fmt.Errorf("total invalide : %d", total)
	}
	return float64(ok) / float64(total) * 100, nil // conversion AVANT
}
```

La version fausse, `float64(ok/total) * 100`, donne `0.00 %` dès que `ok < total`. Elle
« fonctionne » sur les jeux de test où `ok == total`, ce qui la rend particulièrement
sournoise.

## E3 — Drapeaux

```go
type Option uint8

const (
	Recursive Option = 1 << iota
	Hidden
	Follow
	Quiet
)

func Set(o, flag Option) Option    { return o | flag }
func Clear(o, flag Option) Option  { return o &^ flag }   // et non o & ^flag, plus lisible
func Has(o, flag Option) bool      { return o&flag != 0 }
func Toggle(o, flag Option) Option { return o ^ flag }
```

`&^` évite l'erreur classique `o & !flag` — `!` est le NON **booléen**, pas binaire, et ne
compile pas sur un entier. Le NON binaire est `^`.

Noter que ces fonctions retournent une nouvelle valeur au lieu de modifier : c'est le seul
choix possible tant qu'on n'a pas vu les pointeurs.

## E4 — `strconv`

| Entrée | `Atoi` | `ParseFloat` | Commentaire |
|---|---|---|---|
| `"42"` | ✔ 42 | ✔ 42 | |
| `"3.14"` | ✘ | ✔ | `Atoi` refuse le point |
| `"0x1f"` | ✘ | ✔ **31** | `ParseFloat` accepte l'hexadécimal flottant ; `Atoi` non. Surprenant. |
| `"1e5"` | ✘ | ✔ 100000 | notation scientifique |
| `" 7"` | ✘ | ✘ | **aucune des deux ne coupe les espaces** |
| `"7 "` | ✘ | ✘ | idem |
| `""` | ✘ | ✘ | |
| `"abc"` | ✘ | ✘ | |

Les deux surprises : `ParseFloat("0x1f")` réussit, et **ni l'un ni l'autre n'accepte les
espaces**. Toujours `strings.TrimSpace` avant d'analyser une saisie — c'est la source
d'erreur numéro un sur les fichiers de configuration.

## E5 — Les verbes

```go
n := 200
fmt.Printf("%d %b %o %x %X %c %U %08d %-8d| %+d\n", n, n, n, n, n, n, n, n, n, n)
// 200 11001000 310 c8 C8 È U+00C8 00000200 200     | +200

f := 1234.5678
fmt.Printf("%f %.3f %e %g %10.2f|\n", f, f, f, f, f)
// 1234.567800 1234.568 1.234568e+03 1234.5678    1234.57|

s := "bon\tjour é"
fmt.Printf("%s | %q | %x | %+q\n", s, s, s, s)
// bon	jour é | "bon\tjour é" | 626f6e096a6f757220c3a9 | "bon\tjour é"
```

`%c` sur 200 donne `È` : l'entier est interprété comme un point de code Unicode — même
mécanisme que le piège `string(42)`. `%q` révèle la tabulation, invisible avec `%s` : c'est
pour cela qu'on débogue une chaîne avec `%q`.

## Exercice intermédiaire — `numfmt`

```go
package main

import (
	"fmt"
	"math"
	"os"
	"strconv"
)

// groupDigits insère un séparateur tous les trois chiffres.
// On parcourt DEPUIS LA FIN : les groupes se comptent à partir des unités.
func groupDigits(s string, sep string) string {
	neg := false
	if len(s) > 0 && s[0] == '-' {
		neg, s = true, s[1:]
	}
	out := ""
	for i, n := len(s)-1, 0; i >= 0; i, n = i-1, n+1 {
		if n > 0 && n%3 == 0 {
			out = sep + out
		}
		out = string(s[i]) + out
	}
	if neg {
		out = "-" + out
	}
	return out
}

func describe(n int64, base int) string {
	bits := 0
	for v := n; v != 0; v >>= 1 { // attention : boucle infinie sur un négatif si on oublie
		bits++
		if v == -1 {
			break
		}
	}
	sign := "positif"
	switch {
	case n < 0:
		sign = "négatif"
	case n == 0:
		sign = "nul"
	}
	out := fmt.Sprintf("décimal     : %s\n", groupDigits(strconv.FormatInt(n, 10), " "))
	out += fmt.Sprintf("hexadécimal : %#x\n", n)
	out += fmt.Sprintf("binaire     : %#b\n", n)
	out += fmt.Sprintf("octal       : %#o\n", n)
	out += fmt.Sprintf("bits        : %d bits significatifs\n", bits)
	out += fmt.Sprintf("signe       : %s\n", sign)
	if base >= 2 && base <= 36 {
		out += fmt.Sprintf("base %-7d: %s\n", base, strconv.FormatInt(n, base))
	}
	return out
}

func main() {
	// … analyse des drapeaux …
	n, err := strconv.ParseInt(os.Args[len(os.Args)-1], 0, 64) // base 0 : contrainte 1
	if err != nil {
		fmt.Fprintf(os.Stderr, "valeur invalide : %v\n", err)
		os.Exit(1)
	}
	fmt.Print(describe(n, 0))
	fmt.Println(describe(math.MinInt64, 36))
}
```

**Les trois points de correction :**

1. **Contrainte 1 en une ligne :** `strconv.ParseInt(s, 0, 64)`. La base `0` déduit le
   préfixe. Beaucoup de solutions écrivent un `switch` sur `strings.HasPrefix` — ça marche,
   ce n'est pas idiomatique.
2. **Le séparateur se pose depuis la fin.** Parcourir depuis le début oblige à connaître
   d'avance le reste modulo 3, ce qui est plus fragile.
3. **`math.MinInt64` est le piège.** `-n` **n'existe pas** : `-(-9223372036854775808)`
   déborde et revaut la même valeur négative. Toute solution qui commence par
   « si négatif, prendre la valeur absolue » est fausse sur cette entrée. Ici on isole le
   signe **textuellement**, sur la chaîne déjà formatée par `FormatInt`, qui gère le cas.

## Défi

**a) PopCount**

```go
func PopCount(x uint64) int {
	n := 0
	for x != 0 {
		x &= x - 1 // efface le bit à 1 le PLUS À DROITE
		n++
	}
	return n
}
```

`x - 1` met à 1 tous les bits à droite du bit à 1 le plus bas, et met ce bit à 0. Le `&`
efface donc exactement un bit par tour : la boucle tourne autant de fois qu'il y a de bits à
1, jamais 64 fois.

`math/bits.OnesCount64` est encore plus rapide : sur les processeurs modernes, il compile
vers l'instruction machine `POPCNT`, en un cycle. La leçon : **avant d'écrire un algorithme
astucieux, vérifier que la bibliothèque standard ne le fait pas déjà, en mieux.**

**b) Conversions sûres**

```go
func ToInt8(n int) (int8, error) {
	if n < math.MinInt8 || n > math.MaxInt8 {
		return 0, fmt.Errorf("%d hors des bornes d'un int8", n)
	}
	return int8(n), nil
}
```

Pour couvrir toutes les paires de types numériques (11 types entiers, 2 flottants), il
faudrait plus de cent fonctions — toutes identiques à un type près. C'est exactement le
problème que résolvent les **génériques** (niveau 3) :

```go
func To[T constraints.Integer](n int) (T, error)   // une seule fonction
```

Cet exercice n'a pas d'autre but que de faire ressentir le besoin avant de présenter la
solution.

**c) Précédence : Go contre C**

```go
func hasBit(v, bit int) bool { return v & bit == bit }
```

**En Go, c'est correct.** `&` a la précédence de `*`, donc plus forte que `==` : l'expression
se lit `(v & bit) == bit`.

**En C, ce serait faux.** `&` y lie *moins* fort que `==`, donc `v & bit == bit` se lit
`v & (bit == bit)`, c'est-à-dire `v & 1`. Ce défaut de précédence du C est si célèbre que
`gcc -Wparentheses` émet un avertissement dessus. Go l'a corrigé en donnant à `&` la
précédence de `*` et à `|` celle de `+`.

`gofmt` réécrit l'expression en `v&bit == bit` : il **resserre les espaces autour de
l'opérateur le plus prioritaire**. C'est un indicateur visuel de précédence, appliqué
uniformément. Deux autres exemples parlants :

```go
x*2 + y        // * plus prioritaire que +
a&mask != 0    // & plus prioritaire que !=
i < n-1        // - plus prioritaire que <
```

Savoir lire cet espacement dispense de mémoriser la table de précédence.

## Réponses du quiz

1. `3` dans les deux cas : la division entière a lieu avant la conversion.
2. `-1` en Go, `2` en Python. Cela compte pour indexer un tableau circulaire :
   `((x%n)+n)%n` est nécessaire en Go.
3. Non : `++` est une **instruction**, pas une expression.
4. `a &^ b` (*bit clear*) donne les bits de `a` absents de `b`. Il **n'existe pas** en C, où
   l'on écrit `a & ~b`.
5. `&` a la même précédence que `*`, donc **plus forte** que `==` — contrairement au C, où
   c'est l'inverse. `a & b == 0` fait donc ce qu'on attend en Go.
6. Une **valeur surprenante** : `byte(300)` vaut `44`. Aucune erreur, aucune panique — seules
   les constantes sont vérifiées à la compilation.
7. `string(65)` donne `"A"` (point de code) ; `strconv.Itoa(65)` donne `"65"`.
8. `%v` = format par défaut, `%+v` ajoute les noms de champs, `%#v` donne la syntaxe Go
   complète. (Sur les structs, niveau 2.)
9. `fmt.Fprintf(w io.Writer, …)`.
10. Entre entiers : **panique** (`integer divide by zero`). Entre flottants : `+Inf`, `-Inf`
    ou `NaN` selon les opérandes, conformément à IEEE 754, sans panique.
