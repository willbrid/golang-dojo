# Leçon 6 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé. C'est la leçon où lire la solution trop tôt coûte le
> plus cher : la mécanique des slices ne s'acquiert qu'en se trompant.

## E1 — len et cap

```
append  1 : len=1 cap=3
append  2 : len=2 cap=3
append  3 : len=3 cap=3
append  4 : len=4 cap=6     ← réallocation
append  7 : len=7 cap=12    ← réallocation
```

La capacité double tant que le slice est petit (< 256 éléments), puis croît d'environ 25 %.
**Ce comportement n'est garanti nulle part.** La spécification dit seulement qu'`append`
alloue « si nécessaire ». Le facteur a changé entre Go 1.17 et 1.18, et il a encore été
ajusté depuis. Tout code qui en dépend est cassé par construction.

## E2 — Tableau vs slice

```go
func modifyArray(a [3]int) { a[0] = 99 }  // reçoit une COPIE des 3 éléments
func modifySlice(s []int)  { s[0] = 99 }  // reçoit une copie du DESCRIPTEUR,
                                          // qui pointe vers le même tableau
```

Un tableau est une valeur : le passer copie tout son contenu. Un slice est un descripteur
{ptr, len, cap} : le copier duplique trois mots machine, mais le tableau pointé reste
partagé.

## E3 — Aliasing

```
s = [0 1 2 3 4 5]
a = s[1:3] → [1 2]        len=2 cap=5
b = s[2:5] → [2 3 4]      len=3 cap=4
a[1] = 100
→ s = [0 1 100 3 4 5]   a = [1 100]   b = [100 3 4]
```

`a[1]` et `b[0]` sont **le même emplacement mémoire** : l'indice 2 du tableau. Le dessin :

```
tableau : [ 0 | 1 | 100 | 3 | 4 | 5 ]
index   :   0   1    2    3   4   5
s       : ptr→0  len=6 cap=6
a       : ptr→1  len=2 cap=5
b       : ptr→2  len=3 cap=4
```

Les capacités s'expliquent : depuis l'indice 1, il reste 5 emplacements jusqu'au bout.

## E4 — Le piège d'`append`

```
avant  : s = [0 1 2 3 4 5], part = [1 2] (len=2 cap=5)
append : len(2) < cap(5) → écrit à l'indice 3 du tableau
après  : s = [0 1 2 99 4 5], part = [1 2 99]
```

Le `3` de `s` est **écrasé**. Correction :

```go
part := s[1:3:3]            // len=2 cap=2
part = append(part, 99)     // cap atteinte → réalloue
// s = [0 1 2 3 4 5] intact, part = [1 2 99] dans un NOUVEAU tableau
```

Le troisième indice borne la capacité. Règle pratique : **toute fonction qui retourne un
sous-slice d'un slice qu'elle ne possède pas devrait borner la capacité** — sinon l'appelant
peut corrompre les données du propriétaire par un simple `append`, ce qui est
indébogable.

## E5 — `copy`

```go
// Correct
dst := make([]int, len(src))
copy(dst, src)

// BUG SILENCIEUX
var dst []int      // len=0
copy(dst, src)     // copie min(0, len(src)) = 0 élément, sans erreur
fmt.Println(dst)   // []

// Moderne
dst := slices.Clone(src)
```

`copy` ne redimensionne **jamais** la destination : il copie `min(len(dst), len(src))`
éléments. Il retourne ce nombre — ignorer ce retour est la cause du bug.

## Exercice intermédiaire — `slicelib`

