# Leçon 9 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — MinMax

```go
func MinMax(nums ...int) (lo, hi int, ok bool) {
	if len(nums) == 0 {
		return 0, 0, false
	}
	lo, hi = nums[0], nums[0]
	for _, n := range nums[1:] {
		lo = min(lo, n) // min et max sont intégrés depuis Go 1.21
		hi = max(hi, n)
	}
	return lo, hi, true
}
```

**`(T, bool)` ou `(T, error)` ?** La distinction est sémantique, pas stylistique :

- **`(T, bool)`** quand l'absence de résultat est un **cas normal** que l'appelant sait
  traiter sans avoir besoin d'explication. `v, ok := m[k]` en est l'archétype : une clé absente
  n'est pas une anomalie.
- **`(T, error)`** quand quelque chose a **mal tourné** et que l'appelant a besoin de savoir
  *quoi*, éventuellement pour le journaliser ou le remonter.

Ici, appeler `MinMax()` sans argument n'est pas une panne : c'est un cas limite trivial. Un
`bool` suffit — construire un message d'erreur que personne ne lira serait du gaspillage.
Retenir la question qui tranche : **l'appelant a-t-il besoin d'un message ?**

## E2 — Passage par valeur

```go
func modifyInt(n int)                  { n = 99 }          // invisible
func modifySliceElement(s []int)       { s[0] = 99 }       // VISIBLE
func appendToSlice(s []int)            { s = append(s, 9) } // invisible
func modifyMap(m map[string]int)       { m["k"] = 99 }     // VISIBLE
```

**La règle en une phrase :** tout est copié ; modifier le **contenu partagé** est visible,
remplacer le **descripteur local** ne l'est pas.

Un slice, une map ou un channel contiennent un pointeur interne. Copier le descripteur (3 mots
pour un slice, 1 pour une map) duplique la référence, pas la donnée. En revanche, `s = append(…)`
réaffecte la variable locale `s` : l'appelant garde son ancien descripteur.

## E3 — Join variadique

```go
func Join(sep string, parts ...string) string {
	if len(parts) == 0 {
		return ""
	}
	out := parts[0]
	for _, p := range parts[1:] {
		out += sep + p // acceptable ici ; en boucle longue, strings.Builder (leçon 8)
	}
	return out
}
```

`Join(",")` doit retourner `""` : c'est le seul choix cohérent, car `strings.Join(nil, ",")`
fait de même. Retourner le séparateur, ou paniquer, casserait la composition.

Le piège est le séparateur **entre** les éléments : partir de `parts[0]` puis préfixer chaque
suivant évite le séparateur en trop à la fin, sans `if` dans la boucle.

## E4 — Le piège de l'étalement

```go
func corrupt(nums ...int) {
	if len(nums) > 0 {
		nums[0] = 999
	}
}

xs := []int{1, 2, 3}
corrupt(xs...)
fmt.Println(xs)   // [999 2 3]  ← l'original est modifié !

corrupt(1, 2, 3)  // ici Go construit un slice temporaire : rien à corrompre
```

`f(xs...)` ne copie rien : le slice reçu partage le tableau de `xs`.

Deux protections :
1. **Côté fonction** : copier au début si l'on doit modifier — `nums = slices.Clone(nums)`.
2. **Côté appelant** : passer une copie — `corrupt(slices.Clone(xs)...)`.

La première est préférable : elle place la responsabilité là où est la connaissance. Et dans
tous les cas, une fonction variadique qui modifie son paramètre **doit le documenter**.

## E5 — Masquage

Le bug est à la ligne `cfg, err := applyDebug(cfg)`. Le `:=` déclare **deux nouvelles
variables** locales au bloc `if`. Le `cfg` transformé n'existe que dans ce bloc ; à la sortie,
`use(cfg)` reçoit la configuration **d'origine**, non transformée.

Le compilateur ne dit rien : les deux variables sont bien utilisées à l'intérieur du bloc.

```go
if cfg.Debug {
	var err error
	cfg, err = applyDebug(cfg)   // = et non := : on réaffecte les variables externes
	if err != nil {
		return err
	}
	fmt.Println(cfg.Level)
}
```

`golangci-lint` avec le linter `shadow` le détecte ; `go vet` seul, non. Retenir la règle de
lecture : **`:=` dans un bloc imbriqué déclare toujours de nouvelles variables**, même si les
noms existent déjà à l'extérieur.

## Exercice intermédiaire — `textstats`

