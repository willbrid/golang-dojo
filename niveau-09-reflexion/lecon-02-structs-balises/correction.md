# Leçon 2 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Lister les balises

```go
func Tags(v any, key string) map[string]string {
	out := map[string]string{}
	for _, sf := range reflect.VisibleFields(reflect.TypeOf(v)) {
		if sf.IsExported() && !sf.Anonymous {
			out[sf.Name] = sf.Tag.Get(key)
		}
	}
	return out
}
```

Avec `NumField`/`Field`, une struct `User{Base; Name}` ne donne que `Base` (le champ embarqué, sans balise)
et `Name` : les champs **promus** de `Base` n'apparaissent pas. `VisibleFields` les rend tous, avec leur
chemin (`Index`). On écarte l'entrée `Base` elle-même (`Anonymous`), qui n'est pas une donnée.

## E2 — Les balises cassées

| Balise | `Tag.Get("env")` | `go vet` |
|---|---|---|
| `` `env: "A"` `` | `""` | signalée (`bad syntax for struct tag value`) |
| `` `env:"B",default:"1"` `` | `""` | signalée (`key:"value" pairs not separated by spaces`) |
| `` `env:'A'` `` | `""` | signalée (`bad syntax for struct tag value`) |
| `` `env:"B" env:"C"` `` | `"B"` (la première) | **pas** signalée |

La clé en double passe inaperçue : `go vet` ne vérifie les doublons que pour les clés qu'il connaît
(`json`, `xml`), pas pour une clé maison. Un chargeur rigoureux peut le vérifier lui-même au premier
usage d'un type.

## E3 — La troncature

Avec `ParseInt(raw, 10, 64)`, `300` est accepté, puis `SetInt(300)` sur un `int8` stocke **44**
(300 − 256) sans aucune erreur (mesuré). L'autre correction : `f.OverflowInt(n)` rend `true` si `n` ne
tient pas dans le champ ; on rend alors une erreur (voir `set` ci-dessous). Les deux sont correctes ;
`OverflowInt` a l'avantage de produire un message qui nomme le type du champ.

## E4 — Les flottants et les non signés

- `PORT=-1` : `strconv.ParseUint: parsing "-1": invalid syntax` (un non signé n'a pas de signe).
- `PORT=70000` : `value out of range` grâce à `Bits()` (16).
- `RATIO=NaN` : `ParseFloat` l'**accepte** (`NaN`, `Inf`, `+Inf` sont des syntaxes valides). Dans une
  configuration, ce n'est jamais voulu, et `NaN` fausse toute comparaison (`NaN > 0.5` et `NaN <= 0.5`
  sont tous deux faux). Le corrigé refuse les valeurs non finies.

## E5 — Les secrets

`Describe` (ci-dessous) affiche `TOKEN=****`. Le besoin est réel : au démarrage, journaliser la
configuration **effective** (après défauts et variables) est précieux pour diagnostiquer un déploiement
(« pourquoi ce service parle-t-il à la mauvaise base ? »). Mais les journaux sont copiés, indexés,
envoyés à des tiers : un secret qui y passe une fois doit être considéré comme compromis et changé. Le
marquage `secret` à côté de la déclaration du champ rend l'oubli plus difficile qu'une liste de champs à
masquer tenue ailleurs.

## Exercice intermédiaire — Déléguer aux types

