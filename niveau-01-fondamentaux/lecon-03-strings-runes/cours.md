# Leçon 3 — Strings, runes et bytes

## Objectifs

1. Expliquer ce qu'est **réellement** une `string` en Go : une suite d'octets immuable.
2. Distinguer `byte`, `rune` et « caractère », et savoir lequel utiliser.
3. Parcourir du texte UTF-8 sans le corrompre.
4. Construire des chaînes efficacement et savoir pourquoi la concaténation en boucle est un piège.

## Explication

### Une string est une suite d'octets, pas de caractères

C'est **la** source de confusion. En Go :

```go
s := "héllo"
fmt.Println(len(s))        // 6, pas 5 !
fmt.Println(s[1])          // 195, un octet — pas 'é'
```

`len()` sur une chaîne compte les **octets**. `é` s'encode en UTF-8 sur deux octets
(0xC3 0xA9). Indexer une chaîne donne un `byte` (`uint8`), pas un caractère.

Le code source Go est **toujours en UTF-8** — c'est dans la spécification. Une chaîne
littérale est donc une suite d'octets UTF-8.

### Immuables

```go
s := "bonjour"
s[0] = 'B'    // ERREUR : cannot assign to s[0]
```

Une `string` ne peut pas être modifiée. Toute « modification » crée une nouvelle chaîne.
C'est ce qui permet de passer des chaînes par valeur sans copier les octets : en interne,
une `string` est un couple (pointeur vers les octets, longueur). La copier copie ces deux
mots machine, jamais le contenu. Conséquence : passer une chaîne de 10 Mo à une fonction
ne coûte rien.

### `byte` vs `rune`

| | `byte` | `rune` |
|---|---|---|
| Alias de | `uint8` | `int32` |
| Représente | un octet brut | un **point de code Unicode** |
| Littéral | `'A'` vaut 65 | `'é'` vaut 233, `'🙂'` vaut 128578 |
| Obtenu par | `s[i]` | `for … range s`, `[]rune(s)` |

```go
s := "héllo"
fmt.Println(len(s))                    // 6 octets
fmt.Println(utf8.RuneCountInString(s)) // 5 runes
fmt.Println(len([]rune(s)))            // 5 aussi, mais alloue un slice
```

Attention : même une `rune` n'est pas toujours un « caractère perçu ». `é` peut s'écrire
en une rune (U+00E9) ou en deux (`e` + accent combinant U+0301). Et un emoji drapeau en
compte deux. Pour du texte affiché à un humain, il faut la notion de *grapheme cluster*
(paquet externe `golang.org/x/text`). En pratique, ça n'arrive que pour du traitement
typographique sérieux — mais il faut savoir que le problème existe.

### Parcourir une chaîne : deux boucles, deux résultats

```go
s := "héllo"

for i := 0; i < len(s); i++ {
	fmt.Printf("%d:%d ", i, s[i])   // octet par octet
}
// 0:104 1:195 2:169 3:108 4:108 5:111

for i, r := range s {
	fmt.Printf("%d:%c ", i, r)      // rune par rune, i = position en OCTETS
}
// 0:h 1:é 3:l 4:l 5:o
```

Le point crucial : dans `for i, r := range s`, **`i` est un décalage en octets, pas un
numéro de rune**. C'est pourquoi l'indice saute de 1 à 3. `range` sur une chaîne décode
l'UTF-8 automatiquement ; c'est presque toujours ce qu'on veut.

Un octet invalide en UTF-8 est remplacé par `utf8.RuneError` (`U+FFFD`, le fameux « � ») —
`range` ne panique jamais.

### Construire des chaînes : `+` est un piège en boucle

```go
// MAUVAIS : O(n²) allocations
s := ""
for _, w := range words {
	s += w    // chaque += alloue une NOUVELLE chaîne et recopie tout
}

// BON
var b strings.Builder
for _, w := range words {
	b.WriteString(w)
}
s := b.String()

// AUSSI BON, et plus lisible quand c'est possible
s := strings.Join(words, "")
```

`strings.Builder` accumule dans un buffer qui grandit par doublement et évite la copie
finale. Sa zéro-valeur est utilisable (`var b strings.Builder`) — l'illustration directe
du principe vu en leçon 2. Pour 3 concaténations, `+` est parfaitement bien ; le problème
est la **boucle**.

### Le paquet `strings`, l'essentiel

```go
strings.Contains(s, "abc")        strings.HasPrefix(s, "http")
strings.Index(s, "x")             strings.HasSuffix(s, ".go")
strings.Split(s, ",")             strings.Fields(s)     // découpe sur les espaces
strings.Join(parts, ", ")         strings.TrimSpace(s)
strings.ToUpper(s)                strings.Trim(s, ".,!")
strings.ReplaceAll(s, "a", "b")   strings.EqualFold(a, b) // comparaison insensible à la casse
strings.Cut(s, "=")               // (avant, après, trouvé) — préférer à Split pour 2 parties
strings.CutLast(s, "/")           // Go 1.27 : coupe sur la DERNIÈRE occurrence
```

`strings.Cut`, arrivé en Go 1.18, est l'idiome moderne pour découper en deux :

```go
key, value, found := strings.Cut("name=Alice", "=")
if !found {
	return fmt.Errorf("ligne malformée : %q", line)
}
```

