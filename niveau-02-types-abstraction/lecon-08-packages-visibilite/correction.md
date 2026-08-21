# Leçon 8 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Multi-fichiers

```
mathx/
├── doc.go      // Package mathx fournit des opérations numériques élémentaires.
│               package mathx
├── basic.go    package mathx  — func Abs, func clamp (non exportée)
└── stats.go    package mathx  — func Mean, qui appelle clamp SANS import
```

`stats.go` appelle `clamp` sans aucune ligne d'import : les fichiers d'un même répertoire
partagent le même espace de noms. Le découpage en fichiers est **purement organisationnel** —
le compilateur les traite comme un seul texte.

Conséquence à retenir : renommer un fichier, en scinder un ou en fusionner deux ne change
strictement rien au programme ni à son API.

## E2 — Visibilité

```
u.privateField  →  u.privateField undefined (type User has no field or method privateField,
                   but does have unexported field privateField)
mathx.clamp     →  undefined: mathx.clamp
u.doStuff()     →  u.doStuff undefined (type User has no field or method doStuff,
                   but does have unexported method doStuff)
```

Le compilateur est explicite : « **but does have unexported field** ». Il sait que le champ
existe et refuse l'accès. C'est plus utile qu'un simple « undefined », qui laisserait croire à
une faute de frappe.

Un type exporté, lui, est accessible normalement — l'exportation du type et celle de ses
champs sont **indépendantes**. `store.User` peut être public avec la moitié de ses champs
privés.

## E3 — Cycle d'import

```
package exemple.com/cycle/a
	imports exemple.com/cycle/b
	imports exemple.com/cycle/a: import cycle not allowed
```

Le message donne la **chaîne complète**, ce qui permet de localiser le cycle même sur cinq
niveaux.

**Les trois corrections :**

```
1. Extraire le partagé            2. Inverser avec une interface     3. Fusionner
   a → c ← b                          a définit UserGetter               ab
                                      b l'implémente sans importer a
```

**Pour deux packages fortement couplés, la 3 est souvent la bonne** — et c'est celle à laquelle
on pense le moins. Un cycle signifie que les deux moitiés se connaissent mutuellement en
profondeur ; c'est fréquemment le signe qu'on a découpé un concept unique en deux répertoires
sans raison.

La 1 est la bonne réponse quand le cycle révèle un **troisième concept** non nommé — souvent
les types de données partagés. La 2 est la bonne réponse quand la dépendance est réellement
unidirectionnelle sur le plan métier, mais que l'implémentation l'a inversée.

## E4 — `internal/`

```
package exemple.com/autre
	imports monprojet/internal/secret: use of internal package
	monprojet/internal/secret not allowed
```

**La frontière est le parent d'`internal/`.** Un package sous `a/b/internal/c` est importable
par tout ce qui vit sous `a/b/…`, et par rien d'autre. Le test à faire soi-même :

```
monprojet/internal/secret        ← importable depuis tout monprojet/
monprojet/api/internal/detail    ← importable depuis monprojet/api/ SEULEMENT,
                                   pas depuis monprojet/cmd/
```

Cette granularité est peu connue et très utile : elle permet de créer des frontières
**internes** au projet, pas seulement une frontière avec l'extérieur.

## E5 — `init()` et import anonyme

L'ordre d'exécution, garanti par la spécification :

1. les **variables de package**, dans l'ordre de leurs dépendances (pas l'ordre d'écriture) ;
2. les fonctions `init()`, dans l'ordre des fichiers **tel que présenté au compilateur** —
   c'est-à-dire l'ordre alphabétique avec l'outil `go`, mais **la spécification ne le garantit
   pas** ;
3. puis, seulement, `main()`.

Et avant tout cela : les packages importés sont entièrement initialisés d'abord, en profondeur.

```
init de a.go
init de b.go
main
```

Avec l'import anonyme :
```go
import _ "exemple.com/registre" // enregistre le pilote « csv » — effet de bord VOULU
```
Le package est chargé, ses `init()` s'exécutent, aucun de ses identifiants n'est accessible.
C'est exactement ainsi que fonctionnent les pilotes SQL (`_ "github.com/lib/pq"`) et les
formats d'image (`_ "image/png"`).