```go
// Fichier : envcfg/envcfg.go
package envcfg

import (
	"encoding"
	"errors"
	"fmt"
	"reflect"
	"strconv"
	"strings"
	"time"
)

var (
	durationType = reflect.TypeFor[time.Duration]()
	textType     = reflect.TypeFor[encoding.TextUnmarshaler]()
)

func Load(dst any, lookup func(string) (string, bool)) error {
	v := reflect.ValueOf(dst)
	if v.Kind() != reflect.Pointer || v.IsNil() || v.Elem().Kind() != reflect.Struct {
		return fmt.Errorf("Load : pointeur vers une struct attendu, reçu %T", dst)
	}
	_, err := load(v.Elem(), "", lookup)
	return err
}

// load rend aussi « au moins une variable trouvée », pour décider d'allouer un *Struct.
func load(v reflect.Value, prefix string, lookup func(string) (string, bool)) (bool, error) {
	t := v.Type()
	var errs []error
	foundAny := false
	for i := range t.NumField() {
		sf, field := t.Field(i), v.Field(i)
		if !sf.IsExported() {
			continue
		}
		// Champ embarqué sans balise : ses champs sont chargés sans préfixe supplémentaire.
		if sf.Anonymous && field.Kind() == reflect.Struct {
			if _, ok := sf.Tag.Lookup("env"); !ok {
				f, err := load(field, prefix, lookup)
				foundAny = foundAny || f
				errs = append(errs, err)
				continue
			}
		}
		tag, ok := sf.Tag.Lookup("env")
		if !ok {
			continue
		}
		name, opts, _ := strings.Cut(tag, ",")
		key := prefix + name

		if isNested(field.Type()) {
			if field.Kind() == reflect.Pointer {
				tmp := reflect.New(field.Type().Elem())
				f, err := load(tmp.Elem(), key, lookup)
				if f { // bloc utilisé : l'allouer, et ses erreurs comptent (DB_URL obligatoire…)
					field.Set(tmp)
					errs = append(errs, err)
				} // sinon : bloc optionnel absent, ses « obligatoire » ne s'appliquent pas
				foundAny = foundAny || f
			} else {
				f, err := load(field, key, lookup)
				foundAny = foundAny || f
				errs = append(errs, err)
			}
			continue
		}
		raw, found := lookup(key)
		if found {
			foundAny = true
		} else {
			def, hasDef := sf.Tag.Lookup("default")
			switch {
			case hasDef:
				raw = def
			case hasOpt(opts, "required"):
				errs = append(errs, fmt.Errorf("%s : obligatoire", key))
				continue
			default:
				continue
			}
		}
		if err := set(field, raw); err != nil {
			errs = append(errs, fmt.Errorf("%s=%q : %w", key, raw, err))
		}
	}
	return foundAny, errors.Join(errs...)
}

func hasOpt(opts, o string) bool {
	for x := range strings.SplitSeq(opts, ",") {
		if x == o {
			return true
		}
	}
	return false
}

// isNested : une struct (ou un pointeur vers une struct) qui ne sait pas s'analyser elle-même.
func isNested(t reflect.Type) bool {
	if t.Kind() == reflect.Pointer {
		t = t.Elem()
	}
	return t.Kind() == reflect.Struct && t != durationType && !reflect.PointerTo(t).Implements(textType)
}

func set(f reflect.Value, raw string) error {
	// 1. Le type sait s'analyser lui-même : on lui délègue (son pointeur implémente l'interface).
	if f.CanAddr() && f.Addr().Type().Implements(textType) {
		return f.Addr().Interface().(encoding.TextUnmarshaler).UnmarshalText([]byte(raw))
	}
	// 2. Types particuliers.
	if f.Type() == durationType {
		d, err := time.ParseDuration(raw)
		if err != nil {
			return err
		}
		f.SetInt(int64(d))
		return nil
	}
	// 3. Sortes.
	switch f.Kind() {
	case reflect.String:
		f.SetString(raw)
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		n, err := strconv.ParseInt(raw, 10, 64)
		if err != nil {
			return err
		}
		if f.OverflowInt(n) {
			return fmt.Errorf("%d hors des bornes de %s", n, f.Type())
		}
		f.SetInt(n)
	case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
		n, err := strconv.ParseUint(raw, 10, f.Type().Bits())
		if err != nil {
			return err
		}
		f.SetUint(n)
	case reflect.Float32, reflect.Float64:
		x, err := strconv.ParseFloat(raw, f.Type().Bits())
		if err != nil {
			return err
		}
		if x != x || x > 1e308 || x < -1e308 { // NaN et infinis refusés dans une configuration
			return fmt.Errorf("valeur non finie")
		}
		f.SetFloat(x)
	case reflect.Bool:
		b, err := strconv.ParseBool(raw)
		if err != nil {
			return err
		}
		f.SetBool(b)
	case reflect.Slice:
		if f.Type().Elem().Kind() != reflect.String {
			return fmt.Errorf("type %s non géré", f.Type())
		}
		f.Set(reflect.ValueOf(strings.Split(raw, ",")).Convert(f.Type()))
	default:
		return fmt.Errorf("type %s non géré", f.Type())
	}
	return nil
}

// Describe affiche la configuration effective, secrets masqués.
func Describe(cfg any) string {
	var b strings.Builder
	describe(&b, reflect.Indirect(reflect.ValueOf(cfg)), "")
	return b.String()
}

func describe(b *strings.Builder, v reflect.Value, prefix string) {
	t := v.Type()
	for i := range t.NumField() {
		sf, f := t.Field(i), v.Field(i)
		if !sf.IsExported() {
			continue
		}
		tag := sf.Tag.Get("env")
		name, opts, _ := strings.Cut(tag, ",")
		if f.Kind() == reflect.Pointer {
			if f.IsNil() {
				continue
			}
			f = f.Elem()
		}
		if isNested(f.Type()) {
			describe(b, f, prefix+name)
			continue
		}
		if tag == "" {
			continue
		}
		val := fmt.Sprint(f.Interface())
		if hasOpt(opts, "secret") && val != "" {
			val = "****"
		}
		fmt.Fprintf(b, "%s%s=%s\n", prefix, name, val)
	}
}
```

