# Leçon 5 — Boucles

## Objectifs

1. Connaître les **quatre formes** de `for` — le seul mot-clé de boucle de Go.
2. Utiliser `range` sur un entier (Go 1.22+) et savoir ce qu'il apportera sur les autres types.
3. Sortir proprement d'une boucle imbriquée avec un label.
4. Comprendre le changement de sémantique de la variable de boucle en Go 1.22.

## Explication

### Un seul mot-clé

Go n'a ni `while`, ni `do…while`, ni `foreach`, ni `loop`. **Tout passe par `for`**, sous
quatre formes.

**1. La forme classique, à trois clauses**

```go
for i := 0; i < 10; i++ {
	fmt.Println(i)
}
```

`init ; condition ; post`. `i` n'existe que dans la boucle. Les trois clauses sont
facultatives individuellement.

**2. La condition seule — le `while` des autres langages**

```go
x := 1
for x < 100 {
	x *= 2
}
```

Aucun mot-clé nouveau : on omet simplement l'initialisation et le post.

**3. La boucle infinie**

```go
for {
	line, err := read()
	if err != nil {
		break
	}
	process(line)
}
```

C'est la forme des boucles de service : serveurs, boucles d'événements, consommateurs de
messages. Elle **doit** contenir une sortie visible (`break`, `return`, ou un `select` avec
annulation au niveau 6). Une boucle infinie sans sortie identifiable est presque toujours un bug.

**4. `range`**

```go
for i := range 5 {      // 0 1 2 3 4        — Go 1.22+
	fmt.Println(i)
}

for range 3 {           // trois tours, sans variable
	fmt.Println("tic")
}
```

`range` sur un **entier** est arrivé en Go 1.22 et remplace le `for i := 0; i < n; i++`
quand seul le compteur importe. C'est la façon moderne d'écrire « répéter n fois ».

`range` s'applique aussi aux **slices et tableaux** (leçon 6), aux **maps** (leçon 7),
aux **chaînes** (leçon 8), aux **channels** (niveau 6) et aux **fonctions itératrices**
(niveau 3, Go 1.23+). Chacune de ces leçons introduira la forme correspondante avec ses
pièges propres — ils ne sont pas les mêmes selon le type parcouru.

### `break` et `continue`

```go
for i := range 10 {
	if i%2 == 0 {
		continue    // passe au tour suivant
	}
	if i > 7 {
		break       // sort de la boucle
	}
	fmt.Println(i)  // 1 3 5 7
}
```

`break` et `continue` nus n'agissent que sur la boucle **la plus interne**.

### Les labels

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
		fmt.Println(i, j)
	}
}
```

Un label se place immédiatement avant la boucle, sans indentation supplémentaire — `gofmt`
l'aligne sur la colonne du `for`. C'est le seul usage courant de quelque chose qui ressemble
à un saut, et il est parfaitement idiomatique quand l'alternative est un drapeau booléen
propagé à la main.

Cela dit, l'alternative la plus propre est souvent d'**extraire la boucle dans une fonction**
et d'utiliser `return` : le code devient nommé, testable, et le label disparaît.

Les labels servent aussi à sortir d'une boucle depuis un `select` (niveau 6), où `break`
seul ne quitterait que le `select`.

### La variable de boucle : ce qui a changé en Go 1.22

Avant Go 1.22, `i` était **une seule variable réutilisée** à chaque tour de boucle. Depuis
Go 1.22, elle est **recréée à chaque itération**.

Tant qu'on ne fait que lire `i` dans le corps de la boucle, la différence est invisible.
Elle devient spectaculaire dès qu'une **fonction capture** cette variable pour l'utiliser
plus tard — le cas des closures et des goroutines. C'est le bug le plus célèbre de
l'histoire de Go, et il est traité en détail à la **leçon 11**, une fois les closures
introduites.

Deux choses à retenir dès maintenant :

- Le comportement est piloté par la ligne `go` de `go.mod` : un module déclarant `go 1.21`
  conserve l'ancienne sémantique, même compilé avec Go 1.27. C'est l'illustration la plus
  concrète de ce que signifie réellement cette ligne (leçon 1).
- Une immense quantité de code, de tutoriels et de réponses en ligne est antérieure à 1.22
  et applique encore la parade historique (`i := i` en début de boucle), aujourd'hui inutile.
  Savoir la reconnaître évite de la recopier par mimétisme.

### Boucles et performance : deux réflexes

**1. Ne pas recalculer l'invariant.**
```go
for i := 0; i < len(s); i++ { }   // len(s) est appelé à chaque tour…
```
En pratique le compilateur Go élimine cet appel quand `s` ne change pas — mais si le corps
de la boucle est une fonction opaque, il ne peut plus. Dans le doute, sortir le calcul.

**2. Une boucle vide n'est pas optimisée.**
Contrairement au C, le compilateur Go ne supprime pas une boucle sans effet. Utile à savoir
pour les micro-mesures du niveau 11 — et pour comprendre pourquoi un benchmark naïf peut
mesurer n'importe quoi.

## Exemple

Tout ce qui suit n'utilise que des entiers : aucune collection n'a encore été vue.

```go
package main

import "fmt"