```go
// Chunk découpe s en morceaux de taille size. Le dernier peut être plus court.
// CHOIX DOCUMENTÉ : les morceaux PARTAGENT le tableau sous-jacent de s.
// C'est gratuit et c'est ce que fait slices.Chunk de la stdlib. Conséquence :
// écrire dans un morceau modifie s. La capacité est bornée (a:b:b) pour qu'un
// append sur un morceau ne déborde pas sur le morceau suivant.
func Chunk(s []int, size int) ([][]int, error) {
	if size <= 0 {
		return nil, fmt.Errorf("taille de morceau invalide : %d", size)
	}
	out := make([][]int, 0, (len(s)+size-1)/size) // préallocation exacte
	for i := 0; i < len(s); i += size {
		end := min(i+size, len(s))
		out = append(out, s[i:end:end]) // capacité bornée : essentiel
	}
	return out, nil
}

// Flatten aplatit. Deux passes : compter, puis copier — une seule allocation.
func Flatten(s [][]int) []int {
	n := 0
	for _, part := range s {
		n += len(part)
	}
	out := make([]int, 0, n)
	for _, part := range s {
		out = append(out, part...)
	}
	return out
}

// Insert insère vals à l'indice i.
// CHOIX DOCUMENTÉ : comme slices.Insert, la fonction peut réutiliser le tableau
// de s. L'appelant doit donc utiliser la valeur retournée et considérer s comme
// invalide après l'appel.
func Insert(s []int, i int, vals ...int) []int {
	if i < 0 {
		i = 0
	}
	if i > len(s) {
		i = len(s)
	}
	out := make([]int, 0, len(s)+len(vals))
	out = append(out, s[:i]...)
	out = append(out, vals...)
	out = append(out, s[i:]...)
	return out
}

// Delete supprime [i, j). Réutilise le tableau : s est invalidé.
func Delete(s []int, i, j int) []int {
	i, j = max(0, i), min(len(s), j)
	if i >= j {
		return s
	}
	return append(s[:i], s[j:]...) // décale à gauche, en place
}

func Zip(a, b []int) ([][2]int, error) {
	if len(a) != len(b) {
		return nil, fmt.Errorf("longueurs différentes : %d et %d", len(a), len(b))
	}
	out := make([][2]int, len(a))
	for i := range a {
		out[i] = [2]int{a[i], b[i]}
	}
	return out, nil
}

// Window retourne les fenêtres glissantes de taille size.
// Un slice trop court donne un résultat VIDE, pas une erreur : c'est un cas
// normal, pas une anomalie.
func Window(s []int, size int) [][]int {
	if size <= 0 || len(s) < size {
		return nil
	}
	out := make([][]int, 0, len(s)-size+1)
	for i := 0; i+size <= len(s); i++ {
		out = append(out, s[i:i+size:i+size])
	}
	return out
}
```

**Les quatre points de correction :**

1. **La capacité bornée `s[i:end:end]`** est ce qui distingue une implémentation correcte
   d'une implémentation seulement fonctionnelle. Sans elle, `append` sur le morceau 1 écrase
   le premier élément du morceau 2 — silencieusement.
2. **`(len(s)+size-1)/size`** est la division entière arrondie vers le haut. L'écrire avec des
   flottants (`math.Ceil`) fonctionne mais introduit une imprécision inutile.
3. **`Flatten` en deux passes.** Compter d'abord coûte un parcours supplémentaire, mais évite
   toutes les réallocations. Sur un million d'éléments, c'est plus rapide malgré la passe en
   plus — un résultat contre-intuitif à vérifier soi-même.
4. **`Delete` renvoie `append(s[:i], s[j:]...)`.** Le tableau est réutilisé, les éléments sont
   décalés à gauche. C'est exactement ce que fait `slices.Delete`… à un détail près : depuis
   Go 1.22, `slices.Delete` **met à zéro** les éléments devenus inaccessibles à la fin, pour
   que le ramasse-miettes puisse libérer ce qu'ils référençaient. Sur un `[]int` cela ne change
   rien ; sur un `[]*User` c'est une fuite mémoire évitée. C'est précisément ce qu'on découvre
   en lisant `go doc -src slices.Delete`, et c'était le but de la dernière consigne.

## Défi

**a) Rotate en O(1) mémoire**