```go
// Fichier : envcfg/envcfg_test.go
package envcfg

import (
	"errors"
	"fmt"
	"net/netip"
	"reflect"
	"strings"
	"testing"
	"time"
)

type LogLevel int

const (
	Debug LogLevel = iota
	Info
	Warn
	Error
)

func (l *LogLevel) UnmarshalText(b []byte) error {
	i := map[string]LogLevel{"debug": Debug, "info": Info, "warn": Warn, "error": Error}
	v, ok := i[string(b)]
	if !ok {
		return errors.New("niveau inconnu")
	}
	*l = v
	return nil
}

type Common struct {
	Env string `env:"APP_ENV" default:"dev"`
}

type DB struct {
	URL string `env:"URL,required"`
}

type Config struct {
	Common
	Workers int8          `env:"WORKERS" default:"4"`
	Port    uint16        `env:"PORT" default:"8080"`
	Ratio   float64       `env:"RATIO" default:"0.5"`
	Timeout time.Duration `env:"TIMEOUT" default:"5s"`
	Bind    netip.Addr    `env:"BIND" default:"127.0.0.1"`
	Level   LogLevel      `env:"LOG_LEVEL" default:"info"`
	Token   string        `env:"TOKEN,secret"`
	DB      *DB           `env:"DB_"`
}

func env(m map[string]string) func(string) (string, bool) {
	return func(k string) (string, bool) { v, ok := m[k]; return v, ok }
}

func TestLoadDefaults(t *testing.T) {
	var c Config
	if err := Load(&c, env(nil)); err != nil {
		t.Fatal(err)
	}
	want := Config{Common: Common{"dev"}, Workers: 4, Port: 8080, Ratio: 0.5, Timeout: 5 * time.Second,
		Bind: netip.MustParseAddr("127.0.0.1"), Level: Info}
	if !reflect.DeepEqual(c, want) {
		t.Fatalf("%+v\nattendu %+v", c, want)
	}
	if c.DB != nil {
		t.Fatal("DB ne doit pas être alloué sans aucune variable DB_*")
	}
}

func TestLoadValues(t *testing.T) {
	var c Config
	err := Load(&c, env(map[string]string{
		"APP_ENV": "prod", "BIND": "::1", "LOG_LEVEL": "warn", "DB_URL": "postgres://x", "TOKEN": "s3cr3t",
	}))
	if err != nil || c.Env != "prod" || c.Bind.String() != "::1" || c.Level != Warn || c.DB == nil || c.DB.URL != "postgres://x" {
		t.Fatalf("%+v %v", c, err)
	}
	d := Describe(&c)
	if strings.Contains(d, "s3cr3t") || !strings.Contains(d, "TOKEN=****") || !strings.Contains(d, "DB_URL=postgres://x") {
		t.Fatalf("Describe :\n%s", d)
	}
	t.Log("\n" + d)
}

func TestLoadErrors(t *testing.T) {
	cases := map[string]string{
		"WORKERS": "300", "PORT": "70000", "RATIO": "NaN", "BIND": "pas-une-ip", "LOG_LEVEL": "bavard",
	}
	for k, v := range cases {
		var c Config
		err := Load(&c, env(map[string]string{k: v}))
		if err == nil || !strings.Contains(err.Error(), fmt.Sprintf("%s=%q", k, v)) {
			t.Errorf("%s=%s : %v", k, v, err)
		}
	}
	var c Config
	err := Load(&c, env(map[string]string{"DB_X": "1", "PORT": "-1"}))
	t.Log(err)
	if err == nil || !strings.Contains(err.Error(), "PORT") {
		t.Errorf("PORT=-1 : %v", err)
	}
	// DB_X ne correspond à aucun champ : DB n'est pas alloué, DB_URL n'est donc pas exigé.
	if strings.Contains(err.Error(), "DB_URL") {
		t.Errorf("DB_URL ne doit être exigé que si DB est utilisé : %v", err)
	}
}
```

