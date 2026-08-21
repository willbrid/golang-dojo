# Leçon 5 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Promotion

```go
type Base struct{ Name string }
func (b Base) Hello() string { return "bonjour " + b.Name }

type Derived struct {
	Base
	Extra string
}

d := Derived{Base: Base{Name: "Rex"}, Extra: "chien"}
d.Name         // "Rex"  — champ promu
d.Base.Name    // "Rex"  — accès explicite
b := d.Base    // copie du champ embarqué
d.Hello()      // méthode promue

var x Base = d // ERREUR
```

```
./main.go:16:14: cannot use d (variable of struct type Derived) as Base value in variable declaration
```

Le message dit tout : `Derived` **est** un type distinct, pas un sous-type de `Base`. Il n'y a
aucune relation de parenté, seulement une relation de **contenance**. Pour rendre les deux
interchangeables, il faut une interface que tous deux satisfont — et cette interface ne
mentionne ni l'un ni l'autre.

## E2 — Absence de dispatch virtuel

```
Derived:Rex/chien      ← d.Greet()      : la méthode extérieure masque la promue
Base:Rex               ← d.Base.Greet() : l'originale reste accessible
je m'appelle Base:Rex  ← d.Describe()   : PAS Derived !
```

**Ce qu'un langage à héritage aurait affiché** pour la troisième ligne : « je m'appelle
Derived:Rex/chien ». En Java, C++ ou Python, `Describe` étant héritée par `Derived`, l'appel à
`Greet()` en son sein serait résolu **dynamiquement** vers la version de la classe réelle de
l'objet.

Go fait autrement pour une raison précise : `Base.Describe` est compilée sans aucune
connaissance de `Derived`, qui peut être déclaré dans un autre package, des années plus tard.
La méthode appelle donc ce qu'elle connaît — la sienne. Il n'existe pas de table de méthodes
virtuelles attachée à la valeur.

Ce que l'on perd : le patron *template method*. Ce que l'on gagne : plus aucune surprise du
type « ma classe mère appelle une méthode que j'ai redéfinie, avec un champ que je n'ai pas
encore initialisé ». En Go, pour obtenir un comportement paramétrable, on **passe** la fonction
ou l'interface explicitement.

## E3 — Ambiguïté

```
./main.go:14:4: ambiguous selector c.Name
./main.go:15:4: ambiguous selector c.Hello
```

La promotion se résout par **profondeur** : le champ ou la méthode le moins profond gagne. À
profondeur égale entre deux types embarqués, Go **refuse d'arbitrer** et signale l'ambiguïté à
la compilation.

Deux corrections :
```go
c.A.Name       // désambiguïser à l'appel
// ou : redéfinir sur C, ce qui la rend moins profonde et donc prioritaire
func (c C) Hello() string { return c.A.Hello() }
```

La seconde est préférable quand l'ambiguïté est structurelle : elle rend le choix explicite
**une fois**, dans la déclaration du type, plutôt qu'à chaque appel.

Point rassurant : l'ambiguïté est une **erreur de compilation**, pas un arbitrage silencieux.
Certains langages à héritage multiple choisissent à notre place selon un ordre de linéarisation
qu'il faut connaître par cœur.

## E4 — Composition d'interfaces

```go
type Sizer interface{ Size() int }
type Namer interface{ Name() string }

type SizedNamer interface {
	Sizer
	Namer
}

var _ SizedNamer = (*File)(nil)
```

Si une méthode manque :
```
cannot use (*File)(nil) (value of type *File) as SizedNamer value:
	*File does not implement SizedNamer (missing method Size)
```

Le message nomme **la méthode manquante**, ce qui rend le diagnostic immédiat. C'est tout
l'intérêt de la ligne `var _ Iface = (*T)(nil)` : sans elle, l'erreur n'apparaîtrait qu'au
premier usage, parfois dans un autre package.

## E5 — Interface embarquée nil

```go
type Partial struct{ Store }   // interface embarquée, non initialisée
var p Partial
p.Get("x")   // panic: runtime error: invalid memory address or nil pointer dereference
```

