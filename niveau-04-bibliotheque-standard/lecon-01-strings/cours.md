# Leçon 1 — `strings` en profondeur

> Le niveau 1 (leçon 8) a posé les bases : UTF-8, `byte` contre `rune`, immutabilité,
> `strings.Builder`, `Cut`. Cette leçon parcourt le **reste** du paquet, organisé par
> familles, avec les cinq ou six pièges qui produisent des bugs réels — presque tous liés à
> des fonctions dont le nom promet autre chose que ce qu'elles font.

## Objectifs

1. Connaître les familles de fonctions de `strings` et savoir laquelle choisir.
2. Éviter les pièges de `Split`, `Trim`, `Title`, `ToUpper`.
3. Remplacer plusieurs motifs en un passage avec `strings.Replacer`.
4. Construire des chaînes sans coût quadratique.

## Explication

### Lire la documentation

Avant tout : le paquet a plus de 50 fonctions. On ne les apprend pas par cœur, on apprend à
les **trouver** :

```bash
go doc strings              # la liste
go doc strings.SplitN       # une fonction
go doc -ex strings.Cut      # Go 1.27 : avec ses exemples exécutables
```

Presque toutes les fonctions existent en double dans `bytes`, pour les `[]byte`. Quand on
travaille sur des octets lus d'un fichier, utiliser `bytes` évite une conversion (donc une
copie) vers `string`.

### Les familles

| Famille | Fonctions | Remarque |
|---|---|---|
| Test | `Contains`, `ContainsAny`, `ContainsRune`, `ContainsFunc`, `HasPrefix`, `HasSuffix`, `EqualFold` | |
| Position | `Index`, `LastIndex`, `IndexByte`, `IndexRune`, `IndexAny`, `IndexFunc` | rendent des indices d'**octets**, `-1` si absent |
| Découpe | `Split`, `SplitN`, `SplitAfter`, `Fields`, `FieldsFunc`, `Cut`, `CutPrefix`, `CutSuffix`, `CutLast` (1.27) | + versions itérateurs `SplitSeq`, `FieldsSeq`, `Lines` |
| Nettoyage | `TrimSpace`, `Trim`, `TrimLeft`, `TrimRight`, `TrimPrefix`, `TrimSuffix`, `TrimFunc` | piège : voir plus bas |
| Transformation | `ToUpper`, `ToLower`, `Map`, `Repeat`, `Replace`, `ReplaceAll`, `NewReplacer` | allouent une nouvelle chaîne |
| Assemblage | `Join`, `Builder`, `Concat`… | |
| Comptage | `Count` | `Count(s, "")` vaut nombre de runes + 1 |

### Piège 1 — `Split` et les chaînes vides

```go
strings.Split("a,,b,", ",")    // ["a" "" "b" ""]   — 4 éléments
strings.Split("", ",")         // [""]              — 1 élément, pas 0 !
strings.Fields(" a  b ")       // ["a" "b"]         — espaces multiples fusionnés
```

`Split` est **littéral** : chaque séparateur produit une coupure, même entre deux séparateurs
voisins ou en fin de chaîne. Et découper la chaîne vide rend un slice d'**un** élément vide.
Le bug classique : `len(strings.Split(line, ","))` pour compter des champs renvoie 1 sur une
ligne vide.

`Fields` découpe sur les **espaces** (au sens Unicode) et ignore les vides : c'est presque
toujours ce qu'on veut pour du texte saisi par un humain.

`SplitN(s, sep, n)` limite le nombre de morceaux : `SplitN("k=v=w", "=", 2)` →
`["k" "v=w"]`. Mais pour **deux** morceaux, `Cut` est plus clair et indique si le séparateur
était présent.

### Piège 2 — `Trim` prend un **ensemble** de caractères

```go
strings.TrimLeft("xxhixx", "x")       // "hixx"
strings.TrimPrefix("xxhi", "x")       // "xhi"
strings.Trim("abcHIcba", "abc")       // "HI"  ← pas « retirer le mot abc »
```

Le second argument de `Trim`, `TrimLeft` et `TrimRight` est un **ensemble de runes** à retirer
tant qu'on en trouve, pas un préfixe. `strings.TrimLeft(url, "https://")` retire aussi les
`h`, `t`, `p`, `s`, `:` et `/` du nom de domaine qui suit. Pour un **préfixe**, c'est
`TrimPrefix` — ou `CutPrefix` pour savoir s'il était là.

### Piège 3 — la casse n'est pas simple

```go
strings.ToUpper("straße")                  // "STRAßE"   (pas « STRASSE »)
strings.EqualFold("Straße", "STRASSE")     // false
```