`go test -race ./...` passe. Réponses :

1. **Appeler `UnmarshalText`.** C'est `*netip.Addr` (et `*LogLevel`) qui implémente l'interface, pas
   `netip.Addr` (vérifié). À partir d'un champ adressable `f` (il l'est, puisqu'on est passé par un
   pointeur vers la struct), `f.Addr()` donne le pointeur ; on vérifie `f.Addr().Type().Implements(textType)`,
   puis on appelle la méthode via `f.Addr().Interface().(encoding.TextUnmarshaler)`.
2. **`time.Duration`** n'implémente **pas** `encoding.TextUnmarshaler` (vérifié : `false`). Le cas
   particulier reste nécessaire. Variante possible : un type maison `type Duration time.Duration` avec
   sa propre méthode `UnmarshalText`, mais alors le reste du programme manipule un type inhabituel.
3. **Champ embarqué** : `sf.Anonymous` sans balise `env` → récursion avec le **même** préfixe.
4. **`*DBConfig`** : on charge dans une valeur temporaire (`reflect.New`) et on ne l'affecte au champ que
   si au moins une variable a été trouvée. **Un bug trouvé par le test** : la première version gardait
   aussi les erreurs du bloc absent, si bien que `DB_URL : obligatoire` apparaissait alors qu'aucune base
   n'était configurée. Pour un bloc **optionnel**, ses champs obligatoires ne le sont que si le bloc est
   utilisé.
5. Remarque sur l'affichage : `LOG_LEVEL=2` dans `Describe`, parce que `LogLevel` n'a pas de méthode
   `String`. Lui en donner une (et un `MarshalText`) rendrait l'aller-retour lisible.

## Défi — Un mini `encoding/csv` → struct

