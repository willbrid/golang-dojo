# Leçon 4 — Exercices

Code dans `mes-solutions/niveau-01/lecon-04/`.

> Ni boucles ni collections dans ces exercices : elles arrivent aux leçons 5 à 7.
> Tout se joue sur `if`, `switch` et la structure du code.

## Exercices faciles

**E1 — Aplatir.**
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
Compter les niveaux d'imbrication avant et après.

**E2 — Switch sans expression.**
Écrire `BMICategory(weightKg, heightM float64) string` retournant la catégorie d'IMC
(< 18,5 maigreur, < 25 normal, < 30 surpoids, sinon obésité). Utiliser un `switch` sans
expression. Traiter `heightM <= 0`. **Puis intervertir volontairement deux cas** et
constater ce qui casse : c'est le point de l'exercice.

**E3 — Plusieurs valeurs par cas.**
Écrire `DaysInMonth(month, year int) (int, error)` avec un `switch` groupant les mois de 31
et de 30 jours. Gérer février et les **années bissextiles** (divisible par 4, sauf par 100,
sauf par 400 — les trois règles, pas seulement la première). Rejeter les mois hors [1,12].
Vérifier sur 1900, 2000, 2024 et 2100.

**E4 — `fallthrough`.**
Écrire un `switch` utilisant `fallthrough` et prédire sa sortie avant de l'exécuter. Puis
remplacer le `fallthrough` par un cas à plusieurs valeurs. Laquelle des deux versions est la
plus lisible ? Dans quel cas `fallthrough` reste-t-il justifié ?

**E5 — Portée.**
Ce code compile-t-il ? Prédire, vérifier, expliquer précisément :
```go
if v := compute(); v > 10 {
	fmt.Println(v)
} else if v > 5 {
	fmt.Println("moyen", v)
}
fmt.Println(v)
```

---

## Exercice intermédiaire — `taxes`

Un calculateur d'impôt sur le revenu simplifié, entièrement en branchements.

```
$ go run . 45000 2
revenu imposable : 45000 €
parts            : 2
quotient familial: 22500 €
tranche marginale: 30 %
impôt            : 3542 €
taux moyen       : 7,87 %
```

**Barème par part** (fictif, volontairement) : 0 % jusqu'à 11 000 €, 11 % de 11 001 à
28 000, 30 % de 28 001 à 78 000, 41 % au-delà.

**Contraintes :**
1. Le calcul se fait **par tranches successives**, pas en appliquant un seul taux au total. *(C'est l'erreur que fait tout le monde. Que vaut l'impôt sur 28 001 € si l'on applique 30 % au tout ?)*
2. Aucune boucle : le barème a quatre tranches, elles s'écrivent explicitement. *(Question à noter pour plus tard : à partir de combien de tranches ce choix devient-il intenable ?)*
3. Le calcul vit dans une fonction **pure** `Tax(income float64, parts float64) (amount float64, marginalRate float64)`.
4. Les résultats sont **nommés** parce que `(float64, float64)` seul serait ambigu.
5. Un revenu négatif, un nombre de parts nul ou négatif : erreur explicite, code de sortie 1, aucune panique.
6. Le nombre de parts peut être décimal (1,5 part). Attention au type.
7. Aucun `else` dans tout le programme. Si un `else` semble indispensable, c'est que la structure est à revoir.
8. Les montants sont arrondis à l'euro pour l'affichage seulement — jamais dans le calcul.

*Piège volontaire non signalé dans les spécifications : que se passe-t-il exactement au
seuil, à 11 000 € pile ? Et à 11 000,50 € ? Décider, documenter, et rester cohérent d'une
tranche à l'autre.*

---

## Défi

**a) Une machine à états.**
Modéliser un feu tricolore : `Rouge → Vert → Orange → Rouge`, avec une durée par état.
```go
type State int
func Next(s State) State
func Seconds(s State) int
func Name(s State) string
```
**Aucun `if` autorisé** dans `Next` : uniquement un `switch`. *(Des fonctions, pas des
méthodes : celles-ci arrivent au niveau 2. On y reprendra ce même exercice pour voir ce que
les méthodes y changent — ou n'y changent pas.)*

**b) Le piège de l'exhaustivité.**
Ajouter un quatrième état `Clignotant` sans toucher à `Next` ni à `Name`. Le programme
compile-t-il ? Que se passe-t-il à l'exécution ?
Écrire trois phrases sur ce que cela révèle des énumérations en Go et sur ce qui les
distingue d'un `enum` de Rust ou de Java. Quel outil pourrait détecter le problème ?
*(Chercher « exhaustive linter Go ».)*

**c) Conception.**
Comparer par écrit trois façons d'implémenter `Next` : un `switch`, une table associative,
un tableau indexé par l'état. Critères : lisibilité, coût à l'exécution, comportement face à
une valeur invalide, facilité d'ajout d'un état. À partir de combien d'états l'arbitrage
change-t-il ?
*(Les maps arrivent à la leçon 7 et les tableaux sont connus depuis la leçon 2 : décrire
l'idée pour la table associative, implémenter les deux autres.)*

---

## Quiz

1. Faut-il des parenthèses autour d'une condition `if` en Go ?
2. `if x { … }` avec `x` de type `int` : compile-t-il ? Pourquoi ce choix ?
3. Quelle est la portée de `v` dans `if v, ok := f(); ok { … } else { … }` ?
4. Faut-il écrire `break` à la fin de chaque `case` ? Pourquoi ?
5. Que fait `fallthrough` exactement ?
6. Qu'est-ce qu'un `switch` sans expression, et quand le préférer à des `if/else if` ?
7. Dans quel ordre les cas d'un `switch` sans expression sont-ils évalués ?
8. Pourquoi un `else` après un `return` est-il déconseillé ?
9. Un `case` peut-il lister plusieurs valeurs ? Et une plage de valeurs ?
10. `goto` existe-t-il en Go ? Faut-il l'utiliser ?
