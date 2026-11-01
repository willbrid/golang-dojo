# Leçon 5 — Itérateurs et `range` sur fonction

> Depuis Go 1.23, `for … range` accepte une **fonction**. Ce petit ajout de syntaxe a donné
> au langage un protocole d'itération standard : les paquets `slices`, `maps`, `strings`,
> `bytes` en exposent désormais. La mécanique repose entièrement sur deux notions déjà vues :
> les **closures** (niveau 1, leçon 11) et les **génériques** (leçons 1 et 2).

## Objectifs

1. Comprendre ce qu'est un itérateur Go : une fonction qui appelle `yield`.
2. Écrire des `iter.Seq` et `iter.Seq2`, et respecter le contrat de `yield`.
3. Consommer les itérateurs de la bibliothèque standard (`maps.Keys`, `slices.Collect`,
   `strings.SplitSeq`…).
4. Savoir traiter les **erreurs** pendant une itération.

## Explication

### Le problème

Comment exposer « tous les éléments » d'une structure sans exposer la structure ?

| Approche avant 1.23 | Défaut |
|---|---|
| Rendre un slice `Items() []T` | copie tout en mémoire, même si l'appelant s'arrête au 1er |
| Un type itérateur `it.Next()`, `it.Value()` | chaque paquet invente le sien (`bufio.Scanner`, `sql.Rows`…) |
| Une fonction de rappel `Each(func(T))` | impossible de faire `break` |

### Un itérateur est une fonction

Le paquet `iter` définit deux types :

```go
type Seq[V any]     func(yield func(V) bool)
type Seq2[K, V any] func(yield func(K, V) bool)
```

Un `Seq[V]` est une fonction qui reçoit une autre fonction, `yield`, et l'appelle **une fois
par élément**. `yield` rend `false` quand le consommateur ne veut plus d'éléments.

```go
// Countdown produit n, n-1, …, 1.
func Countdown(n int) iter.Seq[int] {
	return func(yield func(int) bool) {
		for i := n; i > 0; i-- {
			if !yield(i) {
				return // le consommateur a fait break : on s'arrête
			}
		}
	}
}
```

Et on le consomme avec `range`, comme un slice :

```go
for i := range Countdown(5) {
	if i == 3 {
		break
	}
	fmt.Println(i) // 5, puis 4
}
```

### Ce que fait le compilateur

Le compilateur réécrit la boucle : **le corps de la boucle devient la fonction `yield`**.

```go
// for i := range Countdown(5) { corps }
// devient, en substance :
Countdown(5)(func(i int) bool {
	corps
	return true // continue ; un break dans le corps devient « return false »
})
```

C'est pourquoi on dit que ces itérateurs sont des itérateurs « push » : c'est l'itérateur qui
**pousse** les valeurs dans le corps de la boucle, et non la boucle qui les tire. Un `return`
dans le corps quitte bien la fonction englobante, un `defer` s'exécute à la bonne place : le
compilateur gère tout cela.

### Le contrat de `yield`

**Quand `yield` rend `false`, l'itérateur doit s'arrêter et ne plus jamais l'appeler.**
C'est la seule règle, et elle est vérifiée à l'exécution :

```go
func Bad() iter.Seq[int] {
	return func(yield func(int) bool) {
		yield(1)   // résultat ignoré !
		yield(2)
	}
}

for range Bad() { break }
// panic: runtime error: range function continued iteration after function for loop body returned false
```

Réflexe à acquérir : **tout appel à `yield` est dans un `if !yield(…) { return }`**.

### `Seq2` : des paires

`Seq2` produit des couples, consommés avec deux variables, comme `range` sur une map ou un
slice :

```go
for i, v := range slices.All(s) { … }       // (indice, valeur)
for k, v := range maps.All(m) { … }         // (clé, valeur)
```

Un `Seq` n'accepte **qu'une** variable :
`for i, w := range strings.SplitSeq(s, ",")` ne compile pas
(`permits only one iteration variable`).

### Les itérateurs de la bibliothèque standard

| Fonction | Rend | Remplace |
|---|---|---|
| `slices.All(s)`, `slices.Backward(s)` | `Seq2[int, E]` | `range s`, boucle à l'envers |
| `slices.Values(s)` | `Seq[E]` | |
| `slices.Chunk(s, n)` | `Seq[S]` — sous-slices de `n` | découpage à la main |
| `maps.Keys(m)`, `maps.Values(m)` | `Seq[K]`, `Seq[V]` | boucle qui remplit un slice |
| `maps.All(m)` | `Seq2[K, V]` | |
| `strings.SplitSeq`, `strings.FieldsSeq`, `strings.Lines` | `Seq[string]` | `Split`, `Fields` qui allouent un slice |
| `bytes.SplitSeq`, … | idem pour `[]byte` | |

Et dans l'autre sens, pour **matérialiser** un itérateur :