```go
// Fichier : csvdec/csvdec.go
package csvdec

import (
	"encoding/csv"
	"errors"
	"fmt"
	"io"
	"reflect"
	"strconv"
	"strings"
	"sync"
)

// column : ce qu'il faut savoir d'un champ, calculé UNE fois par type.
type column struct {
	name     string
	index    []int
	required bool
	parse    func(reflect.Value, string) error
}

var cache sync.Map // reflect.Type → []column

// UseCache peut être désactivé par le benchmark pour mesurer son effet.
var UseCache = true

func plan(t reflect.Type) ([]column, error) {
	if UseCache {
		if c, ok := cache.Load(t); ok {
			return c.([]column), nil
		}
	}
	var cols []column
	for _, sf := range reflect.VisibleFields(t) {
		tag, ok := sf.Tag.Lookup("csv")
		if !ok || !sf.IsExported() {
			continue
		}
		name, opts, _ := strings.Cut(tag, ",")
		if name == "" {
			name = strings.ToLower(sf.Name)
		}
		p, err := parserFor(sf.Type)
		if err != nil {
			return nil, fmt.Errorf("champ %s : %w", sf.Name, err)
		}
		cols = append(cols, column{name, sf.Index, opts == "required", p})
	}
	if UseCache {
		cache.Store(t, cols)
	}
	return cols, nil
}

func parserFor(t reflect.Type) (func(reflect.Value, string) error, error) {
	switch t.Kind() {
	case reflect.String:
		return func(f reflect.Value, s string) error { f.SetString(s); return nil }, nil
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		bits := t.Bits()
		return func(f reflect.Value, s string) error {
			n, err := strconv.ParseInt(s, 10, bits)
			f.SetInt(n)
			return err
		}, nil
	case reflect.Float64:
		return func(f reflect.Value, s string) error {
			x, err := strconv.ParseFloat(s, 64)
			f.SetFloat(x)
			return err
		}, nil
	case reflect.Bool:
		return func(f reflect.Value, s string) error {
			b, err := strconv.ParseBool(s)
			f.SetBool(b)
			return err
		}, nil
	}
	return nil, fmt.Errorf("type %s non géré", t)
}

func DecodeCSV[T any](r io.Reader) ([]T, error) {
	t := reflect.TypeFor[T]()
	if t.Kind() != reflect.Struct {
		return nil, fmt.Errorf("DecodeCSV : %s n'est pas une struct", t)
	}
	cols, err := plan(t)
	if err != nil {
		return nil, err
	}
	cr := csv.NewReader(r)
	header, err := cr.Read()
	if err != nil {
		return nil, fmt.Errorf("en-tête : %w", err)
	}
	pos := map[string]int{}
	for i, h := range header {
		pos[h] = i
	}
	type bound struct {
		column
		at int
	}
	var use []bound
	var missing []error
	for _, c := range cols {
		at, ok := pos[c.name]
		if !ok {
			if c.required {
				missing = append(missing, fmt.Errorf("colonne obligatoire %q absente", c.name))
			}
			continue
		}
		use = append(use, bound{c, at})
	}
	if err := errors.Join(missing...); err != nil {
		return nil, err
	}
	var out []T
	for line := 2; ; line++ {
		rec, err := cr.Read()
		if errors.Is(err, io.EOF) {
			return out, nil
		}
		if err != nil {
			return nil, err
		}
		var item T
		v := reflect.ValueOf(&item).Elem()
		for _, b := range use {
			if err := b.parse(v.FieldByIndex(b.index), rec[b.at]); err != nil {
				return nil, fmt.Errorf("ligne %d, colonne %q : %w", line, b.name, err)
			}
		}
		out = append(out, item)
	}
}
```

```go
// Fichier : csvdec/csvdec_test.go
package csvdec

import (
	"encoding/csv"
	"errors"
	"fmt"
	"io"
	"strconv"
	"strings"
	"testing"
)

type Meta struct {
	Lang string `csv:"lang"`
}

type Book struct {
	Meta
	Title  string  `csv:"title,required"`
	Year   int16   `csv:"year"`
	Rating float64 `csv:"rating"`
	Note   string  // pas de balise : ignoré
}

func TestDecode(t *testing.T) {
	in := "title,year,inconnue,lang,rating\nL'Étranger,1942,x,fr,4.5\nLa Peste,1947,y,fr,4\n"
	got, err := DecodeCSV[Book](strings.NewReader(in))
	if err != nil || len(got) != 2 || got[0].Title != "L'Étranger" || got[1].Year != 1947 || got[0].Lang != "fr" || got[0].Rating != 4.5 {
		t.Fatalf("%+v %v", got, err)
	}
}

func TestDecodeErrors(t *testing.T) {
	if _, err := DecodeCSV[Book](strings.NewReader("year\n1\n")); err == nil || !strings.Contains(err.Error(), `"title"`) {
		t.Errorf("colonne obligatoire : %v", err)
	}
	_, err := DecodeCSV[Book](strings.NewReader("title,year\nA,1\nB,99999\n"))
	if err == nil || err.Error() != `ligne 3, colonne "year" : strconv.ParseInt: parsing "99999": value out of range` {
		t.Errorf("erreur de ligne : %v", err)
	}
}

func data(n int) string {
	var b strings.Builder
	b.WriteString("title,year,rating,lang\n")
	for i := range n {
		fmt.Fprintf(&b, "Titre %d,%d,%d.5,fr\n", i, 1900+i%100, i%5)
	}
	return b.String()
}

// handWritten : la même chose pour UN type, sans réflexion.
func handWritten(r io.Reader) ([]Book, error) {
	cr := csv.NewReader(r)
	if _, err := cr.Read(); err != nil {
		return nil, err
	}
	var out []Book
	for {
		rec, err := cr.Read()
		if errors.Is(err, io.EOF) {
			return out, nil
		}
		if err != nil {
			return nil, err
		}
		y, err := strconv.ParseInt(rec[1], 10, 16)
		if err != nil {
			return nil, err
		}
		x, err := strconv.ParseFloat(rec[2], 64)
		if err != nil {
			return nil, err
		}
		out = append(out, Book{Title: rec[0], Year: int16(y), Rating: x, Meta: Meta{rec[3]}})
	}
}

func BenchmarkDecode(b *testing.B) {
	for _, n := range []int{10, 10_000} {
		in := data(n)
		for _, c := range []struct {
			name  string
			cache bool
		}{{"reflexion-cache", true}, {"reflexion-sans-cache", false}} {
			b.Run(fmt.Sprintf("%s-%d", c.name, n), func(b *testing.B) {
				UseCache = c.cache
				defer func() { UseCache = true }()
				for b.Loop() {
					if _, err := DecodeCSV[Book](strings.NewReader(in)); err != nil {
						b.Fatal(err)
					}
				}
			})
		}
		b.Run(fmt.Sprintf("a-la-main-%d", n), func(b *testing.B) {
			for b.Loop() {
				if _, err := handWritten(strings.NewReader(in)); err != nil {
					b.Fatal(err)
				}
			}
		})
	}
}
```

