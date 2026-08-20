# Leçon 4 — Corrigé

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

L'ordre des cas compte : `n%15` doit venir en premier, sinon 15 sort en `Fizz`. Un `switch`
s'arrête au premier cas vrai — il n'y a pas de « chute » vers les suivants.

Variante plus élégante, sans test de 15 :

```go
s := ""
if n%3 == 0 { s += "Fizz" }
if n%5 == 0 { s += "Buzz" }
if s == "" { s = strconv.Itoa(n) }
```

## E2 — Ordre des maps

Les clés sortent dans un ordre différent à chaque exécution. Le runtime Go choisit un point
de départ aléatoire dans les compartiments de la table de hachage.

C'est **délibéré** : sans cette randomisation, l'ordre serait stable par accident, du code
finirait par en dépendre, et une simple mise à jour de Go (ou un ajout de clé provoquant un
redimensionnement) casserait le programme. Go préfère faire échouer tout de suite ce qui
n'est pas garanti. C'est la même philosophie que le refus des imports inutilisés.

## E3 — La copie de `range`

```go
type Counter struct{ N int }
cs := []Counter{{1}, {2}, {3}}

for _, c := range cs { c.N++ }        // sans effet : c est une copie
fmt.Println(cs)                        // [{1} {2} {3}]

for i := range cs { cs[i].N++ }        // correct
fmt.Println(cs)                        // [{2} {3} {4}]
```

`range` copie chaque élément dans la variable de boucle. Modifier la copie ne touche pas le
slice. Pour muter : passer par l'indice.

## E4 — Aplatir

```go
func check(name string, age int, active bool) string {
	if name == "" {
		return "nom manquant"
	}
	if age < 18 {
		return "mineur"
	}
	if !active {
		return "compte inactif"
	}
	return "ok"
}
```

Quatre niveaux d'imbrication deviennent zéro. Les conditions d'échec se lisent en séquence,
et le cas normal est la dernière ligne. C'est la forme attendue en revue de code Go.

## E5 — Labels

```go
target := 12
found := false
search:
for i, a := range nums {
	for j, b := range nums[i+1:] {
		if a+b == target {
			fmt.Printf("indices %d et %d\n", i, i+1+j)
			found = true
			break search
		}
	}
}
if !found {
	fmt.Println("aucune paire")
}
```

Attention à `i+1+j` : `j` est l'indice **dans le sous-slice**, pas dans `nums`. Erreur
classique.

Alternative idiomatique : extraire dans une fonction et utiliser `return` — souvent plus
propre qu'un label.

## Exercice intermédiaire — `histogram`

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"slices"
	"strconv"
	"strings"
)

const maxBarWidth = 50

// readValues lit des entiers depuis r. Retourne les valeurs valides et le nombre de rejets.
func readValues(r *os.File) ([]int, int) {
	sc := bufio.NewScanner(r)
	sc.Split(bufio.ScanWords) // découpe sur les espaces ET les retours à la ligne

	var values []int
	rejected := 0
	for sc.Scan() {
		n, err := strconv.Atoi(sc.Text())
		if err != nil || n < 0 {
			fmt.Fprintf(os.Stderr, "valeur ignorée : %q\n", sc.Text())
			rejected++
			continue
		}
		values = append(values, n)
	}
	return values, rejected
}

// stats calcule le maximum et la moyenne. Fonction pure : aucun affichage.
func stats(values []int) (max int, mean float64) {
	if len(values) == 0 {
		return 0, 0
	}
	sum := 0
	for _, v := range values {
		sum += v
		max = maxOf(max, v)
	}
	return max, float64(sum) / float64(len(values))
}

func maxOf(a, b int) int { if a > b { return a }; return b } // ou simplement max(a, b), Go 1.21+

