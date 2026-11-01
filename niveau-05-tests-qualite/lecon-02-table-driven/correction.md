# Leçon 2 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Convertir

```go
func TestSlug(t *testing.T) {
	tests := []struct {
		name, in, want string
	}{
		{"simple", "Bonjour le Monde", "bonjour-le-monde"},
		{"accents", "Ça, c'est l'été!", "ca-c-est-l-ete"},
		{"majuscules accentuées", "ÉLÈVE ÂGÉ", "eleve-age"},
		{"vide", "", ""},
		{"ponctuation seule", "?!…", ""},
		{"chiffres", "Go 1.27", "go-1-27"},
		{"tirets multiples", "a -- b", "a-b"},
		{"espaces en bord", "  x  ", "x"},
		{"emoji", "go 🚀 vite", "go-vite"},
		{"ligature", "œuvre", "oeuvre"},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := slug.Slug(tt.in); got != tt.want {
				t.Errorf("Slug(%q) = %q, attendu %q", tt.in, got, tt.want)
			}
		})
	}
}
```

Pour trois cas sur un nom de champ aussi évident, des littéraux **positionnels**
(`{"simple", "…", "…"}`) restent lisibles ; dès qu'il y a un champ facultatif (`wantErr`), on
passe aux champs nommés. Avec le `Slug` de N4 L1, le cas « majuscules accentuées » fonctionne
grâce au `ToLower` placé **avant** le `Replacer` — un bug classique si l'ordre est inversé.

## E2 — Noms en double

