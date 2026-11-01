# Leçon 2 — Expressions régulières

> Le paquet `regexp` de Go a une particularité qui surprend les développeurs venus de Perl,
> PHP, Python ou JavaScript : il **refuse** certaines syntaxes qu'ils utilisent tous les
> jours, comme les rétro-références. Ce n'est pas un manque, c'est une garantie : une
> expression régulière Go s'exécute toujours en **temps linéaire**. Cette leçon explique
> l'échange, puis l'API.

## Objectifs

1. Comprendre la garantie de RE2 et ce qu'elle interdit.
2. Compiler une fois, utiliser souvent ; savoir quand `MustCompile` est légitime.
3. Extraire avec des groupes, nommés ou non, et remplacer avec `$1`/`${nom}`.
4. Savoir quand **ne pas** utiliser d'expression régulière.

## Explication

### RE2 : le temps linéaire contre les rétro-références

La plupart des moteurs (PCRE en PHP, `re` en Python, celui de JavaScript) procèdent par
**retour arrière** (*backtracking*) : ils essaient une possibilité, et reviennent en arrière
en cas d'échec. Sur certaines expressions, le nombre de possibilités explose :

```
(a+)+$   sur   "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaa!"
```

Un moteur à retour arrière essaie toutes les façons de répartir les `a` entre les deux `+` :
2³⁰ tentatives. C'est une **attaque par déni de service** connue (ReDoS) : une seule requête
HTTP contenant une chaîne bien choisie bloque un cœur de processeur pendant des minutes.

Go utilise **RE2**, un moteur à base d'automates qui garantit un temps **proportionnel à la
taille de l'entrée**, quelle que soit l'expression. La même expression sur 50 000 `a` répond
instantanément en Go.

Le prix : tout ce qui exige un retour arrière est **refusé à la compilation** :

