# Leçon 4 — Exercices

Code dans `mes-solutions/niveau-01/lecon-04/`.

## Exercices faciles

**E1 — FizzBuzz, version Go.**
Afficher 1 à 100, avec `Fizz` pour les multiples de 3, `Buzz` pour 5, `FizzBuzz` pour 15.
**Contrainte : utiliser `for i := range` (forme 1.22) et un `switch` sans expression** —
pas de cascade de `if/else`.

**E2 — Ordre des maps.**
Créer une `map[string]int` de cinq entrées, la parcourir avec `range` et afficher les clés.
Exécuter le programme **dix fois de suite**. Que constate-t-on ? Pourquoi Go fait-il ça
délibérément ?

**E3 — La copie de `range`.**
Écrire un slice de structs `type Counter struct{ N int }`, tenter d'incrémenter `N` dans
un `for _, c := range` — constater l'absence d'effet — puis corriger.
Expliquer en une phrase.

**E4 — Aplatir des `if`.**
Réécrire cette fonction en *early return*, sans aucun `else` :
```go
func check(name string, age int, active bool) string {
	if name != "" {
		if age >= 18 {
			if active {
				return "ok"
			} else {
				return "compte inactif"
			}
		} else {
			return "mineur"
		}
	} else {
		return "nom manquant"
	}
}
```

**E5 — Labels.**
Chercher la première paire d'entiers `(i, j)` d'un slice dont la somme vaut une cible,
puis sortir des deux boucles avec un label. Afficher les indices ou un message d'échec.

---

## Exercice intermédiaire — `histogram`

Écrire un programme qui lit des nombres et affiche un histogramme textuel.

```
$ echo "3 7 1 9 4 4 12" | go run .
 1 │ ██
 3 │ ██████
 4 │ ████████
 4 │ ████████
 7 │ ██████████████
 9 │ ██████████████████
12 │ ████████████████████████
   └────────────────────────
     max = 12   n = 7   moyenne = 5.71
```

**Contraintes :**
1. Lecture sur l'entrée standard, nombres séparés par des espaces ou des retours à la ligne.
2. Les valeurs sont triées croissantes à l'affichage. *(Trier soi-même avec une boucle, ou découvrir `slices.Sort` — les deux sont acceptés, mais savoir dire lequel est idiomatique.)*
3. La largeur maximale des barres est de **50 caractères**, quelle que soit la valeur maximale : les barres sont mises à l'échelle.
4. La colonne des nombres est alignée à droite, largeur automatique selon la plus grande valeur.
5. Une valeur négative ou non numérique : message sur `os.Stderr`, la valeur est ignorée, le traitement continue.
6. Entrée vide : message clair, code de sortie 1, aucune panique.
7. Aucune fonction ne fait à la fois du calcul et de l'affichage.

*Indices : `bufio.Scanner` avec `scanner.Split(bufio.ScanWords)`, `strconv.Atoi`,
`strings.Repeat`, le verbe `%*d` pour une largeur dynamique.*

---

## Défi

**Le crible d'Ératosthène, puis une conjecture.**

**a)** Écrire `Primes(n int) []int` qui retourne tous les nombres premiers ≤ n avec un
crible (aucune division, aucun `%` dans la boucle principale — que des marquages).

**b)** Vérifier la conjecture de Goldbach pour tous les entiers pairs de 4 à `n` : chacun
est-il la somme de deux nombres premiers ? Afficher la décomposition du premier contre-exemple
s'il en existe, sinon confirmer.

**Contraintes :**
- `n = 1 000 000` doit s'exécuter en moins d'une seconde ;
- mémoire du crible : un `[]bool`, pas une map — expliquer pourquoi en commentaire ;
- pour (b), la recherche d'une décomposition doit tirer parti du fait que la liste des premiers est **triée**. Une double boucle naïve est trop lente : trouver mieux.

*Ce défi mélange boucles, slices et une vraie contrainte de performance. Mesurer avec
`time.Since` — première rencontre avec le réflexe « mesurer avant d'affirmer » du niveau 8.*

---

## Quiz

1. Combien de mots-clés de boucle Go possède-t-il ? Lesquels ?
2. Faut-il écrire `break` à la fin de chaque `case` d'un `switch` ? Pourquoi ?
3. Que fait `fallthrough` ?
4. Dans `for _, v := range items`, modifier `v` modifie-t-il `items` ? Pourquoi ?
5. L'ordre d'itération d'une map est-il stable d'une exécution à l'autre ? Est-ce un bug ?
6. Qu'affiche `for i := range 3 { fmt.Print(i) }` ? Depuis quelle version de Go ?
7. Quelle est la portée de `v` dans `if v, ok := m[k]; ok { … } else { … }` ?
8. Qu'a changé Go 1.22 concernant la variable de boucle, et quel bug classique cela corrige-t-il ?
9. À quoi sert un label devant une boucle ?
