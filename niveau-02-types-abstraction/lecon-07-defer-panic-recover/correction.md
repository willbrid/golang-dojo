# Leçon 7 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Ordre et évaluation

```
fin du corps 3
C 2
C 1
C 0
B 3
A 1
```

Ligne par ligne :

- **`fin du corps 3`** — le corps s'exécute d'abord, `x` vaut alors 3.
- **`C 2`, `C 1`, `C 0`** — trois `defer` planifiés dans la boucle, exécutés en **LIFO**. Leur
  argument `i` a été **évalué à la planification**, d'où 0, 1, 2 mémorisés puis restitués à
  l'envers.
- **`B 3`** — la closure ne lit `x` qu'au moment de son exécution, à la sortie de `main` : `x`
  vaut alors 3.
- **`A 1`** — l'argument a été évalué **immédiatement**, quand `x` valait encore 1.

Les deux lignes `A` et `B` résument tout le piège : même variable, deux valeurs, selon qu'on
passe l'argument ou qu'on le lit dans une closure.

## E2 — `defer` en boucle

```
too many open files
```

Le nombre exact dépend de `ulimit -n` (souvent 1024 ou 8192). Les `defer` s'accumulent : aucun
`Close` ne s'exécute avant la sortie de la **fonction**, pas de l'itération.

```go
// A : extraire une fonction (préférable)
func processOne(path string) error {
	f, err := os.Open(path)
	if err != nil {
		return err
	}
	defer f.Close()   // s'exécute à la fin de processOne, donc à chaque tour
	return process(f)
}

// B : fonction anonyme immédiate
for _, p := range paths {
	func() {
		f, err := os.Open(p)
		if err != nil {
			return
		}
		defer f.Close()
		process(f)
	}()
}
```

**A est préférable** : la fonction est nommée, testable isolément, et peut retourner une
erreur. B introduit un niveau d'indentation et une closure anonyme dont le `return` prête à
confusion — il quitte la closure, pas la boucle.

## E3 — Résultat nommé

```go
type failingWriter struct{ closed bool }

func (w *failingWriter) Write(p []byte) (int, error) { return len(p), nil }
func (w *failingWriter) Close() error {
	if w.closed {
		return errors.New("déjà fermé")
	}
	w.closed = true
	return errors.New("échec du vidage : disque plein") // simulation
}

// Version NAÏVE : l'erreur de Close est perdue
func writeNaive(w io.WriteCloser, data []byte) error {
	defer w.Close()          // erreur ignorée
	_, err := w.Write(data)
	return err               // retourne nil alors que l'écriture est incomplète
}

// Version CORRECTE : résultat nommé
func writeSafe(w io.WriteCloser, data []byte) (err error) {
	defer func() {
		if cerr := w.Close(); cerr != nil && err == nil {
			err = fmt.Errorf("fermeture : %w", cerr)
		}
	}()
	if _, werr := w.Write(data); werr != nil {
		return fmt.Errorf("écriture : %w", werr)
	}
	return nil
}
```

`writeNaive` retourne `nil` : le programme croit avoir écrit, le fichier est tronqué. C'est un
bug de **perte de données**, la pire catégorie, et il est invisible en test tant qu'on ne
simule pas l'échec de `Close`.

Le `&& err == nil` est essentiel : si l'écriture a déjà échoué, son erreur est plus
informative que celle de la fermeture. On n'écrase jamais une erreur existante.

## E4 — `recover` mal placé

```go
// A : recover appelé directement dans la fonction → INEFFICACE
func a() {
	recover()   // retourne nil : aucune panique en cours à ce point
	panic("boom")
}

// B : recover dans une fonction appelée par le defer → INEFFICACE
func helper() { recover() }
func b() {
	defer helper()   // recover n'est PAS appelé « directement » par le defer
	panic("boom")
}

// C : recover dans la fonction différée elle-même → FONCTIONNE
func c() (err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("rattrapé : %v", r)
		}
	}()
	panic("boom")
}
```

