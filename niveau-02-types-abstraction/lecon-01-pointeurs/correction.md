# Leçon 9 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Swap

```go
func Swap(a, b *int)       { *a, *b = *b, *a }
func SwapValues(a, b int)  { a, b = b, a }      // sans effet
```

`SwapValues` échange bien ses paramètres — mais ce sont des **copies** locales, détruites au
retour. Rien de ce que fait une fonction sur ses paramètres valeur n'est visible de
l'extérieur.

## E2 — Panique nil

```
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x...]

goroutine 1 [running]:
main.main()
	/home/.../main.go:8 +0x18
exit status 2
```

La trace se lit **de haut en bas** : la fonction la plus profonde d'abord. `main.go:8` est
la ligne fautive. `exit status 2` est le code de sortie d'un programme Go qui panique.

## E3 — Adressabilité

| Expression | Compile ? | Pourquoi |
|---|---|---|
| `&42` | **non** | Un littéral n'a pas d'adresse : il n'existe pas en mémoire en tant que variable |
| `&x` | oui | Une variable est adressable |
| `&[]int{1,2,3}[0]` | oui | Les éléments d'un **slice** sont adressables (le tableau existe en mémoire) |
| `&m["a"]` | **non** | Les entrées de map ne sont pas adressables : le runtime peut les déplacer lors d'un redimensionnement |
| `&s[0]` | oui | idem slice |
| `&arr[0]` | oui | si `arr` est une variable (donc adressable) |
| `&struct{N int}{N:1}` | oui | **Exception** : un littéral **composite** est adressable ; Go alloue implicitement |

La règle : est adressable ce qui a une existence stable en mémoire. Les littéraux
composites (`&T{}`) sont l'exception pratique voulue par le langage, parce que `&User{…}`
est trop utile pour l'interdire.

## E4 — Slices et pointeurs

```go
func modifyElement(s []int)  { s[0] = 99 }      // VISIBLE : même tableau
func appendItem(s []int)     { s = append(s, 1) } // invisible : réaffecte la copie locale
func appendItemPtr(s *[]int) { *s = append(*s, 1) } // visible
```

`*[]int` n'est **réellement nécessaire** que si la fonction doit **remplacer le descripteur**
de l'appelant : `append`, tri qui réalloue, remise à nil. Pour modifier des éléments
existants, jamais.

En pratique, l'idiome Go est de **retourner le nouveau slice** plutôt que de prendre un
pointeur : `func addItem(s []int) []int` — c'est exactement ce que fait `append` lui-même.

## E5 — Optionnel

```go
type Settings struct{ Verbose *bool }

switch {
case s.Verbose == nil:  fmt.Println("non spécifié")
case *s.Verbose:        fmt.Println("activé explicitement")
default:                fmt.Println("désactivé explicitement")
}
```

**Alternatives à `*bool` :**
1. Un type énuméré à trois valeurs (`Unset`, `True`, `False`) — plus explicite, aucun risque
   de panique, mais plus verbeux à construire.
2. Un couple `Verbose, VerboseSet bool` — simple, mais deux champs à maintenir cohérents.
3. Un type générique `Option[bool]` (niveau 2).

**Préférence :** le type énuméré quand la valeur circule dans le domaine métier, le `*bool`
quand il s'agit de désérialisation JSON/SQL — parce que `encoding/json` gère nativement le
pointeur nil pour un champ absent. Le choix dépend donc du contexte, et c'est ce qu'il
fallait argumenter.

## Exercice intermédiaire — `linkedlist`

```go
package main

type Node struct {
	Value int
	Next  *Node
}

// List est une liste chaînée simple.
// head est non exporté : l'invariant « size correspond au nombre de nœuds »
// ne peut être garanti que si personne d'autre ne peut modifier la chaîne.
// La zéro-valeur (var l List) est immédiatement utilisable.
type List struct {
	head *Node
	size int
}

func (l *List) Len() int { return l.size }

func (l *List) PushFront(v int) {
	l.head = &Node{Value: v, Next: l.head}
	l.size++
}

func (l *List) PushBack(v int) {
	n := &Node{Value: v}
	l.size++
	if l.head == nil {
		l.head = n
		return
	}
	cur := l.head
	for cur.Next != nil {
		cur = cur.Next
	}
	cur.Next = n
}

func (l *List) PopFront() (int, bool) {
	if l.head == nil {
		return 0, false
	}
	n := l.head
	l.head = n.Next
	n.Next = nil // coupe la référence : le nœud retiré ne retient plus la suite
	l.size--
	return n.Value, true
}

// Remove supprime la première occurrence de v.
func (l *List) Remove(v int) bool {
	// Le pointeur de pointeur élimine TOUS les cas particuliers :
	// tête, milieu, queue, liste d'un seul élément — un seul chemin de code.
	pp := &l.head
	for *pp != nil {
		if (*pp).Value == v {
			*pp = (*pp).Next
			l.size--
			return true
		}
		pp = &(*pp).Next
	}
	return false
}

func (l *List) Slice() []int {
	out := make([]int, 0, l.size)
	for cur := l.head; cur != nil; cur = cur.Next {
		out = append(out, cur.Value)
	}
	return out
}

// Reverse inverse la liste en place : seuls les pointeurs Next changent.
func (l *List) Reverse() {
	var prev *Node
	cur := l.head
	for cur != nil {
		next := cur.Next // sauvegarder AVANT d'écraser
		cur.Next = prev
		prev = cur
		cur = next
	}
	l.head = prev
}
```

