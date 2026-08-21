# Leçon 3 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Le récepteur qui ne modifie rien

```go
type Counter struct{ N int }
func (c Counter) IncValue()  { c.N++ }   // modifie une copie : aucun effet
func (c *Counter) IncPtr()   { c.N++ }   // modifie l'original
```

Après dix appels : `N` vaut 0 avec `IncValue`, 10 avec `IncPtr`. Le compilateur ne dit
**rien** : le code est parfaitement valide, il ne fait simplement pas ce qu'on croit. C'est
le bug silencieux le plus fréquent de la leçon, et la raison pour laquelle le conseil « en
cas de doute, récepteur pointeur » existe.

## E2 — Ensemble de méthodes

```go
type T struct{ N int }
func (t T) Val() int   { return t.N }   // récepteur VALEUR
func (t *T) Inc()      { t.N++ }        // récepteur POINTEUR

func makeT() T { return T{} }

var v T
v.Val()       // OK
v.Inc()       // OK — Go réécrit (&v).Inc(), car v est ADRESSABLE

p := &T{}
p.Val()       // OK — Go réécrit (*p).Val()
p.Inc()       // OK

makeT().Val() // OK
makeT().Inc() // ERREUR : cannot call pointer method Inc on T
```

Le message exact :
```
./main.go:15:9: cannot call pointer method Inc on T
```

Le résultat d'un appel de fonction est une valeur **temporaire, non adressable** : elle n'a pas
d'emplacement mémoire stable dont on pourrait prendre l'adresse. Go refuse donc la conversion
automatique.

Le tableau à retenir :

| Type | Ensemble de méthodes | Peut appeler |
|---|---|---|
| `T` | méthodes à récepteur valeur | les deux **si la valeur est adressable** |
| `*T` | méthodes à récepteur valeur **et** pointeur | les deux, toujours |

La distinction « ensemble de méthodes » et « ce qu'on peut appeler » est subtile : l'appel
bénéficie du sucre syntaxique, la **satisfaction d'interface** non. C'est ce qui produira le
message `T does not implement I (method Inc has pointer receiver)` à la leçon 4.

## E3 — Méthodes sur un type non-struct

```go
type Temperature float64

func (t Temperature) Fahrenheit() Temperature { return t*9/5 + 32 }
func (t Temperature) String() string          { return fmt.Sprintf("%.1f°C", float64(t)) }

type Tags []string

func (t Tags) Contains(s string) bool { return slices.Contains(t, s) }
func (t Tags) Normalized() Tags {
	out := make(Tags, len(t))
	for i, s := range t {
		out[i] = strings.ToLower(strings.TrimSpace(s))
	}
	return out
}

type Counters map[string]int

func (c Counters) Inc(k string)     { c[k]++ }        // pas besoin de pointeur :
func (c Counters) Total() (n int) {                   // une map partage déjà sa donnée
	for _, v := range c {
		n += v
	}
	return n
}
```

`Counters.Inc` a un récepteur **valeur** et modifie pourtant le contenu : une map contient un
pointeur interne, la copier copie la référence. Même chose pour `Tags` si l'on modifiait un
élément — mais un `append` exigerait un pointeur, puisqu'il remplace le descripteur.

La tentative d'ajouter une méthode à `float64` :
```
./main.go:8:6: cannot define new methods on non-local type float64
```
On ne peut définir des méthodes que sur un type déclaré **dans le package courant**. C'est
cette règle qui rend impossible le *monkey patching* : le comportement d'un type ne peut jamais
être modifié à distance par un autre package.

## E4 — `String()` et récursion

```go
type Duration int
func (d Duration) String() string {
	h, m, s := d/3600, (d%3600)/60, d%60
	return fmt.Sprintf("%dh%02dm%02ds", h, m, s)
}
```

Avec `fmt.Sprintf("%v", d)` **à l'intérieur** de `String()` :