```go
package main

import (
	"strings"
	"unicode"
)

// CountWords retourne le nombre de mots d'un texte.
// Un mot est une suite de caractères séparée par des espaces.
func CountWords(text string) int {
	return len(strings.Fields(text))
}

// CountSentences compte les phrases.
// CHOIX DOCUMENTÉ : une suite de ponctuations finales consécutives ("...", "!?")
// compte pour UNE phrase. Les abréviations ("Dr.", "etc.") sont comptées à tort
// comme des fins de phrase : c'est une limite assumée, un traitement correct
// exigerait un dictionnaire d'abréviations.
func CountSentences(text string) int {
	n := 0
	prevWasEnd := false
	for _, r := range text {
		isEnd := r == '.' || r == '!' || r == '?'
		if isEnd && !prevWasEnd {
			n++
		}
		prevWasEnd = isEnd
	}
	if n == 0 && strings.TrimSpace(text) != "" {
		n = 1 // un texte sans ponctuation finale reste une phrase
	}
	return n
}

// CountSyllables approxime le nombre de syllabes en comptant les GROUPES de
// voyelles. Heuristique volontairement simple : elle se trompe sur les hiatus
// ("oasis"), les diphtongues et le e muet final. Suffisante pour un indice de
// lisibilité, inutilisable pour de la phonétique.
func CountSyllables(word string) int {
	n := 0
	inVowel := false
	for _, r := range strings.ToLower(word) {
		isVowel := strings.ContainsRune("aeiouyàâäéèêëîïôöùûü", r)
		if isVowel && !inVowel {
			n++
		}
		inVowel = isVowel
	}
	if n == 0 && word != "" {
		n = 1 // tout mot prononçable a au moins une syllabe
	}
	return n
}

// AverageWordLength retourne la longueur moyenne des mots, en RUNES.
// Retourne 0 sur un texte vide : aucune division par zéro possible.
func AverageWordLength(text string) float64 {
	words := strings.Fields(text)
	if len(words) == 0 {
		return 0
	}
	total := 0
	for _, w := range words {
		total += len([]rune(w))
	}
	return float64(total) / float64(len(words)) // conversion AVANT la division
}

// Readability retourne un indice de lisibilité et son interprétation.
// Les résultats sont nommés parce que (float64, string) seul serait ambigu.
func Readability(text string) (score float64, level string) {
	words := strings.Fields(text)
	sentences := CountSentences(text)
	if len(words) == 0 || sentences == 0 {
		return 0, "texte vide"
	}
	syll := 0
	for _, w := range words {
		syll += CountSyllables(strings.TrimFunc(w, func(r rune) bool {
			return !unicode.IsLetter(r)
		}))
	}
	wps := float64(len(words)) / float64(sentences)
	spw := float64(syll) / float64(len(words))
	score = 207 - 1.015*wps - 73.6*spw // formule de Kandel-Moles (français)

	switch {
	case score >= 80:
		level = "très facile"
	case score >= 60:
		level = "facile"
	case score >= 40:
		level = "moyen"
	default:
		level = "difficile"
	}
	return score, level
}
```

**Ce qui est évalué :**

1. **Aucune fonction n'affiche.** Toutes retournent des valeurs. C'est ce qui les rendra
   testables au niveau 5 sans capturer de sortie ni simuler de terminal. Si une seule avait
   contenu `fmt.Println`, l'exercice serait raté quel que soit le reste.
2. **Aucune division par zéro.** Chaque fonction traite le cas vide explicitement, en amont.
3. **Les conversions précèdent les divisions** — le piège de la leçon 3.
4. **Les limites sont documentées.** Une heuristique dont on ne dit pas qu'elle en est une
   devient un bug dans six mois, quand quelqu'un s'en servira pour autre chose.
5. **Le comptage en runes** et non en octets, sinon « été » compte pour 5 caractères.

*Note : `strings.TrimFunc` prend une fonction en paramètre — c'est le sujet de la leçon 11.
Une version utilisant `strings.Trim` avec une liste de ponctuations est tout aussi acceptable
à ce stade.*

## Défi

**a) Mémoïsation**

```go
var fibCache = map[int]int{}

func fibMemo(n int) int {
	if n <= 1 {
		return n
	}
	if v, ok := fibCache[n]; ok {
		return v
	}
	v := fibMemo(n-1) + fibMemo(n-2)
	fibCache[n] = v
	return v
}
```

