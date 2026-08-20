# Leçon 9 — Exercices

Code dans `mes-solutions/niveau-01/lecon-09/`.

## Exercices faciles

**E1 — Retours multiples.**
Écrire `MinMax(nums ...int) (min, max int, ok bool)` qui retourne le minimum, le maximum et
`false` si aucune valeur n'est fournie. Traiter le troisième résultat à l'appel.
*Question à trancher par écrit : pourquoi `(T, bool)` ici plutôt que `(T, error)` ?
Qu'est-ce qui distingue les deux idiomes ?*

**E2 — Passage par valeur.**
Écrire quatre fonctions : `modifyInt(int)`, `modifySliceElement([]int)`,
`appendToSlice([]int)`, `modifyMap(map[string]int)`. Pour chacune, **prédire** si l'appelant
verra le changement, puis vérifier. Rédiger la règle générale en une phrase.

**E3 — Variadique.**
Écrire `Join(sep string, parts ...string) string` sans utiliser `strings.Join`. L'appeler
sans partie, avec une seule, avec trois, puis avec un slice étalé. Que doit retourner
`Join(",")` ?

**E4 — Le piège de l'étalement.**
Écrire une fonction variadique qui modifie `nums[0]`, l'appeler avec un slice étalé, et
constater ce qui arrive au slice de l'appelant. Puis proposer deux façons de s'en protéger.

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
Puis installer `golangci-lint` et vérifier que le linter `shadow` le détecte.

---

## Exercice intermédiaire — `textstats`

Une petite bibliothèque de statistiques de texte, entièrement composée de fonctions pures.

```go
func CountWords(text string) int
func CountSentences(text string) int
func CountSyllables(word string) int          // approximation, règles à définir
func AverageWordLength(text string) float64
func Readability(text string) (score float64, level string)
```

**Contraintes :**
1. **Aucune fonction n'affiche quoi que ce soit** ni ne lit de fichier. `main` seul fait des entrées/sorties.
2. Aucune fonction ne modifie son argument.
3. Une phrase se termine par `.`, `!` ou `?` — décider du sort de `...` et de `Dr.` et **documenter le choix**.
4. `CountSyllables` utilise une heuristique simple (compter les groupes de voyelles) ; sa documentation doit dire explicitement qu'elle est approximative et sur quel corpus elle se trompe.
5. `Readability` retourne deux résultats **nommés**, parce que `(float64, string)` seul serait ambigu.
6. Un texte vide ne fait paniquer aucune fonction et ne provoque aucune division par zéro.
7. Chaque fonction exportée a un commentaire de documentation commençant par son nom.
8. `main` affiche un tableau aligné des résultats pour trois textes de test.

*L'objectif réel : produire un ensemble de fonctions qu'on pourra tester au niveau 5 sans
rien réécrire. Si une seule d'entre elles appelle `fmt.Println`, l'exercice est raté.*

---

## Défi

**a) `Memoize` manuel.**
Écrire une fonction `fibMemo(n int) int` qui utilise une `map[int]int` déclarée **hors** de
la fonction pour mémoriser les résultats. Comparer le temps de calcul de `fib(40)` avec et
sans mémoïsation.
Puis répondre : pourquoi cette solution est-elle mauvaise en l'état ? *(Deux défauts au
moins : l'un concerne l'état global, l'autre la concurrence. Le second sera traité au
niveau 6, le premier trouvera sa solution à la leçon 11.)*

**b) Récursion et pile.**
Écrire une fonction récursive qui ne se termine pas et observer le message d'erreur exact.
Puis écrire une récursion de profondeur 100 000 qui, elle, **fonctionne**. Que dit ce
résultat sur la pile des goroutines Go, comparée à celle d'un thread C ?

**c) Conception d'API.**
On veut une fonction de découpage de texte configurable : séparateur, nombre maximal de
morceaux, suppression des espaces, suppression des morceaux vides. Comparer par écrit
quatre signatures possibles :
```go
Split(s, sep string, maxParts int, trim, skipEmpty bool) []string
Split(s string, cfg SplitConfig) []string
Split(s, sep string, opts ...SplitOption) []string
SplitTrimmed(s, sep string) []string  +  Split(s, sep string) []string  + …
```
Critères : lisibilité de l'appel, évolutivité, valeurs par défaut, découvrabilité.
Laquelle choisirait la bibliothèque standard ? *(Indice : regarder `strings.SplitN` et
`strings.Cut`. Ce que fait la stdlib est parfois la quatrième option.)*

---

## Quiz

1. Go permet-il la surcharge de fonctions ? Les arguments par défaut ?
2. Une fonction peut-elle modifier un `int` passé en paramètre, vu de l'appelant ?
3. Et un élément de slice ? Et faire un `append` visible par l'appelant ?
4. Où doit se placer le paramètre variadique dans une signature ?
5. Que reçoit une fonction variadique appelée sans argument ?
6. `f(xs...)` copie-t-il le slice `xs` ?
7. À quoi servent les résultats nommés ? Quand le `return` nu est-il acceptable ?
8. Que fait `:=` dans un bloc imbriqué quand une variable du même nom existe à l'extérieur ?
9. Quel outil détecte le masquage de variables ? `go vet` suffit-il ?
10. Combien de valeurs une fonction Go peut-elle retourner, et à partir de combien vaut-il mieux une struct ?
