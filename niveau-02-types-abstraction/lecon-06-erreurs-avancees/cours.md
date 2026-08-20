# Leçon 6 — Erreurs avancées

> Prérequis : la leçon 10 du niveau 1 (`error` comme valeur) et la leçon 4 de ce niveau
> (interfaces). On sait maintenant qu'`error` **est** une interface — c'est ce qui rend tout
> ce chapitre possible.

## Objectifs

1. Envelopper une erreur avec `%w` et comprendre ce que cela construit réellement.
2. Utiliser `errors.Is` et `errors.As` à bon escient — et savoir lequel choisir.
3. Créer des **types d'erreur** porteurs de données.
4. Agréger plusieurs erreurs avec `errors.Join`.

## Explication

### L'enveloppement construit une chaîne

```go
if err != nil {
	return fmt.Errorf("lecture de %s : %w", path, err)
}
```

Le verbe `%w` (*wrap*) ne se contente pas d'insérer le texte : il construit une valeur qui
**conserve un lien** vers l'erreur d'origine. En interne, `fmt.Errorf` retourne alors un
`*fmt.wrapError` porteur d'une méthode :

```go
func (e *wrapError) Unwrap() error { return e.err }
```

Toute erreur possédant `Unwrap() error` participe à la chaîne. En pratique on obtient une
liste chaînée que `errors.Is` et `errors.As` remontent :

```
"chargement du profil : lecture de app.yaml : open app.yaml: no such file or directory"
        ↓ Unwrap                    ↓ Unwrap                    ↓
   *fmt.wrapError            *fmt.wrapError              *fs.PathError → syscall.Errno
```

