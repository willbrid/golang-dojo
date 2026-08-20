# Leçon 2 — Exercices

Code dans `mes-solutions/niveau-01/lecon-02/`.

## Exercices faciles

**E1 — Zéro-valeurs.**
Déclarer sans les initialiser une variable de chaque type : `int`, `float64`, `bool`,
`string`, `[]int` (slice), `map[string]int`, `*int` (pointeur). Afficher chacune avec
`fmt.Printf("%v %q %T\n", …)` et **prédire la sortie avant de l'exécuter**. Noter les écarts.

**E2 — Conversions.**
Que produisent ces expressions ? Prédire, puis vérifier :
```go
int8(127) + 1
int8(300)
int(3.99)
float64(7) / 2
7 / 2
7 % -2
uint8(255) + 1
```
Pour chacune : le compilateur refuse-t-il, ou accepte-t-il en produisant un résultat surprenant ?
*Attention : certaines de ces lignes ne compilent pas telles quelles. Comprendre pourquoi fait partie de l'exercice.*

**E3 — `iota`.**
Créer un type `Planet` et une énumération des huit planètes, avec `Mercury = 1` (et non 0).
Ajouter une méthode `String()`. Afficher les huit valeurs.

**E4 — Constantes non typées.**
Expliquer pourquoi ceci compile :
```go
const big = 1 << 62
var f float64 = big
```
et pourquoi ceci ne compile pas :
```go
var n = 1 << 62
var f float64 = n
```
Rédiger la réponse en trois phrases maximum.

**E5 — Tailles.**
Écrire un programme affichant, pour `int`, `int32`, `int64`, `float64`, `bool`, `string`,
la taille en octets (`unsafe.Sizeof`) et les bornes des entiers (paquet `math`).
Pourquoi `unsafe.Sizeof("bonjour")` ne donne-t-il pas 7 ?

---

## Exercice intermédiaire — `unitconv`

Convertisseur d'unités de données, avec les bons types.

```
$ go run . 1536
1536 B = 1.50 KiB = 0.00 MiB
$ go run . 5368709120
5368709120 B = 5242880.00 KiB = 5120.00 MiB = 5.00 GiB
```

**Contraintes :**
1. Les seuils sont des constantes définies avec `iota` et des décalages de bits.
2. Le programme affiche l'unité la plus grande **pertinente** (pas de `0.00 GiB`).
3. Une valeur négative ou non numérique produit une erreur claire ; sortie avec le code 1.
4. Gérer une valeur allant jusqu'à `math.MaxInt64` sans dépassement ni perte de précision — **prouver** que c'est le cas.
5. Un type `ByteSize` avec une méthode `String()` fait tout le travail de formatage ; `main` ne formate rien lui-même.

*Question à trancher avant de coder : `1536 / 1024` en entier donne `1`. Comment obtenir
`1.50` sans introduire d'erreur d'arrondi sur les grandes valeurs ?*

---

## Défi

Écrire une fonction `SafeAdd(a, b int) (int, bool)` qui additionne deux entiers et retourne
`false` en second résultat **si et seulement si** l'opération déborde — sans jamais provoquer
elle-même de dépassement non détecté.

Puis la même chose pour `SafeMul(a, b int) (int, bool)`.

**Contraintes :**
- pas de conversion vers `float64` (perte de précision au-delà de 2⁵³) ;
- pas de `math/big` ;
- fonctionner pour les valeurs négatives, `math.MinInt` et `math.MaxInt` compris ;
- écrire au moins six cas de vérification manuels dans `main`.

*Ce défi est un classique d'entretien technique. Le cas `math.MinInt` est celui qui piège tout le monde.*

---

## Quiz

1. Quelle est la zéro-valeur d'une `string` ? D'une `map` ? D'un `*int` ?
2. Pourquoi `var f float64 = someInt` ne compile-t-il pas ?
3. Que vaut `int8(200)` ? Le programme panique-t-il ?
4. Quelle différence entre `const n = 100` et `const n int = 100` ?
5. `byte` et `uint8` sont-ils deux types différents ?
6. Combien de bits fait un `int` ? La réponse est-elle garantie par la spécification ?
7. Pourquoi ne faut-il jamais représenter un montant en euros par un `float64` ?
8. Que vaut `iota` sur la troisième ligne d'un bloc `const` ? Et dans le bloc `const` suivant ?
