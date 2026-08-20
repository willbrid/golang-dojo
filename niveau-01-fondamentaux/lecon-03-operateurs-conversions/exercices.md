# Leçon 3 — Exercices

Code dans `mes-solutions/niveau-01/lecon-03/`.

## Exercices faciles

**E1 — Prédire.**
Prédire le résultat de chaque expression, **puis** vérifier. Expliquer chaque écart :
```go
7 / 2          -7 / 2         7 % 3          -7 % 3         7 % -3
1 << 10        -8 >> 1        ^0             5 &^ 3         5 ^ 3
float64(7/2)   float64(7)/2   int(-3.7)      int(3.7)
```

**E2 — Pourcentage.**
Écrire une fonction qui calcule un taux de réussite en pourcentage à partir de deux entiers,
avec deux décimales. Écrire d'abord la version **fausse** (division entière), constater le
`0.00 %`, puis corriger. Gérer le cas `total == 0`.

**E3 — Drapeaux.**
Créer un type `Option uint8` avec quatre drapeaux (`Recursive`, `Hidden`, `Follow`,
`Quiet`). Écrire des fonctions `Set`, `Clear`, `Has` et `Toggle`, puis afficher l'état en
binaire sur 4 bits à chaque étape. Utiliser `&^` pour `Clear`.

**E4 — `strconv`.**
Écrire un programme qui lit une liste de chaînes et les classe en trois catégories :
entier, flottant, ni l'un ni l'autre. Tester avec `"42"`, `"3.14"`, `"0x1f"`, `"1e5"`,
`" 7"`, `"7 "`, `""`, `"abc"`. Certains résultats vont surprendre — expliquer lesquels et
pourquoi.

**E5 — Les verbes.**
Pour un entier : `%d`, `%b`, `%o`, `%x`, `%X`, `%c`, `%U`, `%08d`, `%-8d|`, `%+d`.
Pour un flottant : `%f`, `%.3f`, `%e`, `%g`, `%10.2f`.
Pour une chaîne contenant une tabulation et un accent : `%s`, `%q`, `%x`, `%+q`.
Constituer un tableau de référence personnel — il servira toute la formation.

---

## Exercice intermédiaire — `numfmt`

Un formateur de nombres en ligne de commande.

```
$ go run . 1234567
décimal     : 1 234 567
hexadécimal : 0x12d687
binaire     : 0b100101101011010000111
octal       : 0o4553207
bits        : 21 bits significatifs
signe       : positif
```

**Contraintes :**
1. L'entrée accepte les préfixes `0x`, `0b`, `0o` et le décimal — une seule ligne de code doit suffire pour ça.
2. Le séparateur de milliers est une **espace insécable fine** ou une espace ordinaire — au choix, mais implémenté **à la main**, sans dépendance externe. *(Réfléchir : par quel bout parcourt-on le nombre ?)*
3. Fonctionne pour les valeurs négatives et pour `math.MinInt64` — ce dernier cas est le piège.
4. Une entrée invalide produit un message clair sur `os.Stderr` et le code de sortie 1.
5. `-base N` (2 à 36) ajoute une ligne avec la représentation dans cette base.
6. Aucune fonction ne fait à la fois le calcul et l'affichage.

*Indices : `strconv.ParseInt` avec la base 0, `strconv.FormatInt`, `math.MinInt64`.*

---

## Défi

**a) Compteur de bits.** Écrire `PopCount(x uint64) int` qui compte les bits à 1,
**sans boucle sur les 64 bits**. *Indice : l'algorithme de Kernighan repose sur `x & (x-1)`.
Que fait cette expression, exactement ?*
Comparer ensuite avec `math/bits.OnesCount64` et mesurer les deux avec `time.Since` sur dix
millions d'appels.

**b) Conversion sûre générique — sans génériques.** Écrire une famille de fonctions
`ToInt8`, `ToUint8`, `ToInt32` qui convertissent depuis un `int` en retournant une erreur si
la valeur ne tient pas dans le type cible. Puis répondre par écrit : combien de fonctions
faudrait-il pour couvrir toutes les paires de types numériques ? Qu'est-ce que ça dit du
besoin de génériques (niveau 3) ?

**c) Précédence : Go contre C.**
```go
func hasBit(v, bit int) bool { return v & bit == bit }
```
Ce code est-il correct en Go ? Le serait-il en C ? Écrire l'équivalent C mentalement et
comparer.
Puis lancer `gofmt` dessus et observer **comment l'espacement change**. Qu'est-ce que le
formateur vient de communiquer ? Trouver deux autres expressions où `gofmt` révèle ainsi la
précédence.

---

## Quiz

1. Que vaut `7 / 2` ? Et `float64(7 / 2)` ?
2. Que vaut `-7 % 3` en Go ? Et en Python ? Pourquoi cette différence compte-t-elle ?
3. `x := i++` compile-t-il ?
4. Que fait `a &^ b` ? Existe-t-il en C ?
5. Quelle est la précédence de `&` par rapport à `==` en Go ? Et en C ?
6. Une conversion `byte(300)` sur une **variable** : erreur, panique, ou valeur surprenante ?
7. Quelle différence entre `string(65)` et `strconv.Itoa(65)` ?
8. Quelle différence entre `%s` et `%q` sur une chaîne ? Quand `%q` est-il indispensable ?
9. Quelle fonction de `fmt` écrit vers une destination arbitraire ?
10. `1/0` entre entiers : que se passe-t-il ? Et entre flottants ?