```go
regexp.Compile(`(\w)\1`)    // error parsing regexp: invalid escape sequence: `\1`
regexp.Compile(`a(?=b)`)    // error parsing regexp: invalid or unsupported Perl syntax: `(?=`
```

| Refusé | Pourquoi | Alternative en Go |
|---|---|---|
| Rétro-références `\1` | exige de mémoriser et revenir | code Go après extraction |
| Lookahead `(?=…)`, `(?!…)` | idem | deux expressions, ou du code |
| Lookbehind `(?<=…)` | idem | idem |

Pour un serveur qui applique des expressions à des données **non fiables**, cette garantie
vaut largement les fonctionnalités perdues.

### Compiler une fois

```go
var dateRe = regexp.MustCompile(`(\d{4})-(\d{2})-(\d{2})`)
```

Compiler coûte cher (analyse, construction d'automate) ; utiliser est bon marché. On compile
donc **une fois**, dans une variable de paquet, et pas à chaque appel de fonction.

`MustCompile` **panique** si l'expression est invalide. C'est le cas d'école du niveau 2,
leçon 7 : une panique est légitime pour une **erreur de programmation** détectée au démarrage
— l'expression est une constante écrite par le développeur, et le programme ne démarrera pas
du tout si elle est fausse. Si l'expression vient d'un **utilisateur** ou d'un fichier de
configuration, on utilise `regexp.Compile` et on traite l'erreur.

Un `*regexp.Regexp` peut être utilisé par plusieurs goroutines simultanément (niveau 6).

### Les littéraux bruts

Toujours écrire les expressions entre **accents graves** :

```go
regexp.MustCompile(`\d+\.\d+`)    // lisible
regexp.MustCompile("\\d+\\.\\d+") // la même, en chaîne interprétée : à éviter
```

### L'API : un nom construit par morceaux

Les méthodes ont des noms longs mais **systématiques** :

```
Find(All)?(String)?(Submatch)?(Index)?
```

| Morceau | Signification |
|---|---|
| `All` | toutes les correspondances (avec un argument `n` : `-1` = sans limite) |
| `String` | travaille sur `string` ; sans lui, sur `[]byte` |
| `Submatch` | rend aussi les **groupes** de capture |
| `Index` | rend des **positions** au lieu du texte |

Donc `FindAllStringSubmatch(s, -1)` rend `[][]string` : pour chaque correspondance, le texte
complet suivi de chaque groupe.

```go
re := regexp.MustCompile(`(\d{4})-(\d{2})`)
re.FindStringSubmatch("le 2026-11 et 2027-01")
// [2026-11 2026 11]             ← [0] = tout, [1] = groupe 1, [2] = groupe 2
re.FindAllStringSubmatch("le 2026-11 et 2027-01", -1)
// [[2026-11 2026 11] [2027-01 2027 01]]
```

Et pour un simple test : `re.MatchString(s)`.

### Attention : une correspondance **partielle** suffit

```go
regexp.MustCompile(`\d+`).MatchString("abc123def")   // true
```

`MatchString` cherche l'expression **n'importe où**. Pour valider un format complet, il faut
**ancrer** : `^\d+$`. Oublier les ancres est la première source de failles dans les
validations par expression régulière : `[a-z]+` « valide » `<script>alert(1)</script>a`.

### Groupes nommés

```go
re := regexp.MustCompile(`(?P<year>\d{4})-(?P<month>\d{2})`)
m := re.FindStringSubmatch(s)
year := m[re.SubexpIndex("year")]
```

Depuis Go 1.22, la syntaxe `(?<year>…)` est aussi acceptée. Les noms rendent le code
résistant à l'ajout d'un groupe au milieu de l'expression, qui décalerait tous les indices.

### Remplacer

```go
re.ReplaceAllString("2026-11", "$month/$year")  // "11/2026"
```

Piège : dans le gabarit, `$1x` désigne le groupe **nommé** `1x` (qui n'existe pas → chaîne
vide), pas « groupe 1 suivi de x ». Il faut des accolades : `${1}x`.

```go
re.ReplaceAllString("2026-11", "$1x")    // ""      (groupe « 1x » inexistant)
re.ReplaceAllString("2026-11", "${1}x")  // "2026x"
```

Pour un remplacement calculé : `ReplaceAllStringFunc(s, func(match string) string { … })`.
Pour insérer du texte sans interpréter `$` : `ReplaceAllLiteralString`.

### `\w`, `\b` et l'Unicode

```go
regexp.MustCompile(`\w+`).FindAllString("été élève", -1)    // [t l ve]
regexp.MustCompile(`\p{L}+`).FindAllString("été élève", -1) // [été élève]
```

En RE2, `\w`, `\d`, `\b` et `\s` sont **ASCII seulement**. Pour du texte français, utiliser
les classes Unicode : `\p{L}` (lettre), `\p{N}` (chiffre), `\p{Lu}` (majuscule).

### Gourmand ou paresseux

`a+` prend le plus possible (`"aaa"` dans `"baaab"`), `a+?` le moins possible (`"a"`). Classique
pour extraire entre deux délimiteurs : `".*?"` plutôt que `".*"`, qui irait jusqu'au
**dernier** guillemet de la ligne.

### Quand ne pas en utiliser

Une expression régulière est souvent la **mauvaise** réponse :

| Besoin | Mieux que regexp |
|---|---|
| préfixe, suffixe, contient | `strings.HasPrefix`, `HasSuffix`, `Contains` |
| découper sur un séparateur fixe | `strings.Cut`, `Split`, `Fields` |
| un nombre | `strconv.Atoi` — il valide **et** convertit |
| une date | `time.Parse` (leçon 4) |
| une URL, une adresse e-mail, une IP | `net/url`, `net/mail`, `net/netip` |
| JSON, HTML, XML | un vrai analyseur (`encoding/json`, `golang.org/x/net/html`) |

Côté performance, la réalité est nuancée, et se **mesure** : sur un motif purement littéral
(`needle`), le moteur repère le littéral et va aussi vite que `strings.Contains`. Mais dès que
le motif contient une classe, `(?i)` ou `\b`, on mesure facilement un facteur 50 sur un gros
texte, et un appel court (`^#` sur une ligne) coûte de l'ordre de 100 fois un
`strings.HasPrefix`. On les réserve aux motifs réellement variables.

## Exemple

```go
package main

import (
	"fmt"
	"regexp"
	"strings"
)

// Analyse de lignes de journal d'accès HTTP au format :
//   127.0.0.1 - [03/Oct/2026:08:12:01] "GET /users/42 HTTP/1.1" 200 512 12ms
var accessRe = regexp.MustCompile(
	`^(?P<ip>\S+) - \[(?P<time>[^\]]+)\] "(?P<method>[A-Z]+) (?P<path>\S+) [^"]*" (?P<status>\d{3}) (?P<size>\d+) (?P<dur>\d+)ms$`,
)

// Les identifiants numériques dans les chemins, pour regrouper /users/42 et /users/7.
var idRe = regexp.MustCompile(`/\d+(/|$)`)

type Access struct {
	IP, Method, Path, Status string
}

func parse(line string) (Access, error) {
	m := accessRe.FindStringSubmatch(line)
	if m == nil {
		return Access{}, fmt.Errorf("ligne non reconnue : %q", line)
	}
	get := func(name string) string { return m[accessRe.SubexpIndex(name)] }
	return Access{IP: get("ip"), Method: get("method"), Path: get("path"), Status: get("status")}, nil
}

