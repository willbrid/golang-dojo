# Leçon 10 — Exercices

Code dans `mes-solutions/niveau-01/lecon-10/`.

## Exercices faciles

**E1 — Le récepteur qui ne modifie rien.**
Écrire un type `Counter` avec deux méthodes `IncValue()` (récepteur valeur) et `IncPtr()`
(récepteur pointeur), toutes deux incrémentant un champ. Les appeler dix fois chacune et
afficher le résultat. Expliquer l'écart en deux phrases.

**E2 — Comparabilité.**
Créer trois types : l'un avec uniquement des `int` et `string`, l'un avec un `[]string`,
l'un avec un `[3]string`. Tenter `==` sur chacun. Prédire lesquels compilent, vérifier,
expliquer. Lequel peut servir de clé de map ?

**E3 — `String()`.**
Créer un type `Duration` basé sur `int` (des secondes) et lui donner une méthode `String()`
affichant `2h15m30s`. Vérifier que `fmt.Println` l'utilise. Puis écrire volontairement
`fmt.Sprintf("%v", d)` **à l'intérieur** de `String()` et observer ce qui se passe. Expliquer.

**E4 — Adressabilité.**
```go
type Item struct{ N int }
m := map[string]Item{"a": {N: 1}}
m["a"].N = 2                 // ?
func (i *Item) Inc() { i.N++ }
m["a"].Inc()                 // ?
```
Prédire, compiler, relever les messages. Proposer **deux** corrections différentes et dire
laquelle est préférable et pourquoi.

**E5 — Embedding.**
Créer `Base` avec une méthode `Describe()`, et `Derived` qui l'embarque et définit son
propre `Describe()`. Appeler les deux versions depuis `Derived` (indice : le champ anonyme
porte le nom du type). Puis tenter d'affecter un `Derived` à une variable de type `Base`.
Que dit le compilateur ? Conclusion sur l'héritage en Go ?

---

## Exercice intermédiaire — `inventory`

Un système d'inventaire, avec des invariants réellement protégés.

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
```

**API à concevoir et implémenter :**
```go
func NewInventory() *Inventory
func (inv *Inventory) Add(sku, name string, qty int, priceCts int64) error
func (inv *Inventory) Remove(sku string, qty int) error
func (inv *Inventory) Get(sku string) (Item, bool)     // note : une VALEUR, pas un pointeur
func (inv *Inventory) TotalValue() int64
func (inv *Inventory) LowStock(threshold int) []Item   // trié par SKU
func (it Item) String() string
```

**Contraintes :**
1. La quantité ne peut **jamais** devenir négative — garanti par le type, pas par la discipline de l'appelant.
2. `Add` sur un SKU existant **augmente** la quantité ; le prix est mis à jour ; le nom aussi. Est-ce un bon contrat d'API ? **Répondre par écrit, puis implémenter la version défendue.**
3. `Get` retourne une **copie** : l'appelant ne doit pas pouvoir modifier l'inventaire à travers le résultat. Expliquer pourquoi c'est important, en commentaire.
4. `LowStock` a un ordre **déterministe** (rappel : leçon 7).
5. Erreurs : SKU vide, quantité négative, prix négatif, SKU inconnu, retrait supérieur au stock. Chaque cas a son message, avec du contexte.
6. Une sentinelle `ErrUnknownSKU` testable avec `errors.Is`.
7. Cohérence des récepteurs sur les deux types — et **justifier le choix** en une ligne de commentaire pour chacun.
8. Aucune panique possible, même sur un `Inventory` mal construit.

*Point de conception central : la contrainte 3. Retourner `*Item` serait plus efficace…
et détruirait l'encapsulation. Ce compromis est exactement ce qu'on évalue en revue de code.*

---

## Défi

**a) Matrice.**
```go
type Matrix struct { /* à concevoir */ }
func NewMatrix(rows, cols int) (*Matrix, error)
func (m *Matrix) At(i, j int) (float64, error)
func (m *Matrix) Set(i, j int, v float64) error
func (m *Matrix) Mul(other *Matrix) (*Matrix, error)
func (m *Matrix) Transpose() *Matrix
func (m *Matrix) String() string
```
**Contraintes :** stockage dans **un seul** `[]float64` (pas de `[][]float64` — expliquer
pourquoi en commentaire, la réponse concerne le cache CPU) ; toutes les erreurs de dimension
détectées ; `String()` aligne les colonnes proprement.

**b) Copie profonde.**
Ajouter `func (m *Matrix) Clone() *Matrix` et **prouver** par un test manuel que modifier
le clone ne touche pas l'original. Puis expliquer pourquoi une simple affectation
`m2 := *m` ne suffit pas.

**c) Question ouverte, à répondre par écrit (5-10 lignes).**
`Mul` retourne `(*Matrix, error)`. Trois alternatives existent :
`Mul(other, dst *Matrix) error` (l'appelant fournit la destination),
une méthode qui panique sur dimensions incompatibles,
un type `Matrix` avec des dimensions dans le type via des génériques.
Comparer ces quatre conceptions sur : ergonomie, allocations, testabilité, sécurité.
Laquelle choisirait la bibliothèque standard Go, à ton avis ?

---

## Quiz

1. Quelle est la différence entre `func (u User) F()` et `func (u *User) F()` ?
2. Que se passe-t-il si une méthode à récepteur valeur modifie un champ ?
3. Quand une struct est-elle comparable avec `==` ?
4. Peut-on définir une méthode sur `int` ? Sur `type MyInt int` ?
5. Pourquoi `m["k"].Method()` échoue-t-il quand `Method` a un récepteur pointeur ?
6. Que fait `fmt.Println(x)` si `x` a une méthode `String() string` ?
7. Quel piège guette `String()` si on y écrit `fmt.Sprintf("%v", récepteur)` ?
8. L'embedding est-il de l'héritage ? Justifier avec deux différences.
9. Quand faut-il écrire un constructeur `NewXxx` ?
10. Pourquoi Go n'utilise-t-il pas le préfixe `Get` sur les accesseurs ?
