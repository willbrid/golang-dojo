# Leçon 3 — Exercices

Code dans `mes-solutions/niveau-02/lecon-03/`.

## Exercices faciles

**E1 — Le récepteur qui ne modifie rien.**
Écrire un type `Counter` avec `IncValue()` (récepteur valeur) et `IncPtr()` (récepteur
pointeur), toutes deux incrémentant un champ. Les appeler dix fois chacune et afficher le
résultat. Expliquer l'écart en deux phrases. Le compilateur a-t-il protesté ?

**E2 — Ensemble de méthodes.**
Créer un type `T` avec une méthode à récepteur valeur et une à récepteur pointeur. Tenter
d'appeler les deux sur une variable `T`, sur un `*T`, puis sur le résultat d'une fonction
retournant `T` (donc non adressable). Prédire, vérifier, expliquer chaque refus.

**E3 — Méthodes sur un type non-struct.**
Créer `type Temperature float64`, `type Tags []string` et `type Counters map[string]int`,
chacun avec deux méthodes utiles. Puis tenter d'ajouter une méthode à `float64` directement :
relever le message d'erreur exact.

**E4 — `String()` et récursion.**
Créer `type Duration int` (des secondes) avec un `String()` affichant `2h15m30s`. Vérifier
que `fmt.Println` l'utilise. Puis écrire volontairement `fmt.Sprintf("%v", d)` **à
l'intérieur** de `String()` et observer. Relever le message. Expliquer.

**E5 — Adressabilité.**
```go
type Item struct{ N int }
func (i *Item) Inc() { i.N++ }
m := map[string]Item{"a": {N: 1}}
m["a"].N = 2
m["a"].Inc()
```
Prédire, compiler, relever les messages. Proposer **deux** corrections différentes et dire
laquelle préférer selon le contexte.

---

## Exercice intermédiaire — `inventory`

Un système d'inventaire dont les invariants sont réellement protégés.

```go
type Item struct {
	SKU      string
	Name     string
	quantity int     // non exporté : jamais négatif
	priceCts int64   // centimes
}

type Inventory struct {
	items map[string]*Item
}

func NewInventory() *Inventory
func (inv *Inventory) Add(sku, name string, qty int, priceCts int64) error
func (inv *Inventory) Remove(sku string, qty int) error
func (inv *Inventory) Get(sku string) (Item, bool)     // note : une VALEUR
func (inv *Inventory) TotalValue() int64
func (inv *Inventory) LowStock(threshold int) []Item   // trié par SKU
func (it Item) String() string
```

**Contraintes :**
1. La quantité ne peut **jamais** devenir négative — garanti par le type, pas par la discipline de l'appelant.
2. `Add` sur un SKU existant : cumuler la quantité ? écraser le prix ? **Répondre par écrit avant de coder**, puis implémenter la version défendue.
3. `Get` retourne une **copie**. Expliquer en commentaire quel bug cela évite, et ce que ça coûte.
4. `LowStock` a un ordre **déterministe** (rappel : niveau 1, leçon 7).
5. Erreurs distinctes et contextualisées : SKU vide, quantité négative, prix négatif, SKU inconnu, retrait supérieur au stock.
6. Une sentinelle `ErrUnknownSKU` testable avec `errors.Is`.
7. Cohérence des récepteurs sur les deux types — **justifier le choix en une ligne de commentaire pour chacun**.
8. Aucune panique possible, même sur un `Inventory` construit sans `NewInventory`.

*La contrainte 8 est plus subtile qu'elle n'en a l'air : `var inv Inventory` a une map nil,
et écrire dedans panique. Comment y remédier ? Deux stratégies existent — l'initialisation
paresseuse et le refus explicite. Choisir et défendre.*

---

## Défi

**a) Buffer circulaire.**
```go
type Ring struct { /* un seul slice, alloué UNE fois */ }
func NewRing(size int) (*Ring, error)
func (r *Ring) Push(v int)
func (r *Ring) Len() int
func (r *Ring) Slice() []int   // du plus ancien au plus récent, COPIE
```
**Contraintes :** aucune réallocation après `NewRing` ; `Slice()` ne doit pas exposer l'état
interne ; correct sur `size == 1`, buffer vide, exactement plein, deux tours complets.
*(Le piège : avec seulement deux indices `start` et `end`, « vide » et « plein » deviennent
indiscernables. Comment lever l'ambiguïté ?)*

**b) Liste chaînée.**
```go
type Node struct { Value int; Next *Node }
type List struct { head *Node; size int }

func (l *List) PushFront(v int)
func (l *List) PushBack(v int)
func (l *List) PopFront() (int, bool)
func (l *List) Remove(v int) bool
func (l *List) Reverse()               // EN PLACE, sans allouer de nœud
func (l *List) Len() int
func (l *List) Slice() []int
```
**Contraintes :** `var l List` doit être immédiatement utilisable ; aucune méthode ne panique
sur liste vide ; `Remove` correct sur la tête, la queue, l'élément unique et l'élément absent
— *c'est là que 90 % des implémentations échouent* ; après `PopFront`, le nœud retiré ne doit
plus référencer la suite.

Ajouter ensuite `HasCycle(head *Node) bool` en **O(1) mémoire** (interdiction d'une map de
nœuds visités).

**c) Cache LRU.**
```go
type LRU struct { /* … */ }
func NewLRU(capacity int) (*LRU, error)
func (c *LRU) Get(key string) (int, bool)
func (c *LRU) Put(key string, value int)
func (c *LRU) Len() int
```
**Contraintes :** `Get` et `Put` en **O(1) amorti** — une map seule ne suffit pas, il faut
une seconde structure pour l'ordre d'usage ; un `Get` compte comme un usage ; `Put` d'une clé
existante met à jour **et** rafraîchit ; capacité nulle ou négative refusée.
Écrire dans `main` un scénario prouvant l'éviction correcte sur au moins huit opérations.

*Le LRU est l'exercice d'entretien le plus demandé au monde. Le faire une fois sérieusement,
sans regarder de solution, vaut dix exercices faciles.*

---

## Quiz

1. Quelle est la différence entre `func (u User) F()` et `func (u *User) F()` ?
2. Que se passe-t-il si une méthode à récepteur valeur modifie un champ ?
3. Qu'est-ce que l'ensemble de méthodes de `T` ? Et de `*T` ?
4. Peut-on définir une méthode sur `int` ? Sur `type MyInt int` ? Sur `time.Time` ?
5. Pourquoi `m["k"].Method()` échoue-t-il quand `Method` a un récepteur pointeur ?
6. Une méthode à récepteur pointeur appelée sur un pointeur `nil` panique-t-elle toujours ?
7. Que fait `fmt.Println(x)` si `x` a une méthode `String() string` ?
8. Quel piège guette `String()` si on y écrit `%v` sur le récepteur ?
9. Quand faut-il écrire un constructeur `NewXxx` ? Quand l'appelle-t-on simplement `New` ?
10. Pourquoi Go n'utilise-t-il pas le préfixe `Get` ? Quand un accesseur est-il justifié ?