Attention à `strings.ToUpper` : c'est de l'Unicode, pas de l'ASCII. Pour comparer sans
tenir compte de la casse, `strings.EqualFold` est plus correct que
`ToLower(a) == ToLower(b)` (et plus rapide, il n'alloue pas).

### Conversions

```go
b := []byte(s)     // copie les octets
r := []rune(s)     // décode l'UTF-8, alloue un slice de int32 (4 octets par rune !)
s2 := string(b)    // copie
s3 := string(r)    // ré-encode en UTF-8

string(65)         // "A" — conversion d'un entier en rune, souvent une ERREUR
strconv.Itoa(65)   // "65" — ce que l'on voulait presque toujours
```

Ces conversions **allouent et copient**. Convertir en `[]rune` pour compter des caractères
alloue 4 octets par caractère ; `utf8.RuneCountInString` fait le même travail sans rien allouer.

## Exemple

```go
package main

import (
	"fmt"
	"strings"
	"unicode"
	"unicode/utf8"
)

// normalize met en minuscules et retire la ponctuation de tête et de fin.
// Elle retourne une nouvelle chaîne : l'originale n'est pas modifiée.
func normalize(word string) string {
	trimmed := strings.TrimFunc(word, func(r rune) bool {
		return !unicode.IsLetter(r) && !unicode.IsDigit(r)
	})
	return strings.ToLower(trimmed)
}

// initials retourne les initiales en majuscule des mots d'une phrase.
func initials(phrase string) string {
	var b strings.Builder
	for _, w := range strings.Fields(phrase) {
		r, size := utf8.DecodeRuneInString(w)
		if size == 0 {
			continue
		}
		b.WriteRune(unicode.ToUpper(r))
	}
	return b.String()
}

func main() {
	s := "Élégant, vraiment !"
	fmt.Printf("%q → octets=%d runes=%d\n", s, len(s), utf8.RuneCountInString(s))
	fmt.Println(normalize("Bonjour,"))
	fmt.Println(initials(s))

	for i, r := range "héllo" {
		fmt.Printf("%d:%c(%d) ", i, r, r)
	}
	fmt.Println()
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `strings.TrimFunc` | Retire en tête et en fin tant que la fonction retourne `true`. Prendre une fonction en paramètre est courant en Go — on verra les closures en leçon 5. |
| `unicode.IsLetter` | Fonctionne sur tout l'Unicode, pas seulement `a-z`. `r >= 'a' && r <= 'z'` casserait sur `é`. |
| `utf8.DecodeRuneInString(w)` | Extrait la **première** rune et sa taille en octets. `w[0]` aurait donné un demi-caractère sur un mot accentué. |
| `size == 0` | Cas de la chaîne vide. Sans ce garde-fou, on écrirait une rune invalide. |
| `b.WriteRune` | Écrit une rune encodée en UTF-8 dans le Builder. |
| `%q` | Affiche la chaîne entre guillemets avec les échappements — indispensable pour déboguer les espaces invisibles. |

## Erreurs fréquentes

1. **`len(s)` pris pour un nombre de caractères.** C'est un nombre d'**octets**.
2. **`s[i]` pris pour un caractère.** C'est un `byte`. `fmt.Println(s[0])` affiche un nombre.
3. **Inverser une chaîne octet par octet** : ça corrompt tout texte non ASCII. Il faut passer par `[]rune`.
4. **Concaténer avec `+` dans une boucle** : quadratique. Utiliser `strings.Builder`.
5. **`string(42)`** → `"*"` et non `"42"`. `go vet` signale ce cas précis, écouter l'avertissement.
6. **Comparer avec `ToLower(a) == ToLower(b)`** : deux allocations et une sémantique Unicode discutable. `strings.EqualFold`.
7. **Découper avec `strings.Split` quand on attend deux parties** : `Split` ne dit pas si le séparateur était présent, et retourne un slice à gérer. `strings.Cut` est plus sûr.
8. **Croire qu'une rune est un caractère visible.** Vrai à 99 %, faux pour les accents combinants et les emoji composés.

## Bonnes pratiques Go

- `for … range` pour parcourir du texte ; l'indexation par octets uniquement pour du binaire.
- `utf8.RuneCountInString(s)` plutôt que `len([]rune(s))` : même résultat, zéro allocation.
- `strings.Builder` dès qu'il y a une boucle de construction ; `strings.Join` quand les morceaux existent déjà.
- `strings.Cut` / `strings.CutLast` (1.27) plutôt que `Index` + découpage manuel.
- `%q` pour déboguer une chaîne, `%v` pour l'afficher à l'utilisateur.
- Ne pas convertir en `[]byte` ou `[]rune` « pour voir » : chaque conversion alloue.
- Pour des données binaires, utiliser `[]byte` de bout en bout ; `string` est pour du **texte**.

## Ce que je dois retenir

- Une `string` est une **suite d'octets UTF-8, immuable**. `len` compte les octets.
- `s[i]` donne un `byte` ; `for i, r := range s` donne des **runes**, avec `i` en octets.
- `byte` = `uint8` (donnée brute), `rune` = `int32` (point de code Unicode).
- La concaténation en boucle est **quadratique** : `strings.Builder`.
- Les conversions `[]byte(s)` et `[]rune(s)` **allouent et copient**.
- `string(nombre)` n'est pas `strconv.Itoa(nombre)`.

➡️ [Exercices](exercices.md)