**Ne jamais compter sur l'ordre entre fichiers.** Si deux `init()` doivent s'exécuter dans un
ordre précis, c'est qu'il en faut un seul, ou une initialisation explicite.

## Exercice intermédiaire — `taskman`

Un découpage défendable :

```
taskman/
├── go.mod
├── main.go                     ~50 lignes : analyse des arguments, câblage, sortie
└── internal/
    ├── task/                   LE DOMAINE
    │   ├── task.go             le type Task et ses règles
    │   ├── list.go             la collection et ses opérations
    │   └── filter.go           filtrage par étiquette et statut
    ├── store/                  la persistance
    │   └── memory.go
    └── render/                 l'affichage
        ├── table.go
        └── text.go
```

**Graphe de dépendances :**
```
main ──→ task
  │        ▲
  ├──→ store ┘   (store dépend de task : il stocke des Task)
  └──→ render    (render ne dépend PAS de task — voir ci-dessous)
```

**Contrainte 4 — comment `render` évite de dépendre de `task`.** En définissant l'interface
dont il a besoin, **chez lui** :

```go
// package render
type Row interface {
	Columns() []string
}

func Table(headers []string, rows []Row) string { … }
```

`task.Task` obtient une méthode `Columns()` et satisfait `render.Row` sans que `task`
n'importe `render`, ni l'inverse. La dépendance est **inversée** par l'interface. Si
l'inversion gêne (le domaine ne devrait pas savoir qu'il existe des colonnes), l'adaptation se
fait dans `main`, avec un type `taskRow` local — c'est ce que montre l'exemple du cours.

**Contrainte 5 — les phrases.** Le test « une phrase sans *et* » est étonnamment discriminant :

- ✔ « Package task modélise une tâche et ses règles métier. » — le « et » porte sur un seul
  concept, c'est acceptable.
- ✘ « Package util contient des fonctions de formatage et des helpers de validation. » — deux
  concepts, deux packages.

**Contrainte 6 — exporter le minimum.** L'exercice consiste à parcourir chaque identifiant
exporté et à demander « qui, hors de ce package, en a besoin ? ». En pratique, un premier jet
exporte deux à trois fois trop. Les candidats habituels au retrait : les constructeurs
d'objets internes, les types intermédiaires, les fonctions de validation.

Il est **facile d'exporter plus tard, impossible de retirer** sans casser les appelants — d'où
l'asymétrie du choix par défaut.

## Défi

**a) Le cycle `user` / `order` / `notification`**

Le cycle naît de la contrainte « un utilisateur doit pouvoir lister ses commandes » : `user`
devrait importer `order`, qui importe déjà `user`.

**Solution 1 — extraire les types partagés :**
```
domain/     (User, Order, Notification : les types, sans logique)
   ▲  ▲  ▲
user  order  notification      (la logique, chacun important domain)
```
*Avantage :* simple, immédiat. *Inconvénient :* `domain` devient un package anémique de
structures sans comportement — le fameux *anemic domain model*. Et il grossit à chaque
nouveau concept.

**Solution 2 — inverser avec des interfaces :**
```go
// package user — définit ce dont IL a besoin
type OrderLister interface {
	ByUser(userID string) ([]OrderSummary, error)
}

func (s *Service) Profile(id string, orders OrderLister) (Profile, error) { … }
```
`order` implémente `OrderLister` sans le savoir et sans importer `user`. La dépendance devient
`order → user` uniquement, et le cycle disparaît.
*Avantage :* chaque package garde sa logique et son vocabulaire. *Inconvénient :* un type
`OrderSummary` doit vivre quelque part — souvent chez `user`, ce qui étonne au premier abord.

| | Solution 1 | Solution 2 |
|---|---|---|
| Nombre de packages | 4 | 3 |
| Taille des interfaces | aucune | petites, côté consommateur |
| Testabilité | moyenne (il faut de vrais objets) | excellente (faux triviaux) |
| Ajout d'une facturation | un type de plus dans `domain`, qui enfle | un package de plus, isolé |

**La solution 2 vieillit mieux**, et c'est celle que recommande le style Go. La solution 1
reste acceptable sur un petit projet, à condition de surveiller la croissance de `domain`.

**b) API publique minimale**

