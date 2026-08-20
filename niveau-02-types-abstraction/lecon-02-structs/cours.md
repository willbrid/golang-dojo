# Leçon 2 — Structs

## Objectifs

1. Modéliser des données avec des structs et connaître les quatre façons de les instancier.
2. Savoir quand une struct est **comparable**, et pourquoi cela conditionne son usage comme clé de map.
3. Comprendre ce que copier une struct copie réellement — et ce qu'elle ne copie pas.
4. Découvrir les structs anonymes, l'imbrication et les balises de champ.

## Explication

### Déclarer et instancier

```go
type User struct {
	ID      int
	Name    string
	Email   string
	private bool     // non exporté : invisible hors du package
}
```

Une struct est un **agrégat de champs nommés**, disposés de façon contiguë en mémoire. Ce
n'est pas une classe : pas d'héritage, pas de constructeur implicite, pas de méthodes dans
la déclaration (elles se déclarent à part, leçon 3).

Quatre façons de créer une valeur :

```go
u1 := User{}                              // zéro-valeur : tous les champs à zéro
u2 := User{ID: 1, Name: "Alice"}          // littéral NOMMÉ — la forme à utiliser
u3 := User{1, "Alice", "a@x.fr", false}   // positionnel — à éviter
u4 := &User{ID: 1}                        // pointeur vers une struct
var u5 User                               // zéro-valeur, comme u1
```

Le littéral **positionnel** est fragile : ajouter un champ au type casse tous les appels,
l'ordre est invisible à la lecture, et il oblige à renseigner **tous** les champs, y compris
les non exportés — donc il ne fonctionne même pas depuis un autre package. Il n'est
acceptable que pour de très petites structs stables (`image.Point{1, 2}`). Le littéral
**nommé** est la règle, et `go vet` peut signaler l'autre via l'analyse `composites`.

Les champs omis prennent leur zéro-valeur. On retrouve le principe de la leçon 2 du
niveau 1 : **concevoir ses structs pour que `var x T` soit directement utilisable**.
`var b strings.Builder`, `var mu sync.Mutex`, `var buf bytes.Buffer` fonctionnent sans
constructeur — c'est un critère de qualité d'API.

### Accès aux champs, et déréférencement automatique

```go
u.Name = "Bob"
p := &u
p.Name = "Carol"    // équivalent à (*p).Name — Go déréférence automatiquement
```

Go déréférence les pointeurs pour l'accès aux champs : on n'écrit jamais `(*p).Name`. Le
sucre syntaxique s'arrête là — sur un pointeur `nil`, l'accès panique.

### Copier une struct : une copie **superficielle**

```go
a := User{ID: 1, Name: "Alice"}
b := a          // COPIE de tous les champs
b.Name = "Bob"
fmt.Println(a.Name)   // "Alice" — a est intact
```

Une struct est une **valeur** : l'affecter, la passer à une fonction ou la retourner copie
tous ses champs. Mais la copie s'arrête au premier niveau :

```go
type Doc struct {
	Title string
	Tags  []string          // slice : descripteur copié, tableau PARTAGÉ
	Meta  map[string]string // map : référence partagée
}

d1 := Doc{Title: "a", Tags: []string{"x"}, Meta: map[string]string{"k": "v"}}
d2 := d1
d2.Title = "b"        // indépendant
d2.Tags[0] = "MODIFIÉ" // ← d1.Tags[0] change aussi !
d2.Meta["k"] = "autre" // ← d1.Meta change aussi !
```

C'est le piège d'aliasing du niveau 1 (leçons 6 et 7) remonté au niveau des structs.
**Toute struct contenant un slice, une map, un channel ou un pointeur n'a pas de copie
naturelle.** Si l'indépendance compte, il faut écrire une méthode `Clone()` explicite.

### Comparabilité

```go
type Point struct{ X, Y int }
a, b := Point{1, 2}, Point{1, 2}
fmt.Println(a == b)    // true — comparaison champ par champ

type Bad struct{ Tags []string }
c, d := Bad{}, Bad{}
fmt.Println(c == d)    // ERREUR de compilation : struct contenant un slice
```

Une struct est comparable si **tous** ses champs le sont. Slices, maps et fonctions ne le
sont pas et contaminent la struct entière. Conséquences directes :

- seule une struct comparable peut servir de **clé de map** (leçon 7 du niveau 1) ;
- seule une struct comparable peut être utilisée avec `==` dans un `switch`.

Pour comparer des structs contenant des slices : `reflect.DeepEqual` (lent, à réserver aux
tests) ou une méthode `Equal` écrite à la main — plus rapide, plus explicite, et elle permet
de décider ce que « égal » signifie pour le domaine.

