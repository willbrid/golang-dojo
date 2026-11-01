# Leçon 2 — Table-driven tests

> Ouvrir n'importe quel fichier `_test.go` de la bibliothèque standard ou d'un projet Go
> réputé, c'est presque toujours tomber sur le même patron : un **tableau de cas** et **une
> boucle**. C'est le patron canonique du test en Go, et le premier qu'un relecteur attend
> dans une contribution.

## Objectifs

1. Écrire un test piloté par une table, avec sous-tests nommés.
2. Tester les cas d'erreur dans la même table, avec `errors.Is`.
3. Comparer des valeurs complexes correctement.
4. Paralléliser des sous-tests en connaissant les pièges.

## Explication

### Le patron

```go
func TestParsePort(t *testing.T) {
	tests := []struct {
		name    string
		in      string
		want    int
		wantErr bool
	}{
		{name: "minimum", in: "1", want: 1},
		{name: "http", in: "80", want: 80},
		{name: "maximum", in: "65535", want: 65535},
		{name: "zéro", in: "0", wantErr: true},
		{name: "trop grand", in: "65536", wantErr: true},
		{name: "vide", in: "", wantErr: true},
		{name: "espace", in: " 80", wantErr: true},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, err := ParsePort(tt.in)
			if (err != nil) != tt.wantErr {
				t.Fatalf("ParsePort(%q) erreur = %v, erreur attendue : %v", tt.in, err, tt.wantErr)
			}
			if got != tt.want {
				t.Errorf("ParsePort(%q) = %d, attendu %d", tt.in, got, tt.want)
			}
		})
	}
}
```

Ce que le patron apporte :

- **Ajouter un cas = ajouter une ligne.** Le coût marginal d'un test tend vers zéro : on en
  écrit donc beaucoup plus, et surtout les cas limites.
- **La logique de vérification est écrite une fois** : impossible d'oublier de vérifier
  l'erreur dans le cinquième test.
- **Chaque cas est un sous-test** : il échoue seul, se relance seul
  (`go test -run 'TestParsePort/trop_grand'`), et son nom apparaît dans la sortie.
- **La table se lit comme une spécification** : un relecteur voit d'un coup d'œil quels cas
  sont couverts — et lesquels manquent.

### Les détails qui comptent

**Une struct anonyme**, déclarée dans le test : elle n'a pas d'existence hors de lui.

**Des champs nommés** dans les littéraux (`{name: "http", in: "80", want: 80}`) : les cas
d'erreur peuvent omettre `want`, et l'ajout d'un champ ne casse pas les lignes existantes.

**Un champ `name`** : sans lui, on utiliserait l'indice ou l'entrée comme nom, illisibles dès
que l'entrée est longue ou contient des caractères spéciaux.

**`tt` comme nom de variable** de boucle (pour *test table*) est la convention de la stdlib ; `tc`
(*test case*) se rencontre aussi.

**Une slice, pas une map.** Avec `map[string]struct{…}`, le nom sert de clé, c'est séduisant…
mais l'ordre d'itération est aléatoire (N1 L7) : les échecs n'apparaissent pas dans un ordre
stable. Certains projets le font **exprès**, pour révéler des dépendances entre cas. Choisir en
connaissance de cause ; la slice est le défaut.

### Tester les erreurs

`wantErr bool` vérifie qu'une erreur est **présente**. C'est souvent insuffisant : on veut savoir
**laquelle**. Le champ devient alors une erreur sentinelle, comparée avec `errors.Is` :

```go
tests := []struct {
	name    string
	in      string
	want    int
	wantErr error
}{
	{name: "http", in: "80", want: 80},
	{name: "hors bornes", in: "70000", wantErr: ErrOutOfRange},
	{name: "non numérique", in: "abc", wantErr: strconv.ErrSyntax},
}
…
got, err := ParsePort(tt.in)
if !errors.Is(err, tt.wantErr) {        // errors.Is(nil, nil) == true : le cas sans erreur marche aussi
	t.Fatalf("ParsePort(%q) erreur = %v, attendu %v", tt.in, err, tt.wantErr)
}
```

`errors.Is(nil, nil)` vaut `true` : une seule condition couvre les cas avec et sans erreur.

**Ne jamais comparer le texte** d'une erreur (`err.Error() == "…"`) : le moindre changement de
formulation casse le test, et le texte n'est pas un contrat.

### Comparer des valeurs complexes

`==` ne compile pas sur des slices et des maps. Les options :

