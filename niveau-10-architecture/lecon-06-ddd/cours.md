# Leçon 6 — Domain-Driven Design

> Le *Domain-Driven Design* (DDD) est une approche de conception pour les logiciels dont la difficulté est
> dans le **métier** plutôt que dans la technique : des règles nombreuses, qui changent, et que les
> experts expriment avec leurs propres mots. Il apporte deux choses. Un **vocabulaire tactique** (valeur,
> entité, agrégat, événement) pour écrire un domaine dont les règles ne peuvent pas être contournées. Et
> une idée **stratégique**, les contextes bornés, pour découper un grand système là où le sens des mots
> change. Cette leçon prend les deux, et laisse de côté le reste.

## Objectifs

1. Écrire le code avec le **langage du métier** (langage ubiquitaire).
2. Distinguer **objets valeur** et **entités**, et rendre les valeurs valides par construction.
3. Concevoir un **agrégat** : une racine, des invariants, une frontière de cohérence transactionnelle.
4. Protéger les invariants avec la visibilité des paquets Go.
5. Enregistrer des **événements de domaine**.
6. Reconnaître des **contextes bornés**, et quand le DDD n'est pas utile.

## Explication

### Le langage ubiquitaire

Les bibliothécaires disent « emprunter », « rendre », « prolonger », « pénalité », « exemplaire »,
« cote ». Le code dit `Borrow`, `Return`, `Renew`, `Fines`, `CopyID` — pas `UpdateLoanStatus`,
`SetDueDate` ou `ItemRecord`. Quand un expert dit « on ne peut pas prolonger un emprunt en retard », on
doit trouver cette phrase presque mot pour mot dans `Renew`. Le langage est le même dans les
conversations, les tickets, les tests et le code : c'est ce qui rend les règles **vérifiables par
quelqu'un qui ne lit pas le Go**.

(Le code reste en anglais, par convention de l'écosystème ; ce qui compte est la correspondance un
pour un des termes.)

### Objets valeur et entités

| | Objet **valeur** | **Entité** |
|---|---|---|
| Identité | aucune : défini par sa valeur | une identité qui persiste quand ses attributs changent |
| Égalité | par valeur (`==` sur une struct comparable) | par identifiant |
| Mutabilité | immuable : les opérations rendent de nouvelles valeurs | modifiée au fil du temps, par ses méthodes |
| Exemples | `CopyID`, `Money`, une période, une adresse | un adhérent, un emprunt |

Les objets valeur règlent une classe entière de bugs en Go : une `string` peut contenir n'importe quoi,
un `CopyID` **non** — il ne s'obtient que par `ParseCopyID`, qui valide (le champ non exporté interdit
`CopyID{"n'importe quoi"}` hors du paquet). Une fois la valeur construite, plus personne n'a à la
revérifier. `Money` empêche d'additionner des euros avec un nombre de jours, et de calculer en
`float64` (N4 L3).

### L'agrégat

Un **agrégat** est un groupe d'objets (une entité racine et ce qu'elle contient) qui forme une unité de
cohérence :

- on n'y accède **que par la racine** (`Member`) ; les objets internes (`loan`) ne sont pas visibles de
  l'extérieur ;
- la racine garantit les **invariants** (« 3 emprunts au plus », « plus d'emprunt au-delà de 5 € de
  pénalités », « une prolongation au plus, jamais en retard ») : aucune séquence d'appels publics ne peut
  les violer ;
- c'est l'unité de **chargement et d'enregistrement** : on charge un adhérent entier, on le modifie par ses
  méthodes, on l'enregistre entier, **dans une transaction** (N8 L4). Un agrégat = une transaction ;
- les références vers un **autre** agrégat se font par **identifiant** (`CopyID`), pas par pointeur.

En Go, la protection est assurée par le **paquet** : les champs de `Member` ne sont pas exportés, donc
`ada.loans = nil` ne compile pas en dehors de `lending`. C'est pour cela que l'exemple teste depuis un
paquet externe (`lending_test`) : il ne voit que ce qu'un autre paquet verrait.

Choisir la **taille** d'un agrégat est la décision difficile. Trop gros (une `Library` qui contient tous
les adhérents), et chaque emprunt verrouille toute la bibliothèque. Trop petit, et les invariants qui
couvrent plusieurs objets ne peuvent plus être garantis (la leçon 1, défi : « un exemplaire n'a qu'un
emprunteur » couvre l'exemplaire **et** l'adhérent). Règle pratique : un agrégat aussi petit que possible,
autour d'un invariant qui doit être **immédiatement** vrai ; les règles entre agrégats acceptent souvent
une cohérence **à terme**, par événements (leçon 8).