| Fonction | Effet |
|---|---|
| `slices.Collect(seq)` | `Seq[E]` → `[]E` |
| `slices.Sorted(seq)` | `Seq[E]` → `[]E` trié (`cmp.Ordered`) |
| `slices.SortedFunc(seq, cmp)` | idem avec comparateur |
| `maps.Collect(seq2)` | `Seq2[K, V]` → `map[K]V` |
| `slices.AppendSeq(s, seq)`, `maps.Insert(m, seq2)` | ajout à l'existant |

L'idiome le plus utile de toute la leçon, qui remplace la boucle « extraire les clés puis
trier » écrite des centaines de fois :

```go
keys := slices.Sorted(maps.Keys(m))
```

`strings.SplitSeq` mérite aussi d'être connu : sur un texte de 100 Mo, `strings.Split` alloue
un slice de toutes les lignes ; `SplitSeq` n'en alloue aucun, et on peut s'arrêter à la
première ligne intéressante.

### Composer des itérateurs

Comme un itérateur est une fonction, on peut en écrire qui en **transforment** d'autres :

```go
func Filter[V any](seq iter.Seq[V], keep func(V) bool) iter.Seq[V] {
	return func(yield func(V) bool) {
		for v := range seq {
			if keep(v) && !yield(v) {
				return
			}
		}
	}
}
```

Aucune valeur intermédiaire n'est stockée : chaque élément traverse toute la chaîne avant que
le suivant ne soit produit. C'est la réponse au problème des allocations soulevé à la leçon 3.

La bibliothèque standard ne fournit **pas** de `Map`/`Filter` sur les itérateurs, par choix :
une boucle `for` avec un `if` est souvent plus claire. Ne pas reconstruire un langage
fonctionnel au-dessus de Go.

### Les erreurs pendant l'itération

`yield` ne rend qu'un booléen : comment signaler qu'une lecture a échoué au milieu ? Deux
conceptions coexistent.

**1. `Seq2[V, error]`** — l'erreur voyage avec chaque élément :

```go
for line, err := range Lines(r) {
	if err != nil {
		return err
	}
	…
}
```

Simple, impossible à ignorer… mais le consommateur doit tester `err` à **chaque** tour.

**2. Un objet avec une méthode `Err()`** — le style de `bufio.Scanner` :

```go
sc := NewWordScanner(r)
for w := range sc.Words() { … }
if err := sc.Err(); err != nil { … }
```

Plus léger dans la boucle, mais **oublier** le `Err()` final rend l'erreur silencieuse.

Aucune n'est « la » bonne : choisir en sachant ce qu'on échange. Une API publique qui rend
un itérateur pouvant échouer **doit** documenter comment l'erreur est remontée.

### Itérateurs « pull »

Parfois, il faut tirer les valeurs une par une — par exemple pour avancer **deux** itérateurs
en parallèle et les fusionner. `iter.Pull` convertit un itérateur push en une paire de
fonctions :

```go
next, stop := iter.Pull(seq)
defer stop()            // OBLIGATOIRE : libère les ressources de l'itérateur
v, ok := next()
```

C'est un outil rare et plus coûteux (il s'appuie sur une coroutine du runtime). Le
mentionner ici suffit ; on n'en aura besoin que ponctuellement.

### Itérateurs à usage unique

La plupart des itérateurs peuvent être parcourus plusieurs fois (`slices.Values(s)` repart du
début). Mais un itérateur qui lit un `io.Reader` ne le peut pas : le flux est consommé. Un
itérateur à usage unique doit le **documenter**.

## Exemple

