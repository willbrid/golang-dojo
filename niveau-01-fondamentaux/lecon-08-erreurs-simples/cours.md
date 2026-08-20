# Leçon 8 — Les erreurs, première approche

## Objectifs

1. Comprendre que `error` est une **valeur ordinaire**, pas un mécanisme spécial.
2. Créer, retourner et traiter une erreur correctement.
3. Rédiger des messages d'erreur conformes aux conventions Go.
4. Savoir quand ignorer une erreur — et comment le dire explicitement.

*(L'enveloppement avec `%w`, `errors.Is`, `errors.As` et les erreurs typées seront
approfondis au niveau 2. Cette leçon pose les fondations.)*

## Explication

### `error` est une interface, donc une valeur

```go
type error interface {
	Error() string
}
```

C'est **toute** la définition, dans le paquet `builtin`. Une erreur en Go n'est pas un
mécanisme de contrôle de flux : c'est une valeur qu'on retourne, qu'on compare, qu'on
stocke dans un slice, qu'on passe en paramètre.

La zéro-valeur d'une interface est `nil` : **`nil` signifie « pas d'erreur »**. D'où
l'omniprésence de `if err != nil`.

### Créer une erreur

```go
errors.New("fichier introuvable")                       // message fixe
fmt.Errorf("fichier %q introuvable", name)              // message formaté
fmt.Errorf("lecture de %s : %w", name, err)             // enveloppe une erreur existante
```

`errors.New` pour un message constant, `fmt.Errorf` dès qu'il faut du contexte. Le verbe
`%w` (*wrap*) conserve l'erreur d'origine pour permettre `errors.Is`/`errors.As` plus tard ;
`%v` la transforme en simple texte et perd cette information. **Utiliser `%w` par défaut**
quand on relaie une erreur reçue.

### Le contrat de retour

```go
func ReadConfig(path string) (*Config, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return nil, fmt.Errorf("lecture de %s : %w", path, err)
	}
	cfg, err := parse(data)
	if err != nil {
		return nil, fmt.Errorf("analyse de %s : %w", path, err)
	}
	return cfg, nil
}
```

Deux règles que tout le monde en Go respecte :

1. **`error` est le dernier résultat.**
2. **Si `err != nil`, les autres résultats ne doivent pas être utilisés.** Retourner la
   zéro-valeur (`nil`, `0`, `""`) pour les accompagner. Certaines fonctions de la stdlib
   dérogent avec un contrat explicite — `io.Reader.Read` peut retourner `n > 0` **et** une
   erreur — mais c'est documenté, et c'est l'exception.

Côté appelant :

```go
cfg, err := ReadConfig("app.yaml")
if err != nil {
	return err                          // relayer, éventuellement en ajoutant du contexte
}
// à partir d'ici, cfg est valide
```

Le traitement de l'erreur se fait **immédiatement** après l'appel. Pas de bloc de nettoyage
groupé à la fin, pas d'erreur mise de côté « pour plus tard ».

### Conventions de rédaction des messages

Les *Go Code Review Comments* officiels sont explicites :

```go
// BON
errors.New("connexion refusée")
fmt.Errorf("ouverture de %q : %w", path, err)

// MAUVAIS
errors.New("Connexion refusée.")   // majuscule + point final
errors.New("erreur : échec")       // le mot « erreur » est redondant
fmt.Errorf("failed to open file")  // pas de contexte : quel fichier ?
```

Règles :
- **Pas de majuscule initiale, pas de ponctuation finale.** Parce que les messages
  s'emboîtent : `lecture de config : ouverture de app.yaml : no such file or directory`.
  Une majuscule au milieu de cette chaîne serait absurde.
- **Ne jamais écrire « erreur » ou « échec »** dans le message : le fait que ce soit une
  erreur est déjà porté par le type.
- **Ajouter du contexte que l'appelant n'a pas** : le nom du fichier, l'identifiant, l'URL.
  `%q` pour les chaînes, il montre les espaces invisibles.
