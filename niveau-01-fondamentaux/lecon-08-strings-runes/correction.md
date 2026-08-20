# Leçon 3 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Octets et runes

| Chaîne | octets | runes | remarque |
|---|---|---|---|
| `"hello"` | 5 | 5 | ASCII pur : 1 octet par rune |
| `"héllo"` | 6 | 5 | `é` = 2 octets (C3 A9) |
| `"日本語"` | 9 | 3 | idéogrammes = 3 octets chacun |
| `"🙂👍"` | 8 | 2 | emoji = 4 octets chacun |

```go
fmt.Printf("%q : % x\n", s, s)   // "héllo" : 68 c3 a9 6c 6c 6f
```

UTF-8 encode sur 1 à 4 octets. L'ASCII (0-127) reste sur 1 octet : c'est ce qui rend UTF-8
rétrocompatible, et c'est pourquoi le bug ne se voit jamais sur du texte anglais.

## E2 — Deux boucles

L'indice de `range` est un **décalage en octets**. Sur `"café"` : 0, 1, 2, 3 pour `c`, `a`,
`f`, `é`… puis la boucle s'arrête, car `é` occupe les octets 3 et 4. La boucle classique
itère 0,1,2,3,4 et donne des octets isolés — les deux derniers ne sont même pas des
caractères valides pris séparément.

## E3 — Voyelles

```go
const vowels = "aeiouyàâäéèêëîïôöùûüÿ"

func CountVowels(s string) int {
	n := 0
	for _, r := range strings.ToLower(s) {
		if strings.ContainsRune(vowels, r) {
			n++
		}
	}
	return n
}
```

`range` donne des runes, `ContainsRune` compare des runes : le code est correct sur tout
texte latin. Une comparaison d'octets (`s[i] == 'é'`) ne compile même pas — `'é'` est une
rune valant 233, qui ne tient pas dans un `byte`… ce qui est une chance.

## E4 — `string(42)`

```
./main.go:6:14: conversion from int to string yields a string of one rune,
   not a string of digits (did you mean fmt.Sprint(x)?)
```

`string(42)` produit `"*"` (la rune U+002A). Pour obtenir `"42"` : `strconv.Itoa(42)` ou
`fmt.Sprint(42)`. `go vet` attrape ce cas parce qu'il est presque toujours une erreur.

## E5 — Découpage

```go
for _, field := range strings.Fields(line) {
	key, value, found := strings.Cut(field, "=")
	if !found {
		fmt.Fprintf(os.Stderr, "champ sans '=' : %q\n", field)
		continue
	}
	fmt.Printf("%s → %s\n", key, value)
}
```

`strings.Cut` retourne le troisième résultat `found` — c'est précisément ce qui manque à
`strings.Split`, qui renvoie un slice à un élément quand le séparateur est absent, sans
rien signaler.

## Exercice intermédiaire — `slug`

```go
package main

import (
	"strings"
	"unicode"
)

// accents réduit les lettres latines accentuées à leur base.
// Table explicite : suffisante pour le français, sans dépendance externe.
// La solution générale passe par golang.org/x/text/unicode/norm.
var accents = map[rune]rune{
	'à': 'a', 'â': 'a', 'ä': 'a', 'á': 'a', 'ã': 'a', 'å': 'a',
	'é': 'e', 'è': 'e', 'ê': 'e', 'ë': 'e',
	'î': 'i', 'ï': 'i', 'í': 'i',
	'ô': 'o', 'ö': 'o', 'ó': 'o', 'õ': 'o',
	'ù': 'u', 'û': 'u', 'ü': 'u', 'ú': 'u',
	'ç': 'c', 'ñ': 'n', 'ÿ': 'y',
}

// Slugify transforme un titre en identifiant d'URL.
// Choix documenté : les caractères non latins (idéogrammes, cyrillique…) sont
// SUPPRIMÉS et non translittérés — une translittération correcte exigerait une
// table par écriture, hors périmètre. Ils ne génèrent donc pas de tiret.
func Slugify(s string) string {
	var b strings.Builder
	b.Grow(len(s)) // évite les réallocations : la sortie est au plus aussi longue

	pendingDash := false
	for _, r := range strings.ToLower(s) {
		if base, ok := accents[r]; ok {
			r = base
		}
		switch {
		case r >= 'a' && r <= 'z', r >= '0' && r <= '9':
			if pendingDash && b.Len() > 0 {
				b.WriteByte('-') // tiret différé : jamais en tête
			}
			pendingDash = false
			b.WriteRune(r)
		case unicode.IsLetter(r) || unicode.IsDigit(r):
			// lettre non latine : ignorée, sans produire de tiret
		default:
			pendingDash = true // marque une coupure, écrite seulement si un mot suit
		}
	}
	return b.String()
}
```