```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"iter"
	"maps"
	"slices"
	"strings"
	"unicode"
)

// WordScanner itère sur les mots d'un flux. Usage unique : le flux est consommé.
// Après la boucle, Err() indique si la lecture a échoué.
type WordScanner struct {
	sc *bufio.Scanner
}

func NewWordScanner(r io.Reader) *WordScanner {
	sc := bufio.NewScanner(r)
	sc.Split(bufio.ScanWords)
	return &WordScanner{sc: sc}
}

// Words rend un itérateur sur les mots normalisés (minuscules, sans ponctuation en bord).
func (w *WordScanner) Words() iter.Seq[string] {
	return func(yield func(string) bool) {
		for w.sc.Scan() {
			word := strings.ToLower(strings.TrimFunc(w.sc.Text(), unicode.IsPunct))
			if word == "" {
				continue
			}
			if !yield(word) {
				return
			}
		}
	}
}

func (w *WordScanner) Err() error { return w.sc.Err() }

// Filter : un adaptateur générique d'itérateur.
func Filter[V any](seq iter.Seq[V], keep func(V) bool) iter.Seq[V] {
	return func(yield func(V) bool) {
		for v := range seq {
			if keep(v) && !yield(v) {
				return
			}
		}
	}
}

// Take arrête un itérateur après n éléments.
func Take[V any](seq iter.Seq[V], n int) iter.Seq[V] {
	return func(yield func(V) bool) {
		if n <= 0 {
			return
		}
		i := 0
		for v := range seq {
			if !yield(v) {
				return
			}
			i++
			if i == n {
				return
			}
		}
	}
}

const text = `Le chat dort. Le chien aboie, le chat s'éveille !
Un chat, deux chats ; le chien dort.`

func main() {
	ws := NewWordScanner(strings.NewReader(text))
	freq := map[string]int{}
	for w := range Filter(ws.Words(), func(s string) bool { return len(s) > 2 }) {
		freq[w]++
	}
	if err := ws.Err(); err != nil {
		fmt.Println("lecture :", err)
		return
	}

	// L'idiome : clés triées en une ligne.
	for _, k := range slices.Sorted(maps.Keys(freq)) {
		fmt.Printf("%-9s %d\n", k, freq[k])
	}

	// Les trois premiers mots seulement : l'itérateur s'arrête, rien n'est lu au-delà.
	first := slices.Collect(Take(NewWordScanner(strings.NewReader(text)).Words(), 3))
	fmt.Printf("%q\n", first)

	// Un Seq2 de la stdlib, à l'envers.
	for i, w := range slices.Backward(first) {
		fmt.Print(i, ":", w, " ")
	}
	fmt.Println()

	// SplitSeq : pas de slice intermédiaire.
	for line := range strings.SplitSeq(text, "\n") {
		fmt.Println(len(line), "octets")
	}
}
```

Sortie :

```
aboie     1
chat      3
chats     1
chien     2
deux      1
dort      2
s'éveille 1
["le" "chat" "dort"]
2:dort 1:chat 0:le 
50 octets
36 octets
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `Words() iter.Seq[string]` | La méthode rend une **fonction**. Rien n'est lu tant qu'on n'a pas commencé le `range`. |
| `if !yield(word) { return }` | Le contrat : arrêt immédiat quand le consommateur ne veut plus rien. |
| `Err()` après la boucle | Le style `bufio.Scanner`. Supprimer cette vérification ne provoquerait **aucune** erreur de compilation — c'est le défaut de ce style. |
| `Filter`, `Take` | Ils prennent un `Seq` et en rendent un : composition sans slice intermédiaire. |
| `Take` : `i == n` **après** `yield` | Si le test était avant, on lirait un élément de trop dans le flux. |
| `slices.Sorted(maps.Keys(freq))` | Sortie déterministe malgré l'ordre aléatoire des maps. |
| `s'éveille` | `TrimFunc` ne retire la ponctuation qu'**aux bords** : l'apostrophe interne reste. C'est la difficulté D2 du projet 1. |
| `50 octets` | `len` compte des octets ; `é` en vaut deux (niveau 1, leçon 8). |

## Erreurs fréquentes

1. **Ignorer le résultat de `yield`** → panique à l'exécution dès qu'un consommateur fait `break`.
2. **Deux variables sur un `Seq`** : refusé par le compilateur.
3. **Oublier `Err()`** dans le style scanner : l'erreur disparaît.
4. **Parcourir deux fois un itérateur à usage unique** : le second parcours est vide, sans erreur.
5. **Oublier `stop()`** après `iter.Pull` : fuite de ressources.
6. **Reconstruire `Map`/`Filter`/`Reduce`** pour tout : une boucle est souvent plus claire.
7. **Rendre un itérateur là où un slice suffit** : si la collection est petite et déjà en
   mémoire, `[]T` est plus simple et se parcourt autant de fois qu'on veut.

## Bonnes pratiques Go

- Rendre un `iter.Seq` quand la collection est **grande**, **calculée à la volée**, ou quand le
  consommateur risque de s'arrêter tôt.
- Nommer les méthodes qui rendent un itérateur par ce qu'elles produisent : `All`, `Keys`,
  `Values`, `Lines`, `Words`, `Backward`.
- `slices.Collect` / `slices.Sorted` pour revenir à un slice quand on en a besoin.
- Documenter : usage unique ou non, et comment remontent les erreurs.
- Un `if !yield(v) { return }` à chaque appel, sans exception.

## Ce que je dois retenir

- `iter.Seq[V]` = `func(yield func(V) bool)` ; `iter.Seq2[K, V]` = `func(yield func(K, V) bool)`.
- `range` sur fonction : le **corps** de la boucle devient `yield`.
- Quand `yield` rend `false`, on s'arrête — sinon panique.
- `slices.Sorted(maps.Keys(m))` : clés triées en une ligne.
- `strings.SplitSeq`, `Lines`, `FieldsSeq` : itérer sans allouer de slice.
- Erreurs : `Seq2[V, error]` ou méthode `Err()` — choisir et documenter.

---

➡️ [Exercices](exercices.md)
