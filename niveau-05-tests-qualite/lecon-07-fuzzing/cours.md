# Leçon 7 — Fuzzing

> Un test vérifie les cas **auxquels on a pensé**. Les bugs, eux, se cachent dans les cas
> auxquels on n'a pas pensé : un octet invalide, un nombre gigantesque, un caractère qui
> ressemble à un séparateur. Le **fuzzing** confie à la machine la tâche de chercher ces
> entrées, des millions de fois par minute. Go l'intègre à `go test` depuis la version 1.18.
> Cette leçon raconte une vraie séance : trois bugs trouvés en moins d'une minute dans un
> code qui passait tous ses tests.

## Objectifs

1. Écrire une cible de fuzzing (`FuzzXxx`) et la lancer.
2. Formuler des **propriétés** vérifiables sur des entrées arbitraires.
3. Comprendre le corpus, la minimisation et `testdata/fuzz`.
4. Savoir quoi fuzzer en priorité : tout ce qui **analyse** des données venues de l'extérieur.

## Explication

### Une cible de fuzzing

```go
func FuzzRoundTrip(f *testing.F) {
	f.Add("aaab")                       // corpus de départ (seed corpus)
	f.Fuzz(func(t *testing.T, s string) {
		got, err := Decode(Encode(s))
		if err != nil || got != s {
			t.Fatalf("Decode(Encode(%q)) = %q, %v", s, got, err)
		}
	})
}
```

- Dans un `_test.go`, une fonction `FuzzXxx(f *testing.F)`.
- `f.Add` fournit des exemples de départ ; leurs types fixent ceux des arguments de la fonction
  de `f.Fuzz` (types autorisés : `string`, `[]byte`, entiers, flottants, `bool`, `rune`, `byte`).
- `f.Fuzz` reçoit la fonction à appeler avec des entrées générées.

Deux modes d'exécution :

```bash
go test ./rle                                  # mode normal : exécute seulement le corpus (rapide, en CI)
go test -run '^$' -fuzz FuzzRoundTrip ./rle    # mode fuzzing : génère des entrées jusqu'à un échec
go test -run '^$' -fuzz FuzzRoundTrip -fuzztime 30s ./rle   # durée bornée
```

`-fuzz` n'accepte qu'**une** cible à la fois. Sans `-fuzztime`, le fuzzing tourne jusqu'à ce
qu'il trouve un échec ou qu'on l'interrompe (Ctrl-C).

### Comment ça marche

Le moteur part du corpus, **mute** les entrées (insère, supprime, remplace des octets, recopie des
morceaux, injecte des valeurs intéressantes comme `0`, `-1`, `\xff`) et observe la **couverture**
du code. Une mutation qui fait passer par une branche jamais vue est gardée dans le corpus et
re-mutée. C'est un fuzzing **guidé par la couverture** : il explore le code au lieu de tirer au
hasard. Il utilise tous les cœurs (un *worker* par cœur).

### Que vérifier ? Des propriétés

On ne connaît pas la sortie attendue pour une entrée aléatoire. On vérifie donc des
**propriétés** qui doivent être vraies pour **toute** entrée :

| Propriété | Exemple |
|---|---|
| **Ne jamais paniquer** (ni boucler, ni exploser en mémoire) | tout analyseur d'entrée externe |
| **Aller-retour** | `Decode(Encode(x)) == x`, `Unmarshal(Marshal(x)) == x` |
| **Idempotence** | `Normalize(Normalize(x)) == Normalize(x)` |
| **Équivalence** avec une implémentation de référence | version optimisée contre version simple |
| **Invariants** de sortie | un `Slug` ne contient que `[a-z0-9-]` ; un tri rend une sortie triée |

La plus simple — « ne pas paniquer » — est déjà extraordinairement rentable : un analyseur qui
panique sur une entrée réseau, c'est un serveur qu'on fait tomber avec une seule requête.

### Quand le fuzzer trouve

1. Il **minimise** l'entrée : il cherche la plus petite variante qui échoue encore.
2. Il l'écrit dans `testdata/fuzz/FuzzXxx/<hash>`.
3. Il affiche la commande pour la rejouer : `go test -run=FuzzXxx/<hash>`.

Ce fichier doit être **commité**. À partir de là, `go test` (sans `-fuzz`) rejoue cette entrée à
chaque exécution : le bug devient un **test de non-régression** automatique.

### Le contrat d'abord

