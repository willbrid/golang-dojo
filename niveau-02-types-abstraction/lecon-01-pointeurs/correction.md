# Leçon 1 — Corrigé

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

## Exercice intermédiaire — `ptrlib`

```go
func Swap(a, b *int) {
	if a == nil || b == nil {
		return // ne jamais paniquer sur un nil : contrainte 1
	}
	*a, *b = *b, *a
}

// Zero met à 0 tout ce qui est pointé, en ignorant les nil.
func Zero(nums []*int) {
	for _, p := range nums { // range sur un slice nil : zéro tour, aucun problème
		if p != nil {
			*p = 0
		}
	}
}

// Compact retire les nil en préservant l'ordre.
// ATTENTION : réutilise le tableau sous-jacent, donc MODIFIE l'entrée.
// L'appelant doit utiliser la valeur retournée et considérer nums comme invalide.
func Compact(nums []*int) []*int {
	out := nums[:0] // len 0, même tableau
	for _, p := range nums {
		if p != nil {
			out = append(out, p)
		}
	}
	// Mettre à nil la queue devenue inaccessible : sinon le tableau retient
	// des pointeurs vers des objets que le GC pourrait libérer.
	for i := len(out); i < len(nums); i++ {
		nums[i] = nil
	}
	return out
}

func SumPtrs(nums ...*int) (sum, skipped int) {
	for _, p := range nums {
		if p == nil {
			skipped++
			continue
		}
		sum += *p
	}
	return sum, skipped
}

// Increment crée l'entrée si elle est absente.
func Increment(counters map[string]*int, key string) error {
	if counters == nil {
		return errors.New("map nil : écrire dedans paniquerait")
	}
	if p, ok := counters[key]; ok && p != nil {
		*p++
		return nil
	}
	n := 1
	counters[key] = &n
	return nil
}

// MinPtr retourne un pointeur VERS UN ÉLÉMENT du slice d'origine.
// CHOIX DOCUMENTÉ : modifier *p modifie donc nums. C'est puissant et dangereux —
// voir la discussion ci-dessous.
func MinPtr(nums []int) *int {
	if len(nums) == 0 {
		return nil
	}
	best := 0
	for i, v := range nums {
		if v < nums[best] {
			best = i
		}
		_ = i
	}
	return &nums[best] // les éléments d'un slice SONT adressables
}
```

**Les trois points de correction :**

1. **Contrainte 4 et la mise à nil de la queue.** `Compact` laisse à la fin du tableau des
   pointeurs vers des objets qui ne sont plus dans le slice retourné. Tant que le tableau vit,
   le ramasse-miettes ne peut pas les libérer. C'est exactement ce que fait `slices.Delete`
   depuis Go 1.22 — et l'oublier est une fuite mémoire silencieuse, invisible sur des `*int`,
   coûteuse sur des `*Image`.

2. **Contrainte 2 — `MinPtr` est-il un bon contrat d'API ?** Non, dans le cas général.
   Retourner un pointeur vers l'intérieur d'une structure qu'on ne possède pas expose l'état
   de l'appelant à une modification à distance, et le lien n'est visible nulle part dans la
   signature. `func Min(nums []int) (int, bool)` est presque toujours préférable. Le pointeur
   ne se justifie que si l'on veut explicitement permettre la modification en place — et il
   faut alors le nommer en conséquence (`MinRef`, `MinAddr`) et le documenter.

3. **Contrainte 3 — la map nil.** Lire dans une map nil est sûr, y écrire **panique**
   (niveau 1, leçon 7). Une fonction publique qui reçoit une map doit donc soit refuser
   explicitement le nil, soit ne faire que lire.

**La question de la fin :** tant qu'on garde le pointeur retourné par `MinPtr`, **le tableau
sous-jacent entier reste en mémoire** — un seul `*int` conservé peut retenir un slice d'un
gigaoctet. Le ramasse-miettes de Go libère un objet entier ou rien. C'est le même phénomène que
le sous-slice de la leçon 6 du niveau 1, et c'est un problème réel en production sur des
buffers réseau et des fichiers analysés.

