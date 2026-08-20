# Leçon 10 — Corrigé

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

## E2 — Comparabilité

```go
type A struct{ X int; S string }      // comparable ✔ — clé de map possible
type B struct{ Tags []string }        // NON comparable ✘ — invalid map key type
type C struct{ Tags [3]string }       // comparable ✔ — un TABLEAU est comparable
```

Un tableau de taille fixe est comparable si son type d'élément l'est ; un slice ne l'est
jamais (il faudrait comparer le contenu pointé, ce que `==` ne fait pas). Seuls `A` et `C`
peuvent servir de clés de map.

## E3 — Stringer et récursion

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

## E4 — Adressabilité

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

## E5 — Embedding

```go
type Base struct{ Name string }
func (b Base) Describe() string { return "Base: " + b.Name }

type Derived struct {
	Base
	Extra string
}
func (d Derived) Describe() string { return "Derived: " + d.Extra }

d := Derived{Base{"x"}, "y"}
d.Describe()        // "Derived: y"  — la méthode du type extérieur masque celle du type embarqué
d.Base.Describe()   // "Base: x"     — la version embarquée reste accessible par son nom

var b Base = d      // ERREUR : cannot use d (Derived) as Base value
```

Le message du compilateur est la réponse : **l'embedding n'est pas de l'héritage**. Il n'y a
aucune relation de sous-typage entre `Derived` et `Base`. Le polymorphisme, en Go, passe
uniquement par les **interfaces** — si `Base` et `Derived` satisfont toutes deux une
interface `Describer`, elles sont interchangeables *à travers cette interface*, sans être
parentes.

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

## Défi — Matrix

```go
type Matrix struct {
	rows, cols int
	data       []float64 // stockage LINÉAIRE : m.data[i*cols+j]
}
```

**Pourquoi un seul `[]float64` et pas `[][]float64` ?** Un `[][]float64` est un slice de
slices : `n` allocations séparées, dispersées en mémoire, et deux indirections par accès.
Un stockage linéaire donne **une** allocation, une mémoire contiguë, et un parcours
séquentiel que le préchargeur du CPU anticipe. Sur une multiplication de matrices 1000×1000,
l'écart dépasse fréquemment un facteur 3. C'est le même raisonnement de localité qu'en
leçon 9.

```go
func (m *Matrix) At(i, j int) (float64, error) {
	if i < 0 || i >= m.rows || j < 0 || j >= m.cols {
		return 0, fmt.Errorf("indice (%d,%d) hors bornes (%d×%d)", i, j, m.rows, m.cols)
	}
	return m.data[i*m.cols+j], nil
}

func (m *Matrix) Mul(o *Matrix) (*Matrix, error) {
	if m.cols != o.rows {
		return nil, fmt.Errorf("dimensions incompatibles : %d×%d × %d×%d",
			m.rows, m.cols, o.rows, o.cols)
	}
	out := &Matrix{rows: m.rows, cols: o.cols, data: make([]float64, m.rows*o.cols)}
	for i := range m.rows {
		for k := range m.cols { // ordre i,k,j : meilleur pour le cache que i,j,k
			a := m.data[i*m.cols+k]
			for j := range o.cols {
				out.data[i*o.cols+j] += a * o.data[k*o.cols+j]
			}
		}
	}
	return out, nil
}

func (m *Matrix) Clone() *Matrix {
	return &Matrix{rows: m.rows, cols: m.cols, data: slices.Clone(m.data)}
}
```

**(b)** `m2 := *m` copie la struct : `rows`, `cols` et le **descripteur** de `data` — mais
pas le tableau sous-jacent. Les deux matrices partageraient donc les mêmes nombres, et
modifier l'une modifierait l'autre. C'est le piège de la leçon 6 appliqué aux structs :
une copie de struct est **superficielle**, elle s'arrête au premier slice, map ou pointeur.

**(c) Comparaison des quatre conceptions**

| Conception | Ergonomie | Allocations | Testabilité | Sécurité |
|---|---|---|---|---|
| `Mul(o) (*Matrix, error)` | excellente | 1 par appel | excellente | erreurs explicites |
| `Mul(o, dst) error` | lourde | 0 (dst réutilisable) | bonne | risque d'aliasing si `dst == m` |
| `Mul` qui panique | concise, chaînable | 1 | mauvaise (`recover` en test) | fragile |
| Dimensions dans le type (génériques) | rigide | 1 | excellente | vérifié à la **compilation** |

La bibliothèque standard Go choisirait la **première** : erreurs explicites, pas de panique,
API simple. C'est d'ailleurs ce que fait `gonum` pour son API de haut niveau, tout en
exposant une variante à destination fournie pour les boucles chaudes — les deux coexistent,
la simple par défaut et l'optimisée en option. La quatrième est séduisante mais Go ne permet
pas d'entiers comme paramètres de type : elle exigerait un type par dimension.

## Réponses du quiz

1. Le récepteur valeur reçoit une **copie** (modifications invisibles) ; le récepteur
   pointeur reçoit l'adresse et peut modifier l'original.
2. Rien de visible : la copie est modifiée puis jetée. Aucune erreur de compilation.
3. Quand **tous** ses champs sont comparables. Un slice, une map ou une fonction rend toute
   la struct non comparable.
4. Non sur `int` (type d'un autre paquet, le paquet `builtin`). Oui sur `type MyInt int`,
   déclaré dans notre paquet.
5. Parce que l'appel implique de prendre l'adresse du récepteur, et que les entrées de map
   ne sont **pas adressables**.
6. `fmt` détecte que `x` satisfait `fmt.Stringer` et appelle `String()`.
7. Une **récursion infinie** : `%v` rappelle `String()`. Résultat : `fatal error: stack overflow`.
8. Non. Deux différences : pas de sous-typage (`var b Base = derived` ne compile pas), et
   pas de dispatch virtuel (le code de `Base` appelle toujours la méthode de `Base`, même si
   `Derived` la redéfinit).
9. Seulement quand la construction exige quelque chose : validation, initialisation d'une
   map ou d'un channel, identifiant à générer. Si la zéro-valeur suffit, s'en passer.
10. Parce que `GetName()` est redondant : le nom du champ dit déjà ce qu'on obtient. La
    convention Go est `u.Name()` pour l'accesseur et `u.SetName(x)` pour le mutateur.
