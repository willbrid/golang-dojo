# Leçon 6 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — `%w` contre `%v`

```go
var ErrBase = errors.New("cause profonde")

func c() error { return ErrBase }
func b() error { return fmt.Errorf("étape B : %w", c()) }
func a() error { return fmt.Errorf("étape A : %w", b()) }

err := a()
fmt.Println(err)                     // étape A : étape B : cause profonde
fmt.Println(errors.Is(err, ErrBase)) // true
```

En remplaçant le `%w` de `b` par `%v` :

```
étape A : étape B : cause profonde     ← message IDENTIQUE
false                                   ← errors.Is ne trouve plus rien
```

**Le message affiché ne change pas d'un caractère.** C'est ce qui rend ce bug redoutable : les
logs restent parfaits, seule la logique de traitement en aval casse. Aucun test qui compare des
chaînes ne le détectera ; seul un test sur `errors.Is` le fera.

Un `%w` manquant au milieu d'une chaîne de dix niveaux coupe tout ce qui est en dessous.

## E2 — `Is` ou `As` ?

| Besoin | Outil | Pourquoi |
|---|---|---|
| (a) le fichier n'existe pas | **`Is`** | on veut reconnaître un cas : `errors.Is(err, os.ErrNotExist)` |
| (b) récupérer le chemin fautif | **`As`** | on veut une **donnée** : `*fs.PathError` porte `.Path` |
| (c) un délai a expiré | **`Is`** | `errors.Is(err, context.DeadlineExceeded)` |
| (d) le numéro de ligne d'une erreur d'analyse | **`As`** | la ligne est une donnée du type d'erreur |
| (e) violation de contrainte d'unicité SQL | **`As`** | le code SQLSTATE est dans `*pgconn.PgError` |

```go
f, err := os.Open("/absent")

// (a)
if errors.Is(err, os.ErrNotExist) {
	fmt.Println("le fichier n'existe pas")
}

// (b)
var pe *fs.PathError
if errors.As(err, &pe) {
	fmt.Printf("op=%s chemin=%s cause=%v\n", pe.Op, pe.Path, pe.Err)
	// op=open chemin=/absent cause=no such file or directory
}
```

La règle mnémotechnique : **`Is` répond par oui ou non, `As` remplit une variable.**

## E3 — Type d'erreur

```go
type HTTPError struct {
	Code int
	URL  string
	Err  error
}

func (e *HTTPError) Error() string {
	return fmt.Sprintf("%s : statut %d", e.URL, e.Code)
}
func (e *HTTPError) Unwrap() error { return e.Err }

func NewHTTPError(code int, url string, err error) *HTTPError {
	return &HTTPError{Code: code, URL: url, Err: err}
}

// Côté appelant : décision PROGRAMMATIQUE, sans analyser de texte
var he *HTTPError
switch {
case errors.As(err, &he) && he.Code >= 500:
	return retry(he.URL)
case errors.As(err, &he) && he.Code >= 400:
	return fmt.Errorf("requête invalide, abandon : %w", err)
case err != nil:
	return err
}
```

Le point à saisir : décider en fonction de `he.Code >= 500` est **robuste**. Décider en
cherchant `"500"` dans `err.Error()` fonctionnerait aujourd'hui et casserait à la première
reformulation du message — ou pire, se déclencherait sur une URL contenant « 500 ».

## E4 — Le piège du récepteur

| `Error()` sur | Cible `errors.As` | Résultat |
|---|---|---|
| valeur `MyErr` | `var e MyErr` | ✔ trouve — **si** l'erreur a été retournée par valeur |
| valeur `MyErr` | `var e *MyErr` | ✔ trouve — **si** elle a été retournée par pointeur |
| pointeur `*MyErr` | `var e MyErr` | ✘ **compile**, mais ne trouve jamais rien |
| pointeur `*MyErr` | `var e *MyErr` | ✔ trouve, toujours |

Le troisième cas est le piège : `errors.As` ne signale aucune erreur, il retourne simplement
`false` en permanence. Le code semble correct, la branche n'est jamais prise.

Avec `Error()` sur la **valeur**, les deux formes satisfont `error`, et le comportement dépend
alors de ce que la fonction a retourné — donc d'un détail d'implémentation que l'appelant ne
voit pas. **D'où la convention universelle : récepteur pointeur pour les types d'erreur, et
`return &MyErr{…}` systématiquement.**

## E5 — `errors.Join`

```go
errs := []error{
	fmt.Errorf("a : %w", ErrBase),
	errors.New("b"),
	fmt.Errorf("c : %w", ErrBase),
	errors.New("d"),
	errors.New("e"),
}
joined := errors.Join(errs...)

fmt.Println(joined)                        // les cinq messages, un par ligne
fmt.Println(errors.Is(joined, ErrBase))    // true — Join est traversée
fmt.Println(errors.Unwrap(joined))         // <nil>  ← la surprise
```