| Outil | Pour | Remarque |
|---|---|---|
| `slices.Equal`, `maps.Equal` | slices et maps d'éléments comparables | le bon choix par défaut |
| `reflect.DeepEqual` | tout | pièges : nil ≠ vide, NaN ≠ NaN, compare les champs non exportés (niveau 9) |
| `github.com/google/go-cmp/cmp` | tout, avec un **diff** lisible | `cmp.Diff(want, got)` ; dépendance externe, très répandue |

Pour une struct comparable, `==` suffit et donne le meilleur message avec `%+v`.

```go
if diff := cmp.Diff(tt.want, got); diff != "" {
	t.Errorf("Parse(%q) différence (-attendu +obtenu) :\n%s", tt.in, diff)
}
```

### Paralléliser

```go
for _, tt := range tests {
	t.Run(tt.name, func(t *testing.T) {
		t.Parallel()        // ce sous-test s'exécute en parallèle des autres sous-tests parallèles
		…
	})
}
```

`t.Parallel()` met le sous-test en pause jusqu'à ce que la boucle soit finie, puis exécute
tous les sous-tests parallèles ensemble (limite : `-parallel`, par défaut `GOMAXPROCS`).

**Le piège historique** : avant Go 1.22, la variable de boucle `tt` était **partagée** entre
tous les tours ; les sous-tests parallèles, lancés après la boucle, voyaient tous le **dernier**
cas. Il fallait écrire `tt := tt` au début du corps. Depuis Go 1.22, chaque tour a sa propre
variable (si le `go.mod` déclare `go 1.22` ou plus) : la copie est inutile, et `go vet`
(`copyloopvar`) la signale. On la rencontre encore dans beaucoup de code.

