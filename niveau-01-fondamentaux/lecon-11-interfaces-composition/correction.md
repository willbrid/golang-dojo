# Leçon 11 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Satisfaction implicite

```go
type Shape interface {
	Area() float64
	Perimeter() float64
}

type Circle struct{ R float64 }
func (c Circle) Area() float64      { return math.Pi * c.R * c.R }
func (c Circle) Perimeter() float64 { return 2 * math.Pi * c.R }

type Rectangle struct{ W, H float64 }
func (r Rectangle) Area() float64      { return r.W * r.H }
func (r Rectangle) Perimeter() float64 { return 2 * (r.W + r.H) }

var (
	_ Shape = Circle{}
	_ Shape = Rectangle{}
	_ Shape = Triangle{}
)

shapes := []Shape{Circle{2}, Rectangle{3, 4}, Triangle{3, 4, 5}}
total := 0.0
for _, s := range shapes { total += s.Area() }
```

Aucun des trois types ne mentionne `Shape`. On pourrait supprimer l'interface : les types
continueraient de compiler. C'est ça, le découplage structurel.

## E2 — `io.Reader`

```go
func CountLines(r io.Reader) (int, error) {
	sc := bufio.NewScanner(r)
	n := 0
	for sc.Scan() { n++ }
	return n, sc.Err()
}

CountLines(strings.NewReader("a\nb\nc"))
CountLines(f)          // *os.File
CountLines(os.Stdin)   // *os.File aussi
```

Une seule implémentation suffit parce que `CountLines` ne dépend **que** de la capacité à
lire des octets — exactement ce que décrit `io.Reader`. Un fichier, une chaîne, une socket,
un flux gzip, une réponse HTTP : tous fournissent `Read([]byte) (int, error)`, donc tous
fonctionnent. C'est l'abstraction la plus rentable de la bibliothèque standard, et c'est le
modèle à imiter quand on conçoit une API.

## E3 — Switch de type

```go
func Describe(v any) string {
	switch val := v.(type) {
	case nil:
		return "nil"
	case int:
		return fmt.Sprintf("entier %d", val)
	case float64:
		return fmt.Sprintf("flottant %g", val)
	case string:
		return fmt.Sprintf("chaîne %q (%d octets)", val, len(val))
	case []any:
		return fmt.Sprintf("tableau de %d éléments", len(val))
	case map[string]any:
		return fmt.Sprintf("objet de %d clés", len(val))
	case error:
		return "erreur : " + val.Error()
	default:
		return fmt.Sprintf("type non géré %T", val)
	}
}
```

L'ordre compte : `case error` doit précéder `default`, et si un type concret implémente
`error`, le cas concret l'emporte s'il vient avant. Ce switch est littéralement ce qu'on
écrit après un `json.Unmarshal` dans une `map[string]any` — le niveau 5 y reviendra.

## E4 — Le piège nil

```go
type MyError struct{}
func (e *MyError) Error() string { return "boom" }

func broken() error {
	var e *MyError // nil
	return e       // → interface (type=*MyError, valeur=nil) : NON NIL
}

err := broken()
fmt.Println(err == nil)          // false
fmt.Printf("%T %v\n", err, err)  // *main.MyError <nil>
```

**L'explication en trois phrases.** Une valeur d'interface est un couple (type dynamique,
valeur). En affectant un `*MyError` nil à une variable de type `error`, on remplit le champ
« type » avec `*MyError` tout en laissant le champ « valeur » à nil. L'interface n'est donc
pas nil, puisqu'elle ne l'est que si **ses deux champs** le sont.

Correction : ne jamais déclarer de variable de type pointeur concret pour la retourner en
`error` — retourner `nil` littéralement dans le chemin de succès.

Ce bug a touché du code très diffusé ; il apparaît typiquement quand une fonction retourne
`*MyError` et qu'un appelant intermédiaire change la signature en `error`.

## E5 — Stringer et récepteur

Si `String()` a un récepteur **pointeur**, seul `*Temperature` satisfait `fmt.Stringer` :
mettre une **valeur** `Temperature` dans un `[]fmt.Stringer` ne compile pas
(`Temperature does not implement fmt.Stringer (method String has pointer receiver)`).