### Structs imbriquées

```go
type Address struct {
	Street, City, Zip string
}

type Customer struct {
	Name    string
	Billing Address     // champ NOMMÉ de type struct
	Shipping Address
}

c := Customer{
	Name:    "Alice",
	Billing: Address{Street: "1 rue X", City: "Lyon"},
}
c.Billing.City = "Paris"
```

Une struct imbriquée est stockée **à l'intérieur** de la struct englobante, pas via un
pointeur : `Customer` occupe la place de tous ses champs. C'est différent d'un champ
`*Address`, qui ajouterait une indirection et permettrait `nil`.

Il existe une autre forme d'imbrication, le **champ anonyme** (`Address` sans nom de champ),
qui déclenche la *promotion* des champs et des méthodes : c'est l'**embedding**, sujet de la
leçon 5.

### Structs anonymes

```go
config := struct {
	Host string
	Port int
}{Host: "localhost", Port: 8080}
```

Utile pour une valeur ponctuelle qui ne mérite pas un type nommé. Deux usages réels :

- les **tables de tests** (niveau 5), où c'est la forme canonique ;
- la construction d'une réponse JSON ad hoc (niveau 4).

Partout ailleurs, un type nommé est préférable : il se documente, se réutilise et peut
porter des méthodes.

### Balises de champ (*struct tags*)

```go
type User struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email,omitempty"`
	Token string `json:"-"`               // jamais sérialisé
}
```

Une balise est une **chaîne littérale** accolée au champ. Le compilateur ne l'interprète
pas : elle est simplement conservée dans les métadonnées du type, et lue à l'exécution par
réflexion (niveau 9) — par `encoding/json`, `database/sql`, les validateurs, etc.

Deux conséquences pratiques :
- Une faute de frappe dans une balise (`jsom:"id"`) **compile parfaitement** et échoue
  silencieusement à l'exécution. `go vet` possède une analyse `structtag` qui détecte les
  balises malformées — une raison de plus de le lancer systématiquement.
- Les balises se lisent avec des **guillemets obliques** (`` ` ``), jamais des guillemets
  droits : la syntaxe `json:"id"` contient déjà des guillemets.

Les balises seront exploitées au niveau 4 (JSON) et expliquées en profondeur au niveau 9
(réflexion).

### Disposition mémoire et alignement

```go
type Bad struct {
	a bool   // 1 octet + 7 de bourrage
	b int64  // 8
	c bool   // 1 octet + 7 de bourrage
}            // 24 octets

type Good struct {
	b int64  // 8
	a bool   // 1
	c bool   // 1 + 6 de bourrage
}            // 16 octets
```

Les champs sont alignés sur leur taille naturelle, ce qui insère du **bourrage** (*padding*).
Réordonner les champs du plus grand au plus petit peut réduire nettement la taille.