Une cible de fuzzing révèle souvent qu'on n'a jamais **décidé** du comportement sur certaines
entrées. Un octet UTF-8 invalide doit-il être accepté, refusé, remplacé ? Le fuzzing pose la
question ; c'est au développeur d'y répondre, puis de l'écrire : dans la documentation, et dans
la cible (`t.Skip` pour les entrées hors contrat, ou une vérification d'erreur).

### Quoi fuzzer

En priorité, tout ce qui **transforme des données venues de l'extérieur** : analyseurs de formats
(JSON maison, CSV, protocoles binaires, en-têtes), décodeurs, décompresseurs, expressions de
filtre, chemins de fichiers. La bibliothèque standard de Go est elle-même fuzzée en continu ; de
nombreux bugs de `encoding/*`, `image/*`, `net/*` ont été trouvés ainsi.

Inutile de fuzzer : le code sans entrée variable, ou dont l'entrée vient uniquement du programme.

## Exemple : une vraie séance

Un compresseur RLE (*run-length encoding*) : `"aaab"` devient `"3a1b"`. Première version, qui
passait ses tests unitaires (`""`, `"a"`, `"aaab"`, `"été"`, `"abcabc"`) :

```go
// Première version (extrait) : Encode écrit <nombre><caractère>
b.WriteString(strconv.Itoa(j - i))
b.WriteRune(runes[i])
```

**Bug n° 1, trouvé en 0,02 s** par `FuzzRoundTrip` :

```
--- FAIL: FuzzRoundTrip (0.02s)
    rle_test.go:22: Decode(Encode("0")) : erreur position 0 : nombre puis caractère attendus (encodé : "10")
    Failing input written to testdata/fuzz/FuzzRoundTrip/771e938e4458e983
```

Le texte `"0"` s'encode en `"10"` (« un zéro »)… que `Decode` lit comme « dix, puis rien ».
Le format est **ambigu** dès que les données contiennent des chiffres. Aucun test unitaire n'en
avait. Correctif : un séparateur, `<nombre>:<caractère>` — `"0"` devient `"1:0"`.

**Bug n° 2, trouvé en 0,04 s** sur la version corrigée :

```
--- FAIL: FuzzRoundTrip (0.04s)
    rle_test.go:25: Decode(Encode("\xfb")) = "�" (encodé : "1:�")
```

L'octet `0xFB` seul n'est pas de l'UTF-8 valide ; la conversion `[]rune(s)` le remplace par U+FFFD
(N1 L8). L'aller-retour est impossible. Ce n'est pas un bug de code : c'est un **contrat non
décidé**. Décision : `Encode` n'accepte que de l'UTF-8 valide, c'est documenté, et la cible ignore
les entrées hors contrat (`t.Skip`).

**Bug n° 3** : une seconde cible, `FuzzDecode`, vérifie seulement que `Decode` ne panique jamais
sur une entrée quelconque. Le processus de fuzzing **s'arrête brutalement** :

```
--- FAIL: FuzzDecode (74.32s)
    fuzzing process hung or terminated unexpectedly while minimizing: EOF
    Failing input written to testdata/fuzz/FuzzDecode/1c26b68885d4f04c
```

L'entrée enregistrée : `"111111111111111:1"`. `Decode` tente de produire **111 billions** de
caractères avec `strings.Repeat` : la mémoire est épuisée. Quinze octets d'entrée suffisent à
faire tomber un serveur — une **bombe de décompression**. Correctif : borner la taille décodée.

Version finale :

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : rle/rle.go
// Package rle compresse les répétitions : "aaab" → "3:a1:b".
package rle

import (
	"errors"
	"fmt"
	"strconv"
	"strings"
)

// Encode remplace chaque suite de caractères identiques par <nombre>:<caractère>.
// s doit être de l'UTF-8 valide : un octet invalide est remplacé par U+FFFD.
func Encode(s string) string {
	var b strings.Builder
	runes := []rune(s)
	for i := 0; i < len(runes); {
		j := i
		for j < len(runes) && runes[j] == runes[i] {
			j++
		}
		b.WriteString(strconv.Itoa(j - i))
		b.WriteByte(':')
		b.WriteRune(runes[i])
		i = j
	}
	return b.String()
}

// MaxDecoded borne la taille du texte décodé : une entrée de quelques octets
// ne doit pas pouvoir réclamer des téraoctets de mémoire.
const MaxDecoded = 1 << 20

// ErrTooLarge est retournée quand le texte décodé dépasserait MaxDecoded runes.
var ErrTooLarge = errors.New("texte décodé trop volumineux")

// Decode inverse Encode.
func Decode(s string) (string, error) {
	var b strings.Builder
	total := 0
	runes := []rune(s)
	for i := 0; i < len(runes); {
		j := i
		for j < len(runes) && '0' <= runes[j] && runes[j] <= '9' {
			j++
		}
		if j == i || j+1 >= len(runes) || runes[j] != ':' {
			return "", fmt.Errorf("position %d : <nombre>:<caractère> attendu", i)
		}
		n, err := strconv.Atoi(string(runes[i:j]))
		if err != nil {
			return "", err
		}
		if n > MaxDecoded-total {
			return "", fmt.Errorf("position %d : %w", i, ErrTooLarge)
		}
		total += n
		b.WriteString(strings.Repeat(string(runes[j+1]), n))
		i = j + 2
	}
	return b.String(), nil
}
```

```go
// Fichier : rle/rle_test.go
package rle

import (
	"testing"
	"unicode/utf8"
)

func TestRoundTrip(t *testing.T) {
	for _, s := range []string{"", "a", "aaab", "été", "abcabc"} {
		got, err := Decode(Encode(s))
		if err != nil || got != s {
			t.Errorf("Decode(Encode(%q)) = %q, %v", s, got, err)
		}
	}
}

func FuzzRoundTrip(f *testing.F) {
	for _, s := range []string{"", "a", "aaab", "été"} {
		f.Add(s)
	}
	f.Fuzz(func(t *testing.T, s string) {
		if !utf8.ValidString(s) {
			t.Skip("contrat : Encode n'accepte que de l'UTF-8 valide")
		}
		enc := Encode(s)
		got, err := Decode(enc)
		if err != nil {
			t.Fatalf("Decode(Encode(%q)) : erreur %v (encodé : %q)", s, err, enc)
		}
		if got != s {
			t.Fatalf("Decode(Encode(%q)) = %q (encodé : %q)", s, got, enc)
		}
	})
}

// FuzzDecode : Decode reçoit des données quelconques ; il ne doit jamais paniquer.
func FuzzDecode(f *testing.F) {
	f.Add("3:a1:b")
	f.Add("")
	f.Fuzz(func(t *testing.T, s string) {
		Decode(s) // une erreur est acceptable, une panique non
	})
}
```

Après correction, 25 s de fuzzing sur chaque cible ne trouvent plus rien, et `go test ./rle`
rejoue à chaque exécution les trois entrées enregistrées dans `testdata/fuzz/` :

```
testdata/fuzz/FuzzDecode/1c26b68885d4f04c        string("111111111111111:1")
testdata/fuzz/FuzzRoundTrip/771e938e4458e983     string("0")
testdata/fuzz/FuzzRoundTrip/c600bf7f580fb7c0     string("\xfb")
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `TestRoundTrip` | Les cas pensés à la main. Tous passaient avec la première version. |
| `f.Add(…)` | Le corpus de départ : des entrées **valides et variées**, pour que le moteur explore à partir de vraies formes. |
| `t.Skip` sur l'UTF-8 invalide | Le contrat, décidé après le bug n° 2, est écrit **dans** la cible. |
| `FuzzDecode` sans vérification | « Ne pas paniquer, ne pas exploser » : la propriété la moins exigeante, et elle a trouvé le bug le plus grave. |
| `MaxDecoded` et `ErrTooLarge` | Une limite **exportée** : l'appelant la connaît et peut tester l'erreur. |
| `n > MaxDecoded-total` | Et non `total+n > MaxDecoded` : avec un `n` immense, `total+n` pourrait déborder (N1 L3). |
| `testdata/fuzz/` commité | Chaque bug trouvé devient un test permanent. |

## Erreurs fréquentes

1. **Ne jamais lancer `-fuzz`** : une cible exécutée seulement sur son corpus ne cherche rien.
2. **Un corpus de départ vide ou pauvre** : le moteur part de rien.
3. **Une cible non déterministe** (heure, aléatoire, état global) : les échecs ne se rejouent pas.
4. **Oublier de commiter `testdata/fuzz/`** : le bug peut revenir.
5. **Corriger la cible au lieu du code** quand l'échec dérange.
6. **Fuzzer avec une vérification trop faible** (`_ = f(x)`) en croyant tester un aller-retour.
7. **Ignorer un échec « bizarre »** (processus tué, délai dépassé) : c'est souvent le plus grave.

## Bonnes pratiques Go

- Une cible « ne panique pas » pour **chaque** fonction qui analyse une entrée externe.
- Une cible d'aller-retour pour chaque paire encodeur/décodeur.
- Des limites explicites (taille, profondeur, nombre d'éléments) sur tout décodeur.
- `go test ./...` en CI (rejoue le corpus) ; des séances de fuzzing régulières, plus longues, à part.
- Décider et documenter le contrat sur les entrées étranges.

## Ce que je dois retenir

- `func FuzzXxx(f *testing.F)`, `f.Add` (corpus), `f.Fuzz(func(t *testing.T, …))`.
- `go test` rejoue le corpus ; `go test -fuzz FuzzXxx -fuzztime 30s` cherche.
- On vérifie des **propriétés** : pas de panique, aller-retour, idempotence, équivalence, invariants.
- Entrée fautive minimisée et enregistrée dans `testdata/fuzz/` : à commiter.
- Le fuzzing révèle aussi les **contrats non décidés**.
- Tout décodeur d'entrée externe a besoin d'une limite de taille.

---

➡️ [Exercices](exercices.md)