Et plus insidieux : `fmt.Println(t)` sur une valeur affichera le nombre brut sans erreur ni
avertissement — `fmt` teste l'interface à l'exécution, ne la trouve pas, et se rabat sur le
format par défaut. Le bug est **silencieux**.

C'est la raison profonde de la règle de cohérence de la leçon 10.

## Exercice intermédiaire — Pipeline

```go
type Processor interface {
	Process(d *Document) error
}

// Pipeline applique une séquence de Processor.
// Il satisfait lui-même Processor : c'est le patron COMPOSITE — un composant
// composé se comporte exactement comme un composant simple, donc un pipeline
// peut en contenir un autre sans code particulier.
type Pipeline struct {
	Name  string
	Steps []Processor
}

func (p *Pipeline) Process(d *Document) error {
	for i, step := range p.Steps {
		if err := step.Process(d); err != nil {
			return fmt.Errorf("%s, étape %d (%T) : %w", p.Name, i+1, step, err)
		}
	}
	return nil
}

// Le faux de test tient en 4 lignes — c'est la MESURE de la qualité de l'interface.
type fakeProcessor struct {
	called int
	err    error
}
func (f *fakeProcessor) Process(d *Document) error { f.called++; return f.err }
```

**Contrainte 3 vérifiée** : ajouter un cinquième processeur n'exige aucune modification de
`Pipeline`, ni des quatre autres. C'est le principe ouvert/fermé, obtenu gratuitement par
une interface à une méthode.

**Contrainte 6 — `*Document` ou valeur ?**

| | `Process(*Document) error` | `Process(Document) (Document, error)` |
|---|---|---|
| Allocations | aucune copie | une copie par étape (et `Meta` reste **partagée** : copie superficielle !) |
| Testabilité | il faut vérifier l'état après | comparaison entrée/sortie directe |
| Risque | mutation partielle si erreur au milieu | l'original est toujours intact |
| Composabilité | naturelle | naturelle |

Le piège de la seconde forme : `Document` contient une `map`, donc sa copie est
**superficielle** — deux « copies » partagent la même `Meta`, et l'immuabilité espérée est
une illusion. Il faudrait une copie profonde explicite, coûteuse et facile à oublier.

**Le pointeur est donc le bon choix ici**, à condition de documenter le risque de mutation
partielle en cas d'erreur — ou de travailler sur un clone au niveau du pipeline si
l'atomicité est requise. Formuler ce compromis valait plus que le code.

## Défi

**a) Interfaces optionnelles**

```go
type Flusher interface{ Flush() error }

func Write(w io.Writer, data []byte) error {
	if _, err := w.Write(data); err != nil {
		return fmt.Errorf("écriture : %w", err)
	}
	if f, ok := w.(Flusher); ok { // SI w sait aussi se vider
		if err := f.Flush(); err != nil {
			return fmt.Errorf("vidage : %w", err)
		}
	}
	return nil
}
```

L'assertion de type sur une interface **enrichie** permet d'exploiter une capacité
supplémentaire quand elle existe, sans l'exiger de tout le monde. La stdlib en est truffée :
`http.Flusher`, `http.Hijacker`, `io.ReaderFrom`, `io.WriterTo` (qui permet à `io.Copy` de
court-circuiter le buffer intermédiaire).

Limite honnête : ce patron est **invisible dans les signatures**. Il faut le documenter,
sinon personne ne sait que l'optimisation existe.

**b) Décorateur**

```go
type LoggingNotifier struct {
	Notifier          // interface EMBARQUÉE : les méthodes non redéfinies sont promues
	Prefix   string
}

func (l LoggingNotifier) Notify(user, msg string) error {
	log.Printf("%s → envoi à %s", l.Prefix, user)
	err := l.Notifier.Notify(user, msg) // délégation explicite
	if err != nil {
		log.Printf("%s → échec : %v", l.Prefix, err)
	}
	return err
}

n := LoggingNotifier{
	Notifier: LoggingNotifier{
		Notifier: EmailNotifier{From: "x@y.fr"},
		Prefix:   "interne",
	},
	Prefix: "externe",
}
```

