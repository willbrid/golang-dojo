# Leçon 5 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — FizzBuzz

```go
for i := range 100 {
	n := i + 1
	switch {
	case n%15 == 0:
		fmt.Println("FizzBuzz")
	case n%3 == 0:
		fmt.Println("Fizz")
	case n%5 == 0:
		fmt.Println("Buzz")
	default:
		fmt.Println(n)
	}
}
```

L'ordre est essentiel : `n%15` doit venir en premier, sinon 15 sort en « Fizz ». Un `switch`
s'arrête au premier cas vrai.

Variante plus élégante, sans tester 15 — mais elle a besoin des chaînes (leçon 8) :
```go
s := ""
if n%3 == 0 { s += "Fizz" }
if n%5 == 0 { s += "Buzz" }
if s == "" { s = strconv.Itoa(n) }
```

## E2 — Les quatre formes

```go
// 1. trois clauses
sum := 0
for i := 1; i <= 100; i++ { sum += i }

// 2. condition seule
sum, i := 0, 1
for i <= 100 { sum += i; i++ }

// 3. infinie
sum, i = 0, 1
for { if i > 100 { break }; sum += i; i++ }

// 4. range sur entier
sum = 0
for i := range 100 { sum += i + 1 }
```

La forme 1 est la plus lisible ici : la borne et l'incrément sont visibles d'un coup d'œil.
La forme 4 est la plus idiomatique **quand on part de 0** ; le `+1` qu'elle impose ici est un
signal qu'elle n'est pas parfaitement adaptée. La forme 3 est clairement excessive.

## E3 — Chiffres

```go
func DigitCount(n int) int {
	if n == 0 {
		return 1 // cas particulier : la boucle ne tournerait pas
	}
	if n < 0 {
		n = -n // ATTENTION : faux pour math.MinInt (voir plus bas)
	}
	c := 0
	for n > 0 {
		n /= 10
		c++
	}
	return c
}

func DigitSum(n int) int {
	if n < 0 {
		n = -n
	}
	s := 0
	for n > 0 {
		s += n % 10
		n /= 10
	}
	return s
}

func Reverse(n int) int {
	neg := n < 0
	if neg {
		n = -n
	}
	r := 0
	for n > 0 {
		r = r*10 + n%10
		n /= 10
	}
	if neg {
		return -r
	}
	return r
}
```

`n == 0` doit être traité à part : la boucle `for n > 0` ne s'exécute pas et retournerait 0
chiffre.

**Le piège de `math.MinInt`** (rappel de la leçon 2) : `-math.MinInt` déborde et revaut
`math.MinInt`. Les trois fonctions boucleraient indéfiniment ou retourneraient n'importe quoi.
Une version robuste travaillerait sur `n % 10` sans changer le signe, en prenant la valeur
absolue de chaque chiffre.

Et `Reverse` peut **déborder** : `Reverse(9646324351)` ne tient pas dans un `int32`. Sur
`int64` le cas est plus rare mais existe. Une version production retournerait `(int, error)`.

## E4 — Labels

```go
search:
for n := 2; ; n++ {
	for d := 2; d <= 10; d++ {
		if n%d != 0 {
			continue search // ce n ne convient pas, passer au suivant
		}
	}
	fmt.Println("trouvé :", n) // 2520
	break search
}
```

Version sans label, avec une fonction :
```go
func divisibleByAll(n, upTo int) bool {
	for d := 2; d <= upTo; d++ {
		if n%d != 0 {
			return false
		}
	}
	return true
}

for n := 2; ; n++ {
	if divisibleByAll(n, 10) {
		fmt.Println("trouvé :", n)
		break
	}
}
```

**La seconde est préférable.** La condition porte un nom, elle est testable isolément, et la
boucle externe se lit sans effort. Règle générale : dès qu'un label sert à sortir d'une boucle
interne parce qu'on a « trouvé » quelque chose, une fonction avec `return` est presque toujours
plus claire. Le label garde son utilité quand la boucle interne produit un **effet** dont on
doit sortir, ou dans un `select` (niveau 6).

## E5 — Le piège du non signé

```go
for i := uint(3); i >= 0; i-- {   // NE SE TERMINE JAMAIS
	fmt.Println(i)
}
```