`%v` à la place de `%w` produit **exactement le même texte** mais rompt la chaîne :
`errors.Is` renvoie alors `false`. C'est un bug invisible dans les logs et destructeur en
aval. **Règle : `%w` quand on relaie une erreur reçue, `%v` seulement quand on veut
délibérément masquer la cause** (par exemple à la frontière d'une API publique, pour ne pas
exposer un détail d'implémentation).

Un `fmt.Errorf` peut contenir **plusieurs** `%w` depuis Go 1.20 : la chaîne devient un arbre.

### `errors.Is` — « est-ce cette erreur-là ? »

```go
var ErrNotFound = errors.New("introuvable")   // sentinelle : préfixe Err

if errors.Is(err, ErrNotFound) { … }
if errors.Is(err, os.ErrNotExist) { … }
if errors.Is(err, context.DeadlineExceeded) { … }
```

`errors.Is` compare par **identité** en remontant toute la chaîne. Elle remplace `==`, qui
échouerait dès le premier enveloppement.

Un type peut personnaliser la comparaison en implémentant `Is(error) bool` — rare, utile pour
des erreurs paramétrées.

**Les sentinelles sont un contrat public.** Exporter `ErrNotFound`, c'est promettre de la
retourner dans les mêmes circonstances pour toujours. Elles conviennent quand l'appelant n'a
besoin que de distinguer un **cas**, sans donnée associée.

### `errors.As` — « y a-t-il une erreur de ce type-là ? »

```go
var pathErr *fs.PathError
if errors.As(err, &pathErr) {
	fmt.Println("chemin fautif :", pathErr.Path)   // accès aux DONNÉES
}
```

`errors.As` cherche dans la chaîne la première erreur **assignable au type de la cible** et
la lui affecte. Le second argument est **toujours un pointeur** vers une variable du type
recherché — l'oublier provoque une panique explicite : `errors.As: target must be a non-nil
pointer`.

**`Is` pour reconnaître un cas, `As` pour récupérer des données.** C'est tout l'arbitrage.

### Les types d'erreur

```go
type ValidationError struct {
	Field string
	Value string
	Err   error
}

func (e *ValidationError) Error() string {
	return fmt.Sprintf("champ %q (valeur %q) : %v", e.Field, e.Value, e.Err)
}

func (e *ValidationError) Unwrap() error { return e.Err }
```

Trois points de conception, tous importants :

1. **Récepteur pointeur, toujours.** Si `Error()` était sur la valeur, `ValidationError` *et*
   `*ValidationError` satisferaient `error`, et `errors.As` deviendrait ambigu selon ce qui a
   été retourné. La convention Go est universelle ici : **type d'erreur = récepteur pointeur**,
   et on retourne `&ValidationError{…}`.
2. **`Unwrap()`** rend la cause visible à `errors.Is`/`errors.As`. À omettre s'il n'y a pas de
   cause sous-jacente.
3. **Attention au piège de l'interface nil** (leçon 4) :
   ```go
   var e *ValidationError   // nil
   return e                 // → error NON nil !
   ```
   Toujours `return nil` littéralement dans le chemin de succès.

### Sentinelle ou type ? Le vrai arbitrage

| | Sentinelle (`errors.New`) | Type d'erreur |
|---|---|---|
| Coût | une variable | un type + `Error()` + tests |
| Donnée transportée | aucune | autant qu'on veut |
| Contrat public | figé, très visible | figé aussi, mais extensible |
| Détection | `errors.Is` | `errors.As` |
| Quand | l'appelant veut juste **savoir** | l'appelant veut **agir** sur la donnée |

**Et une troisième option, souvent la meilleure : aucune des deux.** Si aucun appelant ne
distingue ce cas, un `fmt.Errorf` bien rédigé suffit. Créer une sentinelle « au cas où »,
c'est figer une API publique sans utilisateur.

### `errors.Join` — plusieurs erreurs à la fois

```go
var errs []error
for i, line := range lines {
	if err := validate(line); err != nil {
		errs = append(errs, fmt.Errorf("ligne %d : %w", i+1, err))
	}
}
return errors.Join(errs...)   // nil si errs est vide
```

Introduit en Go 1.20. Le résultat est une erreur unique dont le message concatène les
messages sur des lignes séparées, et **`errors.Is` traverse toutes les branches**. La
méthode sous-jacente est `Unwrap() []error` — noter le pluriel, différent de
`Unwrap() error`.

Quand choisir quoi :
- **`errors.Join`** à la frontière d'une API qui doit retourner un seul `error` ;
- **`[]error` explicite** à l'intérieur, quand on veut compter, trier, paginer ou n'afficher
  que les dix premières.

### Où enrichir, et de combien

La règle de l'enveloppement utile : **chaque niveau ajoute exactement l'information que le
niveau inférieur ne pouvait pas connaître.**

```go
// storage : connaît le chemin
return fmt.Errorf("lecture de %s : %w", path, err)
// service : connaît l'identifiant métier
return fmt.Errorf("chargement de l'utilisateur %d : %w", id, err)
// handler : connaît la requête — et ne réenveloppe PAS, il journalise et traduit en HTTP
```

Deux anti-patrons symétriques :
- **Ne rien ajouter** : `return err` sur dix niveaux produit `no such file or directory` sans
  dire lequel.
- **Tout répéter** : `fmt.Errorf("erreur lors du chargement : chargement de %d : %w", …)`.

Et un point souvent négligé : **on n'enveloppe pas *et* on ne journalise pas la même erreur.**
Soit on la retourne (en l'enrichissant), soit on la traite (en la journalisant). Faire les
deux produit la même erreur dix fois dans les logs, à dix niveaux différents. La journalisation
appartient au **point de décision**, généralement le plus haut niveau.

### Aperçu : les erreurs à la frontière d'une API

Un service HTTP ne doit pas exposer `pq: duplicate key value violates unique constraint`. La
technique consiste à **traduire** les erreurs internes en erreurs de domaine à la frontière :
`errors.Is(err, ErrDuplicate)` en interne, message générique et code 409 vers l'extérieur.
Sujet traité au niveau 7.

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

// ErrEmptyInput est une sentinelle : l'appelant veut seulement reconnaître le cas.
var ErrEmptyInput = errors.New("entrée vide")

// FieldError est un type d'erreur : il transporte des données exploitables.
type FieldError struct {
	Line  int
	Field string
	Value string
	Err   error
}

func (e *FieldError) Error() string {
	return fmt.Sprintf("ligne %d, champ %q (valeur %q) : %v", e.Line, e.Field, e.Value, e.Err)
}
func (e *FieldError) Unwrap() error { return e.Err }

// parseAge convertit et valide. Elle enveloppe l'erreur de strconv.
func parseAge(line int, raw string) (int, error) {
	n, err := strconv.Atoi(strings.TrimSpace(raw))
	if err != nil {
		return 0, &FieldError{Line: line, Field: "age", Value: raw, Err: err}
	}
	if n < 0 || n > 150 {
		return 0, &FieldError{Line: line, Field: "age", Value: raw,
			Err: fmt.Errorf("hors des bornes [0,150]")}
	}
	return n, nil
}

// ParseAll collecte TOUTES les erreurs au lieu de s'arrêter à la première.
func ParseAll(input string) ([]int, error) {
	if strings.TrimSpace(input) == "" {
		return nil, ErrEmptyInput
	}
	var ages []int
	var errs []error
	for i, raw := range strings.Split(input, ",") {
		n, err := parseAge(i+1, raw)
		if err != nil {
			errs = append(errs, err)
			continue
		}
		ages = append(ages, n)
	}
	return ages, errors.Join(errs...) // nil si errs est vide
}

func main() {
	for _, in := range []string{"30,25", "", "30,abc,200"} {
		ages, err := ParseAll(in)

		switch {
		case errors.Is(err, ErrEmptyInput):
			fmt.Fprintf(os.Stderr, "%-14q ignoré : %v\n\n", in, err)

		case err != nil:
			fmt.Fprintf(os.Stderr, "%-14q partiellement invalide :\n%v\n", in, err)

			// errors.As traverse errors.Join ET la chaîne de chaque branche
			var fe *FieldError
			if errors.As(err, &fe) {
				fmt.Fprintf(os.Stderr, "→ première erreur exploitable : ligne %d, champ %s\n",
					fe.Line, fe.Field)
			}
			// Is remonte jusqu'à l'erreur de strconv
			fmt.Fprintf(os.Stderr, "→ erreur de syntaxe numérique ? %v\n\n",
				errors.Is(err, strconv.ErrSyntax))

		default:
			fmt.Printf("%-14q → %v\n\n", in, ages)
		}
	}
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `ErrEmptyInput` sentinelle | L'appelant veut juste distinguer ce cas : aucune donnée à transporter. |
| `*FieldError` type d'erreur | L'appelant veut la **ligne** et le **champ** pour agir : `errors.As` les lui donne sans analyser de texte. |
| Récepteur pointeur sur `Error()` | Convention universelle pour les types d'erreur ; évite l'ambiguïté avec `errors.As`. |
| `Unwrap()` | Rend l'erreur de `strconv` visible à travers `FieldError`. |
| `errors.Join(errs...)` | Retourne `nil` si le slice est vide : aucun cas particulier à écrire. |
| `errors.Is(err, strconv.ErrSyntax)` | Traverse `Join` **puis** `FieldError.Unwrap` **puis** l'erreur de `strconv`. Trois niveaux, une ligne. |
| `switch { case errors.Is(…) }` | Discriminer les cas d'erreur par un `switch` sans expression reste plus lisible qu'une cascade de `if`. |

## Erreurs fréquentes

1. **`%v` au lieu de `%w`** quand on relaie : la chaîne est rompue, `errors.Is` ne marche plus.
2. **`err == ErrX`** au lieu d'`errors.Is`.
3. **`errors.As` avec autre chose qu'un pointeur** → panique explicite.
4. **`Error()` à récepteur valeur** sur un type d'erreur : rend `errors.As` ambigu.
5. **Retourner une variable de type pointeur concret nil** en `error` → interface non nil.
6. **Envelopper *et* journaliser** la même erreur à chaque niveau : logs illisibles.
7. **Créer une sentinelle par cas d'erreur** sans qu'aucun appelant ne la teste : API figée pour rien.
8. **Ajouter deux fois la même information** dans le message.
9. **Oublier `Unwrap()`** sur un type d'erreur qui a une cause : la chaîne s'arrête là.
10. **Exposer les erreurs internes** à l'extérieur d'une API publique.

## Bonnes pratiques Go

- `%w` par défaut pour relayer ; `%v` seulement pour masquer volontairement.
- Une information nouvelle par niveau d'enveloppement, jamais deux fois la même.
- Sentinelles nommées `ErrXxx`, déclarées en `var` au niveau du package, documentées.
- Types d'erreur avec **récepteur pointeur** et `Unwrap()` quand il y a une cause.
- `errors.Is` pour reconnaître, `errors.As` pour extraire.
- `errors.Join` à la frontière, `[]error` à l'intérieur.
- Journaliser **une seule fois**, au point de décision.
- Ne créer une sentinelle ou un type que lorsqu'un appelant réel en a besoin.

## Ce que je dois retenir

- `%w` construit une **chaîne** d'erreurs ; `%v` la rompt en produisant le même texte.
- `errors.Is` compare par identité en remontant la chaîne ; `errors.As` extrait par type.
- Un type d'erreur se déclare avec un **récepteur pointeur** et un `Unwrap()` si besoin.
- `errors.Join` agrège et retourne `nil` sur une liste vide ; sa méthode est `Unwrap() []error`.
- Chaque niveau ajoute **une** information que le niveau inférieur ignorait.
- On enveloppe **ou** on journalise, jamais les deux.

➡️ [Exercices](exercices.md)