// collatzSteps compte les étapes de la suite de Syracuse partant de n,
// et retourne aussi la valeur maximale atteinte.
// Forme « while » : la condition seule.
func collatzSteps(n int) (steps, peak int) {
	peak = n
	for n != 1 {
		if n%2 == 0 {
			n /= 2
		} else {
			n = 3*n + 1
		}
		peak = max(peak, n) // max est intégré depuis Go 1.21
		steps++
	}
	return steps, peak
}

// isPrime teste la primalité par divisions d'essai.
// Forme classique à trois clauses, avec une sortie anticipée.
func isPrime(n int) bool {
	if n < 2 {
		return false
	}
	for d := 2; d*d <= n; d++ { // s'arrêter à la racine : indispensable
		if n%d == 0 {
			return false
		}
	}
	return true
}

func main() {
	// Forme 4 : range sur un entier (Go 1.22+)
	for i := range 3 {
		fmt.Printf("essai %d\n", i+1)
	}

	// Forme 2 : condition seule
	steps, peak := collatzSteps(27)
	fmt.Printf("Syracuse(27) : %d étapes, maximum %d\n", steps, peak)

	// Forme 1 + label : première paire (i,j) dont le produit dépasse 30
search:
	for i := 1; i <= 9; i++ {
		for j := i; j <= 9; j++ {
			if i*j > 30 {
				fmt.Printf("première paire : %d × %d = %d\n", i, j, i*j)
				break search // sans le label, seule la boucle interne s'arrêterait
			}
		}
	}

	// continue avec label : n'afficher que les lignes entièrement paires
	fmt.Println("lignes paires de la table de multiplication :")
row:
	for i := 1; i <= 6; i++ {
		for j := 1; j <= 3; j++ {
			if (i*j)%2 != 0 {
				continue row // passe à la ligne suivante dès qu'un produit est impair
			}
		}
		fmt.Printf("  ligne %d\n", i)
	}

	// Forme 3 : boucle infinie avec sortie explicite
	n := 100
	for {
		n++
		if isPrime(n) {
			break
		}
	}
	fmt.Printf("premier nombre premier après 100 : %d\n", n)
}
```


## Explication du code

| Élément | Ce qui compte |
|---|---|
| `for n != 1` | La forme « while ». Aucun mot-clé spécifique : on omet init et post. |
| `(steps, peak int)` | Résultats nommés : la signature dit ce que la fonction retourne. Ils valent leur zéro-valeur dès l'entrée. |
| `for d := 2; d*d <= n; d++` | S'arrêter à la racine carrée. Écrire `d <= n` donnerait le bon résultat mais serait environ mille fois plus lent sur un grand `n`. |
| `break search` | Sort des **deux** boucles. Sans le label, seule la boucle interne s'arrêterait et la recherche continuerait. |
| `continue row` | Passe à l'itération suivante de la boucle **externe**. Sans label, `continue` ne sauterait qu'un `j`. |
| Les labels non indentés | `gofmt` les aligne sur la colonne du `for` : c'est volontaire, ils doivent sauter aux yeux. |
| `for { … break }` | Boucle infinie dont la condition de sortie est **visible** dans le corps. |

## Erreurs fréquentes

1. **Chercher `while`** : il n'existe pas, `for cond {}` en tient lieu.
2. **`break` qui ne sort que de la boucle interne** alors qu'on voulait sortir des deux.
3. **Boucle infinie sans sortie identifiable** : `for {}` sans `break`, `return` ni `select`.
4. **Modifier le compteur dans le corps** d'une boucle à trois clauses : illisible, et source de boucles infinies.
5. **`i--` avec un compteur non signé** : `for i := uint(0); i < n; i--` ne se termine jamais, car `0-1` donne un nombre énorme.
6. **Croire encore au piège de la variable de boucle** en Go ≥ 1.22 — ou l'ignorer sur du code ancien.
7. **Capturer une variable déclarée hors de la boucle** dans une closure : le correctif de 1.22 ne s'applique pas.
8. **Oublier d'incrémenter** dans une boucle à condition seule.

## Bonnes pratiques Go

- `for i := range n` (1.22+) quand seul le compteur compte.
- `for cond {}` plutôt qu'une forme à trois clauses partiellement vide.
- Toute boucle infinie doit avoir une sortie **visible** dès la lecture du corps.
- Préférer `return` depuis une fonction dédiée à un label quand c'est possible.
- Nommer les labels avec un sens (`search:`, `outer:`) et les garder rares.
- Extraire le corps d'une boucle de plus d'une vingtaine de lignes dans une fonction.
- Ne jamais dépendre du nombre d'itérations restantes après un `break`.

## Ce que je dois retenir

- **Un seul mot-clé de boucle** : `for`, sous quatre formes (trois clauses, condition seule,
  infinie, `range`).
- `for i := range n` sur un entier depuis **Go 1.22**.
- `break`/`continue` n'agissent que sur la boucle **la plus interne** ; les labels étendent
  leur portée.
- Depuis **Go 1.22**, la variable de boucle est **recréée à chaque itération** — mais pas
  les variables déclarées hors de la boucle.
- `range` sur chaînes, slices, maps et channels arrive dans les leçons suivantes, avec des
  pièges différents à chaque fois.

➡️ [Exercices](exercices.md)