### Les événements de domaine

Quand quelque chose d'important arrive (« exemplaire rendu avec 28 jours de retard »), l'agrégat
**l'enregistre** sous la forme d'un événement, nommé au passé dans le langage du métier. Il ne l'envoie
pas lui-même (il ne connaît ni courriel ni file de messages) : le service les récupère après coup
(`PullEvents`), les enregistre avec l'agrégat dans la même transaction, et ils seront publiés plus tard
(leçon 8, *outbox*). D'autres parties du système réagissent : notifier l'adhérent, mettre à jour des
statistiques, débloquer une réservation.

### Les contextes bornés

Dans une bibliothèque, « livre » désigne au **catalogue** une œuvre (titre, auteurs, résumé, ISBN), et au
**prêt** un exemplaire physique (une cote, un état, un emprunteur). Un seul type `Book` qui servirait les
deux grossirait de champs que chaque partie ignore, et chaque changement de l'un menacerait l'autre.

Un **contexte borné** est une frontière à l'intérieur de laquelle un modèle et ses mots ont un sens unique.
Le catalogue et le prêt sont deux contextes, chacun avec son modèle ; ils communiquent par des
identifiants (« l'exemplaire EX-1 est un exemplaire de l'œuvre ISBN 978… ») et des événements, pas en
partageant des structs ou des tables. En Go, un contexte borné correspond naturellement à un paquet (ou un
groupe de paquets) — et c'est la base du **monolithe modulaire** de la leçon 7.

### Quand ne pas le faire

- **Un domaine pauvre** (CRUD, administration de données, intégration technique) : agrégats et objets
  valeur ajoutent des types et des conversions sans protéger aucune règle.
- **Le DDD « complet »** avec usines, répertoires, services de domaine, services d'application, couches
  anti-corruption pour un service de dix règles : le vocabulaire tactique ne vaut que là où les invariants
  sont réels.
- **Des objets valeur pour tout** (`FirstName`, `LastName`, `Comment`…) : seulement pour les valeurs qui
  ont des **règles** (format, plage, unité).

La partie la plus rentable, presque toujours : le langage du métier dans le code, des objets valeur pour
les quelques notions qui ont des règles, et des invariants protégés par la visibilité du paquet.

## Exemple

Le contexte « prêt » : deux objets valeur, un agrégat `Member` qui protège trois invariants, et ses
événements. La fonction d'exemple est écrite **depuis un autre paquet** (`lending_test`) ; sa sortie est
vérifiée par `go test`.

```go
// Fichier : lending/values.go
// Package lending : le contexte « prêt ». Langage du métier : emprunter, rendre, prolonger, pénalité.
package lending

import (
	"errors"
	"fmt"
	"strings"
)

// CopyID : objet VALEUR. Défini par sa valeur, immuable, valide par construction.
type CopyID struct{ v string }

func ParseCopyID(s string) (CopyID, error) {
	s = strings.TrimSpace(s)
	if len(s) < 3 || !strings.HasPrefix(s, "EX-") {
		return CopyID{}, fmt.Errorf("cote d'exemplaire %q invalide (EX-…)", s)
	}
	return CopyID{s}, nil
}

func (c CopyID) String() string { return c.v }

// Money : montant en centimes. Les opérations rendent de NOUVELLES valeurs.
type Money struct{ cents int64 }

func Cents(c int64) Money                { return Money{c} }
func (m Money) Add(o Money) Money        { return Money{m.cents + o.cents} }
func (m Money) GreaterThan(o Money) bool { return m.cents > o.cents }
func (m Money) String() string           { return fmt.Sprintf("%d,%02d €", m.cents/100, m.cents%100) }

var ErrInvalid = errors.New("valeur invalide")
```