func render(values []int, w *os.File) {
	max, mean := stats(values)
	labelWidth := len(strconv.Itoa(max))

	for _, v := range values {
		width := 0
		if max > 0 {
			width = v * maxBarWidth / max // mise à l'échelle en ENTIERS : pas d'arrondi flottant
		}
		fmt.Fprintf(w, "%*d │ %s\n", labelWidth, v, strings.Repeat("█", width))
	}
	fmt.Fprintf(w, "%s └%s\n", strings.Repeat(" ", labelWidth), strings.Repeat("─", maxBarWidth))
	fmt.Fprintf(w, "%s   max = %d   n = %d   moyenne = %.2f\n",
		strings.Repeat(" ", labelWidth), max, len(values), mean)
}

func main() {
	values, rejected := readValues(os.Stdin)
	if len(values) == 0 {
		fmt.Fprintln(os.Stderr, "aucune valeur exploitable")
		os.Exit(1)
	}
	slices.Sort(values)
	render(values, os.Stdout)
	if rejected > 0 {
		os.Exit(1)
	}
}
```

**Points de correction :**
- `bufio.ScanWords` gère espaces et retours à la ligne d'un coup — pas besoin de découper à
  la main.
- `v * maxBarWidth / max` en **entiers** : multiplier avant de diviser évite à la fois le
  flottant et la perte de précision. Attention au débordement si les valeurs sont énormes.
- `%*d` prend la largeur en argument : c'est ce qui aligne la colonne sans calcul manuel.
- `slices.Sort` est idiomatique depuis Go 1.21. Écrire un tri à bulles à la main
  « fonctionne » mais n'est pas idiomatique — et la question portait précisément là-dessus.
- Les trois responsabilités (lire, calculer, afficher) sont dans trois fonctions distinctes.

## Défi — Crible et Goldbach

```go
func Primes(n int) []int {
	if n < 2 {
		return nil
	}
	composite := make([]bool, n+1) // []bool : 1 octet par entrée, contigu en mémoire.
	// Une map coûterait ~50 octets par entrée et détruirait la localité de cache.
	var primes []int

	for p := 2; p <= n; p++ {
		if composite[p] {
			continue
		}
		primes = append(primes, p)
		for m := p * p; m <= n; m += p { // démarrer à p² : les multiples plus petits
			composite[m] = true          // ont déjà été marqués par un facteur inférieur
		}
	}
	return primes
}
```

Complexité : O(n log log n). Sur un million, quelques millisecondes.

```go
func Goldbach(n int) (int, int, int, bool) {
	primes := Primes(n)
	isPrime := make([]bool, n+1)
	for _, p := range primes {
		isPrime[p] = true
	}
	for even := 4; even <= n; even += 2 {
		found := false
		for _, p := range primes {
			if p > even/2 {
				break
			}
			if isPrime[even-p] { // test O(1) grâce au tableau
				found = true
				break
			}
		}
		if !found {
			return even, 0, 0, false // contre-exemple
		}
	}
	return 0, 0, 0, true
}
```

L'astuce est le **tableau de test d'appartenance** : au lieu de chercher `even - p` dans la
liste triée (O(log n) par recherche dichotomique), on teste en O(1). C'est un échange
mémoire contre temps — le raisonnement central du niveau 8.

## Réponses du quiz

1. **Un seul** : `for`. Pas de `while`, pas de `do…while`, pas de `foreach`.
2. Non. Go ne « tombe » pas d'un cas au suivant : le `break` est implicite. `fallthrough`
   force le passage, mais reste rare.
3. Il force l'exécution du cas **suivant**, sans réévaluer sa condition.
4. Non : `v` est une **copie** de l'élément. Il faut passer par `items[i]`.
5. Non, il change à chaque exécution. Ce n'est **pas** un bug : le runtime randomise
   délibérément pour empêcher toute dépendance à un ordre non garanti.
6. `012`. Depuis **Go 1.22** (`range` sur un entier).
7. `v` n'existe que dans le `if` et son `else`. C'est tout l'intérêt de la forme.
8. La variable de boucle est désormais **recréée à chaque itération**. Cela corrige le
   classique des closures/goroutines qui capturaient toutes la même variable et affichaient
   la dernière valeur.
9. À sortir (`break`) ou passer au tour suivant (`continue`) d'une boucle **externe** depuis
   une boucle imbriquée — ou depuis un `select`.