Mesures (`-benchmem`, deux séries) :

| Fichier | Réflexion, avec cache | Réflexion, sans cache | Écrit à la main |
|---|---|---|---|
| 10 lignes | 6,9 µs, 52 allocations | 10,0 µs, 69 allocations | 5,5 µs, 39 allocations |
| 10 000 lignes | 5,1 ms, 30 036 allocations | 5,2 ms, 30 053 allocations | 3,9 ms, 20 033 allocations |

- **Le cache** ne compte que pour les **petits** appels fréquents : l'analyse du type (balises,
  `VisibleFields`, fonctions d'analyse) coûte ≈ 3 µs, une fois par appel. Sur 10 000 lignes, elle est
  noyée ; sur 10 lignes, elle représente 30 % du temps. Un encodeur JSON appelé pour chaque petite
  réponse HTTP est dans le second cas.
- **La réflexion** coûte ici ≈ 1,3 fois le temps et 1,5 fois les allocations du code écrit à la main.
  L'écart est modéré parce que le travail dominant est ailleurs (lire le CSV, analyser les nombres).
  Les `reflect.Value` créées pour chaque champ expliquent les allocations supplémentaires.
- **Quand ça vaut la peine** : quand il y a beaucoup de types à décoder (un import par table, des
  dizaines de structs), ou des types définis par d'autres. Pour **un** format à fort volume, le code
  écrit à la main (ou généré, leçon 4) est plus rapide et plus simple à déboguer.

Choix de conception : le plan (`[]column`) contient, pour chaque colonne, le **chemin** du champ
(`Index`, compatible avec les champs promus) et une **fonction** d'analyse choisie une fois selon la sorte.
La boucle par ligne ne regarde plus ni balises ni sortes : elle applique des fonctions. C'est
l'architecture d'`encoding/json` (ses « encodeurs par type » mis en cache).

## Quiz

1. `t.Field(i)` décrit le champ (nom, type, balise, chemin) ; `v.Field(i)` est sa valeur dans une
   struct donnée.
2. Des paires `clé:"valeur"` séparées par des espaces, sans espace autour des deux-points.
3. `""` ; `Lookup` rend en plus un booléen qui distingue « absente » de « vide ».
4. Elle est ignorée en silence (`Get` rend `""`) ; `go vet` la détecte.
5. Pour que l'analyse refuse une valeur qui ne tient pas dans le champ, au lieu d'une troncature
   silencieuse.
6. Parce que `time.Duration` est de sorte `int64` mais s'écrit `"5s"` : le type précis doit passer avant
   sa sorte.
7. Comme **un** champ `Anonymous` ; `VisibleFields` rend aussi les champs promus, avec leur chemin.
8. Pour que l'utilisateur corrige tout en une fois, au lieu d'une erreur par démarrage.
9. Parce que l'analyse d'un type (balises, champs) est coûteuse et identique à chaque appel.
10. Que l'argument est un pointeur non nil vers une struct.