func normalize(path string) string {
	return idRe.ReplaceAllString(path, "/{id}${1}")
}

func main() {
	logs := `127.0.0.1 - [03/Oct/2026:08:12:01] "GET /users/42 HTTP/1.1" 200 512 12ms
10.0.0.7 - [03/Oct/2026:08:12:02] "GET /users/7/orders/913 HTTP/1.1" 200 2048 31ms
10.0.0.7 - [03/Oct/2026:08:12:02] "DELETE /users/7 HTTP/1.1" 403 0 2ms
ligne corrompue
10.0.0.9 - [03/Oct/2026:08:12:03] "POST /login HTTP/1.1" 401 64 120ms`

	byRoute := map[string]int{}
	for line := range strings.Lines(logs) {
		a, err := parse(strings.TrimSuffix(line, "\n"))
		if err != nil {
			fmt.Println("ignorée :", err)
			continue
		}
		byRoute[a.Method+" "+normalize(a.Path)]++
		if a.Status[0] == '4' {
			fmt.Printf("refus %s pour %s %s\n", a.Status, a.IP, a.Path)
		}
	}
	fmt.Println(byRoute)

	// La garantie RE2 : refusé à la compilation plutôt que lent à l'exécution.
	if _, err := regexp.Compile(`(\w+) \1`); err != nil {
		fmt.Println("compile :", err)
	}
}
```

Sortie :

```
refus 403 pour 10.0.0.7 /users/7
ignorée : ligne non reconnue : "ligne corrompue"
refus 401 pour 10.0.0.9 /login
map[DELETE /users/{id}:1 GET /users/{id}:1 GET /users/{id}/orders/{id}:1 POST /login:1]
compile : error parsing regexp: invalid escape sequence: `\1`
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `var accessRe = regexp.MustCompile(…)` | Compilée une fois au chargement. Si elle est fausse, le programme ne démarre pas — c'est voulu. |
| `^…$` | Ancrée aux deux bouts : une ligne partiellement conforme est **rejetée**. |
| `(?P<ip>\S+)` | Groupes nommés ; `\S` = tout sauf espace. |
| `[^\]]+`, `[^"]*` | Classes niées plutôt que `.*?` : plus explicites et sans ambiguïté. |
| `m == nil` | Pas de correspondance ; tester **avant** d'indexer. |
| `get := func(name string) …` | Closure sur `m` : le code reste lisible malgré sept groupes. |
| `idRe` avec `(/\|$)` et `${1}` | Remplace `/42` qu'il soit suivi d'un `/` ou en fin de chaîne, en remettant le séparateur capturé. `${1}` avec accolades : obligatoire ici puisque rien ne sépare `1` d'un éventuel texte suivant. |
| `a.Status[0] == '4'` | Pas besoin d'une regexp pour tester le premier chiffre. |

## Erreurs fréquentes

1. **Compiler dans la fonction appelée en boucle.**
2. **Oublier `^` et `$`** dans une validation.
3. **`$1x` au lieu de `${1}x`** dans un gabarit de remplacement.
4. **`\w` sur du texte accentué.**
5. **`MustCompile` sur une expression fournie par l'utilisateur** : un caractère mal placé fait
   tomber le serveur.
6. **Chercher une rétro-référence** pendant une heure : RE2 ne les a pas, c'est documenté.
7. **Regexp pour ce que `strings` ou `strconv` font déjà.**
8. **Analyser du HTML ou du JSON** avec des expressions régulières.
9. **`.*` gourmand** entre deux délimiteurs.

## Bonnes pratiques Go

- Variables de paquet + `MustCompile` pour les expressions constantes.
- `Compile` + traitement d'erreur pour les expressions venues de l'extérieur.
- Littéraux bruts entre accents graves.
- Groupes nommés dès qu'il y en a plus de deux.
- `regexp.QuoteMeta(s)` pour insérer un texte littéral dans une expression.
- Toujours se demander si `strings` ne suffit pas.

## Ce que je dois retenir

- RE2 : **temps linéaire garanti**, donc **pas** de rétro-références ni de lookaround.
- Compiler une fois ; `MustCompile` pour les constantes seulement.
- `Find(All)?(String)?(Submatch)?(Index)?`.
- `MatchString` cherche n'importe où : **ancrer** pour valider.
- `${1}` et `${nom}` dans les remplacements.
- `\w`, `\b`, `\d` sont ASCII ; `\p{L}` pour l'Unicode.

---

➡️ [Exercices](exercices.md)