Ne paralléliser que des cas **indépendants** : pas d'état global modifié, pas de fichier
partagé, pas de `t.Setenv` (qui refuse d'ailleurs de fonctionner dans un test parallèle).

### Quand ne pas utiliser de table

Quand chaque cas a une **logique de vérification différente**, une table oblige à ajouter des
champs `check func(…)` et des conditions : le test devient plus difficile à lire que des tests
séparés. Une table est faite pour « mêmes vérifications, données différentes ».

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : duration/duration.go
// Package duration analyse des durées humaines : "1h30", "45min", "2j".
package duration

import (
	"errors"
	"fmt"
	"strconv"
	"strings"
	"time"
)

var (
	ErrEmpty    = errors.New("durée vide")
	ErrUnit     = errors.New("unité inconnue")
	ErrNegative = errors.New("durée négative")
)

var units = map[string]time.Duration{
	"s":   time.Second,
	"min": time.Minute,
	"h":   time.Hour,
	"j":   24 * time.Hour,
}

func isDigit(r rune) bool { return '0' <= r && r <= '9' }

// Parse lit une suite de composantes "<entier><unité>", par exemple "1h30min".
func Parse(s string) (time.Duration, error) {
	s = strings.TrimSpace(s)
	if s == "" {
		return 0, ErrEmpty
	}
	if strings.HasPrefix(s, "-") {
		return 0, fmt.Errorf("%q : %w", s, ErrNegative)
	}
	var total time.Duration
	for s != "" {
		// le nombre : les chiffres en tête
		i := strings.IndexFunc(s, func(r rune) bool { return !isDigit(r) })
		if i == 0 {
			return 0, fmt.Errorf("%q : nombre attendu : %w", s, strconv.ErrSyntax)
		}
		if i < 0 {
			return 0, fmt.Errorf("%q : unité manquante : %w", s, ErrUnit)
		}
		n, err := strconv.Atoi(s[:i])
		if err != nil {
			return 0, err
		}
		// l'unité : tout ce qui suit, jusqu'au prochain chiffre
		rest := s[i:]
		j := strings.IndexFunc(rest, isDigit)
		if j < 0 {
			j = len(rest)
		}
		d, ok := units[rest[:j]]
		if !ok {
			return 0, fmt.Errorf("%q : %w", rest[:j], ErrUnit)
		}
		total += time.Duration(n) * d
		s = rest[j:]
	}
	return total, nil
}
```

```go
// Fichier : duration/duration_test.go
package duration_test

import (
	"errors"
	"strconv"
	"testing"
	"time"

	"exemple/duration"
)

func TestParse(t *testing.T) {
	tests := []struct {
		name    string
		in      string
		want    time.Duration
		wantErr error
	}{
		{name: "heures", in: "2h", want: 2 * time.Hour},
		{name: "minutes", in: "45min", want: 45 * time.Minute},
		{name: "composée", in: "1h30min", want: 90 * time.Minute},
		{name: "jours", in: "2j", want: 48 * time.Hour},
		{name: "espaces autour", in: "  10s ", want: 10 * time.Second},
		{name: "vide", in: "", wantErr: duration.ErrEmpty},
		{name: "blanc", in: "   ", wantErr: duration.ErrEmpty},
		{name: "négative", in: "-1h", wantErr: duration.ErrNegative},
		{name: "unité inconnue", in: "3sem", wantErr: duration.ErrUnit},
		{name: "unité seule", in: "h", wantErr: strconv.ErrSyntax},
		{name: "nombre sans unité", in: "12", wantErr: duration.ErrUnit},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			t.Parallel()
			got, err := duration.Parse(tt.in)
			if !errors.Is(err, tt.wantErr) {
				t.Fatalf("Parse(%q) erreur = %v, attendu %v", tt.in, err, tt.wantErr)
			}
			if got != tt.want {
				t.Errorf("Parse(%q) = %v, attendu %v", tt.in, got, tt.want)
			}
		})
	}
}
```

Sortie de `go test ./...` :

```
ok  	exemple/duration	0.004s
```

Et avec `-v`, onze sous-tests nommés : `TestParse/heures`, `TestParse/unité_inconnue`…

> **Ce que la table a trouvé.** La première version de `Parse` écrite pour ce cours
> parcourait une liste d'unités avec `strings.CutPrefix`. Elle passait les cinq premiers cas…
> et échouait sur deux des cas limites :
>
> ```
> --- FAIL: TestParse/unité_inconnue
>     Parse("3sem") erreur = "em" : nombre attendu : invalid syntax, attendu unité inconnue
> --- FAIL: TestParse/nombre_sans_unité
>     Parse("12") erreur = "12" : nombre attendu : invalid syntax, attendu unité inconnue
> ```
>
> `"3sem"` était lu comme « 3 secondes » suivi de `"em"`. Le correctif — isoler l'unité
> **entière** avant de la chercher dans une map — est celui ci-dessus. Ces deux lignes de la
> table, qui ne coûtaient rien à écrire, ont trouvé deux bugs. C'est tout l'argument du patron.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| Sentinelles exportées | Les tests (et les appelants) testent la **nature** de l'erreur avec `errors.Is`. |
| `%w` partout | Le contexte (`"3sem" : …`) est ajouté sans casser `errors.Is`. |
| `strconv.ErrSyntax` réutilisé | Pas besoin d'inventer une sentinelle quand la stdlib en a une qui dit la même chose. |
| `wantErr error` | Plus précis que `bool` : on vérifie **quelle** erreur. |
| `errors.Is(err, tt.wantErr)` | Couvre aussi les cas sans erreur (`nil`, `nil`). |
| `t.Fatalf` sur l'erreur | Si l'erreur est fausse, comparer la valeur n'a pas de sens. |
| `t.Parallel()` sans `tt := tt` | Go 1.22+ et `go 1.27` dans `go.mod` : chaque tour a sa propre `tt`. |
| Cas « blanc », « unité seule », « nombre sans unité » | Les cas limites coûtent une ligne : on les écrit. Ce sont eux qui trouvent les bugs. |

## Erreurs fréquentes

1. **Des tests copiés-collés** au lieu d'une table.
2. **Pas de champ `name`**, ou des noms dupliqués (le second est renommé `#01`).
3. **`wantErr bool`** quand on pourrait vérifier quelle erreur.
4. **Comparer le texte de l'erreur.**
5. **`reflect.DeepEqual` sans connaître ses pièges** (nil contre vide).
6. **Cas parallèles qui partagent un état.**
7. **Des tables à logique conditionnelle** (`if tt.special { … }`) : écrire un test séparé.
8. **Une table qui ne contient que des cas qui réussissent.**

## Bonnes pratiques Go

- La table d'abord, puis la boucle ; champs nommés ; un `name` lisible et unique.
- Autant de cas d'**échec** que de cas de succès.
- `wantErr error` + `errors.Is`.
- `slices.Equal`/`maps.Equal`, ou `go-cmp` pour un diff lisible.
- `t.Parallel()` pour les cas indépendants ; jamais de `tt := tt` en Go 1.22+.
- Ajouter **un cas** à la table pour chaque bug corrigé.

## Ce que je dois retenir

- Table de structs anonymes + boucle + `t.Run(tt.name, …)`.
- Ajouter un cas = une ligne : on teste donc les cas limites.
- `wantErr error` avec `errors.Is` ; jamais le texte de l'erreur.
- Slice plutôt que map (ordre stable).
- `t.Parallel()` : cas indépendants seulement ; la variable de boucle est par tour depuis Go 1.22.
- Pas de table quand chaque cas se vérifie différemment.

---

➡️ [Exercices](exercices.md)