`ToUpper` et `EqualFold` travaillent **rune par rune** : une rune ne devient jamais deux.
Pour la plupart des textes, c'est sans conséquence ; pour des comparaisons linguistiques
exactes, il faut `golang.org/x/text/cases`. Même paquet pour remplacer `strings.Title`,
**déprécié** : il met une majuscule au début de chaque « mot », un mot commençant après
tout caractère qui n'est pas une lettre, sans connaître les règles d'aucune langue.
`Title("o'neil")` donne `"O'Neil"` (juste), mais `Title("l'été")` donne `"L'Été"` (faux en
français) : la même règle ne peut pas satisfaire les deux langues.

Pour comparer sans tenir compte de la casse, toujours `EqualFold` plutôt que
`ToLower(a) == ToLower(b)` : il n'alloue rien et gère les équivalences simples d'Unicode.

### Piège 4 — les indices sont des octets

```go
strings.Index("chaîne", "n")   // 5, pas 4 : « î » occupe deux octets
```

Un indice rendu par `Index` est valide pour **découper** la chaîne (`s[:i]`), jamais pour
compter des caractères. Pour une position en caractères : `utf8.RuneCountInString(s[:i])`.

### `strings.Replacer` : plusieurs remplacements en un passage

```go
r := strings.NewReplacer("&", "&amp;", "<", "&lt;", ">", "&gt;")
r.Replace("a<b && c>d")   // "a&lt;b &amp;&amp; c&gt;d"
```

Trois appels successifs à `ReplaceAll` seraient **faux** ici : remplacer `<` par `&lt;` puis
`&` par `&amp;` transformerait le `&` de `&lt;` en `&amp;lt;`. Le `Replacer` parcourt la
chaîne **une seule fois** et ne revient jamais sur un texte déjà remplacé. Autre conséquence :
`NewReplacer("a", "b", "b", "a").Replace("ab")` donne `"ba"` — un échange, impossible avec
deux `ReplaceAll`.

Un `Replacer` se construit **une fois** (variable de paquet ou champ) et se réutilise : sa
construction prépare une table, c'est le coût à amortir. Il est utilisable depuis plusieurs
goroutines (niveau 6).

### `strings.Map` : transformer rune par rune

```go
noDigits := strings.Map(func(r rune) rune {
	if unicode.IsDigit(r) {
		return -1 // une valeur négative SUPPRIME la rune
	}
	return r
}, "a1b2") // "ab"
```

Avec le paquet `unicode` (`IsLetter`, `IsSpace`, `IsPunct`, `IsUpper`…), `Map`,
`FieldsFunc`, `TrimFunc` et `IndexFunc` couvrent l'essentiel du traitement de texte sans
expression régulière.

### Construire : le coût de `+=`

Une chaîne est immuable : `s += x` alloue une **nouvelle** chaîne et recopie tout `s`. Dans
une boucle de `n` tours, on recopie `1 + 2 + … + n` morceaux : coût **quadratique**.
Mesurable dès quelques milliers de tours (voir exercices).

| Situation | Outil |
|---|---|
| Un slice de chaînes à assembler avec un séparateur | `strings.Join` |
| Construction incrémentale, boucle | `strings.Builder` (+ `Grow` si on connaît la taille) |
| Formatage | `fmt.Fprintf(&sb, …)` — un `*Builder` est un `io.Writer` |
| Deux ou trois morceaux, une fois | `a + b + c` — parfaitement idiomatique |

Rappel de N3 L6 : un `Builder` ne se **copie** pas une fois utilisé — il panique s'il détecte
la copie. On le passe par pointeur.

## Exemple