- Un message d'erreur bien construit décrit une **chaîne d'opérations**, du plus général au
  plus précis, chaque niveau ajoutant ce qu'il sait.

### Comparer des erreurs : `errors.Is` plutôt que `==`

Certaines erreurs sont des **sentinelles**, des valeurs exportées comparables :

```go
var ErrNotFound = errors.New("introuvable")   // convention : préfixe Err

if errors.Is(err, os.ErrNotExist) {
	// le fichier n'existe pas
}
```

`errors.Is` déballe la chaîne des `%w`, alors que `err == ErrNotFound` échoue dès que
l'erreur a été enveloppée une fois. **Toujours `errors.Is`** — c'est le sujet du niveau 2,
mais autant prendre le bon réflexe tout de suite.

### Ignorer une erreur : explicitement, ou pas du tout

```go
_ = os.Remove(tmpFile)      // délibéré : l'échec du nettoyage n'a pas d'importance ici
os.Remove(tmpFile)          // paresseux : le lecteur ne sait pas si c'est voulu
```

Les linters (`errcheck`, inclus dans `golangci-lint`) signalent les erreurs non traitées.
Le `_ =` explicite dit « j'ai vu, j'assume », et mérite souvent un commentaire d'une ligne.

Une erreur particulièrement traître :

```go
f, _ := os.Create(path)
defer f.Close()             // l'erreur de Close est perdue
```

Sur un fichier en écriture, `Close` peut échouer (disque plein, quota) et l'écriture est
alors **incomplète sans que personne ne le sache**. Traiter l'erreur de `Close` en écriture
est un critère « prêt pour la production ». La technique propre passe par `defer` et un
résultat nommé — niveau 2.

### `panic` n'est pas une erreur

```go
panic("quelque chose de terrible")
```

`panic` interrompt le programme. Il est réservé à ce qui **ne peut pas arriver** : un
invariant violé, un bug de programmation. Une erreur d'entrée/sortie, une entrée utilisateur
invalide, un réseau coupé ne sont **jamais** des cas de `panic` — ce sont des situations
normales du monde réel, à retourner comme `error`.

Règle de niveau 1 : **aucun `panic` dans le code écrit**. On verra les rares exceptions au niveau 2.

### Le compromis, honnêtement

Le style Go est verbeux : on écrit beaucoup de `if err != nil`. Les critiques sont légitimes,
et plusieurs propositions d'amélioration ont été rejetées par l'équipe Go. Ce qu'on gagne en
échange :

- Aucun chemin d'exécution invisible : pas de saut sur cinq niveaux de pile.
- La gestion d'erreur est **du code ordinaire**, testable et lisible.
- Une signature dit toujours si une fonction peut échouer.

Ce n'est pas « mieux » dans l'absolu : c'est un compromis différent de celui des exceptions.
Savoir l'expliquer, c'est comprendre Go.

## Exemple