Un `uint` ne peut pas être négatif : après `0`, `i--` donne `18446744073709551615`. La
condition `i >= 0` est donc **toujours vraie** — le compilateur ne prévient même pas, car
l'expression est valide.

Deux corrections :
```go
for i := 3; i >= 0; i-- { }              // A : utiliser un int signé
for i := uint(4); i > 0; i-- { v := i-1 } // B : décaler d'un cran et tester > 0
```

La leçon générale : `uint` ne signifie pas « nombre positif », il signifie « arithmétique
modulaire non signée ». Pour exprimer « ce nombre doit être positif », un `int` plus une
validation est plus sûr.

## Exercice intermédiaire — `numbers`

```go
package main

import (
	"fmt"
	"math"
	"os"
	"strconv"
)

const maxCollatzSteps = 10_000

// collatz AFFICHE au fil de l'eau via la fonction de rappel qu'est fmt :
// contrainte de l'énoncé, les slices n'étant pas encore vus.
func collatz(n int) (steps, peak int, err error) {
	if n < 1 {
		return 0, 0, fmt.Errorf("collatz : %d doit être >= 1", n)
	}
	peak = n
	for n != 1 {
		if steps > maxCollatzSteps {
			// La conjecture n'est pas démontrée : on ne lui fait pas confiance.
			return steps, peak, fmt.Errorf("collatz : plus de %d étapes, abandon", maxCollatzSteps)
		}
		if n%2 == 0 {
			n /= 2
		} else {
			// Débordement possible sur un très grand n impair
			if n > (math.MaxInt-1)/3 {
				return steps, peak, fmt.Errorf("collatz : débordement à %d", n)
			}
			n = 3*n + 1
		}
		peak = max(peak, n)
		steps++
	}
	return steps, peak, nil
}

func isPrime(n int) bool {
	if n < 2 {
		return false
	}
	if n%2 == 0 {
		return n == 2
	}
	for d := 3; d*d <= n; d += 2 { // uniquement les diviseurs impairs
		if n%d == 0 {
			return false
		}
	}
	return true
}

// fib affiche la suite au fil de l'eau et détecte le débordement AVANT
// de produire une valeur fausse.
func fib(n int) error {
	if n < 0 {
		return fmt.Errorf("fib : n négatif")
	}
	a, b := 0, 1
	for i := range n {
		fmt.Print(a)
		if i < n-1 {
			fmt.Print(" ")
		}
		if a > math.MaxInt-b { // le test AVANT l'addition
			fmt.Println()
			return fmt.Errorf("fib : débordement à l'indice %d", i+1)
		}
		a, b = b, a+b
	}
	fmt.Println()
	return nil
}

func perfect(limit int) {
	for n := 2; n <= limit; n++ {
		sum := 1
		for d := 2; d*d <= n; d++ {
			if n%d != 0 {
				continue
			}
			sum += d
			if q := n / d; q != d {
				sum += q
			}
		}
		if sum == n {
			fmt.Print(n, " ")
		}
	}
	fmt.Println()
}
```

**Les trois points de correction :**

1. **`d*d <= n` et non `d <= n`** dans `isPrime` : sur un million, l'écart est d'un facteur
   plusieurs centaines. Si `n` a un diviseur supérieur à sa racine, il en a forcément un
   inférieur.
2. **Le test de débordement précède l'addition** : `a > math.MaxInt-b`. Écrire
   `if a+b < 0 { … }` *après* l'addition est un comportement non défini au sens du domaine —
   ça marche par accident en complément à deux, mais ça ne se lit pas et ça masque l'intention.
   `fib(92)` est la dernière valeur qui tient dans un `int64`.
3. **Le plafond de `collatz`** : la conjecture de Syracuse n'est **pas démontrée**. Un
   programme qui suppose qu'une boucle se termine sur la foi d'une conjecture ouverte n'est
   pas un programme de production. Le débordement de `3n+1` est le second piège du même ordre.

`perfect` illustre l'optimisation des diviseurs par paires : trouver `d` donne aussi `n/d`,
d'où l'arrêt à la racine. Le `q != d` évite de compter deux fois la racine d'un carré parfait.

## Défi

**a) Motifs**

Le triangle de Pascal est le seul vrai défi : il faut connaître la largeur du plus grand
nombre **avant** d'afficher la première ligne.

