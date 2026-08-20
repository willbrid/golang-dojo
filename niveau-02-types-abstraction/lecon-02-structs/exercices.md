# Leçon 2 — Exercices

Code dans `mes-solutions/niveau-02/lecon-02/`.

## Exercices faciles

**E1 — Zéro-valeur utile.**
Déclarer une struct `Config` à cinq champs de types variés (dont un slice et une map).
Afficher `var c Config` avec `%+v`. Lesquels de ses champs sont immédiatement utilisables,
lesquels feraient paniquer ? Corriger la conception pour que la zéro-valeur soit sûre.

**E2 — Copie superficielle.**
Créer une struct contenant un `int`, un `[]string`, une `map[string]int` et un `*int`.
L'affecter à une seconde variable, modifier **chacun** des quatre champs sur la copie, et
observer lesquels affectent l'original. **Prédire avant d'exécuter.**

**E3 — Comparabilité.**
Créer trois types : l'un avec uniquement des `int` et `string`, l'un avec un `[]string`,
l'un avec un `[3]string`. Tenter `==` sur chacun. Prédire lesquels compilent, vérifier,
expliquer. Lesquels peuvent servir de clé de map ?

**E4 — Littéral positionnel.**
Écrire une struct à trois champs, l'instancier en positionnel, puis **ajouter un champ au
milieu** de la déclaration du type. Que dit le compilateur ? Puis refaire l'expérience avec
un littéral nommé. Conclure en une phrase.

**E5 — Balises.**
Ajouter des balises `json` à une struct, dont une avec `omitempty` et une avec `-`.
Introduire volontairement une faute (`jsom:"x"`), compiler — puis lancer `go vet ./...`.
Que détecte-t-il, que ne détecte-t-il pas ?

---

## Exercice intermédiaire — `geometry`

Modéliser des figures géométriques et calculer leurs propriétés.

```go
type Point struct{ X, Y float64 }
type Segment struct{ A, B Point }
type Polygon struct{ Vertices []Point }
type Circle struct{ Center Point; Radius float64 }
```

**Fonctions à écrire** (des fonctions, pas encore des méthodes — elles arrivent leçon 3) :
```go
func Distance(a, b Point) float64
func SegmentLength(s Segment) float64
func PolygonPerimeter(p Polygon) (float64, error)
func PolygonArea(p Polygon) (float64, error)      // formule du lacet
func BoundingBox(p Polygon) (min, max Point, err error)
func ClonePolygon(p Polygon) Polygon
func PolygonEqual(a, b Polygon) bool
```

**Contraintes :**
1. Un polygone de moins de 3 sommets est une erreur pour `Area` et `Perimeter`.
2. `ClonePolygon` doit produire une copie **réellement indépendante**. Le prouver dans `main`.
3. `PolygonEqual` doit être écrite à la main — `==` ne compile pas sur `Polygon`. Décider et **documenter** : deux polygones décrivant la même forme mais commençant par un sommet différent sont-ils égaux ?
4. `BoundingBox` retourne des résultats **nommés**, sinon `(Point, Point, error)` est illisible.
5. Aucune fonction ne modifie son argument ; aucune n'affiche quoi que ce soit.
6. Comparer des flottants avec une tolérance, jamais avec `==`. Définir un `const epsilon` et **justifier sa valeur**.
7. `main` affiche un tableau aligné pour trois polygones de test, dont un triangle et un carré.

*La formule du lacet (« shoelace ») donne une aire **signée** : négative si les sommets sont
en sens horaire. Est-ce un bug ou une information utile ? Décider, documenter.*

---

## Défi

**a) Copie profonde générale.**
Écrire `DeepClone` pour une struct imbriquée à trois niveaux, dont un niveau contient un
`map[string][]Point`. Puis répondre par écrit : combien de lignes faudrait-il maintenir si le
type gagne un champ tous les six mois ? Quelles alternatives existent ?
*(Chercher : sérialisation/désérialisation, `reflect`, génération de code. Comparer les trois
sur la performance, la sécurité et la maintenance. La réponse honnête n'est pas la même
selon le contexte.)*

**b) Alignement mémoire.**
Écrire trois versions d'une même struct à sept champs (`bool`, `int64`, `bool`, `int32`,
`string`, `bool`, `float64`), dans trois ordres différents. Afficher `unsafe.Sizeof` de
chacune. **Prédire l'écart avant de mesurer.**
Puis allouer un slice d'un million d'exemplaires de la meilleure et de la pire, et comparer
`runtime.ReadMemStats`. À partir de quel nombre d'instances l'optimisation vaut-elle la peine
de rendre le code moins lisible ?

**c) Conception.**
On modélise un utilisateur avec : identité, coordonnées, préférences, historique de connexion
et droits. Comparer par écrit deux conceptions — une struct plate de vingt champs, contre une
struct de cinq champs dont quatre sont des structs imbriquées.
Critères : lisibilité, copie, comparabilité, sérialisation JSON, évolutivité, coût mémoire.
Y a-t-il un contexte où la struct plate gagne ?

---

## Quiz

1. Quelles sont les quatre façons d'instancier une struct ?
2. Pourquoi le littéral positionnel est-il déconseillé ? Donner deux raisons.
3. Que copie exactement `b := a` quand `a` est une struct contenant un slice ?
4. Quand une struct est-elle comparable avec `==` ?
5. Quelle est la conséquence de la non-comparabilité sur l'usage comme clé de map ?
6. Quelle différence entre un champ `Billing Address` et un champ `Billing *Address` ?
7. Qu'est-ce qu'une balise de champ ? Qui la lit, et quand ?
8. Une faute de frappe dans une balise est-elle détectée à la compilation ?
9. Pourquoi `reflect.DeepEqual` est-il déconseillé hors des tests ?
10. Pourquoi l'ordre des champs peut-il changer la taille d'une struct ?
