# Leçon 1 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Prédire `Split`

| Appel | Résultat |
|---|---|
| `Split("a,b,c", ",")` | `["a" "b" "c"]` |
| `Split("a,b,c,", ",")` | `["a" "b" "c" ""]` — le séparateur final produit un dernier champ vide |
| `Split(",", ",")` | `["" ""]` — un séparateur, deux champs, tous deux vides |
| `Split("", ",")` | `[""]` |
| `Split("abc", "")` | `["a" "b" "c"]` — séparateur vide : découpe après chaque **rune** UTF-8 |
| `SplitN("a,b,c", ",", 0)` | `[]` — et c'est un slice **nil** |
| `SplitN("a,b,c", ",", -1)` | `["a" "b" "c"]` — `n < 0` = pas de limite |
| `Fields("\t a \n b  ")` | `["a" "b"]` — tabulations et retours à la ligne sont des espaces |

La phrase de `go doc strings.Split` qui explique le cas vide : *« If s does not contain sep
and sep is not empty, Split returns a slice of length 1 whose only element is s. »* La chaîne
vide ne contient pas `","` : on rend donc `[s]`, c'est-à-dire `[""]`.

## E2 — Le piège de `TrimLeft`

Mesuré :

```
https://shop.fr      → op.fr
https://tests.io     → ests.io
https://phpstan.org  → an.org
http://site.fr       → ite.fr
```

Toute URL dont le domaine commence par une lettre parmi `h t p s :` `/` est amputée.

```go
func host(url string) (string, error) {
	for _, scheme := range []string{"https://", "http://"} {
		if rest, ok := strings.CutPrefix(url, scheme); ok {
			h, _, _ := strings.Cut(rest, "/")
			return h, nil
		}
	}
	return "", fmt.Errorf("%q : schéma http(s) attendu", url)
}
```

En vrai code : `net/url.Parse`, qui gère ports, identifiants, IPv6, etc. (niveau 7). Écrire
son propre analyseur d'URL est un exercice, pas une pratique.

## E3 — Échapper

```go
func EscapeCSV(field string) string {
	if !strings.ContainsAny(field, ",\"\n\r") {
		return field
	}
	return `"` + strings.ReplaceAll(field, `"`, `""`) + `"`
}
```

Un `Replacer` n'apporte rien : il n'y a qu'**un** remplacement, et l'ajout des guillemets
extérieurs dépend d'une **condition** sur la chaîne entière, ce qu'un `Replacer` ne sait pas
exprimer. Les littéraux entre accents graves (`` `"` ``) évitent d'échapper les guillemets.

En vrai code : `encoding/csv`, qui gère aussi la lecture.

## E4 — Le coût de `+=`

Mesures sur la machine de rédaction (les tiennes différeront) :

| n | `+=` | `Builder` |
|---|---|---|
| 10 000 | 22 ms | 57 µs |
| 20 000 | 37 ms | 77 µs |
| 40 000 | 137 ms | 89 µs |
| 80 000 | 611 ms | 227 µs |

Quand `n` double, `+=` est multiplié par **3 à 4** — tendance vers 4, la signature d'un coût
**quadratique** (O(n²)) : chaque concaténation recopie toute la chaîne précédente. Le
`Builder` est grosso modo multiplié par 2 : **linéaire**, son buffer grandit par doublement
comme un slice (niveau 1, leçon 6).

Ces mesures au `time.Since` sont bruitées (le premier point est souvent faussé par le
démarrage). Le niveau 5 (leçon 6) apprendra à mesurer proprement avec `testing.B`.

## E5 — Positions

```go
s := "Le café coûte 3€"
i := strings.Index(s, "3")                 // 16
n := utf8.RuneCountInString(s[:i])         // 14
```

`L e ␣ c a f é ␣ c o û t e ␣` : 14 caractères avant le `3`. `é` et `û` occupent chacun
**2 octets** en UTF-8 : 14 + 2 = 16. Le `€` (3 octets) est **après** le `3`, il ne compte pas.

## Exercice intermédiaire — `wrap`