```go
// Fichier : lending/member.go
package lending

import (
	"errors"
	"fmt"
	"time"
)

var (
	ErrTooManyLoans  = errors.New("nombre maximal d'emprunts atteint")
	ErrUnpaidFines   = errors.New("pénalités impayées au-delà du seuil")
	ErrNotBorrowed   = errors.New("exemplaire non emprunté par cet adhérent")
	ErrRenewalDenied = errors.New("prolongation refusée")
)

const (
	MaxLoans    = 3
	LoanPeriod  = 21 * 24 * time.Hour
	MaxRenewals = 1
)

var (
	FinePerDay = Cents(20)  // 0,20 € par jour entamé
	FineLimit  = Cents(500) // au-delà de 5 €, plus d'emprunt
)

// loan : ENTITÉ interne à l'agrégat (identité : la cote de l'exemplaire). Invisible de l'extérieur.
type loan struct {
	copy     CopyID
	due      time.Time
	renewals int
}

// Event : ce qui s'est passé, dans le langage du métier. Collecté par l'agrégat (leçon 8).
type Event struct {
	Name string
	At   time.Time
	Data string
}

// Member : RACINE de l'agrégat. Tous les changements passent par ses méthodes,
// qui garantissent les invariants. Les champs non exportés empêchent tout contournement.
type Member struct {
	id     string
	loans  []loan
	fines  Money
	events []Event
}

func NewMember(id string) *Member { return &Member{id: id} }

func (m *Member) ID() string     { return m.id }
func (m *Member) LoanCount() int { return len(m.loans) }
func (m *Member) Fines() Money   { return m.fines }

// PullEvents rend les événements enregistrés et vide la liste.
func (m *Member) PullEvents() []Event {
	ev := m.events
	m.events = nil
	return ev
}

func (m *Member) record(name string, at time.Time, data string) {
	m.events = append(m.events, Event{name, at, data})
}

// Borrow : invariants « 3 au plus » et « pénalités sous le seuil ».
func (m *Member) Borrow(c CopyID, now time.Time) error {
	if len(m.loans) >= MaxLoans {
		return ErrTooManyLoans
	}
	if m.fines.GreaterThan(FineLimit) {
		return fmt.Errorf("%w (%s)", ErrUnpaidFines, m.fines)
	}
	m.loans = append(m.loans, loan{copy: c, due: now.Add(LoanPeriod)})
	m.record("ExemplaireEmprunté", now, c.String())
	return nil
}

// Return : calcule et ajoute la pénalité de retard.
func (m *Member) Return(c CopyID, now time.Time) (Money, error) {
	for i, l := range m.loans {
		if l.copy != c {
			continue
		}
		m.loans = append(m.loans[:i], m.loans[i+1:]...)
		fine := Cents(0)
		if late := now.Sub(l.due); late > 0 {
			days := int64((late + 24*time.Hour - 1) / (24 * time.Hour))
			fine = Money{days * FinePerDay.cents}
			m.fines = m.fines.Add(fine)
		}
		m.record("ExemplaireRendu", now, fmt.Sprintf("%s pénalité=%s", c, fine))
		return fine, nil
	}
	return Cents(0), ErrNotBorrowed
}

// Renew : une prolongation au plus, et jamais d'un emprunt déjà en retard.
func (m *Member) Renew(c CopyID, now time.Time) error {
	for i := range m.loans {
		l := &m.loans[i]
		if l.copy != c {
			continue
		}
		switch {
		case l.renewals >= MaxRenewals:
			return fmt.Errorf("%w : déjà prolongé", ErrRenewalDenied)
		case now.After(l.due):
			return fmt.Errorf("%w : emprunt en retard", ErrRenewalDenied)
		}
		l.renewals++
		l.due = l.due.Add(LoanPeriod)
		m.record("EmpruntProlongé", now, c.String())
		return nil
	}
	return ErrNotBorrowed
}
```