```
runtime: goroutine stack exceeds 1000000000-byte limit
fatal error: stack overflow
```

`%v` teste si la valeur satisfait `fmt.Stringer` → appelle `String()` → qui appelle `%v` →
récursion infinie. La parade : convertir vers le type sous-jacent (`int(d)`) ou utiliser
les champs directement. `go vet` détecte certains cas de cette récursion, mais pas tous.

## E5 — Adressabilité

```
m["a"].N = 2    → cannot assign to struct field m["a"].N in map
m["a"].Inc()    → cannot call pointer method Inc on Item
```

**Correction 1 — `map[string]*Item`** :
```go
m := map[string]*Item{"a": {N: 1}}
m["a"].N = 2      // OK : on modifie ce qui est POINTÉ, pas l'entrée de map
m["a"].Inc()      // OK
```

**Correction 2 — lire / modifier / réécrire** :
```go
it := m["a"]
it.Inc()
m["a"] = it
```

**Laquelle préférer ?** La map de pointeurs si les valeurs sont mutables, volumineuses ou
partagées ; la réécriture si les valeurs sont petites et qu'on veut préserver l'immuabilité
(personne ne peut modifier une entrée sans passer par la map). Le piège de la première : un
`*Item` récupéré et gardé reste valide après un `delete` — l'objet survit, ce qui peut
surprendre.

## Exercice intermédiaire — `inventory`

```go
package main

import (
	"errors"
	"fmt"
	"maps"
	"slices"
	"strings"
)

var ErrUnknownSKU = errors.New("SKU inconnu")

// Item décrit un article. quantity et priceCts sont non exportés :
// c'est la seule façon de garantir « quantité jamais négative » depuis
// l'extérieur du paquet.
type Item struct {
	SKU      string
	Name     string
	quantity int
	priceCts int64
}

// Récepteur VALEUR : Item est petit, String ne modifie rien, et Get en
// retourne des copies — un récepteur pointeur n'apporterait rien ici.
func (it Item) String() string {
	return fmt.Sprintf("%-10s %-20s x%-4d %d,%02d €",
		it.SKU, it.Name, it.quantity, it.priceCts/100, it.priceCts%100)
}
func (it Item) Quantity() int  { return it.quantity }  // accesseur : pas de préfixe Get
func (it Item) PriceCts() int64 { return it.priceCts }

// Récepteur POINTEUR partout : Add et Remove modifient l'inventaire,
// donc toutes les méthodes en prennent un, par cohérence.
type Inventory struct {
	items map[string]*Item
}

func NewInventory() *Inventory {
	return &Inventory{items: make(map[string]*Item)} // sans ceci, Add paniquerait
}

// Add ajoute un article ou augmente la quantité d'un SKU existant.
// Choix documenté : sur un SKU existant, la quantité est CUMULÉE (c'est une
// réception de stock), tandis que le nom et le prix sont ÉCRASÉS (ce sont les
// informations les plus récentes du catalogue). Une API plus stricte séparerait
// Add (création) et Restock (réapprovisionnement) — c'est défendable et sans
// doute meilleur ; le contrat choisi ici est simplement documenté sans ambiguïté.
func (inv *Inventory) Add(sku, name string, qty int, priceCts int64) error {
	if strings.TrimSpace(sku) == "" {
		return errors.New("SKU vide")
	}
	if qty < 0 {
		return fmt.Errorf("quantité négative pour %s : %d", sku, qty)
	}
	if priceCts < 0 {
		return fmt.Errorf("prix négatif pour %s : %d", sku, priceCts)
	}
	if it, ok := inv.items[sku]; ok {
		it.quantity += qty
		it.Name, it.priceCts = name, priceCts
		return nil
	}
	inv.items[sku] = &Item{SKU: sku, Name: name, quantity: qty, priceCts: priceCts}
	return nil
}

func (inv *Inventory) Remove(sku string, qty int) error {
	if qty <= 0 {
		return fmt.Errorf("quantité invalide : %d", qty)
	}
	it, ok := inv.items[sku]
	if !ok {
		return fmt.Errorf("retrait sur %q : %w", sku, ErrUnknownSKU)
	}
	if qty > it.quantity {
		return fmt.Errorf("retrait de %d sur %d disponibles pour %s", qty, it.quantity, sku)
	}
	it.quantity -= qty
	return nil
}

// Get retourne une COPIE. Retourner *Item laisserait l'appelant écrire
// directement dans l'inventaire — y compris une quantité négative — et
// l'invariant protégé par le champ non exporté s'effondrerait.
func (inv *Inventory) Get(sku string) (Item, bool) {
	it, ok := inv.items[sku]
	if !ok {
		return Item{}, false
	}
	return *it, true
}

func (inv *Inventory) TotalValue() int64 {
	var total int64
	for _, it := range inv.items {
		total += int64(it.quantity) * it.priceCts
	}
	return total
}

// LowStock est trié par SKU : l'ordre d'une map est aléatoire (leçon 7).
func (inv *Inventory) LowStock(threshold int) []Item {
	var out []Item
	for _, sku := range slices.Sorted(maps.Keys(inv.items)) {
		if it := inv.items[sku]; it.quantity < threshold {
			out = append(out, *it)
		}
	}
	return out
}
```