La spécification est explicite : `recover` n'a d'effet que s'il est appelé **directement par
une fonction différée**. En B, `helper` est différée, mais `recover` y est appelé par `helper`,
pas par le `defer` — un niveau d'indirection de trop.

Cette règle n'est pas une bizarrerie : elle empêche qu'une fonction utilitaire quelconque
avale une panique à distance, à l'insu de celui qui l'appelle.

## E5 — `recover` et goroutines

```go
func main() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("rattrapé :", r) // ne s'affiche JAMAIS
		}
	}()
	go func() { panic("boom depuis une goroutine") }()
	time.Sleep(time.Second)
	fmt.Println("jamais atteint")
}
```

```
panic: boom depuis une goroutine

goroutine 6 [running]:
main.main.func2()
	/tmp/main.go:14 +0x25
created by main.main in goroutine 1
exit status 2
```

Le programme **meurt**, malgré le `recover` de `main`.

**Les trois phrases attendues :** `recover` ne rattrape que les paniques de **sa propre
goroutine** — la pile déroulée est celle de la goroutine qui panique, et elle ne passe jamais
par `main`. Une panique non rattrapée dans **n'importe quelle** goroutine termine tout le
processus, quel que soit le nombre de `recover` ailleurs. Toute goroutine dont le corps peut
paniquer — c'est-à-dire toute goroutine exécutant du code non trivial — doit donc porter sa
propre protection.

```go
go func() {
	defer func() {
		if r := recover(); r != nil {
			log.Printf("goroutine : %v\n%s", r, debug.Stack())
		}
	}()
	work()
}()
```

C'est la première cause d'arrêt inexpliqué en production Go. Sujet approfondi au niveau 6.

## Exercice intermédiaire — `resource`

```go
package main

import (
	"errors"
	"fmt"
	"sync/atomic"
)

type Resource struct {
	Name   string
	closed bool
}

var ErrAlreadyClosed = errors.New("ressource déjà fermée")

func (r *Resource) Close() error {
	if r.closed {
		return fmt.Errorf("%s : %w", r.Name, ErrAlreadyClosed)
	}
	r.closed = true
	return nil
}

func (r *Resource) Use(op string) error {
	if r.closed {
		return fmt.Errorf("%s : usage après fermeture", r.Name)
	}
	if op == "boom" {
		return errors.New("opération invalide")
	}
	return nil
}

type Pool struct {
	acquired atomic.Int64
	released atomic.Int64
}

func (p *Pool) Acquire(name string) (*Resource, error) {
	if name == "" {
		return nil, errors.New("nom de ressource vide")
	}
	p.acquired.Add(1)
	return &Resource{Name: name}, nil
}

func (p *Pool) Balance() (acq, rel int64) {
	return p.acquired.Load(), p.released.Load()
}

// WithResource acquiert, exécute fn, et libère TOUJOURS — succès, erreur ou panique.
// Le résultat nommé err est indispensable : c'est le defer qui l'ajuste.
func (p *Pool) WithResource(name string, fn func(*Resource) error) (err error) {
	r, err := p.Acquire(name)
	if err != nil {
		return fmt.Errorf("acquisition de %q : %w", name, err)
	}

	defer func() {
		// 1. Libérer d'abord : la ressource doit partir quoi qu'il arrive.
		p.released.Add(1)
		cerr := r.Close()

		// 2. Rattraper une éventuelle panique de fn, APRÈS libération.
		if rec := recover(); rec != nil {
			err = fmt.Errorf("ressource %q : panique rattrapée : %v", name, rec)
		}

		// 3. Ne jamais écraser une erreur existante ; les combiner.
		if cerr != nil {
			if err == nil {
				err = fmt.Errorf("fermeture : %w", cerr)
			} else {
				err = errors.Join(err, fmt.Errorf("fermeture : %w", cerr))
			}
		}
	}()

	return fn(r)
}
```

**Les quatre points de correction :**