```go
// Fichier : lending/example_test.go
// Un test du paquet EXTERNE (lending_test) : il ne voit que ce qu'un autre paquet verrait.
package lending_test

import (
	"fmt"
	"time"

	"exemple/lending"
)

func Example() {
	day := func(d int) time.Time { return time.Date(2026, 3, d, 10, 0, 0, 0, time.UTC) }
	cp := func(s string) lending.CopyID { c, _ := lending.ParseCopyID(s); return c }

	if _, err := lending.ParseCopyID("42"); err != nil {
		fmt.Println("valeur :", err)
	}

	ada := lending.NewMember("ada")
	for _, c := range []string{"EX-1", "EX-2", "EX-3", "EX-4"} {
		fmt.Printf("emprunt %s : %v\n", c, ada.Borrow(cp(c), day(1)))
	}
	fmt.Println("prolongation EX-1 :", ada.Renew(cp("EX-1"), day(10)))
	fmt.Println("prolongation EX-1 :", ada.Renew(cp("EX-1"), day(11)))

	// EX-2 rendu avec 28 jours de retard (attendu le 22 mars, rendu le 19 avril).
	fine, _ := ada.Return(cp("EX-2"), day(1).AddDate(0, 0, 49))
	fmt.Println("pénalité EX-2 :", fine, "— total dû :", ada.Fines())
	fmt.Println("emprunt EX-5 :", ada.Borrow(cp("EX-5"), day(1).AddDate(0, 0, 49)))

	// ada.loans = nil   → ne compile pas : l'invariant ne peut pas être contourné de l'extérieur.

	for _, e := range ada.PullEvents() {
		fmt.Printf("événement %s %s %s\n", e.At.Format("01-02"), e.Name, e.Data)
	}
	fmt.Println("événements restants :", len(ada.PullEvents()))
	// Output:
	// valeur : cote d'exemplaire "42" invalide (EX-…)
	// emprunt EX-1 : <nil>
	// emprunt EX-2 : <nil>
	// emprunt EX-3 : <nil>
	// emprunt EX-4 : nombre maximal d'emprunts atteint
	// prolongation EX-1 : <nil>
	// prolongation EX-1 : prolongation refusée : déjà prolongé
	// pénalité EX-2 : 5,60 € — total dû : 5,60 €
	// emprunt EX-5 : pénalités impayées au-delà du seuil (5,60 €)
	// événement 03-01 ExemplaireEmprunté EX-1
	// événement 03-01 ExemplaireEmprunté EX-2
	// événement 03-01 ExemplaireEmprunté EX-3
	// événement 03-10 EmpruntProlongé EX-1
	// événement 04-19 ExemplaireRendu EX-2 pénalité=5,60 €
	// événements restants : 0
}
```
`go test ./...` passe.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `CopyID{v string}` + `ParseCopyID` | Valeur valide par construction ; impossible d'en fabriquer une invalide hors du paquet. |
| `Money` et ses méthodes | Les opérations rendent de nouvelles valeurs ; aucune arithmétique en `float64`. |
| `loan` non exporté | Entité interne à l'agrégat : invisible et inaccessible de l'extérieur. |
| `Member` aux champs non exportés | La seule façon de changer un adhérent est d'appeler ses méthodes ; `ada.loans = nil` ne compile pas. |
| `Borrow`, `Return`, `Renew` | Les invariants, dans les mots du métier ; chaque refus est une erreur nommée. |
| Pénalité 5,60 € = 28 jours × 0,20 € | Rendu le 19 avril, attendu le 22 mars : 28 jours entamés. |
| `EX-5` refusé | L'invariant « pénalités sous le seuil » joue immédiatement après le retour. |
| `record` et `PullEvents` | L'agrégat note ce qui s'est passé ; il ne publie rien lui-même. Les événements sont vidés une fois lus. |
| Test dans `lending_test` | Il vérifie le comportement **tel qu'un autre paquet le voit**. |

## Erreurs fréquentes

1. **Des noms techniques** (`UpdateStatus`, `SetDue`) à la place des mots du métier.
2. **Des champs exportés** sur un agrégat : n'importe quel paquet contourne les invariants.
3. **Des valeurs primitives partout** (`string` pour une cote, `float64` pour de l'argent).
4. **Un agrégat géant** qui verrouille tout, ou **trop petit** pour garantir ses invariants.
5. **Des pointeurs entre agrégats** au lieu d'identifiants.
6. **Un agrégat qui envoie lui-même** courriels ou messages.
7. **Un seul modèle** pour des contextes où les mots n'ont pas le même sens.
8. **Tout le vocabulaire DDD** appliqué à un domaine sans règles.

## Bonnes pratiques Go

- Le langage du métier dans les noms de types, méthodes et erreurs.
- Objets valeur avec champ non exporté et constructeur qui valide, pour les notions qui ont des règles.
- Agrégat : racine aux champs non exportés, méthodes qui garantissent les invariants, testé depuis un
  paquet externe.
- Un agrégat chargé et enregistré entier, dans une transaction.
- Événements de domaine enregistrés par l'agrégat, récupérés et publiés par le service.
- Un contexte borné par paquet (ou groupe de paquets) ; échanges par identifiants et événements.

## Ce que je dois retenir

- Langage ubiquitaire : les mots des experts, un pour un, dans le code.
- Valeur : sans identité, immuable, valide par construction. Entité : une identité qui persiste.
- Agrégat : une racine, des invariants, une transaction ; on n'y touche que par la racine.
- En Go, la visibilité du paquet protège les invariants.
- Contexte borné : là où un mot a un seul sens ; un paquet en Go.

---

➡️ [Exercices](exercices.md)
