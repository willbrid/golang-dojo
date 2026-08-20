# Leçon 6 — Exercices

Code dans `mes-solutions/niveau-01/lecon-06/`.

> Pour tous ces exercices : **prédire la sortie avant d'exécuter**, puis expliquer chaque écart.
> C'est là que se joue la compréhension des slices.

## Exercices faciles

**E1 — len et cap.**
Écrire un programme qui affiche `len` et `cap` après chaque opération :
```go
s := make([]int, 0, 3)
// afficher après chaque append de 1 à 10
```
Relever à quels moments la capacité change et de combien. Le facteur observé est-il garanti
par la spécification ? *(Vérifier dans la documentation officielle avant de répondre.)*

**E2 — Tableau vs slice.**
Écrire `modifyArray([3]int)` et `modifySlice([]int)`, chacune modifiant son premier élément.
Les appeler et observer. Expliquer la différence en deux phrases.

**E3 — Aliasing.**
```go
s := []int{0, 1, 2, 3, 4, 5}
a := s[1:3]
b := s[2:5]
a[1] = 100
fmt.Println(s, a, b)
```
**Prédire les trois sorties avant d'exécuter.** Puis dessiner sur papier le tableau
sous-jacent et les trois descripteurs (ptr/len/cap).

**E4 — Le piège d'`append`.**
```go
s := []int{0, 1, 2, 3, 4, 5}
part := s[1:3]
part = append(part, 99)
fmt.Println(s, part)
```
Prédire, exécuter, expliquer. Puis corriger avec un slice à trois indices et vérifier.

**E5 — `copy`.**
Écrire trois versions d'une duplication de slice : avec `copy` et une destination bien
allouée, avec `copy` et `var dst []int` (constater le bug), avec `slices.Clone`.
Vérifier dans chaque cas que la modification de la copie n'affecte pas l'original.

---

## Exercice intermédiaire — `ringbuffer`

Implémenter un **buffer circulaire** de taille fixe : quand il est plein, chaque nouvel
élément écrase le plus ancien.

```go
type Ring struct {
	// à concevoir — mais un seul slice alloué UNE fois, jamais réalloué
}

func NewRing(size int) (*Ring, error)
func (r *Ring) Push(v int)        // écrase le plus ancien si plein
func (r *Ring) Len() int          // nombre d'éléments réellement présents
func (r *Ring) Slice() []int      // du plus ancien au plus récent
```

**Contraintes :**
1. Le tableau sous-jacent est alloué **une seule fois** dans `NewRing`. Aucun `append` qui réalloue, jamais.
2. `size <= 0` retourne une erreur.
3. `Slice()` retourne une **copie** : l'appelant ne doit pas pouvoir corrompre l'état interne. *(Justifier ce choix en commentaire : quel bug cela évite-t-il ?)*
4. `Slice()` sur un buffer vide retourne un slice vide (ou nil) et surtout **ne panique pas**.
5. Le buffer doit fonctionner correctement après des milliers de `Push` — aucune fuite, aucun débordement d'indice.
6. Écrire dans `main` au moins six scénarios : vide, partiellement rempli, exactement plein, un tour complet, deux tours, `size == 1`.

*Cette structure reviendra au niveau 4 (fenêtre glissante de métriques) et au niveau 8
(pooling). L'écrire correctement maintenant est un bon investissement.*

---

## Défi

**a) `Rotate(s []int, k int)`** — décaler les éléments de `k` positions vers la gauche,
**en place**, avec `O(1)` mémoire supplémentaire (aucune allocation d'un second slice).
`k` peut être négatif ou supérieur à `len(s)`.
*Indice : l'algorithme classique repose sur trois inversions. Trouver lequel.*

**b) `Dedup(s []int) []int`** — retirer les doublons **en préservant l'ordre de première
apparition**, en réutilisant le tableau sous-jacent (technique `s[:0]`).
Comparer avec `slices.Compact` : que fait `Compact` exactement, et pourquoi n'est-ce pas
la même chose ?

**c) Chasse à la fuite mémoire.** Écrire :
```go
func lastLine(data []byte) []byte   // retourne la dernière ligne d'un buffer de 50 Mo
```
Une première version qui **retient** les 50 Mo, une seconde qui ne retient que la ligne.
Prouver l'écart avec `runtime.ReadMemStats` et `runtime.GC()` : afficher `HeapAlloc` après
avoir mis le gros buffer hors de portée dans les deux cas.

*Le (c) est le premier contact avec le raisonnement du niveau 8 : ne pas croire, mesurer.*

---

## Quiz

1. Quelle est la différence de type entre `[3]int` et `[]int` ?
2. Que contient exactement un slice, en interne ?
3. `s[1:3]` copie-t-il les éléments ?
4. Que vaut `cap(s[1:3])` si `s` a une longueur de 6 et une capacité de 6 ?
5. Pourquoi doit-on écrire `s = append(s, x)` et pas seulement `append(s, x)` ?
6. Que fait `append` quand `len == cap` ?
7. Le facteur de croissance d'`append` est-il garanti par la spécification ?
8. Que retourne `copy(dst, src)` et que se passe-t-il si `dst` est plus court que `src` ?
9. Comment retourner un sous-slice sans risquer qu'un `append` de l'appelant écrase l'original ?
10. Peut-on comparer deux slices avec `==` ? Que faire à la place ?
11. Quelle différence pratique entre `var s []int` et `s := []int{}` ?