1. **L'ordre dans le `defer` compte.** Libérer **avant** de rattraper la panique garantit que
   la ressource part même si le traitement du `recover` échouait. Inverser l'ordre fonctionne
   dans ce code, mais est plus fragile.
2. **`errors.Join` pour la contrainte 3.** Quand `fn` échoue **et** que `Close` échoue, on a
   deux informations indépendantes. Les combiner, plutôt que d'en choisir une, est la seule
   réponse honnête — et `errors.Is` traverse les deux branches.
3. **Le compteur.** `acquired` est incrémenté dans `Acquire`, `released` dans le `defer` : ils
   ne peuvent pas diverger, quel que soit le chemin de sortie. C'est ce que `defer` garantit et
   qu'un `Close()` placé à la main en fin de fonction ne garantit pas. `atomic.Int64` anticipe
   l'usage concurrent (niveau 6) ; un simple `int64` suffirait ici.
4. **Le résultat nommé.** Sans `(err error)` dans la signature, le `defer` s'exécuterait mais
   ses modifications de `err` seraient sans effet : la valeur de retour est figée au moment du
   `return`.

**Sur le patron lui-même.** Go n'a ni `with` (Python) ni `try-with-resources` (Java). Il a
`defer` et les fonctions d'ordre supérieur, dont la combinaison donne le même résultat — en
plus explicite et sans syntaxe dédiée. `WithResource` est la forme canonique, et on la retrouve
partout en production : `db.WithTx(func(tx *sql.Tx) error {…})` au niveau 8,
`WithLock(func() error {…})` au niveau 6.

## Défi

**a) Paniques converties en erreurs**

```go
// parseError est un type NON EXPORTÉ : c'est la clé du mécanisme.
type parseError struct{ msg string; pos int }

func (e parseError) Error() string { return fmt.Sprintf("position %d : %s", e.pos, e.msg) }

func fail(pos int, format string, args ...any) {
	panic(parseError{msg: fmt.Sprintf(format, args...), pos: pos})
}

// Parse est la frontière PUBLIQUE : aucune panique n'en sort.
func Parse(input string) (result int, err error) {
	defer func() {
		r := recover()
		if r == nil {
			return
		}
		// On ne rattrape QUE nos propres paniques.
		if pe, ok := r.(parseError); ok {
			err = fmt.Errorf("analyse de %q : %w", input, pe)
			return
		}
		panic(r) // toute autre panique est RELANCÉE : ce n'est pas la nôtre
	}()
	return parseExpr(input), nil
}
```

**Le risque de la technique**, et sa parade : si l'analyseur appelle du code tiers — un
rappel fourni par l'utilisateur, une bibliothèque — une panique venue de ce code serait
silencieusement convertie en « erreur d'analyse ». Un vrai bug (déréférencement nil,
dépassement d'indice) serait maquillé en erreur de syntaxe, et le diagnostic deviendrait
impossible.

La parade est le **type non exporté** : personne d'autre ne peut fabriquer un `parseError`.
L'assertion `r.(parseError)` distingue donc à coup sûr nos paniques des autres, et le
`panic(r)` relance ce qui ne nous appartient pas — en préservant la trace de pile d'origine.

C'est exactement ce que fait `encoding/json` en interne.

**b) Protection de goroutines**

```go
// Go lance fn dans une goroutine protégée et rapporte une éventuelle panique
// sous forme d'erreur, avec la trace de pile du point de panique.
func Go(fn func()) <-chan error {
	ch := make(chan error, 1) // BUFFERISÉ : sinon la goroutine fuit si personne ne lit
	go func() {
		defer close(ch)
		defer func() {
			if r := recover(); r != nil {
				ch <- fmt.Errorf("panique : %v\n%s", r, debug.Stack())
			}
		}()
		fn()
	}()
	return ch
}
```

Deux détails qui font la différence :

- **Le channel est bufferisé (capacité 1).** Sur un channel non bufferisé, l'envoi bloque
  jusqu'à ce que quelqu'un lise ; si l'appelant ignore le channel, la goroutine reste bloquée
  pour toujours — une fuite de goroutine, exactement ce que le profil `goroutineleak` de
  Go 1.27 détecte.