**`errors.Unwrap` retourne `nil`.** La raison est dans la signature : `errors.Unwrap` cherche
une méthode `Unwrap() error` (au singulier), alors que le type produit par `Join` implémente
`Unwrap() []error` (au pluriel). Ce sont deux interfaces distinctes.

`errors.Is` et `errors.As` connaissent les deux formes et explorent l'arbre ; `errors.Unwrap`,
volontairement, ne gère que la chaîne linéaire. Pour accéder aux erreurs individuelles d'un
`Join`, il faut faire l'assertion soi-même :

```go
if multi, ok := joined.(interface{ Unwrap() []error }); ok {
	for _, e := range multi.Unwrap() {
		fmt.Println("-", e)
	}
}
```

C'est exactement la réponse à la contrainte 4 de l'exercice intermédiaire.

## Exercice intermédiaire — `configloader`

```go
package main

import (
	"errors"
	"fmt"
	"os"
	"strconv"
	"strings"
	"time"
)

var (
	ErrMissingKey = errors.New("clé manquante")
	ErrBadFormat  = errors.New("format invalide")
)

type ConfigError struct {
	Key    string
	Value  string
	Reason string
	Err    error
}

func (e *ConfigError) Error() string {
	if e.Value != "" {
		return fmt.Sprintf("clé %q (valeur %q) : %s", e.Key, e.Value, e.Reason)
	}
	return fmt.Sprintf("clé %q : %s", e.Key, e.Reason)
}
func (e *ConfigError) Unwrap() error { return e.Err }

type Config struct {
	Host    string
	Port    int
	Timeout time.Duration
	Debug   bool
}

// LoadFromString contient TOUTE la logique et ne touche jamais au système de
// fichiers : elle est donc testable sans fichier temporaire.
// CHOIX : une string plutôt qu'un io.Reader, parce qu'un fichier de
// configuration tient toujours en mémoire et qu'aucun traitement en flux n'a de
// sens ici. Pour un format de taille non bornée (logs, CSV), io.Reader
// s'imposerait — c'est le critère, pas une préférence stylistique.
func LoadFromString(s string) (*Config, error) {
	cfg := &Config{Timeout: 30 * time.Second} // valeurs par défaut
	seen := map[string]bool{}
	var errs []error

	for i, line := range strings.Split(s, "\n") {
		line = strings.TrimSpace(line)
		if line == "" || strings.HasPrefix(line, "#") {
			continue
		}
		key, value, ok := strings.Cut(line, "=")
		if !ok {
			errs = append(errs, &ConfigError{
				Key: fmt.Sprintf("ligne %d", i+1), Value: line,
				Reason: "séparateur '=' absent", Err: ErrBadFormat,
			})
			continue
		}
		key, value = strings.TrimSpace(key), strings.TrimSpace(value)
		seen[key] = true

		switch key {
		case "host":
			if value == "" {
				errs = append(errs, &ConfigError{Key: key, Reason: "valeur vide", Err: ErrBadFormat})
				continue
			}
			cfg.Host = value
		case "port":
			n, err := strconv.Atoi(value)
			if err != nil {
				errs = append(errs, &ConfigError{Key: key, Value: value,
					Reason: "entier attendu", Err: err}) // enveloppe l'erreur de strconv
				continue
			}
			if n < 1 || n > 65535 {
				errs = append(errs, &ConfigError{Key: key, Value: value,
					Reason: "hors des bornes [1,65535]", Err: ErrBadFormat})
				continue
			}
			cfg.Port = n
		case "timeout":
			n, err := strconv.Atoi(value)
			if err != nil {
				errs = append(errs, &ConfigError{Key: key, Value: value,
					Reason: "entier attendu (secondes)", Err: err})
				continue
			}
			cfg.Timeout = time.Duration(n) * time.Second
		case "debug":
			b, err := strconv.ParseBool(value)
			if err != nil {
				errs = append(errs, &ConfigError{Key: key, Value: value,
					Reason: "booléen attendu", Err: err})
				continue
			}
			cfg.Debug = b
		default:
			errs = append(errs, &ConfigError{Key: key, Reason: "clé inconnue", Err: ErrBadFormat})
		}
	}

	for _, required := range []string{"host", "port"} {
		if !seen[required] {
			errs = append(errs, &ConfigError{Key: required, Reason: "obligatoire", Err: ErrMissingKey})
		}
	}

	if len(errs) > 0 {
		return nil, errors.Join(errs...)
	}
	return cfg, nil
}

// Load n'ouvre que le fichier : une erreur d'ouverture est FATALE et retournée
// immédiatement, sans passer par la collecte.
func Load(path string) (*Config, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return nil, fmt.Errorf("lecture de %q : %w", path, err)
	}
	cfg, err := LoadFromString(string(data))
	if err != nil {
		return nil, fmt.Errorf("configuration %q : %w", path, err)
	}
	return cfg, nil
}

// BadKeys extrait les clés fautives — réponse à la contrainte 4.
func BadKeys(err error) []string {
	var out []string
	var collect func(error)
	collect = func(e error) {
		var ce *ConfigError
		if errors.As(e, &ce) {
			out = append(out, ce.Key)
			return
		}
		// errors.Join implémente Unwrap() []error, au PLURIEL
		if multi, ok := e.(interface{ Unwrap() []error }); ok {
			for _, sub := range multi.Unwrap() {
				collect(sub)
			}
			return
		}
		if inner := errors.Unwrap(e); inner != nil {
			collect(inner)
		}
	}
	collect(err)
	return out
}
```

