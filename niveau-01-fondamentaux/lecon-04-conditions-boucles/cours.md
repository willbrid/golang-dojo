# Leçon 4 — Conditions et boucles

## Objectifs

1. Écrire un `if` idiomatique, avec instruction d'initialisation, et pratiquer le *early return*.
2. Maîtriser le `switch` de Go, qui ne ressemble à celui d'aucun autre langage courant.
3. Connaître les **quatre formes** de `for` — le seul mot-clé de boucle du langage.
4. Utiliser `range` correctement sur chaque type, et éviter ses pièges.

## Explication

### `if` : pas de parenthèses, accolades obligatoires

```go
if x > 10 {
	fmt.Println("grand")
} else if x > 5 {
	fmt.Println("moyen")
} else {
	fmt.Println("petit")
}
```

Pas de parenthèses autour de la condition ; accolades **obligatoires** même pour une seule
instruction. La condition doit être un `bool` : `if x { … }` avec `x` entier ne compile
pas. Adieu le `if (x = 5)` du C.

### L'instruction d'initialisation : l'idiome le plus courant de Go

```go
if v, ok := m[key]; ok {
	fmt.Println(v)
}   // v et ok n'existent plus ici

if err := doSomething(); err != nil {
	return err
}
```

La variable déclarée avant le `;` n'existe que dans le `if` et ses `else`. C'est
l'idiome le plus fréquent du langage : il **limite la portée** au strict nécessaire.

### *Early return* : la forme de code Go par excellence

```go
// À éviter : imbrication en escalier
func process(u *User) error {
	if u != nil {
		if u.Active {
			if err := save(u); err == nil {
				return nil
			} else {
				return err
			}
		} else {
			return errors.New("inactif")
		}
	}
	return errors.New("nil")
}

// Idiomatique : les cas d'erreur sortent tôt, le chemin normal reste à gauche
func process(u *User) error {
	if u == nil {
		return errors.New("utilisateur nil")
	}
	if !u.Active {
		return errors.New("utilisateur inactif")
	}
	return save(u)
}
```

La règle, formulée dans *Effective Go* : **le chemin heureux reste aligné à gauche**, les
cas d'erreur sortent immédiatement. On lit la fonction en diagonale sans dérouler
mentalement une pile de `else`. Un `else` après un `return` est presque toujours du bruit.

### `switch` : sans `break`, et beaucoup plus puissant

```go
switch day {
case "samedi", "dimanche":     // plusieurs valeurs par cas
	fmt.Println("week-end")
case "lundi":
	fmt.Println("courage")
default:
	fmt.Println("semaine")
}
```

Différences majeures avec C, Java ou JavaScript :

- **Pas de `break` implicite à écrire** : Go ne « tombe » pas d'un cas au suivant.
  L'oubli de `break`, bug classique en C, n'existe pas en Go.
- Pour forcer le passage au cas suivant : `fallthrough` (rare, et il faut le justifier).
- Un cas peut lister **plusieurs valeurs**.

**Le switch sans expression** remplace avantageusement les cascades de `if / else if` :

```go
switch {
case score >= 90:
	grade = "A"
case score >= 80:
	grade = "B"
default:
	grade = "F"
}
```

Il accepte aussi une instruction d'initialisation :

```go
switch hour := time.Now().Hour(); {
case hour < 12:
	return "bonjour"
default:
	return "bonsoir"
}
```

Le **switch de type** (`switch v := x.(type)`) existe aussi, mais il attendra la leçon 11
sur les interfaces.

### `for` : un seul mot-clé, quatre formes

Go n'a ni `while`, ni `do…while`, ni `foreach`. Tout passe par `for`.

```go
// 1. Classique
for i := 0; i < 10; i++ { }

// 2. Condition seule — le « while » des autres langages
for x < 100 { x *= 2 }

// 3. Infinie — pour les boucles de service, avec sortie explicite
for {
	if done { break }
}

// 4. range — sur slice, tableau, map, string, channel, entier, fonction
for i, v := range items { }
```

La forme `range` sur un **entier** existe depuis Go 1.22 :

```go
for i := range 5 {       // 0, 1, 2, 3, 4
	fmt.Println(i)
}
for range 3 {            // 3 fois, sans variable
	fmt.Println("tic")
}
```

C'est la façon moderne d'écrire « répéter n fois ». Le `range` sur **fonction**
(itérateurs personnalisés, Go 1.23) sera vu au niveau 2.

### `range` : ce qu'il donne selon le type

| Type parcouru | Première variable | Seconde variable |
|---|---|---|
| `[]T`, `[N]T` | indice `int` | **copie** de l'élément |
| `map[K]V` | clé | valeur — **ordre aléatoire** |
| `string` | décalage en **octets** | `rune` décodée |
| `chan T` | valeur reçue | — (une seule variable) |
| `int` (1.22+) | `0 … n-1` | — |
| `func(yield)` (1.23+) | selon l'itérateur | selon l'itérateur |

Deux points à graver :

**1. `range` copie l'élément.** Modifier `v` ne modifie pas le slice :

```go
for _, v := range items {
	v.Count++          // sans effet : v est une copie
}
for i := range items {
	items[i].Count++   // correct
}
```

**2. L'ordre d'itération d'une map est délibérément aléatoire**, et il change à chaque
exécution. Ce n'est pas un défaut : le runtime randomise volontairement pour empêcher tout
code de dépendre d'un ordre non garanti. Pour une sortie déterministe, il faut trier les
clés (leçon 7).

### Le piège de la variable de boucle : corrigé en Go 1.22

Avant Go 1.22, `i` et `v` étaient **une seule variable réutilisée** à chaque tour, ce qui
produisait ce grand classique :