**La technique clé est le tiret différé.** L'erreur classique est d'écrire le tiret
immédiatement, puis de nettoyer après coup avec `strings.Trim` et `ReplaceAll("--","-")` —
ce qui viole la contrainte « une seule passe » et coûte plusieurs allocations. En marquant
`pendingDash` et en n'écrivant le tiret que juste avant le caractère suivant, les trois
règles (pas de tiret en tête, pas en fin, jamais doublé) tombent d'elles-mêmes.

`b.Grow(len(s))` préalloue : la sortie ne peut pas être plus longue que l'entrée.

## Défi

**a) Reverse**

```go
func Reverse(s string) string {
	r := []rune(s) // décode l'UTF-8 : indispensable
	for i, j := 0, len(r)-1; i < j; i, j = i+1, j-1 {
		r[i], r[j] = r[j], r[i]
	}
	return string(r)
}
```

Inverser les **octets** produirait de l'UTF-8 invalide. Note honnête : même cette version
casse les emoji composés (drapeaux, familles) et les accents combinants — l'inversion
« correcte » d'un texte Unicode arbitraire exige les grappes de graphèmes.

**b) Palindrome**

```go
func IsPalindrome(s string) bool {
	var clean []rune
	for _, r := range strings.ToLower(s) {
		if base, ok := accents[r]; ok {
			r = base
		}
		if unicode.IsLetter(r) || unicode.IsDigit(r) {
			clean = append(clean, r)
		}
	}
	for i, j := 0, len(clean)-1; i < j; i, j = i+1, j-1 {
		if clean[i] != clean[j] {
			return false
		}
	}
	return true
}
```

Comparer par les deux bouts évite d'allouer une seconde chaîne inversée.

**c) Mesures** — ordres de grandeur typiques sur 100 000 mots :

| Méthode | Durée | Pourquoi |
|---|---|---|
| `+=` | ~10 s | O(n²) : chaque `+=` alloue et recopie tout l'accumulé |
| `strings.Builder` | ~2 ms | buffer amorti, une seule copie finale |
| `strings.Join` | ~1 ms | calcule la taille totale, alloue **une fois**, copie |

Le facteur est de l'ordre de **5 000×**. C'est l'un des rares cas où « c'est plus rapide »
ne se discute pas : la complexité algorithmique diffère, pas la constante.

## Réponses du quiz

1. `6`. `len` compte les **octets**, et `é` en occupe deux en UTF-8.
2. `byte`, c'est-à-dire `uint8`.
3. Le **décalage en octets** de la rune dans la chaîne, pas son numéro d'ordre.
4. Une `string` est **immuable** en Go. Toute modification crée une nouvelle chaîne — c'est
   ce qui permet de la partager sans copie.
5. `"H"` (la rune U+0048). Pour `"72"` : `strconv.Itoa(72)`.
6. Chaque `+` alloue une nouvelle chaîne et recopie tout le contenu accumulé : le coût total
   est quadratique.
7. Le résultat est identique, mais `[]rune(s)` **alloue** un slice de 4 octets par rune,
   alors que `RuneCountInString` ne fait que parcourir. Préférer la seconde.
8. `EqualFold` compare selon les règles Unicode de repli de casse, sans allouer les deux
   chaînes intermédiaires. Plus correct et plus rapide.
9. Chaque octet invalide donne `utf8.RuneError` (U+FFFD, « � ») avec une taille de 1.
   `range` ne panique jamais.