**Le cœur de l'exercice est la contrainte 3.** Retourner `*Item` économiserait une copie de
48 octets ; retourner une valeur préserve l'invariant. Ici la copie est dérisoire et
l'invariant est essentiel : **la sécurité l'emporte**. Le raisonnement inverse (retourner un
pointeur) ne se justifierait que sur des objets volumineux, et il faudrait alors soit une
copie profonde, soit une interface en lecture seule.

C'est exactement le type d'arbitrage évalué en revue de code : la bonne réponse n'est pas
« pointeur » ou « valeur », c'est **savoir formuler le compromis**.

## Défi

**a) Buffer circulaire**

```go
package main

import (
	"errors"
	"fmt"
)

// Ring est un tampon circulaire de taille fixe.
// Quand il est plein, chaque Push écrase l'élément le plus ancien.
type Ring struct {
	buf   []int
	start int // indice du plus ancien élément
	count int // nombre d'éléments présents (0 ≤ count ≤ len(buf))
}

func NewRing(size int) (*Ring, error) {
	if size <= 0 {
		return nil, fmt.Errorf("taille invalide : %d", size)
	}
	return &Ring{buf: make([]int, size)}, nil // allocation UNIQUE
}

func (r *Ring) Push(v int) {
	end := (r.start + r.count) % len(r.buf)
	r.buf[end] = v
	if r.count < len(r.buf) {
		r.count++
		return
	}
	// plein : le plus ancien vient d'être écrasé, on avance la fenêtre
	r.start = (r.start + 1) % len(r.buf)
}

func (r *Ring) Len() int { return r.count }

// Slice retourne une COPIE, du plus ancien au plus récent.
// Retourner r.buf directement exposerait l'état interne : l'appelant pourrait
// écrire dedans et casser l'invariant, ou lire des cases non encore écrites.
func (r *Ring) Slice() []int {
	out := make([]int, 0, r.count)
	for i := range r.count {
		out = append(out, r.buf[(r.start+i)%len(r.buf)])
	}
	return out
}
```

**Les trois points qui font la différence :**

1. **`start` + `count`**, pas `start` + `end`. Avec deux indices seulement, « vide » et
   « plein » donnent la même configuration et deviennent indiscernables. C'est le piège
   classique du tampon circulaire.
2. **Le modulo** fait tout le travail d'enroulement. Pas de `if` sur les bords.
3. **`Slice()` copie.** Retourner une vue interne serait plus rapide et détruirait
   l'encapsulation : c'est le même arbitrage que dans le projet 1.