L'appel est dispatché vers le champ embarqué, qui vaut `nil` : aucune méthode à appeler.

**Ce comportement est souhaitable en test.** Un faux partiel qui n'implémente que `Get` doit
**paniquer** si le code testé appelle `Delete` : cela révèle immédiatement que le test ne
vérifie pas ce qu'on croit. Un faux qui retournerait des zéro-valeurs pour toutes les autres
méthodes laisserait passer silencieusement un appel inattendu — et le test deviendrait
mensonger.

En production, en revanche, un champ embarqué nil est un bug : la garde ou l'initialisation
obligatoire dans le constructeur s'impose.

## Exercice intermédiaire — `middleware`

```go
package main

import (
	"errors"
	"fmt"
	"log"
	"strings"
	"time"
)

type Handler interface {
	Handle(req string) (string, error)
}

// EchoHandler : le cœur, sans décoration.
type EchoHandler struct{}

func (EchoHandler) Handle(req string) (string, error) { return req, nil }

// UpperHandler EMBARQUE l'interface : seul Handle est redéfini.
type UpperHandler struct{ Handler }

func (u UpperHandler) Handle(req string) (string, error) {
	out, err := u.Handler.Handle(req) // délégation EXPLICITE : u.Handle() bouclerait
	if err != nil {
		return "", err
	}
	return strings.ToUpper(out), nil
}

type LoggingHandler struct {
	Handler
	Prefix string
}

func (l LoggingHandler) Handle(req string) (string, error) {
	start := time.Now()
	log.Printf("%s → %q", l.Prefix, req)
	out, err := l.Handler.Handle(req)
	log.Printf("%s ← %q err=%v (%s)", l.Prefix, out, err, time.Since(start))
	return out, err
}

// ErrInvalid est DÉFINITIVE : la réessayer n'a aucun sens.
var ErrInvalid = errors.New("requête invalide")

type ValidateHandler struct{ Handler }

func (v ValidateHandler) Handle(req string) (string, error) {
	if strings.TrimSpace(req) == "" {
		return "", fmt.Errorf("requête vide : %w", ErrInvalid) // n'appelle PAS le suivant
	}
	return v.Handler.Handle(req)
}

type RetryHandler struct {
	Handler
	Attempts int
}

func (r RetryHandler) Handle(req string) (string, error) {
	var last error
	for i := range r.Attempts {
		out, err := r.Handler.Handle(req)
		if err == nil {
			return out, nil
		}
		if errors.Is(err, ErrInvalid) {
			return "", err // contrainte 4 : erreur définitive, on abandonne
		}
		last = err
		_ = i
	}
	return "", fmt.Errorf("échec après %d tentatives : %w", r.Attempts, last)
}

// Chain applique les décorateurs. CHOIX DOCUMENTÉ : le PREMIER de la liste est
// le plus EXTERNE, donc le premier à s'exécuter — c'est l'ordre de lecture
// naturel et celui qu'utilisent les routeurs HTTP (chi, gorilla).
// Implémentation : on applique donc la liste À L'ENVERS.
func Chain(h Handler, decorators ...func(Handler) Handler) Handler {
	for i := len(decorators) - 1; i >= 0; i-- {
		h = decorators[i](h)
	}
	return h
}

func main() {
	h := Chain(EchoHandler{},
		func(n Handler) Handler { return LoggingHandler{Handler: n, Prefix: "externe"} },
		func(n Handler) Handler { return ValidateHandler{Handler: n} },
		func(n Handler) Handler { return RetryHandler{Handler: n, Attempts: 3} },
		func(n Handler) Handler { return UpperHandler{Handler: n} },
	)
	fmt.Println(h.Handle("bonjour"))
	fmt.Println(h.Handle(""))
}
```

**Contrainte 1 — l'embedding de l'interface.** Chaque décorateur ne redéfinit que `Handle` ;
si `Handler` gagnait cinq méthodes demain, aucun décorateur ne changerait. Sans embedding, il
faudrait écrire cinq méthodes de délégation dans chacun.