Mesuré : le second sous-test s'appelle `TestX/a#01`. `go test -run 'TestX/a'` lance **les
deux** (c'est une expression régulière qui correspond aux deux), et un relecteur qui voit
`a#01` dans un échec doit compter les lignes de la table pour retrouver le cas. Les noms doivent
être uniques **et** parlants.

## E3 — `reflect.DeepEqual`

Mesuré : `reflect.DeepEqual(nil, []string{})` → **false** ; `slices.Equal` → **true**.

`slices.Equal` compare les éléments (longueur 0 des deux côtés) ; `DeepEqual` distingue nil et
vide. Quand le contrat dit « jamais nil », c'est **exactement** la distinction qu'on veut
tester — mais explicitement : `if got == nil { t.Error("nil, attendu une liste vide") }`, puis
`slices.Equal` pour le contenu. Un `DeepEqual` qui échoue pour cette raison donne un message
incompréhensible (`[] != []`).

## E4 — La variable de boucle

Mesuré avec `go 1.21` dans `go.mod` : les quatre sous-tests parallèles affichent **`c`** — le
dernier cas. Avec `go 1.27` : `a`, `a`, `c`, `b` (chacun le sien, ordre quelconque puisque
parallèle).

Pour les vérifications, c'est pire qu'un affichage faux : les quatre sous-tests vérifieraient
**tous le dernier cas**. Trois cas sur quatre ne seraient jamais testés, et le test passerait
au vert. C'est la raison du changement de sémantique de Go 1.22 — un des rares changements de
langage, rendu possible par la ligne `go` du `go.mod` (N2 L9).

## E5 — Table d'erreurs

Forme attendue :

```go
tests := []struct {
	name    string
	in      string
	want    time.Time
	wantErr error
}{
	{name: "date", in: "2026-11-05", want: time.Date(2026, 11, 5, 0, 0, 0, 0, paris)},
	{name: "relative", in: "+3d", want: time.Date(2026, 11, 4, 12, 0, 0, 0, paris)},
	{name: "format inconnu", in: "05/11/2026", wantErr: task.ErrDueFormat},
	{name: "date impossible", in: "2026-02-30", wantErr: task.ErrDueFormat},
	{name: "dans le passé", in: "2020-01-01", wantErr: task.ErrDueInPast},
	{name: "relatif négatif", in: "+-3d", wantErr: task.ErrDueFormat},
	{name: "relatif trop grand", in: "+99999d", wantErr: task.ErrDueRange},
	{name: "vide", in: "", wantErr: task.ErrDueEmpty},
}
```

Avec un `now` **fixe** passé à `ParseDue` (contrainte 4 du projet 2) : c'est ce qui rend `+3d`
testable. La leçon 3 généralise cette idée.

Comparer des `time.Time` : `got.Equal(tt.want)`, jamais `!=` (N4 L4).

## Exercice intermédiaire — Spécifier par la table

```go
package semver

import (
	"errors"
	"testing"
)

var tests = []struct {
	name    string
	a, b    string
	want    int
	wantErr error
}{
	{name: "égales", a: "1.2.3", b: "1.2.3", want: 0},
	{name: "préfixe v", a: "v1.2.3", b: "1.2.3", want: 0},
	{name: "majeure", a: "2.0.0", b: "1.9.9", want: 1},
	{name: "mineure", a: "1.2.0", b: "1.3.0", want: -1},
	{name: "correctif", a: "1.2.4", b: "1.2.3", want: 1},
	{name: "plusieurs chiffres", a: "1.10.0", b: "1.9.0", want: 1},
	{name: "pré-version < version", a: "1.0.0-alpha", b: "1.0.0", want: -1},
	{name: "alpha < beta", a: "1.0.0-alpha", b: "1.0.0-beta", want: -1},
	{name: "numérique < alphanumérique", a: "1.0.0-1", b: "1.0.0-alpha", want: -1},
	{name: "identifiants numériques", a: "1.0.0-rc.2", b: "1.0.0-rc.10", want: -1},
	{name: "plus d'identifiants", a: "1.0.0-alpha", b: "1.0.0-alpha.1", want: -1},
	{name: "build ignoré", a: "1.0.0+build5", b: "1.0.0+build9", want: 0},
	{name: "chaîne semver.org 1", a: "1.0.0-alpha.1", b: "1.0.0-alpha.beta", want: -1},
	{name: "chaîne semver.org 2", a: "1.0.0-alpha.beta", b: "1.0.0-beta", want: -1},
	{name: "chaîne semver.org 3", a: "1.0.0-beta.2", b: "1.0.0-beta.11", want: -1},
	{name: "chaîne semver.org 4", a: "1.0.0-beta.11", b: "1.0.0-rc.1", want: -1},
	{name: "chaîne semver.org 5", a: "1.0.0-rc.1", b: "1.0.0", want: -1},
	{name: "deux composantes", a: "1.2", b: "1.2.0", wantErr: ErrInvalid},
	{name: "zéro en tête", a: "01.2.3", b: "1.2.3", wantErr: ErrInvalid},
	{name: "négatif", a: "1.-2.3", b: "1.2.3", wantErr: ErrInvalid},
	{name: "vide", a: "", b: "1.2.3", wantErr: ErrInvalid},
	{name: "pré-version vide", a: "1.0.0-", b: "1.0.0", wantErr: ErrInvalid},
	{name: "identifiant vide", a: "1.0.0-a..b", b: "1.0.0", wantErr: ErrInvalid},
}

func TestCompare(t *testing.T) {
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, err := Compare(tt.a, tt.b)
			if !errors.Is(err, tt.wantErr) {
				t.Fatalf("Compare(%q, %q) erreur = %v, attendu %v", tt.a, tt.b, err, tt.wantErr)
			}
			if got != tt.want {
				t.Errorf("Compare(%q, %q) = %d, attendu %d", tt.a, tt.b, got, tt.want)
			}
		})
	}
}

func TestCompareAntisymmetric(t *testing.T) {
	for _, tt := range tests {
		if tt.wantErr != nil {
			continue
		}
		ab, _ := Compare(tt.a, tt.b)
		ba, _ := Compare(tt.b, tt.a)
		if ab != -ba {
			t.Errorf("Compare(%q, %q) = %d mais Compare(%q, %q) = %d", tt.a, tt.b, ab, tt.b, tt.a, ba)
		}
	}
}
```

Sur l'implémentation `return 0, nil` : les **3** cas d'égalité passent, les **20** autres
échouent. C'est rassurant — une table dont la majorité des cas passe sur une implémentation
vide teste surtout le vide.

```go
// Package semver compare des versions sémantiques (semver.org 2.0.0).
package semver

import (
	"cmp"
	"errors"
	"fmt"
	"strconv"
	"strings"
)

// ErrInvalid est retournée pour une version mal formée.
var ErrInvalid = errors.New("version invalide")

type version struct {
	core [3]int
	pre  []string
}

func parse(s string) (version, error) {
	var v version
	s = strings.TrimPrefix(s, "v")
	s, _, _ = strings.Cut(s, "+") // métadonnées de build : ignorées
	s, pre, hasPre := strings.Cut(s, "-")
	parts := strings.Split(s, ".")
	if len(parts) != 3 {
		return v, fmt.Errorf("%q : %w", s, ErrInvalid)
	}
	for i, p := range parts {
		n, err := strconv.Atoi(p)
		if err != nil || n < 0 || (len(p) > 1 && p[0] == '0') {
			return v, fmt.Errorf("composante %q : %w", p, ErrInvalid)
		}
		v.core[i] = n
	}
	if hasPre {
		v.pre = strings.Split(pre, ".")
		for _, id := range v.pre {
			if id == "" {
				return v, fmt.Errorf("pré-version %q : %w", pre, ErrInvalid)
			}
		}
	}
	return v, nil
}

func compareIdent(a, b string) int {
	na, errA := strconv.Atoi(a)
	nb, errB := strconv.Atoi(b)
	switch {
	case errA == nil && errB == nil:
		return cmp.Compare(na, nb)
	case errA == nil:
		return -1 // numérique < alphanumérique
	case errB == nil:
		return 1
	}
	return cmp.Compare(a, b)
}

// Compare rend -1, 0 ou 1 selon que a précède, égale ou suit b.
func Compare(a, b string) (int, error) {
	va, err := parse(a)
	if err != nil {
		return 0, err
	}
	vb, err := parse(b)
	if err != nil {
		return 0, err
	}
	for i := range va.core {
		if c := cmp.Compare(va.core[i], vb.core[i]); c != 0 {
			return c, nil
		}
	}
	switch {
	case len(va.pre) == 0 && len(vb.pre) == 0:
		return 0, nil
	case len(va.pre) == 0:
		return 1, nil // 1.0.0 > 1.0.0-alpha
	case len(vb.pre) == 0:
		return -1, nil
	}
	for i := 0; i < min(len(va.pre), len(vb.pre)); i++ {
		if c := compareIdent(va.pre[i], vb.pre[i]); c != 0 {
			return c, nil
		}
	}
	return cmp.Compare(len(va.pre), len(vb.pre)), nil
}
```

Points de correction :

- **Les cinq cas « chaîne semver.org »** viennent directement de la section 11 de la
  spécification, qui donne l'ordre `1.0.0-alpha < 1.0.0-alpha.1 < 1.0.0-alpha.beta < 1.0.0-beta
  < 1.0.0-beta.2 < 1.0.0-beta.11 < 1.0.0-rc.1 < 1.0.0`. Recopier la spec dans la table est la
  meilleure façon de la couvrir.
- Cas souvent manquants : **zéros en tête** interdits (`01.2.3`), identifiants numériques
  comparés **numériquement** (`rc.2 < rc.10`), numérique **avant** alphanumérique, plus
  d'identifiants = plus grand **à égalité de préfixe**.
- **`TestCompareAntisymmetric`** réutilise `tests`, déclarée au niveau du paquet. C'est un
  premier pas vers les tests de **propriétés**, que le fuzzing (leçon 7) poussera plus loin.
- Le « préfixe `v` » n'est **pas** dans semver.org : c'est une convention de Go (les tags de
  module, N2 L9). La table documente ce choix.

## Défi — Quand la table devient illisible

1. Les champs `setup func(*Store)` et `check func(*testing.T, *Store, Result)` signifient que
   **chaque cas a sa propre logique** : la boucle ne fait plus qu'appeler des fonctions écrites
   ailleurs, et on lit chaque cas comme un test séparé, avec en plus l'indirection. `skipOnCI`
   et `parallel` montrent que les cas n'ont même pas les **mêmes conditions d'exécution**. La
   table n'apporte plus rien : elle cache.
2. Garder dans une table les cas qui partagent setup **et** vérifications (une table par
   « famille »). Les cas à setup particulier deviennent des fonctions `TestStoreXxx` séparées,
   avec un utilitaire `newTestStore(t)` partagé. Les cas lents deviennent un test à part avec
   `if testing.Short() { t.Skip(…) }`.
3. **Une table quand les cas ne diffèrent que par leurs données ; des tests séparés dès qu'ils
   diffèrent par leur logique.**

## Quiz

1. Une slice de structs anonymes (les cas), une boucle, un `t.Run(tt.name, …)` par cas, la
   vérification écrite une fois.
2. L'ordre d'une map est aléatoire : sortie instable (sauf si c'est voulu).
3. Lisibilité, cas qui omettent les champs facultatifs, ajout de champs sans casser l'existant.
4. Un champ `wantErr error` et `errors.Is(err, tt.wantErr)`, vrai aussi pour `nil, nil`.
5. Le texte n'est pas un contrat : la moindre reformulation casse le test.
6. Un slice nil et un slice vide sont différents pour lui.
7. Il suspend le sous-test jusqu'à la fin de la boucle, puis l'exécute en parallèle des autres
   sous-tests parallèles.
8. La variable de boucle était partagée entre les tours : les closures exécutées plus tard
   voyaient toutes la dernière valeur. Depuis Go 1.22 (selon la ligne `go`), chaque tour a sa
   propre variable.
9. Quand chaque cas demande une logique de préparation ou de vérification différente.
10. Y ajouter un cas qui reproduit le bug, **avant** de le corriger.