**Le `**Node` de `Remove` est l'astuce centrale.** L'implémentation naïve traite séparément
« supprimer la tête » et « supprimer ailleurs », avec un pointeur `prev` à maintenir — et
c'est là que les bugs apparaissent. En manipulant l'**emplacement** du pointeur plutôt que
le pointeur, la tête n'est plus un cas particulier : `&l.head` est un emplacement comme un
autre. C'est une technique de C classique, et elle reste la plus élégante en Go.

**Pourquoi `head` non exporté ?** S'il était public, n'importe qui pourrait faire
`l.Head = nil` sans décrémenter `size` : l'invariant serait rompu et `Len()` mentirait.
L'encapsulation ne protège pas les données, elle protège les **invariants**.

## Défi

**a) Floyd**

```go
func HasCycle(head *Node) bool {
	slow, fast := head, head
	for fast != nil && fast.Next != nil {
		slow = slow.Next      // 1 pas
		fast = fast.Next.Next // 2 pas
		if slow == fast {
			return true
		}
	}
	return false
}
```

Si un cycle existe, le pointeur rapide finit forcément par rattraper le lent (il gagne un
pas par tour à l'intérieur du cycle). O(n) en temps, **O(1) en mémoire** — contrairement à
une map de nœuds visités qui coûterait O(n).

**b) Doublement chaînée**

`PopBack` et `Remove(n *Node)` deviennent O(1) : plus besoin de parcourir pour trouver le
prédécesseur. En revanche, chaque opération doit maintenir **deux** pointeurs cohérents,
et un oubli crée une liste corrompue — silencieusement, jusqu'à un parcours en sens inverse.
La complexité en temps baisse, la complexité de maintenance monte.

**c) Mémoire et cache**

```go
unsafe.Sizeof(Node{})   // 16 octets : int (8) + pointeur (8)
```

| | Mémoire | Parcours |
|---|---|---|
| `[]int` de 10⁶ | 8 Mo, **contigus** | ~1 ms |
| Liste de 10⁶ | 16 Mo + surcoût d'allocation ≈ 24 Mo, **dispersés** | ~20 à 50 ms |

Le rapport de mémoire est de 2 à 3×, mais l'écart de **vitesse** est de 20 à 50×. La raison
n'est pas le nombre d'instructions — il est comparable — mais le **cache CPU** : un slice
se parcourt séquentiellement, le préchargeur matériel anticipe ; une liste chaînée saute
d'une adresse imprévisible à l'autre, et chaque saut peut coûter un défaut de cache
(~100 ns, soit des centaines de cycles perdus).

C'est la raison pour laquelle, en Go comme en C++, **on utilise un slice par défaut** et
une liste chaînée seulement quand on a besoin d'insertions/suppressions O(1) au milieu avec
des références stables. Premier contact avec le raisonnement du niveau 8 : la complexité
algorithmique ne dit pas tout, la localité mémoire compte souvent davantage.

## Réponses du quiz

1. `&x` donne l'**adresse** de `x` ; `*p` **suit** le pointeur et donne la valeur pointée.
2. **Non.** Go n'a que le passage par valeur — mais on peut passer la valeur d'une adresse.
3. **Non.** Un slice partage déjà son tableau sous-jacent : `s[0] = 1` suffit.
4. Parce qu'une map se comporte déjà comme une référence. Un `*map` n'est utile que pour
   remplacer entièrement la map de l'appelant, ce qui est très rare.
5. Panique : `invalid memory address or nil pointer dereference`.
6. Non : un littéral n'est pas adressable. Il faut passer par une variable. (Exception : les
   littéraux **composites**, `&T{…}`.)
7. **Oui.** Le compilateur détecte par *escape analysis* que la variable s'échappe et
   l'alloue sur le tas. Contrairement au C, c'est toujours sûr.
8. Non — sauf via le paquet `unsafe`. Bénéfice : **aucun dépassement de tampon**, aucune
   corruption mémoire dans du code Go ordinaire.
9. Muter l'argument ; struct volumineuse ; valeur optionnelle à distinguer de la
   zéro-valeur ; type non copiable (contenant un `sync.Mutex`).
10. Parce que c'est **souvent faux** : une petite struct sur la pile évite une allocation sur
    le tas, une indirection et du travail pour le ramasse-miettes. Et parce que dans tous
    les cas, la seule réponse valable est une **mesure** (`go test -bench`), pas une intuition.