À relativiser : cela n'a d'importance que pour des structs allouées en très grand nombre
(millions d'instances). Sur une struct de configuration, c'est du bruit. **Ne pas optimiser
sans mesurer** — `unsafe.Sizeof` donne la taille, et l'outil `fieldalignment` de
`golang.org/x/tools` signale les cas coûteux. On y reviendra au niveau 11.

## Exemple

```go
package main

import (
	"fmt"
	"slices"
	"maps"
)

// Address est imbriquée par valeur : elle fait partie de Customer.
type Address struct {
	Street string
	City   string
	Zip    string `json:"postal_code"`
}

// Customer contient un slice et une map : sa copie est SUPERFICIELLE.
type Customer struct {
	ID      int
	Name    string
	Billing Address
	Tags    []string
	Meta    map[string]string
}

// Clone effectue une copie PROFONDE. Sans elle, `c2 := c1` partagerait
// Tags et Meta, et l'indépendance apparente serait une illusion.
func (c Customer) Clone() Customer {
	out := c                       // copie superficielle : champs scalaires et Address
	out.Tags = slices.Clone(c.Tags)
	out.Meta = maps.Clone(c.Meta)  // maps.Clone : Go 1.21+
	return out
}

func main() {
	c1 := Customer{
		ID:      1,
		Name:    "Alice",
		Billing: Address{Street: "1 rue de la Paix", City: "Paris", Zip: "75002"},
		Tags:    []string{"vip"},
		Meta:    map[string]string{"source": "web"},
	}

	// Copie superficielle : le piège
	c2 := c1
	c2.Name = "Bob"          // indépendant
	c2.Billing.City = "Lyon" // indépendant : Address est imbriquée PAR VALEUR
	c2.Tags[0] = "spam"      // PARTAGÉ !
	fmt.Printf("c1 : %s %s tags=%v\n", c1.Name, c1.Billing.City, c1.Tags)

	// Copie profonde
	c3 := c1.Clone()
	c3.Tags[0] = "premium"
	c3.Meta["source"] = "mobile"
	fmt.Printf("c1 : tags=%v meta=%v\n", c1.Tags, c1.Meta)
	fmt.Printf("c3 : tags=%v meta=%v\n", c3.Tags, c3.Meta)

	// Comparabilité
	a1 := Address{City: "Paris"}
	a2 := Address{City: "Paris"}
	fmt.Println("adresses égales :", a1 == a2)
	// fmt.Println(c1 == c3)  // ← ne compilerait pas : Customer contient un slice

	// Une struct comparable peut être une clé de map
	seen := map[Address]int{a1: 1}
	seen[a2]++
	fmt.Println("clé partagée :", seen[a1]) // 2 : a1 et a2 sont la MÊME clé

	// Struct anonyme
	stats := struct {
		Total, Active int
	}{Total: 10, Active: 7}
	fmt.Printf("%+v\n", stats)
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `Billing Address` | Imbriquée **par valeur** : modifier `c2.Billing.City` ne touche pas `c1`. Un `*Address` aurait donné l'inverse. |
| `c2.Tags[0] = "spam"` | Le slice est **partagé**. C'est le piège central de la leçon. |
| `out := c` puis `slices.Clone` | Idiome de copie profonde : partir de la copie superficielle, puis remplacer explicitement chaque champ de référence. Ajouter un champ au type sans mettre `Clone` à jour est le bug classique. |
| `maps.Clone` | Go 1.21+. Avant, il fallait une boucle. |
| `map[Address]int` | Fonctionne parce qu'`Address` n'a que des champs comparables. |
| `seen[a2]++` donne 2 | Deux structs de même contenu sont la **même clé** : l'égalité est structurelle, pas par identité. |
| `%+v` | Affiche les noms de champs. Le réflexe de diagnostic n°1 sur une struct. |

## Erreurs fréquentes

1. **Littéral positionnel** : casse dès qu'un champ est ajouté, et impossible depuis un autre package si un champ n'est pas exporté.
2. **Croire qu'affecter une struct la copie entièrement** : slices, maps et pointeurs restent partagés.
3. **Oublier de mettre `Clone()` à jour** après avoir ajouté un champ de référence.
4. **Comparer avec `==` une struct contenant un slice** → ne compile pas.
5. **Utiliser `reflect.DeepEqual` en production** pour comparer : c'est lent et il compare des choses inattendues (deux slices `nil` et vide y sont différents).
6. **Exporter tous les champs par réflexe** : plus aucun invariant n'est protégeable.
7. **Faute de frappe dans une balise** : compile, échoue silencieusement.
8. **Struct de vingt champs** : elle cache presque toujours trois concepts distincts.
9. **Optimiser l'alignement des champs sans mesure** : illisible pour un gain nul dans 99 % des cas.

## Bonnes pratiques Go

- Littéraux **nommés**, toujours.
- Concevoir pour que la **zéro-valeur soit utilisable**.
- Champs non exportés dès qu'un invariant doit être protégé.
- Une struct, un concept. Si le nom contient « Data », « Info » ou « Manager », interroger le découpage.
- Documenter les unités et les invariants dans les commentaires de champ
  (`// balance en centimes, jamais négatif`).
- Écrire `Clone()` quand la struct contient des références et que l'indépendance compte —
  et le dire dans la documentation quand ce n'est **pas** le cas.
- Grouper les champs liés dans une struct imbriquée plutôt que d'aplatir vingt champs.
- Lancer `go vet` : il attrape les balises malformées et les littéraux positionnels fragiles.

## Ce que je dois retenir

- Littéral **nommé** ; les champs omis prennent leur zéro-valeur.
- Une struct est une **valeur** : la copier copie ses champs — mais la copie est
  **superficielle**, elle s'arrête au premier slice, map ou pointeur.
- Une struct est **comparable** si tous ses champs le sont ; c'est la condition pour être
  clé de map.
- Les structs imbriquées par valeur sont **contenues**, pas référencées.
- Les **balises** sont des chaînes lues par réflexion à l'exécution, invisibles au compilateur.
- L'ordre des champs influe sur la taille en mémoire — sujet du niveau 11, pas d'aujourd'hui.

➡️ [Exercices](exercices.md)