```go
func Wrap(text string, width int) (string, error) {
	if width <= 0 {
		return "", fmt.Errorf("largeur invalide : %d", width)
	}
	var b strings.Builder
	b.Grow(len(text))
	for i, para := range splitParagraphs(text) {
		if i > 0 {
			b.WriteString("\n\n")
		}
		lineLen := 0
		for j, w := range para {
			wl := utf8.RuneCountInString(w)
			switch {
			case j == 0:
				// premier mot du paragraphe : rien avant
			case lineLen+1+wl > width:
				b.WriteByte('\n')
				lineLen = 0
			default:
				b.WriteByte(' ')
				lineLen++
			}
			b.WriteString(w)
			lineLen += wl
		}
	}
	return b.String(), nil
}

// splitParagraphs découpe sur les lignes vides et rend les mots de chaque paragraphe.
func splitParagraphs(text string) [][]string {
	var paras [][]string
	var cur []string
	for line := range strings.Lines(text) {
		if strings.TrimSpace(line) == "" {
			if len(cur) > 0 {
				paras = append(paras, cur)
				cur = nil
			}
			continue
		}
		cur = append(cur, strings.Fields(line)...)
	}
	if len(cur) > 0 {
		paras = append(paras, cur)
	}
	return paras
}
```

Points de correction :

- **`RuneCountInString`, pas `len`** : avec `len`, une ligne de 20 caractères dont 3 accentués
  « mesure » 23 et serait coupée trop tôt. C'est le piège de la contrainte 6.
- **Pas d'espace final** par construction : l'espace s'écrit **avant** un mot, jamais après.
  C'est plus simple que de retirer l'espace en trop à la fin de chaque ligne.
- **Mot trop long** (contrainte 2) : en tête de ligne, il est écrit sans test de longueur ;
  la ligne dépasse, mais c'est le comportement demandé.
- `strings.Lines` (Go 1.24) itère sur les lignes **avec** leur `\n` final — d'où le
  `TrimSpace` pour détecter les lignes vides.
- Vérification sur un texte accentué, `width = 20`, sortie passée à `cat -A` : aucune ligne ne
  finit par un espace, `anticonstitutionnellement` (25 caractères) est seul sur sa ligne, et
  `Été évité, élève` (16 caractères, 21 octets) n'a pas été coupé à tort.

## Défi — Un `Replacer` qui ne fait pas ce qu'on croit

1. Mesuré : `NewReplacer("a", "1", "aa", "2").Replace("aaa")` → `"111"` ;
   `NewReplacer("aa", "2", "a", "1").Replace("aaa")` → `"21"`.
   Règle : **à chaque position, le premier couple dans l'ordre des arguments qui correspond
   l'emporte** — pas le plus long. La documentation le dit : *« Replacements are performed in
   the order they appear in the target string, without overlapping matches »* et *« comparisons
   are done in argument order »*. Conséquence pratique : lister les motifs **les plus longs
   d'abord** quand l'un est préfixe d'un autre.

2. Un `Replacer` sait retirer `**`, `*` et `` ` `` (remplacement par `""`, en mettant `**`
   **avant** `*` d'après la règle précédente). Il ne sait pas transformer `[texte](url)` en
   `texte (url)` : il faut **capturer** deux morceaux variables et les **réordonner**, alors
   qu'un `Replacer` ne remplace que des chaînes fixes par des chaînes fixes. C'est exactement
   ce que font les groupes de capture d'une expression régulière : `regexp` et
   `ReplaceAllString` avec `$1 ($2)`, leçon 2.

## Quiz

1. `[""]` (un élément) ; `[]` (aucun).
2. `TrimLeft` retire **toutes** les runes `a` et `b` en tête, quel que soit leur ordre ;
   `TrimPrefix` retire le préfixe `"ab"` une fois s'il est présent.
3. Pas d'allocation, et gestion des équivalences de casse Unicode simples.
4. Sa règle de découpage en mots ne respecte les conventions d'aucune langue ; remplacé par
   `golang.org/x/text/cases`.
5. Non, en octets.
6. Un `ReplaceAll` peut modifier le texte produit par un précédent ; le `Replacer` ne revient
   jamais sur un texte déjà remplacé.
7. Elle supprime la rune.
8. Chaque `+=` alloue et recopie toute la chaîne : O(n²) au total.
9. Quand les données sont déjà des `[]byte` (fichier, réseau) : on évite une conversion qui
   copie.
10. `go doc -ex paquet.Fonction`.
