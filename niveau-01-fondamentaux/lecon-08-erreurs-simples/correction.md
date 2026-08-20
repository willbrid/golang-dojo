# Leçon 8 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Divide

```go
func Divide(a, b float64) (float64, error) {
	if b == 0 {
		return 0, errors.New("division par zéro")
	}
	return a / b, nil
}
```

**Pourquoi `0` et non `NaN` ?** Parce que la convention Go est claire : quand `err != nil`,
les autres résultats **ne doivent pas être utilisés**. Retourner la zéro-valeur le signale
sans ambiguïté. `NaN` serait une valeur « intéressante » qui inviterait à l'utiliser — et
`NaN` se propage silencieusement dans tous les calculs suivants, transformant une erreur
locale en corruption diffuse.

Note : en Go, `1.0/0.0` sur des **variables** flottantes donne `+Inf` sans paniquer (norme
IEEE 754). Seule la division entière par zéro panique.

## E2 — Contexte

```go
func ReadFirstLine(path string) (string, error) {
	f, err := os.Open(path)
	if err != nil {
		return "", fmt.Errorf("ouverture de %q : %w", path, err)
	}
	defer f.Close() // en lecture seule, ignorer l'erreur de Close est acceptable

	sc := bufio.NewScanner(f)
	if !sc.Scan() {
		if err := sc.Err(); err != nil {
			return "", fmt.Errorf("lecture de %q : %w", path, err)
		}
		return "", fmt.Errorf("fichier %q vide", path)
	}
	return sc.Text(), nil
}
```

Les quatre messages :

```
ouverture de "absent.txt" : open absent.txt: no such file or directory
ouverture de "/tmp" : ... (réussit !) puis lecture de "/tmp" : read /tmp: is a directory
fichier "vide.txt" vide
(cas normal : pas d'erreur)
```

Le cas du répertoire est instructif : `os.Open` **réussit** sur un répertoire, c'est la
lecture qui échoue. Découvrir ça en testant vaut mieux que le lire.

Le `!sc.Scan()` sans erreur signifie « fin de flux atteinte immédiatement » : le fichier est
vide. Ce n'est pas forcément une erreur selon le domaine — c'est un choix à documenter.

## E3 — Critique des messages

| Original | Corrigé | Reproche |
|---|---|---|
| `"Erreur: Impossible d'ouvrir le fichier."` | `fmt.Errorf("ouverture de %q : %w", path, err)` | Majuscule, point final, mot « Erreur » redondant, aucun contexte sur *quel* fichier |
| `"échec"` | `fmt.Errorf("connexion à %s : %w", addr, err)` | Ne dit rien : ni quoi, ni pourquoi |
| `"ERREUR FATALE !!!"` | `errors.New("état interne incohérent")` | Cris, redondance ; la gravité est décidée par l'appelant, pas par le message |
| `fmt.Errorf("erreur lors de la lecture: %v", err)` | `fmt.Errorf("lecture de %q : %w", path, err)` | `%v` casse la chaîne d'erreurs ; « erreur » redondant ; pas d'espace avant `:` (typographie française) ; pas de contexte |
| `"Le user n'existe pas"` | `fmt.Errorf("utilisateur %q introuvable", id)` | Majuscule, anglicisme, pas d'identifiant |

## E4 / E5 — Sentinelle et `%w`

```go
var ErrNotFound = errors.New("introuvable")

func FindUser(id int) (string, error) {
	if id != 1 {
		return "", fmt.Errorf("utilisateur %d : %w", id, ErrNotFound)
	}
	return "Alice", nil
}

// Un niveau de plus
func LoadProfile(id int) (string, error) {
	name, err := FindUser(id)
	if err != nil {
		return "", fmt.Errorf("chargement du profil : %w", err)
	}
	return name, nil
}

_, err := LoadProfile(42)
fmt.Println(err)                          // chargement du profil : utilisateur 42 : introuvable
fmt.Println(errors.Is(err, ErrNotFound))  // true — à travers DEUX enveloppes
```

Avec `%v` à la place de `%w`, l'affichage est **identique** mais `errors.Is` retourne
`false` : l'erreur d'origine a été aplatie en texte. C'est un bug invisible à la lecture des
logs et qui casse toute la logique de traitement en aval. D'où la règle : `%w` par défaut
quand on relaie une erreur reçue.

## Exercice intermédiaire — `csvcheck`