- **`debug.Stack()` est appelée dans le `recover`**, donc pendant le déroulement : la trace
  pointe le lieu de la panique. Appelée plus tard, elle ne montrerait plus rien d'utile.

**Pourquoi la bibliothèque standard n'en fournit pas d'équivalent :** parce que la décision
d'avaler une panique n'est **pas générique**. Selon le contexte, on veut journaliser et
continuer (un worker), redémarrer la tâche (un superviseur), ou laisser mourir le processus
(un invariant violé, où continuer serait dangereux). Go préfère laisser ce choix explicite
plutôt que d'imposer un défaut. `golang.org/x/sync/errgroup` fournit une réponse dans un cadre
précis — attendre un groupe de tâches — et pas une réponse universelle.

**c) Coût de `defer`**

| | ns/appel (ordre de grandeur) |
|---|---|
| libération explicite | ~2 ns |
| `defer` (open-coded) | ~2 à 3 ns |
| `defer` en boucle ou conditionnel | ~30 à 50 ns |

**Le « open-coded defer », introduit en Go 1.14**, permet au compilateur d'insérer directement
l'appel différé aux points de sortie, sans passer par la structure de liste chaînée du runtime.
Le surcoût devient alors quasi nul.

Il **ne s'applique pas** quand :
- le `defer` est dans une **boucle** (le nombre d'appels différés n'est pas connu à la
  compilation) ;
- la fonction contient **plus de huit** `defer` ;
- le `defer` est dans un chemin où le compilateur ne peut pas énumérer les sorties.

**Cela change-t-il la recommandation ?** Non, et c'est le point. Un surcoût de 30 ns est
invisible sauf dans une boucle exécutée des dizaines de millions de fois — et dans ce cas, le
`defer` en boucle était déjà à proscrire pour la raison bien plus grave de l'exercice E2. La
règle reste : **utiliser `defer` systématiquement, et ne s'en priver qu'après une mesure**
montrant qu'il domine le profil. Ce qui, en pratique, n'arrive presque jamais.

## Réponses du quiz

1. À la sortie de la **fonction englobante** — pas du bloc, pas de l'itération de boucle.
2. En **LIFO** : dernier planifié, premier exécuté.
3. **Immédiatement**, au moment où le `defer` est rencontré. Seul l'**appel** est différé.
4. **Oui** après une panique, pendant le déroulement de la pile. **Non** après `os.Exit`, qui
   termine le processus sans rien exécuter.
5. Parce que les ressources s'accumulent jusqu'à la sortie de la fonction : on épuise les
   descripteurs de fichiers. Remède : extraire le corps dans une fonction.
6. Parce que le `defer` s'exécute **après** l'affectation du résultat mais **avant** le retour
   effectif. Sans nom, il n'y a pas de variable à modifier — la valeur retournée est déjà figée.
7. En écriture, `Close` déclenche le vidage des tampons et peut échouer (disque plein, quota) :
   l'ignorer produit un fichier tronqué signalé comme un succès. En lecture, l'échec de `Close`
   n'affecte pas les données déjà lues.
8. **Directement dans une fonction différée.** Ailleurs — y compris dans une fonction appelée
   par la fonction différée — il retourne `nil` et ne fait rien.
9. **Non.** Il ne protège que sa propre goroutine. Une panique non rattrapée dans n'importe
   quelle goroutine tue tout le processus.
10. `panic` : les fonctions `MustXxx` (`regexp.MustCompile`) et un invariant interne violé qui
    signale un bug. `recover` : la frontière d'un serveur (une requête ne doit pas tuer le
    processus) et la frontière d'une bibliothèque utilisant `panic` en interne.
11. `fatal error: concurrent map writes` et
    `fatal error: all goroutines are asleep - deadlock!`. Ce sont des **erreurs fatales** du
    runtime, pas des paniques : il refuse délibérément de laisser le programme continuer dans
    un état incohérent.