```go
package main

import (
	"fmt"
	"strings"
	"unicode"
)

// slugReplacer : construit une fois, réutilisé à chaque appel.
var slugReplacer = strings.NewReplacer(
	"à", "a", "â", "a", "ä", "a",
	"é", "e", "è", "e", "ê", "e", "ë", "e",
	"î", "i", "ï", "i", "ô", "o", "ö", "o",
	"ù", "u", "û", "u", "ü", "u", "ç", "c",
	"œ", "oe", "æ", "ae",
)

// Slug transforme un titre en identifiant d'URL : minuscules ASCII,
// mots séparés par un seul tiret.
func Slug(title string) string {
	s := slugReplacer.Replace(strings.ToLower(title))
	words := strings.FieldsFunc(s, func(r rune) bool {
		return !(r < unicode.MaxASCII && (unicode.IsLetter(r) || unicode.IsDigit(r)))
	})
	return strings.Join(words, "-")
}

// ParseHeader lit « Nom: valeur » ; le nom est insensible à la casse.
func ParseHeader(line string) (name, value string, err error) {
	name, value, ok := strings.Cut(line, ":")
	if !ok {
		return "", "", fmt.Errorf("en-tête %q : « : » manquant", line)
	}
	name = strings.TrimSpace(name)
	if name == "" || strings.ContainsFunc(name, unicode.IsSpace) {
		return "", "", fmt.Errorf("en-tête %q : nom invalide", line)
	}
	return name, strings.TrimSpace(value), nil
}

func main() {
	for _, t := range []string{
		"Les Œuvres complètes — tome 2",
		"  Go 1.27 : quoi de neuf ?  ",
		"Ça, c'est l'été!",
	} {
		fmt.Printf("%-34q → %s\n", t, Slug(t))
	}

	for _, h := range []string{"Content-Type: text/html", "X-Id:42", "sans deux-points", "Mauvais Nom: x"} {
		n, v, err := ParseHeader(h)
		if err != nil {
			fmt.Println("erreur :", err)
			continue
		}
		fmt.Printf("%s=%q json? %v\n", n, v, strings.EqualFold(n, "content-type") && strings.HasPrefix(v, "application/json"))
	}

	fmt.Printf("%q %q\n", strings.Split("", ","), strings.Fields(""))
	fmt.Println(strings.TrimLeft("https://shop.example", "https://"), "|", strings.TrimPrefix("https://shop.example", "https://"))
}
```

Sortie :

```
"Les Œuvres complètes — tome 2"    → les-oeuvres-completes-tome-2
"  Go 1.27 : quoi de neuf ?  "     → go-1-27-quoi-de-neuf
"Ça, c'est l'été!"                 → ca-c-est-l-ete
Content-Type="text/html" json? false
X-Id="42" json? false
erreur : en-tête "sans deux-points" : « : » manquant
erreur : en-tête "Mauvais Nom: x" : nom invalide
[""] []
op.example | shop.example
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `var slugReplacer = …` | Construit une fois au chargement du paquet ; chaque appel de `Slug` le réutilise. |
| `ToLower` **avant** le `Replacer` | Sinon il faudrait lister `É`, `À`… en double. L'ordre des transformations fait partie de l'algorithme. |
| `FieldsFunc` | Découpe sur tout ce qui n'est pas lettre ou chiffre ASCII et **ignore les vides** : pas de tirets doubles ni en bord, sans aucun nettoyage supplémentaire. |
| `r < unicode.MaxASCII` | Toute lettre non ASCII restante (un `ñ` non prévu) devient séparateur plutôt que de produire une URL non ASCII. Choix à documenter. |
| `ParseHeader` avec `Cut` | Coupe au **premier** `:` — une valeur peut contenir des `:` (`Host: a:8080`). |
| `EqualFold` | Les noms d'en-têtes HTTP sont insensibles à la casse. |
| Dernière ligne | Le piège de `TrimLeft` en action : `h`, `t`, `p`, `s`, `:`, `/` sont retirés **tant qu'il y en a** — y compris le `sh` de `shop`. |

## Erreurs fréquentes

1. `len(strings.Split(s, ","))` sur une chaîne vide → 1.
2. `TrimLeft`/`Trim` avec une chaîne qu'on croit être un préfixe.
3. `strings.Title` (déprécié) pour mettre en forme des noms.
4. `ToLower(a) == ToLower(b)` au lieu de `EqualFold`.
5. Utiliser un indice d'octet comme une position de caractère.
6. Enchaîner des `ReplaceAll` dont les remplacements interfèrent.
7. `s += …` dans une boucle.
8. Reconstruire un `Replacer` (ou `strings.NewReplacer(...)`) à chaque appel.
9. Convertir `[]byte` → `string` pour utiliser `strings`, alors que `bytes` a la même fonction.

## Bonnes pratiques Go

- `go doc` avant toute recherche.
- `Cut` pour couper en deux, `Fields` pour du texte humain, `Split` pour un format strict.
- `TrimPrefix`/`CutPrefix` pour les préfixes, `TrimSpace` pour les espaces.
- `Builder` dans les boucles, `Join` pour les listes, `+` pour deux ou trois morceaux.
- Les `Replacer` et autres tables en variables de paquet, construits une fois.
- Pour tout ce qui est linguistique (casse, tri, accents) : `golang.org/x/text`.

## Ce que je dois retenir

- `Split("")` rend `[""]` ; `Fields` ignore les vides.
- `Trim*` prend un **ensemble** de caractères ; `TrimPrefix` un préfixe.
- Les indices de `strings` sont des indices d'**octets**.
- `Replacer` : plusieurs remplacements en **un** passage, sans interférence.
- `Map` avec `-1` supprime une rune.
- `+=` en boucle est quadratique.
- `bytes` duplique l'API pour les `[]byte`.

---

➡️ [Exercices](exercices.md)