Cas limites à vérifier : `size == 1` (chaque `Push` écrase), exactement plein, deux tours
complets, `Slice()` sur un buffer vide (retourne un slice vide, pas nil — les deux sont
acceptables si documenté).

**b) Liste chaînée**

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

**Détection de cycle, en O(1) mémoire**

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

**c) Cache LRU**

Une map seule ne suffit pas : elle donne l'accès en O(1) mais ne dit pas quel élément est le
plus ancien. Il faut y ajouter une structure qui maintient **l'ordre d'usage** et permet de
déplacer un élément en tête en O(1) — une **liste doublement chaînée**. La map stocke alors
`clé → pointeur vers le nœud`.

```go
type entry struct {
	key   string
	value int
	prev  *entry
	next  *entry
}

type LRU struct {
	capacity int
	items    map[string]*entry
	head     *entry // le plus récemment utilisé
	tail     *entry // le plus ancien : la victime de l'éviction
}

func (c *LRU) Get(key string) (int, bool) {
	e, ok := c.items[key]
	if !ok {
		return 0, false
	}
	c.moveToFront(e) // un Get COMPTE comme un usage
	return e.value, true
}

func (c *LRU) Put(key string, value int) {
	if e, ok := c.items[key]; ok {
		e.value = value
		c.moveToFront(e)
		return
	}
	if len(c.items) == c.capacity {
		delete(c.items, c.tail.key) // évincer AVANT d'insérer
		c.remove(c.tail)
	}
	e := &entry{key: key, value: value}
	c.items[key] = e
	c.pushFront(e)
}
```

Les deux erreurs classiques : oublier que `Get` rafraîchit l'ordre, et oublier de retirer la
clé de la **map** en même temps que le nœud de la liste (fuite mémoire silencieuse : la map
grossit indéfiniment).

`container/list` de la stdlib fait le travail de la liste, mais l'écrire à la main une fois
apprend beaucoup plus. Version production : y ajouter un `sync.Mutex` (niveau 4) et des
métriques de taux de succès (niveau 9).

## Réponses du quiz

1. Le récepteur valeur reçoit une **copie** (les modifications sont invisibles) ; le récepteur
   pointeur reçoit l'adresse et peut modifier l'original.
2. Rien de visible : la copie est modifiée puis jetée. **Aucune erreur de compilation** — c'est
   ce qui en fait le bug silencieux le plus fréquent du niveau.
3. `T` n'a que les méthodes à récepteur **valeur** ; `*T` a **les deux**.
4. Non sur `int` ni sur `time.Time` : ce sont des types non locaux. Oui sur
   `type MyInt int` déclaré dans le package courant.
5. Parce que l'appel implique de prendre l'adresse du récepteur, et que les entrées de map ne
   sont **pas adressables** — le runtime peut les déplacer lors d'un redimensionnement.
6. **Non.** Elle ne panique que si elle **déréférence** le récepteur. On peut donc écrire
   délibérément des méthodes qui tolèrent `nil`, comme `func (l *List) Len() int` qui retourne
   0 sur une liste nil.
7. `fmt` teste à l'exécution si la valeur satisfait `fmt.Stringer` et appelle `String()`.
8. Une **récursion infinie** : `%v` rappelle `String()`. Résultat : `fatal error: stack overflow`.
9. Seulement si la construction exige une validation, une initialisation (map, channel,
   slice préalloué) ou une ressource. On l'appelle `New` tout court quand le package ne
   construit qu'un seul type — d'où `list.New()`, `errors.New()`, `bytes.NewBuffer()`.
10. Parce que `GetName()` est redondant : le nom du champ dit déjà ce qu'on obtient. Un
    accesseur n'est justifié que s'il **valide**, **calcule** ou **protège** — pas s'il se
    contente d'exposer un champ, auquel cas autant exporter le champ.