```go
func pascal(rows int) {
	// Première passe : calculer la largeur maximale
	width := 1
	c := 1
	for k := 0; k < rows; k++ {
		width = max(width, len(strconv.Itoa(c)))
		c = c * (rows - 1 - k) / (k + 1)
	}

	// Seconde passe : afficher
	for n := range rows {
		for range (rows - n - 1) * (width + 1) / 2 { // décalage de centrage
			fmt.Print(" ")
		}
		c = 1
		for k := 0; k <= n; k++ {
			fmt.Printf("%*d ", width, c)
			c = c * (n - k) / (k + 1)
		}
		fmt.Println()
	}
}
```

Le calcul incrémental `c = c*(n-k)/(k+1)` évite de calculer des factorielles, qui déborderaient
très vite. La division tombe toujours juste — c'est une propriété des coefficients binomiaux,
pas un hasard.

**b) PGCD**

```go
func GCD(a, b int) int {         // itératif
	for b != 0 {
		a, b = b, a%b
	}
	return abs(a)
}

func GCDRec(a, b int) int {      // récursif
	if b == 0 {
		return abs(a)
	}
	return GCDRec(b, a%b)
}

func Simplify(num, den int) (int, int, error) {
	if den == 0 {
		return 0, 0, errors.New("dénominateur nul")
	}
	g := GCD(num, den)
	num, den = num/g, den/g
	if den < 0 { // convention : le signe est porté par le numérateur
		num, den = -num, -den
	}
	return num, den, nil
}
```

Le **pire cas** d'Euclide est atteint par deux nombres de Fibonacci consécutifs : le nombre
d'étapes est alors maximal pour la taille des entrées. C'est le théorème de Lamé, et il donne
une complexité en O(log min(a,b)) — le premier résultat de complexité de l'histoire de
l'informatique, démontré en 1844.

`abs(a)` est nécessaire : `GCD(-12, 8)` donnerait `-4` sans lui, alors qu'un PGCD est
positif par définition.

**c) Localité mémoire**

```
m[i][j], j varie le plus vite : ~1,0 ms
m[j][i], i varie le plus vite : ~6 à 12 ms
```

Un facteur de 5 à 12 selon la machine, pour un code dont la **complexité algorithmique est
strictement identique** : un million d'accès dans les deux cas.

L'explication tient au cache CPU. Un tableau `[1000][1000]int` est stocké **ligne par ligne**
en mémoire contiguë. Le parcours `m[i][j]` avance de 8 octets à chaque itération : chaque
ligne de cache chargée (64 octets) sert pour 8 accès, et le préchargeur matériel anticipe la
suite. Le parcours `m[j][i]` avance de 8 000 octets à chaque itération : chaque accès touche
une ligne de cache différente, aucune n'est réutilisée, et le préchargeur ne peut rien
anticiper.

C'est le premier contact avec le raisonnement du niveau 11 : **le nombre d'opérations ne
prédit pas le temps d'exécution.** L'ordre des accès mémoire pèse souvent davantage.

## Réponses du quiz

1. **Un seul** : `for`. Ni `while`, ni `do…while`, ni `foreach`.
2. `for condition { }`. Il n'y a **pas** de `do…while` : on l'écrit avec `for { … if !cond { break } }`.
3. `012`. Depuis **Go 1.22**.
4. Il ne sort que de la boucle **la plus interne**.
5. À `break` ou `continue` une boucle **externe** depuis une boucle imbriquée (ou depuis un
   `select`). `gofmt` l'aligne sur la colonne du `for`, sans indentation supplémentaire.
6. Elle est **recréée à chaque itération** au lieu d'être réutilisée. Cela corrige le bug des
   closures et des goroutines qui capturaient toutes la même variable.
7. La ligne `go` du `go.mod` : un module en `go 1.21` conserve l'ancienne sémantique, même
   compilé avec Go 1.27.
8. Parce qu'un `uint` ne peut pas être négatif : `0 - 1` donne la plus grande valeur possible,
   et `i >= 0` reste vraie indéfiniment.
9. **Non.** Contrairement au C, le compilateur Go ne supprime pas une boucle sans effet. C'est
   à connaître pour les micro-mesures du niveau 11.
10. Déjà vus : les entiers (leçon 5). À venir : slices et tableaux (6), maps (7), chaînes (8),
    channels (niveau 6), fonctions itératrices (niveau 3).