**Contrainte 4 — le vrai sujet de conception.** `errors.Join` aplatit tout en une seule
`error` : `errors.As` ne rendra que **la première** `*ConfigError` rencontrée. Pour obtenir la
liste complète, deux réponses :

1. **Explorer l'arbre soi-même** via l'assertion `interface{ Unwrap() []error }` — c'est
   `BadKeys` ci-dessus. Avantage : l'API publique reste `(*Config, error)`, parfaitement
   idiomatique. Inconvénient : l'appelant doit connaître cette technique, et le code de parcours
   est du bruit.
2. **Changer le type de retour** : `func LoadFromString(s string) (*Config, []*ConfigError)`.
   Avantage : trivial à consommer. Inconvénient : ce n'est plus une `error`, donc la fonction ne
   se compose plus avec le reste de l'écosystème — `errors.Is`, `%w`, les signatures qui
   attendent `error`.

**La conception recommandée combine les deux** : retourner `error` pour la composition, **et**
exposer une fonction d'extraction (`BadKeys`) pour ceux qui en ont besoin. C'est ce que fait la
bibliothèque standard avec `json.UnmarshalTypeError` : l'API retourne `error`, et le type
concret est là pour qui veut creuser.

**Contrainte 5 — `string` ou `io.Reader` ?** Le critère est la **taille bornée**. Un fichier de
configuration tient en mémoire par nature ; un flux de logs, non. Choisir `io.Reader` par
mimétisme ajoute ici une indirection sans bénéfice. Choisir `string` sur un format non borné
serait une faute de conception. Ce n'est pas une question de style.

## Défi

**a) `MultiError` maison**

```go
type MultiError struct{ errs []error }

func (m *MultiError) Add(err error) {
	if err != nil {
		m.errs = append(m.errs, err)
	}
}

func (m *MultiError) Error() string {
	msgs := make([]string, len(m.errs))
	for i, e := range m.errs {
		msgs[i] = e.Error()
	}
	return strings.Join(msgs, "\n")
}

// Unwrap au PLURIEL : c'est cette signature qu'errors.Is et errors.As explorent.
func (m *MultiError) Unwrap() []error { return m.errs }

func (m *MultiError) Errors() []error { return slices.Clone(m.errs) } // copie défensive

// ErrOrNil : le point crucial. Retourner *MultiError directement produirait une
// interface NON nil contenant un pointeur nil (leçon 4).
func (m *MultiError) ErrOrNil() error {
	if m == nil || len(m.errs) == 0 {
		return nil
	}
	return m
}
```

Ce qu'apporte la version maison sur `errors.Join` : l'accumulation **incrémentale** (`Add`),
l'accès direct à la liste (`Errors`), le comptage, le filtrage, et la possibilité d'ajouter des
métadonnées. Ce qu'elle coûte : un type de plus à maintenir, et le piège d'`ErrOrNil` qu'il ne
faut surtout pas oublier.

Pour un simple regroupement en fin de fonction, `errors.Join` suffit et évite le piège.

**b) Erreur temporaire**

```go
// Piste 1 : une interface détectée par assertion
type temporary interface{ Temporary() bool }

func isTemporary(err error) bool {
	var t temporary
	return errors.As(err, &t) && t.Temporary()
}

// Piste 2 : une sentinelle enveloppée
var ErrTemporary = errors.New("erreur temporaire")
// … retourner fmt.Errorf("timeout : %w", ErrTemporary)
// détection : errors.Is(err, ErrTemporary)

// Piste 3 : un type d'erreur portant l'information
type OpError struct{ Op string; Retryable bool; Err error }
```