```go
func reverse(s []int) {
	for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
		s[i], s[j] = s[j], s[i]
	}
}

func Rotate(s []int, k int) {
	n := len(s)
	if n == 0 {
		return
	}
	k = ((k % n) + n) % n // normalise, y compris pour k négatif
	if k == 0 {
		return
	}
	reverse(s[:k])
	reverse(s[k:])
	reverse(s)
}
```

Les **trois inversions** : inverser la première partie, inverser la seconde, inverser le
tout. O(n) en temps, O(1) en mémoire. La normalisation `((k%n)+n)%n` gère `k` négatif —
en Go, `-1 % 5` vaut `-1`, pas `4`.

**b) Dedup**

```go
func Dedup(s []int) []int {
	seen := make(map[int]struct{}, len(s))
	out := s[:0] // réutilise le tableau sous-jacent
	for _, v := range s {
		if _, ok := seen[v]; ok {
			continue
		}
		seen[v] = struct{}{}
		out = append(out, v)
	}
	return out
}
```

Sûr parce qu'on n'écrit jamais plus loin qu'on n'a lu : `len(out) <= i` à tout instant.
Attention : le slice d'origine est **détruit** — à documenter.

`slices.Compact` ne retire que les doublons **adjacents**. Sur `[1,2,1]` il ne fait rien.
Il est conçu pour être appliqué après un tri, et il ne préserve donc pas l'ordre de première
apparition.

**c) La fuite mémoire**

```go
// FUITE : le sous-slice retient le tableau de 50 Mo entier
func lastLineLeaky(data []byte) []byte {
	i := bytes.LastIndexByte(data, '\n')
	return data[i+1:]
}

// CORRECT : copie explicite, les 50 Mo peuvent être libérés
func lastLine(data []byte) []byte {
	i := bytes.LastIndexByte(data, '\n')
	return bytes.Clone(data[i+1:])
}
```

Mesure :

```go
big := make([]byte, 50<<20)
line := lastLineLeaky(big) // ou lastLine(big)
big = nil
runtime.GC()
var m runtime.MemStats
runtime.ReadMemStats(&m)
fmt.Printf("HeapAlloc = %d Mo (line=%d octets)\n", m.HeapAlloc>>20, len(line))
```

Version fuyante : ~50 Mo restent alloués. Version correcte : quelques centaines de
kilo-octets. Le GC de Go libère un objet entier ou rien du tout — il ne sait pas récupérer
la partie inutilisée d'un tableau encore référencé.

Ce bug est fréquent en production sur du parsing de fichiers et des buffers réseau.

## Réponses du quiz

1. `[3]int` est un **tableau** de taille fixe : la taille fait partie du type. `[]int` est
   un **slice**, une vue de taille variable.
2. Trois champs : un pointeur vers le premier élément, la longueur, la capacité.
3. **Non.** Il crée un nouveau descripteur pointant dans le même tableau.
4. `5` — la capacité s'étend du début de la fenêtre jusqu'à la fin du tableau sous-jacent.
5. Parce qu'`append` peut **réallouer** : il retourne alors un slice différent. Sans
   réaffectation, le résultat est perdu. `go vet` le signale.
6. Il alloue un nouveau tableau plus grand, y copie les éléments, ajoute la valeur, et
   retourne un slice pointant vers ce nouveau tableau.
7. **Non.** C'est un détail d'implémentation qui a déjà changé plusieurs fois.
8. Le nombre d'éléments copiés, soit `min(len(dst), len(src))`. Si `dst` est plus court,
   seuls `len(dst)` éléments sont copiés, sans erreur.
9. En bornant la capacité : `s[low:high:high]`. Ou en copiant avec `slices.Clone`.
10. Non, cela ne compile pas (sauf `s == nil`). Utiliser `slices.Equal`.
11. `var s []int` vaut `nil` et **n'alloue rien** ; `s := []int{}` alloue un slice vide non
    nil. Comportement identique pour `len`, `range` et `append` ; la différence se voit sur
    `s == nil` et à l'encodage JSON (`null` contre `[]`).