L'ordre d'exécution est **externe → interne → cœur → interne → externe** : chaque couche
enveloppe la suivante. C'est exactement la structure d'un middleware HTTP
(`func(http.Handler) http.Handler`), et c'est pourquoi reconnaître le patron ici fait gagner
beaucoup de temps au niveau 5.

L'embedding de l'interface évite d'écrire des méthodes de délégation pour tout ce qu'on ne
décore pas — mais attention : si le champ embarqué est nil et qu'une méthode non redéfinie
est appelée, c'est la panique.

**c) Critique de `Storage`**

Reproches précis :
1. **Huit méthodes** : écrire un faux en test exige d'implémenter les huit, même pour un
   test qui n'en utilise qu'une. C'est le meilleur indicateur qu'une interface est trop grosse.
2. `Connect`/`Disconnect` sont des détails du **cycle de vie de l'implémentation**, pas du
   besoin de l'appelant. Ils devraient vivre dans le constructeur concret et un `Close()`.
3. **Plusieurs responsabilités** mélangées : accès aux données (`Get`/`Put`/`Delete`),
   navigation (`List`), transactions (`BeginTx`), observabilité (`Stats`).
4. `Stats` couple l'interface à un type `Stats` qui n'a rien à faire dans un contrat d'accès
   aux données.
5. **Définie côté fournisseur** : elle décrit ce que PostgreSQL sait faire, pas ce que
   l'appelant a besoin de faire.

Découpage :
```go
type Getter interface { Get(id string) ([]byte, error) }
type Putter interface { Put(id string, data []byte) error }
type Deleter interface { Delete(id string) error }
type Lister interface { List(prefix string) ([]string, error) }
// composables au besoin :
type ReadWriteStore interface { Getter; Putter }
```
Chaque consommateur déclare la plus petite interface qui lui suffit — de préférence chez lui,
non exportée.

**Nuance honnête.** Une grosse interface se défend quand elle décrit une **frontière
d'implémentation interchangeable** dont on veut garantir la complétude : plusieurs backends
(S3, disque, mémoire) qui doivent *tous* tout implémenter, avec une suite de tests de
conformité partagée. Là, l'interface large est un **contrat**, pas une abstraction de
consommation, et le compilateur qui refuse un backend incomplet rend service. `sql/driver`
de la stdlib fonctionne ainsi. La question n'est donc pas « grosse ou petite » mais **qui la
définit et pourquoi**.

## Réponses du quiz

1. Il ne le déclare **pas** : il suffit qu'il possède les méthodes. Satisfaction implicite.
2. Un couple **(type dynamique, valeur)**.
3. Parce que son champ « type » est renseigné (`*MyError`) même si son champ « valeur » est
   nil. Une interface n'est nil que si les **deux** le sont.
4. Que moins une interface exige de méthodes, plus de types peuvent la satisfaire et plus
   elle est réutilisable. `io.Reader` avec sa méthode unique est plus puissante qu'une
   interface `FileLike` à dix méthodes.
5. Dans le paquet qui l'**utilise**. Elle décrit alors un besoin, pas une implémentation ;
   elle reste minimale, et le fournisseur n'a aucune dépendance vers elle.
6. Accepter des interfaces en paramètre (souplesse pour l'appelant, testabilité) et
   retourner des types concrets (l'appelant garde accès à toutes les méthodes et aux
   évolutions futures).
7. `x.(T)` **panique** si le type ne correspond pas ; `v, ok := x.(T)` retourne `ok = false`.
8. Légitime pour des données de type inconnu par nature (JSON, `any`, erreurs, réflexion).
   Suspect dès qu'il porte sur des types qu'on contrôle : il fallait alors une méthode sur
   l'interface.
9. De `interface{}`, depuis **Go 1.18**. C'est un alias, donc strictement le même type.
10. `T` ne la satisfait **pas** ; seul `*T` la satisfait. L'ensemble de méthodes de `T` ne
    contient que les méthodes à récepteur valeur, tandis que celui de `*T` contient les deux.
    C'est la raison technique de la règle « cohérence des récepteurs ».

---

## 🎓 Fin du niveau 1

Une fois ces onze leçons digérées : demander l'**évaluation cumulative**, puis attaquer le
[projet 1 — `wordstat`](../../projets/projet-01-cli-simple/).