**`net.Error.Temporary()` a été déprécié en Go 1.18**, et la raison est instructive : la notion
de « temporaire » n'était **pas définie de manière cohérente**. Selon l'implémentation, elle
retournait `true` pour des erreurs qui ne se résoudraient jamais (un `ECONNREFUSED` sur un port
fermé), et `false` pour des erreurs manifestement transitoires. Les appelants réessayaient donc
indéfiniment des opérations vouées à l'échec, ou abandonnaient des opérations récupérables.

La leçon est plus large que Go : **« temporaire » n'est pas une propriété de l'erreur, c'est
une décision de l'appelant, qui dépend du contexte**. Un timeout est temporaire pour un lot de
nuit, définitif pour une requête interactive à budget de 200 ms.

**Recommandation** : la piste 3, un type portant l'information, **décidée par la couche qui
connaît le contexte** — typiquement la couche transport, pas la couche métier. Et documenter
précisément ce que « réessayable » signifie dans ce domaine.

**c) Frontière d'API**

```go
func ToPublic(err error) (code int, message string) {
	switch {
	case err == nil:
		return 200, "ok"
	case errors.Is(err, ErrNotFound):
		return 404, "ressource introuvable"
	case errors.Is(err, ErrConflict):
		return 409, "conflit avec l'état actuel de la ressource"
	case errors.Is(err, ErrValidation):
		return 400, "requête invalide"
	case errors.Is(err, ErrForbidden):
		return 403, "accès refusé"
	default:
		// On ne renvoie RIEN de l'erreur interne : elle pourrait contenir un
		// nom de table, une requête SQL, un chemin de fichier.
		return 500, "erreur interne"
	}
}
```

**Où vit cette fonction ?** Dans la couche la plus **externe** — le paquet HTTP, l'adaptateur,
le handler. Jamais dans le domaine : le domaine ne doit pas connaître l'existence de HTTP. Si
`ToPublic` vivait dans le paquet métier, celui-ci dépendrait d'une notion de code de statut,
et deviendrait inutilisable derrière gRPC ou une file de messages. C'est le principe qui sera
formalisé au niveau 10.

**Qu'arrive-t-il à une nouvelle erreur interne non prévue ?** Elle tombe dans le `default` et
devient un 500 avec un message générique. C'est **le bon comportement par défaut** : ne rien
fuiter. Mais c'est aussi un angle mort — une erreur de validation ajoutée sans mettre `ToPublic`
à jour sera présentée à l'utilisateur comme une panne serveur. Deux parades : journaliser
systématiquement le cas `default` avec l'erreur complète (côté serveur uniquement), et écrire un
test qui vérifie que chaque sentinelle exportée du domaine a une traduction.

## Réponses du quiz

1. **Le texte produit est identique.** La différence est structurelle : `%w` conserve un lien
   vers l'erreur d'origine (`Unwrap`), `%v` ne conserve que le texte. `errors.Is` et
   `errors.As` cessent de fonctionner avec `%v`.
2. `Unwrap() error` pour une cause unique, ou `Unwrap() []error` pour plusieurs.
3. Elle compare par **identité** en remontant toute la chaîne d'`Unwrap`, y compris à travers
   les branches d'un `Join`. Un type peut personnaliser la comparaison via une méthode
   `Is(error) bool`.
4. Elle cherche dans la chaîne la première erreur **assignable au type de la cible** et la lui
   affecte. Le second argument doit être un **pointeur non nil** vers une variable du type
   recherché — sinon elle panique explicitement.
5. **Sentinelle** quand l'appelant veut seulement reconnaître un cas, sans donnée associée.
   **Type d'erreur** quand il a besoin d'une donnée pour agir (ligne, champ, code, URL).
6. Pour éviter l'ambiguïté : avec un récepteur valeur, `MyErr` **et** `*MyErr` satisfont
   `error`, et le succès d'`errors.As` dépend alors de la façon dont l'erreur a été retournée.
7. `errors.Join()` sans argument retourne `nil`. Avec uniquement des `nil`, également `nil` :
   les entrées nulles sont écartées.
8. `Unwrap() []error` — au **pluriel**. C'est pourquoi `errors.Unwrap` (au singulier) retourne
   `nil` sur le résultat d'un `Join`.
9. Parce que l'erreur apparaîtrait alors autant de fois qu'il y a de niveaux, chacun ajoutant
   sa ligne de log pour le même incident. La journalisation appartient au **point de décision**,
   c'est-à-dire à celui qui traite l'erreur au lieu de la remonter.
10. **Une seule** : celle que le niveau inférieur ne pouvait pas connaître.