```go
package main

import (
	"bufio"
	"errors"
	"fmt"
	"os"
	"strconv"
	"strings"
)

// LineError décrit un problème sur une ligne précise.
type LineError struct {
	Line int
	Msg  string
	Err  error
}

func (e *LineError) Error() string {
	if e.Err != nil {
		return fmt.Sprintf("ligne %d : %s : %v", e.Line, e.Msg, e.Err)
	}
	return fmt.Sprintf("ligne %d : %s", e.Line, e.Msg)
}
func (e *LineError) Unwrap() error { return e.Err }

// Validate lit r et retourne TOUTES les erreurs rencontrées, plus le nombre de
// lignes traitées. Elle n'affiche rien et n'appelle jamais os.Exit : c'est
// l'appelant qui décide quoi en faire.
func Validate(f *os.File) (errs []error, lines int, fatal error) {
	sc := bufio.NewScanner(f)

	if !sc.Scan() {
		return nil, 0, errors.New("fichier vide : en-tête attendue")
	}
	if h := strings.TrimSpace(sc.Text()); h != "name,age" {
		return nil, 0, fmt.Errorf("en-tête invalide : %q", h)
	}

	for n := 2; sc.Scan(); n++ {
		lines++
		fields := strings.Split(sc.Text(), ",")
		if len(fields) != 2 {
			errs = append(errs, &LineError{n, fmt.Sprintf("2 champs attendus, %d trouvés", len(fields)), nil})
			continue
		}
		if strings.TrimSpace(fields[0]) == "" {
			errs = append(errs, &LineError{n, "champ « name » vide", nil})
		}
		age, err := strconv.Atoi(strings.TrimSpace(fields[1]))
		if err != nil {
			errs = append(errs, &LineError{n, "champ « age »", err})
			continue
		}
		if age < 0 || age > 150 {
			errs = append(errs, &LineError{n, fmt.Sprintf("âge %d hors bornes [0,150]", age), nil})
		}
	}
	if err := sc.Err(); err != nil {
		return errs, lines, fmt.Errorf("lecture : %w", err)
	}
	return errs, lines, nil
}

func main() {
	if len(os.Args) != 2 {
		fmt.Fprintln(os.Stderr, "usage: csvcheck <fichier.csv>")
		os.Exit(1)
	}
	f, err := os.Open(os.Args[1])
	if err != nil { // erreur FATALE : on ne peut rien valider du tout
		fmt.Fprintf(os.Stderr, "ouverture de %q : %v\n", os.Args[1], err)
		os.Exit(1)
	}
	defer f.Close()

	errs, lines, fatal := Validate(f)
	if fatal != nil {
		fmt.Fprintln(os.Stderr, fatal)
		os.Exit(1)
	}
	for _, e := range errs {
		fmt.Fprintln(os.Stderr, e)
	}
	if len(errs) > 0 {
		fmt.Fprintf(os.Stderr, "%d erreurs sur %d lignes\n", len(errs), lines)
		os.Exit(1)
	}
	fmt.Printf("%d lignes valides\n", lines)
}
```

**`[]error` ou `errors.Join` ?** Les deux se défendent :

- **`errors.Join`** produit une seule `error`, composable avec le reste du code, et
  `errors.Is` traverse toutes les branches. Idéal quand l'appelant veut juste « ça a échoué,
  voici pourquoi ».
- **`[]error` explicite** garde chaque erreur individuellement accessible : on peut les
  compter, les trier, les paginer, en afficher les dix premières. Ici on veut compter et
  formater ligne par ligne — le slice est plus direct.

En pratique : `[]error` à l'intérieur, `errors.Join(errs...)` à la frontière publique si
l'API doit retourner une seule `error`.

**La distinction fatale/collectée** est le vrai enseignement : un fichier illisible rend
toute validation impossible (fatale) ; une ligne invalide n'empêche pas de valider les
autres (collectée). Les deux passent par `error` mais ne se traitent pas pareil — d'où les
trois résultats de `Validate`.

## Défi — Type d'erreur porteur de données

```go
type ValidationError struct {
	Line  int
	Field string
	Value string
	Err   error
}

func (e *ValidationError) Error() string {
	return fmt.Sprintf("ligne %d, champ %q (valeur %q) : %v", e.Line, e.Field, e.Value, e.Err)
}

func (e *ValidationError) Unwrap() error { return e.Err }

// Côté appelant
var ve *ValidationError
if errors.As(err, &ve) {
	fmt.Printf("erreur à la ligne %d sur le champ %s\n", ve.Line, ve.Field)
	// décision PROGRAMMATIQUE, sans analyser de texte
}
```

**(d) Récepteur valeur ou pointeur ?**

`errors.As(err, &ve)` compare le **type dynamique** stocké dans l'interface avec le type
pointé par la cible. Si `Error()` est défini sur `*ValidationError`, seul un
`*ValidationError` satisfait `error` — et il faut donc `var ve *ValidationError`. Si
`Error()` est défini sur la valeur, **les deux** `ValidationError` et `*ValidationError`
satisfont `error`, et le code devient ambigu : selon ce que la fonction a retourné,
`errors.As` avec `**ValidationError` ou `*ValidationError` réussira ou échouera.

**Convention Go, quasi universelle : récepteur pointeur pour les types d'erreur**, et on
retourne toujours `&MyError{…}`. Cela évite l'ambiguïté, évite de copier la struct à chaque
manipulation, et rend la comparaison d'identité possible.

Attention au piège de la leçon 11 : `var e *ValidationError = nil; return e` produit une
`error` **non nil**. Toujours `return nil` explicitement.

## Réponses du quiz

1. `type error interface { Error() string }` — une interface à une seule méthode.
2. Qu'aucune erreur ne s'est produite. C'est la zéro-valeur d'une interface.
3. En **dernier**. La convention rend le motif immédiatement reconnaissable et permet aux
   outils (`errcheck`, `go vet`) de raisonner dessus.
4. **Non.** Sauf contrat explicitement documenté, comme `io.Reader.Read` qui peut retourner
   `n > 0` avec une erreur.
5. `%w` **enveloppe** : la chaîne est préservée pour `errors.Is`/`errors.As`. `%v` aplatit
   l'erreur en texte, la chaîne est rompue.
6. Parce que les messages s'**emboîtent** : `lecture de config : ouverture de app.yaml :
   no such file`. Une majuscule ou un point au milieu de cette chaîne serait absurde.
7. Parce qu'`errors.Is` **déballe** la chaîne des `%w`. Une comparaison `==` échoue dès que
   l'erreur a été enveloppée une seule fois — ce qui arrive tôt ou tard.
8. Avec `_ = f()`, et un commentaire si la raison n'est pas évidente.
9. Uniquement pour un **bug de programmation** : invariant violé, état impossible,
   initialisation ratée dans un `init()`. Jamais pour une erreur d'entrée/sortie, de réseau
   ou d'entrée utilisateur.
10. Parce que le **type** `error` porte déjà cette information. Le message doit dire *ce qui*
    a échoué, pas répéter *que* quelque chose a échoué.
