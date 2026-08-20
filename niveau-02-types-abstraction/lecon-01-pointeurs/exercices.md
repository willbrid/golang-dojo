# Leçon 1 — Exercices

Code dans `mes-solutions/niveau-01/lecon-09/`.

## Exercices faciles

**E1 — Swap.**
Écrire `Swap(a, b *int)` qui échange deux valeurs. Puis écrire `SwapValues(a, b int)` avec
des valeurs et constater que rien ne change. Expliquer pourquoi en une phrase.

**E2 — Panique nil.**
Écrire un programme qui déclenche volontairement une `nil pointer dereference`. Relever le
message exact **et la trace de pile**. Identifier dans la trace la ligne fautive.
Puis corriger avec un test `nil`.

**E3 — Adressabilité.**
Lesquelles de ces expressions compilent ? Prédire, puis vérifier, puis expliquer chaque refus :
```go
p1 := &42
x := 42;      p2 := &x
p3 := &[]int{1,2,3}[0]
m := map[string]int{"a":1}; p4 := &m["a"]
s := []int{1,2,3};          p5 := &s[0]
arr := [3]int{1,2,3};       p6 := &arr[0]
p7 := &struct{ N int }{N: 1}
```

**E4 — Slices et pointeurs.**
Écrire trois fonctions : `modifyElement([]int)`, `appendItem([]int)`, `appendItemPtr(*[]int)`.
Montrer par l'exécution laquelle est visible par l'appelant et laquelle ne l'est pas.
Conclure : quand `*[]int` est-il **réellement** nécessaire ?

**E5 — Optionnel.**
Écrire un type `Settings` avec un champ `Verbose *bool` et une fonction qui distingue les
trois états : non spécifié, explicitement `false`, explicitement `true`. Montrer les trois cas.
Puis répondre : quelle alternative existe-t-il à `*bool` pour représenter ces trois états ?
Laquelle préférer, et pourquoi ?

---

## Exercice intermédiaire — `ptrlib`

Une série de fonctions qui exigent réellement des pointeurs — sans structs, qui arrivent à la
leçon suivante.

```go
func Swap(a, b *int)
func Zero(nums []*int)                     // met à 0 tout ce qui est pointé, ignore les nil
func Compact(nums []*int) []*int           // retire les nil, préserve l'ordre
func SumPtrs(nums ...*int) (int, int)      // somme, et nombre de nil ignorés
func Increment(counters map[string]*int, key string) error  // crée l'entrée si absente
func MinPtr(nums []int) *int               // pointeur vers le minimum, nil si vide
```

**Contraintes :**
1. Aucune fonction ne panique sur un pointeur `nil`, un slice `nil` ou une map `nil`.
2. `MinPtr` retourne un pointeur **vers un élément du slice d'origine** : modifier `*p` doit modifier le slice. Le prouver dans `main`. *(Question : est-ce un bon contrat d'API ? Répondre en commentaire.)*
3. `Increment` reçoit une map de pointeurs. Que se passe-t-il si la map elle-même est nil ? Traiter le cas explicitement.
4. `Compact` ne doit pas allouer un nouveau slice si aucun `nil` n'est présent. *(Technique `s[:0]` du niveau 1, leçon 6 — attention : ici on modifie l'entrée. Le documenter.)*
5. Écrire dans `main` une démonstration de chaque cas limite : slice vide, tous nil, aucun nil, un seul élément.

*Le vrai enseignement : après `MinPtr`, se demander combien de temps le slice entier reste en
mémoire tant qu'on garde ce pointeur. La réponse annonce le niveau 11.*

## Défi

**a) Pointeur de pointeur.**
Écrire `RemoveFirst(head **int, target int) bool`… puis constater que l'exercice n'a pas de
sens sans structure chaînée. Le vrai exercice de `**T` est la **liste chaînée**, au programme
de la [leçon 3](../lecon-03-methodes/exercices.md), défi (b). En attendant, comprendre
pourquoi `**T` existe :

```go
func setToNil(p *int)   { p = nil }     // sans effet chez l'appelant
func setToNil(p **int)  { *p = nil }    // fonctionne
```
Écrire les deux, montrer la différence, et l'expliquer en trois phrases. Quand a-t-on
réellement besoin de `**T` en Go ? *(Indice : la même raison qui justifie `*[]int`.)*

**b) Aliasing volontaire.**
Écrire une fonction qui reçoit `[]*int`, y range des pointeurs vers la **même** variable, puis
la modifie une seule fois. Afficher tout le slice. Expliquer ce qu'on observe et dans quel cas
réel ce piège se produit accidentellement.
*(Indice : boucle, variable temporaire, et `&v`. Le correctif de Go 1.22 aide-t-il ici ?)*

**c) Coût mémoire d'une indirection.**
Comparer `[]int` et `[]*int` de 1 000 000 d'éléments : mémoire occupée (`runtime.ReadMemStats`)
et temps de parcours pour en faire la somme. **Prédire les deux écarts avant de mesurer.**
Expliquer le résultat — la réponse ne concerne pas seulement le nombre d'octets.

## Quiz

1. Que fait `&x` ? Que fait `*p` ?
2. Go a-t-il le passage par référence ?
3. Faut-il un pointeur pour modifier un élément d'un slice depuis une fonction ?
4. Pourquoi `*map[string]int` est-il presque toujours une erreur ?
5. Que se passe-t-il quand on déréférence un pointeur `nil` ?
6. Peut-on écrire `p := &42` ? Pourquoi ?
7. Retourner l'adresse d'une variable locale est-il sûr en Go ? Pourquoi ?
8. Peut-on faire de l'arithmétique de pointeurs en Go ? Quel bénéfice en tire-t-on ?
9. Citer les quatre raisons légitimes d'utiliser un pointeur.
10. Pourquoi « c'est plus rapide » est-il un mauvais argument pour choisir un pointeur ?