```go
for _, v := range []int{1, 2, 3} {
	go func() { fmt.Println(v) }()   // affichait souvent 3 3 3 avant Go 1.22
}
```

**Depuis Go 1.22, la variable est recréée à chaque itération** et ce code affiche bien 1,
2, 3 (dans un ordre quelconque). Comportement piloté par la ligne `go` de `go.mod` : un
module déclarant `go 1.21` conserve l'ancienne sémantique. C'est un excellent exemple de
ce que signifie réellement la ligne `go 1.27` vue en leçon 1.

Il faut connaître ce piège : beaucoup de code, de tutoriels et de réponses en ligne datent
d'avant 1.22.

### `break`, `continue`, et les labels

```go
outer:
for i := range 3 {
	for j := range 3 {
		if i*j > 2 {
			break outer      // sort des DEUX boucles
		}
		if j == 1 {
			continue outer   // passe au i suivant
		}
	}
}
```

Un `break` nu ne sort que de la boucle la plus interne. Les labels sont le seul usage
acceptable de quelque chose qui ressemble à un `goto` — ils restent rares mais légitimes,
notamment pour sortir d'une boucle depuis un `select` (niveau 4).

## Exemple

```go
package main

import (
	"fmt"
	"strings"
)

// classify range un mot dans une catégorie.
func classify(word string) string {
	switch n := len(word); {
	case n == 0:
		return "vide"
	case n <= 3:
		return "court"
	case n <= 8:
		return "moyen"
	default:
		return "long"
	}
}

// firstDuplicate retourne le premier mot répété et true, ou "" et false.
func firstDuplicate(words []string) (string, bool) {
	seen := make(map[string]bool)
	for _, w := range words {
		if seen[w] {
			return w, true
		}
		seen[w] = true
	}
	return "", false
}

func main() {
	phrase := "le chat dort et le chien dort aussi"
	words := strings.Fields(phrase)

	for i, w := range words {
		fmt.Printf("%d. %-8s %s\n", i+1, w, classify(w))
	}

	if dup, ok := firstDuplicate(words); ok {
		fmt.Printf("premier doublon : %q\n", dup)
	} else {
		fmt.Println("aucun doublon")
	}

	// Recherche d'une paire, avec sortie des deux boucles.
	target := "chien"
search:
	for i, a := range words {
		for _, b := range words[i+1:] {
			if a+" "+b == "le "+target {
				fmt.Printf("trouvé à l'indice %d\n", i)
				break search
			}
		}
	}
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `switch n := len(word); {` | Initialisation **et** switch sans expression. `n` n'existe que dans le switch. Plus lisible qu'une cascade de `if`. |
| `return w, true` | Retours multiples : l'idiome « valeur + drapeau ». Voir aussi `v, ok := m[k]`. |
| `seen[w]` sur une map absente | Retourne la **zéro-valeur** `false` — pas d'erreur, pas de panique. Un accès en lecture à une clé absente est toujours sûr. |
| `words[i+1:]` | Slice à partir de `i+1` : évite de comparer une paire deux fois. Détail : ne copie rien (leçon 6). |
| `%-8s` | Chaîne alignée à gauche sur 8 caractères. Le formatage de `fmt` mérite un `go doc fmt`. |
| `break search` | Sort des deux boucles. Sans le label, seule la boucle interne s'arrêterait. |

## Erreurs fréquentes

1. **Mettre des parenthèses** : `if (x > 10) {` compile mais `gofmt` les retire. Ce n'est pas du Go.
2. **Écrire `break` à la fin de chaque `case`** : inutile, Go ne « tombe » pas. Réflexe importé du C.
3. **Modifier la copie dans `range`** : `for _, v := range s { v.X = 1 }` ne fait rien. Utiliser l'indice.
4. **Dépendre de l'ordre d'une map.** Le test passera dix fois puis échouera en CI.
5. **Modifier un slice pendant qu'on le parcourt** avec `range` : la longueur est évaluée **une seule fois**, au début. Ajouter des éléments dans la boucle ne les fera pas parcourir.
6. **`else` après un `return`** : bruit visuel. Le *early return* est la forme attendue.
7. **Boucle infinie sans condition de sortie visible** : `for { }` sans `break`, `return` ni `select` est presque toujours un bug.
8. **Croire au vieux piège de la variable de boucle** en Go ≥ 1.22 — ou pire, l'ignorer sur du code ancien.

## Bonnes pratiques Go

- *Early return* systématique ; garder le chemin normal à gauche.
- `if v, ok := …; ok` pour limiter la portée des variables temporaires.
- `switch` sans expression plutôt que trois `else if` ou plus.
- `for i := range n` (1.22+) pour « répéter n fois ».
- `for _, v := range` quand l'indice est inutile ; `for i := range` quand la valeur l'est.
- Nommer les labels avec un sens (`search:`, `outer:`), et les utiliser avec parcimonie.
- Préférer une fonction avec `return` à une boucle avec drapeau `found := true`.

## Ce que je dois retenir

- `if` sans parenthèses, accolades obligatoires, condition **booléenne** stricte.
- L'instruction d'initialisation (`if v, ok := …; ok`) limite la portée : idiome n°1 de Go.
- `switch` n'a **pas** de `break` implicite à écrire ; il accepte plusieurs valeurs par cas et fonctionne sans expression.
- **Un seul mot-clé de boucle** : `for`, sous quatre formes.
- `range` **copie** la valeur ; l'ordre des maps est **aléatoire par conception**.
- Depuis **Go 1.22**, la variable de boucle est recréée à chaque itération.

➡️ [Exercices](exercices.md)