Sur le `catalog` du cours : `Book` doit rester exporté (c'est le vocabulaire échangé), mais
`Catalog` pourrait ne plus l'être, avec `New()` retournant une **interface** :

```go
package catalog

type Store interface {
	Add(b Book, copies int) error
	Borrow(isbn string) error
	All() []Book
}

func New() Store { return &catalog{…} }   // type concret NON exporté
```

**Faut-il le faire ? Généralement non.** La règle « accept interfaces, return structs » existe
parce que retourner une interface :
- prive l'appelant des méthodes ajoutées ultérieurement au type concret ;
- empêche l'appelant de définir **sa propre** interface, plus petite, adaptée à son besoin ;
- ajoute une indirection à chaque appel, sans bénéfice.

**L'exception se justifie quand plusieurs implémentations sont prévues dès le départ et que
l'appelant ne doit jamais connaître la concrète.** C'est le cas de `database/sql` : `sql.DB`
est une struct, mais `driver.Conn` est une interface, parce que le pilote est
interchangeable par conception. Autre cas légitime : cacher un type dont l'API est instable.

En résumé : retourner une struct par défaut, une interface seulement quand
l'interchangeabilité fait partie du contrat.

**c) Le coût de `internal/`**

Six mois plus tard, l'autre équipe ne peut pas importer. Ses options :

| Option | Coût |
|---|---|
| Copier-coller le code | duplication, divergence garantie à moyen terme |
| Sortir le composant d'`internal/` | il devient une **API publique** : compatibilité à maintenir, versionnement, documentation |
| Extraire un module séparé | le plus propre, mais coût réel : dépôt, versionnement, CI, revue |
| Demander à l'équipe 1 d'exposer un service | change la nature de la dépendance (réseau au lieu de compilation) |

**Ce n'est pas un défaut d'`internal/`, c'est sa raison d'être** : il rend le coût de la
réutilisation **visible et décidé**, au lieu de laisser une dépendance s'installer par
accident et de découvrir trois ans plus tard qu'on ne peut plus rien changer.

**Ce que font les grands projets :** Kubernetes, Docker et la bibliothèque standard elle-même
(`internal/` existe dans `$GOROOT/src`) utilisent massivement `internal/`. Le répertoire
`pkg/`, lui, n'apporte **rien de vérifiable** — c'est une convention de nommage sans effet sur
le compilateur. Russ Cox et Dave Cheney ont tous deux exprimé publiquement qu'il ajoute un
niveau de chemin sans bénéfice, et que `golang-standards/project-layout` ne représente pas une
recommandation de l'équipe Go. La position la plus défendable : `internal/` oui, `pkg/`
seulement si l'on a une raison qu'on sait formuler.

## Réponses du quiz

1. **Le répertoire.** Tous les fichiers `.go` d'un même répertoire appartiennent au même
   package.
2. **Non.** Ils partagent le même espace de noms et se voient sans aucun import.
3. En le faisant commencer par une **minuscule**. Il n'existe ni `private` ni `public`.
4. **Au package.** Deux types du même package accèdent mutuellement à leurs champs privés —
   c'est pourquoi un package doit rester cohérent.
5. **Erreur de compilation** : `import cycle not allowed`, avec la chaîne complète.
6. Extraire le code partagé dans un troisième package ; inverser une dépendance avec une
   interface définie côté consommateur ; fusionner les deux packages s'ils forment un seul
   concept.
7. Uniquement le code dont le chemin partage le **parent d'`internal`**. Vérifié par le
   compilateur.
8. À charger un package pour ses seuls `init()`. Cas réels : pilotes SQL
   (`_ "github.com/lib/pq"`), formats d'image (`_ "image/png"`).
9. Parce qu'il injecte les identifiants dans l'espace de noms courant : on ne sait plus d'où
   vient une fonction, et les outils d'analyse s'y perdent.
10. Essentiellement pour l'**enregistrement dans un registre**. Partout ailleurs : une
    initialisation explicite depuis `main`, ou `sync.OnceValue` pour du paresseux — car
    `init()` ne peut pas retourner d'erreur, s'exécute même si la fonctionnalité n'est jamais
    utilisée, et ne peut pas être désactivée en test.
11. Parce qu'il ne dit rien de ce qu'il contient : il n'a pas de raison d'exister formulable,
    donc aucune limite à sa croissance. Un package `util` finit toujours par contenir tout ce
    dont personne n'a su quoi faire.
