# Leçon 4 — Exercices

Code dans `mes-solutions/niveau-02/lecon-04/`.

## Exercices faciles

**E1 — Satisfaction implicite.**
Définir `Shape` avec `Area()` et `Perimeter()`, puis `Circle`, `Rectangle` et `Triangle`.
Les mettre dans un `[]Shape` et afficher l'aire totale. **Aucun des trois types ne doit
mentionner `Shape`.** Ajouter les trois vérifications `var _ Shape = …`.

**E2 — `io.Reader`.**
Écrire `CountLines(r io.Reader) (int, error)` et l'appeler avec trois sources différentes :
`strings.NewReader`, `os.Open` sur un fichier, `os.Stdin`. **Une seule** implémentation pour
les trois. Expliquer en deux phrases pourquoi c'est possible.

**E3 — Switch de type.**
Écrire `Describe(v any) string` qui traite : `nil`, `int`, `float64`, `string`, `[]any`,
`map[string]any`, `error`, et un cas par défaut affichant le type avec `%T`. Le tester avec
huit valeurs. *(C'est exactement ce que fait un décodeur JSON — leçon du niveau 5.)*

**E4 — Le piège nil.**
Reproduire le piège de l'interface nil : une fonction qui retourne `error` à partir d'une
variable `*MyError` nil. Constater que `err != nil` est vrai. Afficher `%T` et `%v` de
l'erreur. Puis corriger. **Écrire en trois phrases l'explication du phénomène** — si elle
n'est pas claire, le concept n'est pas acquis.

**E5 — Stringer.**
Créer trois types (`Temperature`, `Money`, `Percentage`) basés sur des numériques, chacun
avec un `String()` adapté. Les afficher dans un `[]fmt.Stringer`. Que se passe-t-il si l'un
des `String()` a un récepteur pointeur et qu'on met une valeur dans le slice ?

---

## Exercice intermédiaire — `pipeline de traitement`

Concevoir un système de traitement de documents extensible.

```go
type Document struct {
	Name    string
	Content string
	Meta    map[string]string
}

type Processor interface {
	Process(d *Document) error
}
```

**À implémenter :**
- `Trimmer` — retire les espaces superflus ;
- `WordCounter` — écrit le nombre de mots dans `Meta["words"]` ;
- `Redactor` — remplace une liste de mots interdits par `***` ;
- `Validator` — retourne une erreur si le document est vide ou sans nom ;
- `Pipeline` — un `Processor` qui contient d'autres `Processor` et les applique en séquence.

**Contraintes :**
1. `Pipeline` doit lui-même satisfaire `Processor` — un pipeline peut donc en contenir un autre. *(Ce patron a un nom classique. Lequel ?)*
2. Le pipeline s'arrête à la première erreur, en indiquant **quel processeur** a échoué et **à quelle position**.
3. Ajouter un `Processor` ne doit modifier **aucun** code existant. Le vérifier en en ajoutant un cinquième après coup.
4. Écrire un `fakeProcessor` pour tester le pipeline sans dépendre des vrais processeurs. Combien de lignes fait-il ? C'est la mesure de la qualité de l'interface.
5. `Processor` doit rester à **une seule méthode**. Si une implémentation a besoin de configuration, elle passe par sa struct, pas par l'interface.
6. Question à trancher et documenter : `Process` prend `*Document` et le modifie. L'alternative serait `Process(Document) (Document, error)`. Comparer les deux sur : allocations, testabilité, risque d'erreur, composabilité. **Défendre le choix retenu.**

---

## Défi

**a) Interfaces optionnelles.**
La stdlib utilise un patron avancé : tester si une valeur satisfait *aussi* une interface
supplémentaire, pour activer une optimisation.

```go
type Flusher interface { Flush() error }

func Write(w io.Writer, data []byte) error {
	// écrire data, puis SI w sait se vider, le vider
}
```
Implémenter `Write`, puis deux types : l'un avec `Flush`, l'autre sans. Montrer que les deux
fonctionnent. *(C'est exactement ce que fait `net/http` avec `http.Flusher`.)*

**b) Décorateur.** *(à reprendre après la [leçon 5](../lecon-05-composition/), qui traite l'embedding)*
Écrire `LoggingNotifier` qui **embarque** un `Notifier` et journalise chaque appel avant de
déléguer. Il doit satisfaire `Notifier` lui-même, et fonctionner avec n'importe quelle
implémentation, y compris une autre décoration. Empiler trois décorateurs et vérifier
l'ordre d'exécution.
*Ce patron est le fondement des middlewares HTTP du niveau 5 — le reconnaître ici fait
gagner une semaine plus tard.*

**c) Question de conception, réponse écrite de 10 à 15 lignes.**
Un collègue propose cette interface :
```go
type Storage interface {
	Connect() error
	Disconnect() error
	Get(id string) ([]byte, error)
	Put(id string, data []byte) error
	Delete(id string) error
	List(prefix string) ([]string, error)
	BeginTx() (Tx, error)
	Stats() Stats
}
```
Critiquer cette conception selon les principes de la leçon : que reprocher, précisément ?
Proposer un découpage. Puis nuancer honnêtement : dans quel contexte une grosse interface
comme celle-ci **serait**-elle défendable ?

---

## Quiz

1. Comment un type déclare-t-il qu'il implémente une interface en Go ?
2. Que contient exactement une valeur d'interface ?
3. Pourquoi une interface contenant un pointeur nil n'est-elle pas `nil` ?
4. Que signifie « the bigger the interface, the weaker the abstraction » ?
5. Où faut-il définir une interface : dans le paquet qui l'implémente, ou dans celui qui l'utilise ? Pourquoi ?
6. Que veut dire « accept interfaces, return structs » ?
7. Différence entre `x.(T)` et `x, ok := x.(T)` ?
8. Quand un `switch` de type est-il légitime, et quand trahit-il une erreur de conception ?
9. `any` est l'alias de quoi ? Depuis quelle version de Go ?
10. Si `func (t *T) Foo()` est la seule méthode de `T`, est-ce que `T` satisfait l'interface `Foo() `? Et `*T` ?
11. Comment vérifier à la compilation qu'un type satisfait une interface ?
