# Leçon 3 — Exercices

Code dans `mes-solutions/niveau-04/lecon-03/`.

## Exercices faciles

**E1 — Constante contre variable.**
Afficher `0.1+0.2 == 0.3` écrit avec des constantes, puis avec des variables. Afficher les deux
sommes avec `%.20f`. Expliquer en deux phrases d'où vient la différence. *(Réponse attendue :
le mot « arrondi » doit apparaître deux fois.)*

**E2 — Le tableau des arrondis.**
Remplir ce tableau en **prédisant**, puis vérifier :

| x | `Round` | `RoundToEven` | `Floor` | `Ceil` | `Trunc` | `int(x)` | `%.0f` |
|---|---|---|---|---|---|---|---|
| 0.5 | | | | | | | |
| 1.5 | | | | | | | |
| -0.5 | | | | | | | |
| -1.7 | | | | | | | |

Quelle règle d'arrondi `%.0f` applique-t-il ?

**E3 — 2⁵³.**
Trouver le plus petit entier `n` positif tel que `float64(n) + 1 == float64(n)`. Le trouver par
une boucle, puis expliquer le résultat à l'aide du nombre de bits de mantisse.

**E4 — Jeton.**
Écrire deux fonctions qui produisent un code de réinitialisation de mot de passe à 8 chiffres :
l'une avec `math/rand/v2`, l'autre avec `crypto/rand` (`rand.Int` avec `big.NewInt(1e8)`, ou
`rand.Read` sur des octets). Laquelle est acceptable, et pourquoi la première ne l'est-elle
**pas**, même si ses sorties « ont l'air » aléatoires ?

**E5 — `sort.Search`.**
Un tableau `stock[jour]` décroît au fil des jours. Avec `sort.Search`, trouver le premier jour
où le stock passe **sous** 10. Que rend `sort.Search` si ça n'arrive jamais ? Comment
l'appelant le détecte-t-il ?

---

## Exercice intermédiaire — Facture

Écrire un calcul de facture correct **au centime** :

```go
type Line struct {
	Label    string
	Quantity int
	UnitHT   Cents     // prix unitaire hors taxes
	VATRate  int       // taux de TVA en pour-dix-mille : 2000 = 20 %, 550 = 5,5 %
}

type Invoice struct {
	Lines              []Line
	TotalHT, TotalVAT  Cents
	TotalTTC           Cents
}

func Compute(lines []Line) (Invoice, error)
```

**Contraintes :**
1. Aucun `float64`, nulle part.
2. La TVA se calcule **par ligne** puis s'additionne, arrondie au centime **au plus proche,
   demis vers le haut**. Écrire la fonction d'arrondi entière soi-même, et la tester sur les
   cas `x,5` exactement.
3. Quantité négative ou taux hors `[0, 10000]` : erreur citant la ligne fautive.
4. Une ligne de 3 articles à 0,99 € HT à 5,5 % : calculer à la main, puis vérifier.
5. Montrer, sur un exemple de ton choix, que la même facture calculée en `float64` puis
   arrondie à l'affichage donne un total différent d'**un** centime. *(Il faut chercher un
   peu ; c'est l'intérêt de l'exercice.)*
6. Afficher la facture alignée, montants à droite.

---

## Défi — Simulation reproductible

Simuler une file d'attente de supermarché : des clients arrivent toutes les `rand.N(3 *
time.Minute)`, chaque passage en caisse dure `1min + rand.N(4 * time.Minute)`, une seule caisse.

1. Simuler 8 heures. Afficher l'attente moyenne, l'attente maximale, et le nombre de clients.
2. Avec une graine fixe, la sortie doit être identique à chaque exécution. Le vérifier.
3. Lancer 100 simulations avec 100 graines différentes et donner la moyenne et l'écart-type
   de l'attente moyenne. *(`math.Sqrt` ; calculer la variance sans perte de précision : chercher
   « algorithme de Welford ».)*
4. Ajouter une deuxième caisse : de combien l'attente moyenne baisse-t-elle ? Est-ce
   proportionnel ? *(Le niveau 11, leçon 7, expliquera pourquoi la réponse est surprenante.)*

---

## Quiz

1. Pourquoi `0.1` n'est-il pas représentable exactement en `float64` ?
2. Pourquoi `0.1+0.2 == 0.3` est-il vrai entre constantes ?
3. Comment comparer deux flottants calculés ?
4. Que vaut `1/x` pour `x := 0.0` ? Et pour `x := 0` (entier) ?
5. Comment tester qu'une valeur est NaN ?
6. Quelle différence entre `math.Round` et `math.RoundToEven` ?
7. Comment représenter des montants d'argent en Go ?
8. Qu'apporte `math/rand/v2` par rapport à `math/rand` ?
9. Comment obtenir une séquence aléatoire reproductible ?
10. Quand faut-il impérativement `crypto/rand` ?
