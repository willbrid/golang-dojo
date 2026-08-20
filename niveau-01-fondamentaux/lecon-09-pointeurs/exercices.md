# Leçon 9 — Exercices

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

## Exercice intermédiaire — `linkedlist`

Implémenter une liste chaînée simple — la structure de données qui *exige* des pointeurs.

```go
type Node struct {
	Value int
	Next  *Node
}

type List struct {
	head *Node
	size int
}

func (l *List) PushFront(v int)
func (l *List) PushBack(v int)
func (l *List) PopFront() (int, bool)
func (l *List) Remove(v int) bool      // supprime la PREMIÈRE occurrence
func (l *List) Len() int
func (l *List) Slice() []int
func (l *List) Reverse()               // inverse la liste EN PLACE, sans allouer de nœud
```

**Contraintes :**
1. La zéro-valeur `var l List` doit être **immédiatement utilisable** (rappel de la leçon 2). Aucun constructeur obligatoire.
2. Toutes les méthodes doivent fonctionner sur une liste vide sans paniquer.
3. `Remove` du premier élément, du dernier, du seul élément, et d'un élément absent : les quatre cas doivent être corrects. *(C'est là que 90 % des implémentations échouent.)*
4. `Reverse` ne crée **aucun** nouveau nœud : seuls les pointeurs `Next` sont réécrits.
5. `Slice()` retourne une copie ; l'appelant ne doit pas pouvoir atteindre les nœuds internes.
6. Aucune fuite : après `PopFront`, le nœud retiré ne doit plus être référencé par la liste.
7. Écrire dans `main` au moins dix scénarios couvrant les cas limites.

*Question de conception à trancher avant de coder : pourquoi `head` est-il non exporté ?
Que se passerait-il s'il était public ?*

---

## Défi

**a) Détection de cycle.**
Écrire `HasCycle(head *Node) bool` qui détecte si une liste chaînée boucle sur elle-même,
en **O(1) mémoire** (interdiction d'utiliser une map de nœuds visités).
*Indice : deux parcours à des vitesses différentes. C'est l'algorithme de Floyd.*

Construire un cycle à la main pour tester — et se demander pourquoi ce type de bug est si
difficile à repérer autrement.

**b) Liste doublement chaînée.**
Ajouter `Prev *Node` et implémenter `PushBack`, `PopBack` et `Remove(n *Node)` en O(1).
Quelle méthode devient beaucoup plus simple ? Laquelle devient plus dangereuse ?

**c) Question de mémoire.**
Une liste chaînée de 1 000 000 d'entiers contre un `[]int` de même taille : lequel occupe
le plus de mémoire, et dans quel rapport ? **Calculer** avant de mesurer (`unsafe.Sizeof`
sur `Node` aide), puis mesurer avec `runtime.ReadMemStats`. Lequel est le plus rapide à
parcourir, et pourquoi ? *(La réponse tient au cache CPU — premier contact avec le niveau 8.)*

---

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