```go
package main

import (
	"errors"
	"fmt"
	"os"
	"strconv"
	"strings"
)

// ErrEmptyInput signale une entrée vide. Sentinelle exportée : les appelants
// peuvent la reconnaître avec errors.Is.
var ErrEmptyInput = errors.New("entrée vide")

// ParseAges convertit une liste « 30,25,40 » en []int.
// En cas de valeur invalide, elle retourne une erreur décrivant la position fautive.
func ParseAges(input string) ([]int, error) {
	if strings.TrimSpace(input) == "" {
		return nil, ErrEmptyInput
	}

	parts := strings.Split(input, ",")
	ages := make([]int, 0, len(parts))

	for i, p := range parts {
		n, err := strconv.Atoi(strings.TrimSpace(p))
		if err != nil {
			// %w conserve l'erreur de strconv ; le contexte dit OÙ ça a échoué.
			return nil, fmt.Errorf("champ %d (%q) : %w", i+1, p, err)
		}
		if n < 0 || n > 150 {
			return nil, fmt.Errorf("champ %d : âge %d hors bornes [0,150]", i+1, n)
		}
		ages = append(ages, n)
	}
	return ages, nil
}

func main() {
	inputs := []string{"30, 25, 40", "", "30,abc,40", "30,200"}

	for _, in := range inputs {
		ages, err := ParseAges(in)
		switch {
		case errors.Is(err, ErrEmptyInput):
			fmt.Fprintf(os.Stderr, "%-14q ignoré : %v\n", in, err)
		case err != nil:
			fmt.Fprintf(os.Stderr, "%-14q refusé : %v\n", in, err)
		default:
			fmt.Printf("%-14q → %v\n", in, ages)
		}
	}
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `var ErrEmptyInput = errors.New(…)` | Sentinelle : préfixe `Err`, déclarée au niveau du package, exportée pour que les appelants la testent. |
| `return nil, ErrEmptyInput` | Le premier résultat est la zéro-valeur : rien d'exploitable quand l'erreur est non nulle. |
| `fmt.Errorf("champ %d (%q) : %w", …)` | Contexte que l'appelant n'a pas (la position, la valeur brute) **plus** l'erreur d'origine. Message final : `champ 2 ("abc") : strconv.Atoi: parsing "abc": invalid syntax`. |
| Hors bornes sans `%w` | Ici il n'y a aucune erreur sous-jacente à envelopper : `fmt.Errorf` sans `%w` est correct. |
| `errors.Is(err, ErrEmptyInput)` | Fonctionne même si l'erreur avait été enveloppée plus haut. `err == ErrEmptyInput` casserait. |
| `switch { case … }` | Un switch sans expression pour discriminer les cas d'erreur — plus lisible qu'une cascade de `if`. |
| `fmt.Fprintf(os.Stderr, …)` | Les erreurs vont sur la sortie d'erreur, les résultats sur la sortie standard. Exigence « production ». |

## Erreurs fréquentes

1. **Ignorer une erreur sans le dire** : `f()` au lieu de `_ = f()` ou d'un vrai traitement.
2. **Message avec majuscule et point final** : casse la lisibilité des messages emboîtés.
3. **Écrire « erreur : » dans le message** : redondant.
4. **`%v` au lieu de `%w`** quand on relaie : la chaîne d'erreurs est rompue, `errors.Is` ne fonctionne plus.
5. **Utiliser les autres résultats alors que `err != nil`.**
6. **Comparer avec `==` au lieu d'`errors.Is`.**
7. **`panic` sur une erreur d'entrée/sortie ou d'utilisateur.**
8. **Ne pas ajouter de contexte** : relayer `err` tel quel sur dix niveaux produit `no such file or directory` sans dire quel fichier.
9. **Envelopper deux fois la même information** : `fmt.Errorf("erreur de lecture : lecture de %s : %w", …)`. Chaque niveau ajoute **une** information nouvelle.

## Bonnes pratiques Go

- Traiter l'erreur **immédiatement** après l'appel ; ne jamais la reporter.
- Ajouter du contexte à chaque niveau, **une seule information** par niveau.
- `errors.New` pour un message fixe, `fmt.Errorf` avec `%w` pour relayer.
- Sentinelles nommées `ErrXxx`, déclarées en `var` au niveau du package.
- Messages en minuscule, sans ponctuation finale, sans le mot « erreur ».
- `_ =` explicite pour une erreur volontairement ignorée, avec un commentaire si ce n'est pas évident.
- Erreurs sur `os.Stderr`, résultats sur `os.Stdout`, code de sortie non nul en cas d'échec.
- Pas de `panic` dans du code de bibliothèque. Jamais.

## Ce que je dois retenir

- `error` est une **interface à une méthode** ; `nil` signifie « pas d'erreur ».
- `error` est **toujours le dernier résultat** ; si elle est non nulle, ignorer les autres.
- `%w` enveloppe et préserve la chaîne, `%v` l'aplatit.
- Messages : **minuscule, sans point final, avec du contexte utile**.
- `errors.Is` plutôt que `==`.
- Ignorer une erreur se fait **explicitement** avec `_ =`.
- `panic` est réservé aux bugs de programmation, pas aux conditions d'exécution.

➡️ [Exercices](exercices.md)