## Défi

**a) Pointeur de pointeur**

```go
func setToNilValue(p *int)  { p = nil }   // réaffecte la COPIE locale : sans effet
func setToNilPtr(p **int)   { *p = nil }  // écrit à l'adresse du pointeur : fonctionne

x := 42
p := &x
setToNilValue(p)
fmt.Println(p == nil)   // false
setToNilPtr(&p)
fmt.Println(p == nil)   // true
```

**L'explication en trois phrases.** Tout est passé par valeur en Go, y compris un pointeur :
`setToNilValue` reçoit une copie de l'adresse et ne peut modifier que sa copie. Pour remplacer
le pointeur **de l'appelant**, il faut recevoir l'adresse de ce pointeur, donc un `**int`.
C'est exactement la même raison qui rend `*[]int` nécessaire quand une fonction doit remplacer
le slice entier de l'appelant, et non seulement son contenu.

En pratique, `**T` est rare en Go : on préfère **retourner** la nouvelle valeur
(`func clear(p *int) *int`), comme le fait `append`. Son usage légitime reste la manipulation
de structures chaînées, où l'on manipule l'**emplacement** d'un pointeur plutôt que le
pointeur — technique reprise à la [leçon 3](../lecon-03-methodes/), défi (b).

**b) Aliasing volontaire**

```go
var ptrs []*int
v := 0
for i := range 3 {
	v = i
	ptrs = append(ptrs, &v) // TOUS pointent vers la MÊME variable
}
for _, p := range ptrs {
	fmt.Print(*p, " ")      // 2 2 2
}
```

Le correctif de Go 1.22 **n'aide pas** : il concerne la variable de boucle `i`, pas `v`, qui
est déclarée à l'extérieur. Il n'existe qu'une seule `v`, donc une seule adresse, donc trois
pointeurs identiques.

Le cas réel où cela arrive : une boucle qui construit un slice de pointeurs vers une structure
temporaire réutilisée — par exemple en lisant des lignes d'un fichier dans un tampon réutilisé
et en stockant `&record`. Le symptôme est déroutant : toutes les entrées de la collection sont
identiques et valent la dernière lue.

**c) Coût d'une indirection**

| | `[]int` (10⁶) | `[]*int` (10⁶) |
|---|---|---|
| Mémoire | 8 Mo | 8 Mo (pointeurs) + 10⁶ allocations de 8 octets ≈ **24 Mo** |
| Somme | ~0,5 ms | ~5 à 15 ms |

Le rapport de mémoire est d'environ 3× : chaque `int` alloué séparément porte un surcoût
d'allocation et d'alignement.

Le rapport de **temps** est bien plus élevé, et n'est pas dû au nombre d'instructions —
identique dans les deux cas. Il vient du **cache CPU** : `[]int` est contigu, chaque ligne de
cache de 64 octets sert pour 8 valeurs et le préchargeur anticipe. `[]*int` oblige à suivre un
pointeur vers une adresse imprévisible pour chaque élément : c'est un défaut de cache par
accès, soit une centaine de nanosecondes perdues à chaque fois.

D'où une règle de conception qui reviendra au niveau 11 : **préférer `[]T` à `[]*T`** sauf
raison précise — objets volumineux, partage voulu, ou nécessité de `nil`.

## Réponses du quiz

1. `&x` donne l'**adresse** de `x` ; `*p` **suit** le pointeur et donne la valeur pointée.
2. **Non.** Go n'a que le passage par valeur — mais on peut passer la valeur d'une adresse.
3. **Non.** Un slice partage déjà son tableau sous-jacent : `s[0] = 1` suffit. Un `*[]T` n'est
   nécessaire que pour **remplacer** le slice entier de l'appelant.
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