`fib(40)` : environ **700 ms** en récursion naïve (plus de 300 millions d'appels), moins
d'une **microseconde** avec le cache. La complexité passe de O(φⁿ) à O(n).

**Deux défauts, tous deux graves :**

1. **L'état global.** `fibCache` est une variable de package : le cache est partagé par tout le
   programme, ne peut pas être vidé ni dimensionné, grossit indéfiniment, et rend la fonction
   impossible à tester de façon isolée — deux tests successifs ne partent pas du même état.
   La solution est une **closure** qui capture sa propre map (leçon 11) : chaque
   « mémoïseur » a alors son cache privé.
2. **La concurrence.** Deux goroutines appelant `fibMemo` simultanément écrivent dans la même
   map, ce qui déclenche `fatal error: concurrent map writes` — une erreur **fatale**, non
   rattrapable par `recover`. Cette fonction est une bombe à retardement dans tout programme
   concurrent. Correction au niveau 6 : `sync.Mutex`, `sync.Map` ou `sync.OnceValue`.

**b) Récursion et pile**

```
runtime: goroutine stack exceeds 1000000000-byte limit
fatal error: stack overflow
```

La limite est de **1 Go par goroutine** par défaut, réglable via `debug.SetMaxStack`.

Une récursion de profondeur 100 000 fonctionne sans problème : la pile d'une goroutine démarre
à environ **8 Ko** et **grandit dynamiquement** — le runtime la recopie dans une zone plus
grande quand elle sature (*stack growth*).

Un thread système C, lui, reçoit une pile de taille **fixe** (typiquement 8 Mo) à sa création,
réservée d'avance. C'est précisément ce qui rend les goroutines si peu coûteuses : on peut en
lancer un million, elles consomment quelques kilo-octets chacune au départ. Un million de
threads système demanderait des téraoctets d'espace d'adressage.

**c) Conception d'API**

| Signature | Appel | Évolutivité | Défauts par défaut | Découvrabilité |
|---|---|---|---|---|
| `Split(s, sep, maxParts, trim, skipEmpty)` | `Split(s, ",", -1, true, false)` — illisible | mauvaise : chaque ajout casse les appels | impossibles | mauvaise : que veut dire `true, false` ? |
| `Split(s, cfg SplitConfig)` | lisible | bonne | la zéro-valeur de la struct | bonne |
| `Split(s, sep, opts ...SplitOption)` | très lisible | excellente | dans le constructeur | excellente |
| Plusieurs fonctions nommées | très lisible | explosion combinatoire | n/a | excellente |

**Ce que fait réellement la bibliothèque standard : la quatrième option, volontairement
limitée.** `strings` expose `Split`, `SplitN`, `SplitAfter`, `SplitAfterN`, `Fields`,
`FieldsFunc`, `Cut`, `CutPrefix`, `CutSuffix` — des fonctions distinctes, chacune faisant une
chose, aucune avec de la configuration.

La leçon est plus profonde qu'il n'y paraît : **la stdlib refuse le problème plutôt que de le
résoudre**. Plutôt qu'une fonction paramétrable qui fait tout, elle en offre plusieurs qui
font chacune une chose évidente, et laisse l'utilisateur composer. C'est une constante du
style Go, et cela explique pourquoi les options fonctionnelles n'apparaissent presque que dans
les bibliothèques tierces, sur des objets réellement complexes (un serveur, un client gRPC) —
jamais sur une fonction utilitaire.

## Réponses du quiz

1. **Non** aux deux. Pas de surcharge (deux fonctions ne peuvent pas partager un nom dans un
   package), pas d'arguments par défaut. L'équivalent idiomatique des seconds est le patron des
   options fonctionnelles (leçon 11).
2. **Non** : le paramètre est une copie.
3. **Oui** pour l'élément (le tableau sous-jacent est partagé). **Non** pour `append` : il
   réaffecte le descripteur local.
4. **En dernier**. Un seul paramètre variadique par signature.
5. Un slice `nil`, de longueur 0. `range`, `len` et `append` fonctionnent dessus.
6. **Non** : le slice reçu partage le tableau sous-jacent de `xs`.
7. À documenter une signature ambiguë, et à permettre à un `defer` de modifier le résultat
   (niveau 2). Le `return` nu est acceptable dans une fonction courte, jamais au-delà.
8. Il déclare de **nouvelles** variables dans ce bloc, masquant celles de l'extérieur. La
   valeur affectée est perdue à la sortie du bloc.
9. `golangci-lint` avec le linter `shadow`. **`go vet` seul ne suffit pas.**
10. Autant qu'on veut. Au-delà de trois, une struct nommée devient plus lisible et survit mieux
    aux évolutions de la signature.