**Contrainte 2 — l'ordre d'exécution.** Il est **externe → interne → cœur → interne → externe**.
Les logs le montrent : le décorateur le plus externe écrit sa première ligne en premier et sa
dernière ligne en dernier. C'est le modèle d'une pile d'appels, pas d'une file.

**Contrainte 3 — l'ordre de `Chain`.** Les deux conventions existent réellement dans
l'écosystème. Celle retenue ici — premier de la liste = plus externe — correspond à l'ordre de
lecture (« d'abord journaliser, ensuite valider, ensuite réessayer ») et c'est celle des
routeurs HTTP Go. Elle impose d'appliquer la liste **à l'envers** dans l'implémentation, ce qui
surprend à l'écriture mais pas à l'usage. Le point évalué n'est pas le choix, c'est qu'il soit
**explicite et cohérent**.

**Contrainte 4 — erreur définitive contre erreur temporaire.** La solution provisoire retenue
est une **sentinelle** `ErrInvalid` détectée par `errors.Is`. Elle fonctionne, mais couple le
décorateur générique à une erreur particulière : chaque nouvelle erreur définitive exigera de
modifier `RetryHandler`. La solution propre — une interface `interface{ Temporary() bool }`
détectée par assertion, ou un type d'erreur portant l'information — est le sujet de la
[leçon 6](../lecon-06-erreurs-avancees/).

**Contrainte 6 — l'extensibilité.** Ajouter un `CacheHandler` ne demande de modifier **aucune**
ligne existante : c'est le principe ouvert/fermé, obtenu gratuitement par une interface à une
méthode.

## Défi

**a) Faux partiel**

```go
type Store interface {
	Get(id string) ([]byte, error)
	Put(id string, data []byte) error
	Delete(id string) error
	List(prefix string) ([]string, error)
	Count() (int, error)
	Close() error
}

// fakeStore n'implémente QUE Get. Les cinq autres méthodes existent — promues
// depuis l'interface embarquée — mais le champ vaut nil : les appeler panique.
type fakeStore struct {
	Store                                    // embarquée, volontairement NON initialisée
	getFn func(string) ([]byte, error)
}

func (f fakeStore) Get(id string) ([]byte, error) { return f.getFn(id) }
```

Quatre lignes pour un faux d'une interface à six méthodes. Sans embedding, il en faudrait
une vingtaine, à réécrire chaque fois que `Store` évolue.

```go
// Test qui n'utilise que Get : passe.
f := fakeStore{getFn: func(id string) ([]byte, error) { return []byte("ok"), nil }}
data, _ := f.Get("x")

// Test qui appelle Delete : panique immédiatement.
_ = f.Delete("x")
// panic: runtime error: invalid memory address or nil pointer dereference
```

**Pourquoi cette panique est préférable à un faux « complet » retournant des zéro-valeurs :**

Un faux qui retourne `nil, nil` pour `Delete` laisse le test **passer** alors que le code
appelle une méthode que le test ne vérifie pas. Le test devient mensonger : il affirme un
comportement qu'il n'a pas observé.

La panique, elle, est un échec **bruyant et immédiat**, avec une trace qui pointe la ligne
exacte. Elle transforme une hypothèse implicite (« ce code n'appelle que `Get` ») en une
assertion vérifiée par l'exécution. C'est le principe du *fail fast* : mieux vaut un test qui
explose qu'un test qui ment.

Nuance : ce comportement est souhaitable **en test**, jamais en production. En production, un
champ embarqué nil est un bug de construction — d'où l'intérêt d'un constructeur qui refuse le
nil.

**b) Embedding et API publique**

```go
type SafeMap struct {
	sync.Mutex // EMBARQUÉ : Lock() et Unlock() deviennent PUBLIQUES
	data map[string]int
}

func (m *SafeMap) Set(k string, v int) {
	m.Lock()
	defer m.Unlock()
	m.data[k] = v
}
```

Le code appelant, de son point de vue, n'écrit rien d'incorrect :

```go
m.Lock()          // je veux faire plusieurs opérations atomiquement
m.Set("a", 1)     // ← INTERBLOCAGE : Set tente de reprendre un mutex déjà tenu
m.Unlock()
```

