# Leçon 5 — Exercices

Code dans `mes-solutions/niveau-01/lecon-05/`.

## Exercices faciles

**E1 — Retours multiples.**
Écrire `MinMax(nums ...int) (min, max int, err error)` qui retourne le minimum, le
maximum et une erreur si aucun nombre n'est fourni. Traiter l'erreur à l'appel.
*Question à se poser : pourquoi une erreur plutôt que de retourner `0, 0` ?*

**E2 — Variadique et étalement.**
Écrire `Sum(nums ...int) int`, l'appeler de trois façons : sans argument, avec trois
littéraux, avec un slice étalé. Puis écrire une fonction variadique qui **modifie** son
paramètre (`nums[0] = 999`) et vérifier ce qui arrive au slice de l'appelant. Expliquer.

**E3 — Compteur.**
Écrire `Counter() func() int` qui retourne une closure incrémentant un compteur privé.
Créer deux compteurs et vérifier qu'ils sont indépendants. Expliquer où vit la variable capturée.

**E4 — Fonction en paramètre.**
Écrire `Apply(nums []int, f func(int) int) []int` qui retourne un nouveau slice avec `f`
appliquée à chaque élément. L'utiliser avec une fonction nommée, puis avec une fonction anonyme.
*L'original ne doit pas être modifié — c'est la contrainte réelle de l'exercice.*

**E5 — Masquage.**
Ce code compile mais est faux. Identifier le bug, l'expliquer, le corriger :
```go
func load() error {
	cfg, err := readConfig()
	if err != nil {
		return err
	}
	if cfg.Debug {
		cfg, err := applyDebug(cfg)
		if err != nil {
			return err
		}
		fmt.Println(cfg.Level)
	}
	return use(cfg)
}
```

---

## Exercice intermédiaire — `retry`

Écrire une fonction générique de réessai — sans génériques, ils viendront au niveau 2.

```go
// Retry exécute op jusqu'à attempts fois, en attendant delay entre deux essais.
// Elle retourne nil au premier succès, ou la dernière erreur.
func Retry(attempts int, delay time.Duration, op func() error) error
```

**Contraintes :**
1. `attempts <= 0` est une erreur de programmation : retourner une erreur explicite, ne pas paniquer.
2. Aucune attente **après** la dernière tentative — un `Retry(3, time.Second, …)` qui échoue doit durer 2 s, pas 3 s.
3. L'erreur finale doit indiquer le nombre de tentatives effectuées et **envelopper** la dernière erreur avec `%w`.
4. Ajouter une variante `RetryBackoff` où le délai **double** à chaque échec.
5. Écrire dans `main` un scénario de démonstration : une opération qui échoue deux fois puis réussit (utiliser une closure avec un compteur — c'est le point de l'exercice).
6. `Retry` ne doit **rien afficher**. Pour observer ce qui se passe, ajouter un paramètre… ou pas. **Trancher et justifier le choix en commentaire.**

*Ce patron est réellement utilisé en production ; il reviendra au niveau 7 avec `context`,
le jitter et les circuit breakers.*

---

## Défi

**Un pipeline de transformations composables.**

```go
type Transform func(string) string

// Compose retourne une Transform qui applique t1, puis t2, puis … dans l'ordre.
func Compose(ts ...Transform) Transform
```

**a)** Implémenter `Compose`, puis construire un pipeline `trim → lowercase → remplacer les
espaces par des tirets` et l'appliquer à `"  Bonjour Le Monde  "`.

**b)** `Compose()` sans argument doit retourner une transformation valide (l'identité), pas `nil`.

**c)** Ajouter `ComposeErr(ts ...func(string) (string, error)) func(string) (string, error)`
qui s'arrête à la première erreur en indiquant **quelle étape** a échoué.

**d)** Question de conception, à répondre en quelques lignes : `Compose` construit sa chaîne
au moment de l'appel de `Compose`, ou au moment de l'appel du résultat ? Les deux
implémentations sont possibles. Laquelle est préférable, et pourquoi ? Que se passe-t-il si
le slice `ts` est modifié entre les deux ?

---

## Quiz

1. Où doit se trouver `error` dans la liste des résultats ? Pourquoi cette convention ?
2. Que reçoit une fonction variadique appelée sans aucun argument ?
3. `f(xs...)` copie-t-il le slice `xs` ?
4. Qu'est-ce qu'une closure capture : une copie de la variable, ou la variable ?
5. Où vit une variable locale capturée par une closure retournée ?
6. Quand un `return` nu est-il acceptable ?
7. Que se passe-t-il si on appelle une variable de fonction valant `nil` ?
8. Pourquoi `data, err := f()` dans un `if` imbriqué peut-il faire perdre une erreur ?
9. Combien de valeurs une fonction Go peut-elle retourner ?
