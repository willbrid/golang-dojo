# Copié ou partagé ?

En Go, **tout est passé par valeur**. Sans exception, jamais. Cette phrase est vraie et
pourtant elle semble contredite tous les jours : on passe une map à une fonction, la fonction
la modifie, et l'appelant voit le changement.

Il n'y a pas de contradiction. Il y a une seule question à se poser, et c'est toujours la
même :

> **Qu'est-ce qui est copié exactement — et ce qui est copié contient-il un pointeur ?**

Copier une valeur qui *contient* un pointeur donne deux valeurs distinctes qui désignent la
**même** chose en mémoire. La copie a bien eu lieu ; elle a copié le papier avec l'adresse,
pas la maison.

Cette page est le fil rouge du cours. Elle se lit une première fois maintenant, puis se
relit après chaque leçon citée dans la colonne de droite.

---

## Le tableau

| Type | Ce que la copie duplique | Une fonction qui le reçoit peut-elle modifier l'original ? | Leçon |
|---|---|---|---|
| `int`, `float64`, `bool`, `rune` | la valeur entière | **non** | N1 L2 |
| `struct` (champs simples) | tous les champs, un par un | **non** | N2 L2 |
| tableau `[3]int` | les 3 cases | **non** — c'est ce qui le distingue d'un slice | N1 L6 |
| `string` | (pointeur vers octets, longueur) | **non**, elle est immuable | N1 L8 |
| **slice** `[]int` | (pointeur, longueur, capacité) | **oui** pour `s[i] = …` · **non** pour `append` | N1 L6 |
| **map** | le pointeur vers la table interne | **oui** | N1 L7 |
| **channel** | le pointeur vers la structure interne | **oui** | N6 |
| pointeur `*T` | l'adresse | **oui**, via `*p = …` | N2 L1 |
| **interface** | (type dynamique, valeur) | dépend de ce qu'elle contient | N2 L4 |
| récepteur valeur `func (t T)` | la valeur `T` entière | **non** | N2 L3 |
| récepteur pointeur `func (t *T)` | l'adresse | **oui** | N2 L3 |

---

## Les trois pièges que ce tableau explique

### 1. Le slice à moitié partagé

C'est la ligne la plus subtile du tableau : un slice est **partagé pour l'écriture d'un
élément** et **non partagé pour sa longueur**. `s[0] = 9` traverse le pointeur et se voit de
partout ; `s = append(s, 9)` réaffecte la *copie locale* du triplet — et parfois réalloue le
tableau sous-jacent, ce qui rompt le partage sans prévenir.

D'où deux surprises symétriques, qu'il faut savoir reconnaître :

- une fonction qui `append` sur son paramètre ne change rien pour l'appelant ;
- deux slices issus du même tableau peuvent s'écraser mutuellement.

### 2. Le récepteur qui n'écrit nulle part

Une méthode à récepteur valeur reçoit une copie. Elle peut la modifier autant qu'elle veut :
la copie meurt au `return`. Le compilateur ne dit rien, parce que le code est légal — il est
simplement inutile.

Un récepteur **n'est pas une créature spéciale** : c'est un paramètre ordinaire, écrit avant
le nom de la méthode au lieu d'être entre les parenthèses. Tout ce qui vaut pour les
paramètres vaut pour lui.

### 3. L'interface qui n'est pas nil

Une interface copie **deux** informations : le type dynamique et la valeur. Elle ne vaut
`nil` que si les **deux** sont vides. Y ranger un pointeur nil remplit la première case —
l'interface n'est donc pas nil, alors qu'elle « ne contient rien ».

---

## Ce que je dois retenir

- « Tout est passé par valeur » est vrai partout, sans exception.
- Ce qui varie, ce n'est pas la règle : c'est **ce que la valeur contient**.
- Trois types de la bibliothèque de base contiennent un pointeur sans le montrer : **slice**,
  **map**, **channel**. Ce sont eux qui donnent l'illusion d'un passage par référence.
- Le slice est le seul à être **partiellement** partagé. C'est pour cela qu'il produit le plus
  de bugs.