`sync.Mutex` n'est **pas réentrant** en Go — c'est un choix délibéré de l'équipe Go, qui
considère qu'un mutex réentrant masque des erreurs de conception. Le programme se fige, sans
message, jusqu'à ce que le runtime détecte que toutes les goroutines dorment :
`fatal error: all goroutines are asleep - deadlock!`.

La correction :
```go
type SafeMap struct {
	mu   sync.Mutex        // champ NOMMÉ et NON EXPORTÉ
	data map[string]int
}
```

**La leçon générale :** embarquer un type **exporte toutes ses méthodes exportées**. On élargit
donc son API publique sans y penser, et l'on s'engage à maintenir un contrat qu'on n'a pas
écrit. La question à se poser avant chaque embedding : *est-ce que je veux vraiment que mon
type soit utilisable comme celui-là ?* Pour un mutex, la réponse est presque toujours non.

**c) Trois conceptions**

Notifications avec journalisation, réessai et limitation de débit.

| | Embedding | Champ nommé + délégation | Fonctions en paramètre |
|---|---|---|---|
| Lignes de code | minimal | +1 méthode par méthode déléguée | minimal |
| Lisibilité de l'ordre | claire (pile visible) | claire | claire |
| Testabilité | excellente (faux partiel) | excellente | excellente |
| Ajout d'un comportement | un type de plus | un type de plus | une fonction de plus |
| Risque d'API accidentelle | **élevé** | nul | nul |
| Composition dynamique | facile | facile | facile |
| Contrat multi-méthodes | naturel | naturel | lourd (une fonction par méthode) |

**Il n'y a pas de vainqueur absolu, et le critère est net :**

- **Une seule méthode** dans le contrat → une **fonction** suffit. C'est plus léger, et cela
  évite de déclarer un type pour rien. C'est ce que fait `http.HandlerFunc`.
- **Plusieurs méthodes**, et le décorateur n'en modifie qu'une ou deux → **embedding
  d'interface**. C'est ce que fait `httptest` avec `http.ResponseWriter`.
- **Plusieurs méthodes**, et l'on veut contrôler exactement ce qui est exposé → **champ nommé**
  et délégation explicite. Verbeux, mais aucune méthode ne fuit par accident.

Le seul choix réellement risqué est l'embedding sur un type dont on ne maîtrise pas l'API —
`sync.Mutex`, un client HTTP, un logger tiers.

## Réponses du quiz

1. Un champ déclaré **sans nom**, dont le type sert implicitement de nom de champ : `Base` est
   accessible via `d.Base`.
2. Les champs et méthodes du type embarqué deviennent accessibles **directement** sur le type
   extérieur : `d.Name` est réécrit en `d.Base.Name` à la compilation.
3. **Non.** L'embedding est une relation de **contenance**, pas de parenté. Il n'existe aucun
   sous-typage en Go ; le polymorphisme passe uniquement par les interfaces.
4. **Celle de `Base`.** `Describe` est compilée dans le contexte de `Base` et ne connaît pas
   `Derived`. Il n'y a pas de dispatch virtuel.
5. **Erreur de compilation** : `ambiguous selector`. Go refuse d'arbitrer entre deux
   promotions de même profondeur.
6. Par le nom du champ embarqué : `d.Base.Greet()`.
7. Toutes les méthodes non redéfinies sont **promues** depuis la valeur embarquée : on n'écrit
   que celles qu'on décore. C'est ce qui rend le décorateur et le faux partiel économiques.
8. Toute méthode **non redéfinie** provoque une panique `nil pointer dereference` au moment de
   l'appel, sans indication claire de la cause.
9. Parce que cela rend `Lock()` et `Unlock()` **publiques** : n'importe quel appelant peut
   verrouiller le mutex depuis l'extérieur et provoquer un interblocage, `sync.Mutex` n'étant
   pas réentrant.
10. Dès que le type extérieur ne doit **pas** être utilisable comme le type intérieur —
    c'est-à-dire la plupart du temps. `s.logger.Info()` est plus clair et plus sûr que
    `s.Info()`.
