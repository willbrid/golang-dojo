# Leçon 2 — Structs, champs et balises

> Les balises de struct (`json:"name,omitempty"`, `db:"created_at"`) sont la façon dont Go attache des
> métadonnées aux champs. Le compilateur les ignore ; seule la réflexion les lit. Cette leçon apprend à
> parcourir les champs d'une struct, à lire et interpréter des balises, et à **remplir** une struct à
> partir de données textuelles : exactement ce que font `encoding/json`, les *mappers* de bases de
> données et les chargeurs de configuration. On en écrit un, complet.

## Objectifs

1. Parcourir les champs d'une struct : nom, type, visibilité, balise, position.
2. Lire une balise avec `Tag.Get` et `Tag.Lookup`, et respecter sa syntaxe.
3. Remplir les champs d'une struct depuis des chaînes, sorte par sorte, avec des erreurs utiles.
4. Gérer les structs imbriquées et les champs embarqués (`VisibleFields`).
5. Connaître les pièges : balise mal formée silencieuse, champ non exporté, type particulier
   (`time.Duration`) caché derrière une sorte ordinaire.

## Explication

### Deux vues d'un champ

Une struct se décrit côté **type** et côté **valeur** :

```go
t := v.Type()           // reflect.Type de la struct
for i := range t.NumField() {
	sf := t.Field(i)    // reflect.StructField : Name, Type, Tag, Index, Anonymous, IsExported()
	fv := v.Field(i)    // reflect.Value : la valeur du champ dans CETTE struct
}
```

Le nom, le type et la balise sont des propriétés du **type** : identiques pour toutes les valeurs. La
valeur du champ appartient à la `Value`. Pour **modifier** un champ, il faut que la struct soit
modifiable (obtenue à travers un pointeur, leçon 1) **et** que le champ soit exporté :
`sf.IsExported()` le dit avant toute tentative.

### La syntaxe des balises

Une balise est une **chaîne littérale** (souvent entre accents graves) qui suit une convention :
des paires `clé:"valeur"` séparées par des **espaces**, sans espace autour des deux-points.

```go
Addr string `env:"ADDR" default:":8080"`
```

- `sf.Tag.Get("env")` rend `"ADDR"`, ou `""` si la clé est absente.
- `sf.Tag.Lookup("default")` rend la valeur **et** un booléen : il distingue « absente » de « présente
  mais vide » (`default:""`), ce que `Get` ne peut pas faire.
- La **valeur** est libre ; chaque bibliothèque y définit ses options, par convention après une virgule :
  `json:"name,omitempty"`, `json:"-"` (ignorer le champ), ou ici `env:"URL,required"`. On la découpe avec
  `strings.Cut(tag, ",")`.

Une balise mal formée n'est **pas** une erreur de compilation. `Get` rend alors `""` en silence, et le
champ est ignoré sans que rien ne le signale. Mesuré :

```go
A int `env: "A"`               // espace après les deux-points
B int `env:"B",default:"1"`    // virgule au lieu d'un espace
// Tag.Get("env") → ""   Tag.Get("default") → ""
```

`go vet` (analyseur `structtag`) les détecte :

```
struct field tag `env: "A"` not compatible with reflect.StructTag.Get: bad syntax for struct tag value
struct field tag `env:"B",default:"1"` not compatible with reflect.StructTag.Get: key:"value" pairs not separated by spaces
```

Une raison de plus pour que `go vet` tourne sur chaque modification (N5 L8).

### Remplir un champ

Le schéma d'un remplisseur : lire la donnée brute (une chaîne), choisir l'analyse selon la **sorte** du
champ, puis `Set…` :

| Sorte | Analyse | Écriture |
|---|---|---|
| `String` | — | `f.SetString(raw)` |
| entiers | `strconv.ParseInt(raw, 10, f.Type().Bits())` | `f.SetInt(n)` |
| `Bool` | `strconv.ParseBool` | `f.SetBool(b)` |
| `Float64`… | `strconv.ParseFloat(raw, f.Type().Bits())` | `f.SetFloat(x)` |
| `Slice` | découper, convertir chaque élément | `f.Set(…)` |

Deux détails qui distinguent un code correct d'un code qui « marche » :

- **La taille du champ** : `f.Type().Bits()` donne 8 pour un `int8`. Passé à `ParseInt`, il fait
  refuser `300` (`value out of range`). Sans cela, `SetInt(300)` sur un `int8` tronquerait **en
  silence** à 44.
- **Le type avant la sorte** : `time.Duration` est de sorte `int64`. Un remplisseur qui ne regarde que la
  sorte lirait `"5s"` avec `ParseInt` et échouerait, ou lirait `"5000000000"` pour 5 secondes. On teste
  donc les **types particuliers** d'abord (`f.Type() == reflect.TypeFor[time.Duration]()`), puis la sorte.
  `encoding/json` fait de même, de façon générale : si le type implémente `json.Unmarshaler` ou
  `encoding.TextUnmarshaler`, il lui délègue l'analyse (exercice intermédiaire).

### Imbrication et champs embarqués

- Une struct **imbriquée** (champ `DB DBConfig`) se traite récursivement ; la balise du champ peut servir
  de préfixe (`DB_` + `URL` → `DB_URL`).
- Un champ **embarqué** (N2 L5) apparaît comme **un seul** champ, `Anonymous == true`, dans
  `t.Field(i)` : `NumField()` d'une struct `User{Base; Name}` vaut 2, et `ID` (promu depuis `Base`)
  n'apparaît pas directement. `reflect.VisibleFields(t)` rend tous les champs **visibles**, promus
  compris, avec leur chemin d'accès (`Index` : `[0 0]` pour `User.Base.ID`), utilisable avec
  `v.FieldByIndex`. C'est ce qu'utilise `encoding/json` pour aplatir les champs embarqués.

### Rendre toutes les erreurs

Un chargeur de configuration qui s'arrête à la première erreur oblige l'utilisateur à corriger,
relancer, corriger, relancer. Accumuler les erreurs et les rendre ensemble avec `errors.Join` (N2 L6)
donne la liste complète du premier coup — et `main` refuse de démarrer, avec un message clair (N7 L9).

### Le coût

Parcourir les champs et analyser les balises à **chaque** appel est coûteux. Les bibliothèques sérieuses
analysent un type **une fois**, gardent le résultat dans un cache indexé par `reflect.Type` (souvent un
`sync.Map`, N6 L7), et ne refont que le travail propre à la valeur. Pour un chargeur de configuration
appelé une fois au démarrage, c'est inutile ; pour un encodeur appelé un million de fois, c'est
indispensable (leçon 4).

## Exemple

Un chargeur de configuration piloté par balises : valeurs par défaut, champs obligatoires, structs
imbriquées avec préfixe, durées, listes, et toutes les erreurs en une fois. La source des valeurs est une
fonction injectée : `os.LookupEnv` en production, une `map` dans les tests.

```go
package main

import (
	"errors"
	"fmt"
	"reflect"
	"strconv"
	"strings"
	"time"
)

type DBConfig struct {
	URL      string `env:"URL,required"`
	MaxConns int    `env:"MAX_CONNS" default:"10"`
}

type Config struct {
	Addr     string        `env:"ADDR" default:":8080"`
	Workers  int8          `env:"WORKERS" default:"4"`
	Debug    bool          `env:"DEBUG"`
	Timeout  time.Duration `env:"TIMEOUT" default:"5s"`
	Origins  []string      `env:"ORIGINS"` // valeurs séparées par des virgules
	DB       DBConfig      `env:"DB_"`     // struct imbriquée : le nom sert de préfixe
	internal int           // non exporté : ignoré
}

var durationType = reflect.TypeFor[time.Duration]()

// Load remplit la struct pointée par dst à partir de lookup (os.LookupEnv en production).
// Toutes les erreurs sont rendues ensemble.
func Load(dst any, lookup func(string) (string, bool)) error {
	v := reflect.ValueOf(dst)
	if v.Kind() != reflect.Pointer || v.IsNil() || v.Elem().Kind() != reflect.Struct {
		return fmt.Errorf("Load : pointeur vers une struct attendu, reçu %T", dst)
	}
	return load(v.Elem(), "", lookup)
}

func load(v reflect.Value, prefix string, lookup func(string) (string, bool)) error {
	t := v.Type()
	var errs []error
	for i := range t.NumField() {
		sf := t.Field(i) // reflect.StructField : nom, type, balise…
		if !sf.IsExported() {
			continue
		}
		tag, ok := sf.Tag.Lookup("env")
		if !ok {
			continue
		}
		name, opts, _ := strings.Cut(tag, ",")
		key := prefix + name
		field := v.Field(i)

		if field.Kind() == reflect.Struct && field.Type() != durationType {
			if err := load(field, key, lookup); err != nil {
				errs = append(errs, err)
			}
			continue
		}
		raw, found := lookup(key)
		if !found {
			def, hasDef := sf.Tag.Lookup("default")
			switch {
			case hasDef:
				raw = def
			case opts == "required":
				errs = append(errs, fmt.Errorf("%s : obligatoire", key))
				continue
			default:
				continue // valeur zéro conservée
			}
		}
		if err := set(field, raw); err != nil {
			errs = append(errs, fmt.Errorf("%s=%q : %w", key, raw, err))
		}
	}
	return errors.Join(errs...)
}

func set(f reflect.Value, raw string) error {
	if f.Type() == durationType { // un type précis AVANT sa sorte (int64)
		d, err := time.ParseDuration(raw)
		if err != nil {
			return err
		}
		f.SetInt(int64(d))
		return nil
	}
	switch f.Kind() {
	case reflect.String:
		f.SetString(raw)
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		n, err := strconv.ParseInt(raw, 10, f.Type().Bits()) // borne selon la taille du champ
		if err != nil {
			return err
		}
		f.SetInt(n)
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
		parts := strings.Split(raw, ",")
		f.Set(reflect.ValueOf(parts).Convert(f.Type()))
	default:
		return fmt.Errorf("type %s non géré", f.Type())
	}
	return nil
}

func env(m map[string]string) func(string) (string, bool) {
	return func(k string) (string, bool) { v, ok := m[k]; return v, ok }
}

func main() {
	var cfg Config
	err := Load(&cfg, env(map[string]string{
		"DEBUG":   "true",
		"ORIGINS": "https://a.example,https://b.example",
		"DB_URL":  "postgres://localhost/app",
	}))
	fmt.Printf("%+v\nerreur : %v\n\n", cfg, err)

	var bad Config
	err = Load(&bad, env(map[string]string{"WORKERS": "300", "TIMEOUT": "5 secondes"}))
	fmt.Println(err)

	fmt.Println(Load(cfg, env(nil)))

	// Lire une balise sans rien remplir.
	sf, _ := reflect.TypeFor[Config]().FieldByName("Addr")
	fmt.Printf("balise complète : %s | env=%q default=%q json=%q\n",
		sf.Tag, sf.Tag.Get("env"), sf.Tag.Get("default"), sf.Tag.Get("json"))
}
```

Sortie :

```
{Addr::8080 Workers:4 Debug:true Timeout:5s Origins:[https://a.example https://b.example] DB:{URL:postgres://localhost/app MaxConns:10} internal:0}
erreur : <nil>

WORKERS="300" : strconv.ParseInt: parsing "300": value out of range
TIMEOUT="5 secondes" : time: unknown unit " secondes" in duration "5 secondes"
DB_URL : obligatoire
Load : pointeur vers une struct attendu, reçu main.Config
balise complète : env:"ADDR" default:":8080" | env="ADDR" default=":8080" json=""
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `Load(dst any, lookup …)` | Vérifie d'abord « pointeur non nil vers une struct » : l'appel `Load(cfg, …)` sans `&` est refusé avec un message clair. |
| `lookup func(string) (string, bool)` | La signature de `os.LookupEnv` : injectable, donc testable sans toucher à l'environnement. |
| `sf.IsExported()` | `internal` est ignoré : il ne pourrait pas être modifié de toute façon. |
| `sf.Tag.Lookup("env")` | Un champ sans balise `env` n'est pas géré par le chargeur. |
| `strings.Cut(tag, ",")` | Nom et options (`required`). |
| `sf.Tag.Lookup("default")` | Distingue « pas de défaut » de « défaut vide ». |
| `field.Kind() == reflect.Struct && … != durationType` | Récursion avec préfixe pour les structs imbriquées. |
| `f.Type() == durationType` avant le `switch` | Le type particulier passe avant sa sorte (`int64`). |
| `ParseInt(raw, 10, f.Type().Bits())` | `WORKERS=300` est refusé pour un `int8`, au lieu d'être tronqué. |
| `reflect.ValueOf(parts).Convert(f.Type())` | Fonctionne aussi pour un type nommé `type Origins []string`. |
| `errors.Join(errs...)` | Les trois erreurs de `bad` sont rendues ensemble, chacune avec sa clé et sa valeur. |
| `FieldByName("Addr")` | Lire des métadonnées sans valeur : tout est sur le `reflect.Type`. |

## Erreurs fréquentes

1. **Une balise mal formée** (`env: "X"`, paires séparées par des virgules) : ignorée en silence ; `go vet`
   la signale.
2. **`Tag.Get` là où il faut `Tag.Lookup`** : impossible de distinguer absente et vide.
3. **Tenter de remplir un champ non exporté** : panique (`CanSet` faux).
4. **Brancher sur la sorte seule** : `time.Duration` lu comme un entier.
5. **Ignorer la taille du champ** : `SetInt` tronque un `int8` sans erreur.
6. **Oublier les champs embarqués**, ou les traiter comme des champs ordinaires.
7. **S'arrêter à la première erreur** d'une configuration.
8. **Réanalyser les balises à chaque appel** dans un code appelé très souvent.

## Bonnes pratiques Go

- Vérifier « pointeur non nil vers une struct » à l'entrée, et rendre une erreur sinon.
- `IsExported` avant toute modification ; `Lookup` pour les balises optionnelles.
- Types particuliers d'abord (`time.Duration`, `encoding.TextUnmarshaler`), sortes ensuite.
- Analyser avec la taille réelle du champ (`Bits()`).
- Accumuler les erreurs avec leur contexte (clé, valeur brute) et les joindre.
- Mettre en cache l'analyse d'un type quand le code est appelé fréquemment.
- `go vet` pour la syntaxe des balises.

## Ce que je dois retenir

- `t.Field(i)` décrit le champ (nom, type, balise) ; `v.Field(i)` est sa valeur.
- Balise : `clé:"valeur"` séparées par des espaces ; `Get`, `Lookup` ; options après une virgule.
- Une balise invalide est ignorée sans erreur ; `go vet` la trouve.
- Remplir = analyser selon le type précis puis la sorte, avec la bonne taille, puis `Set…`.
- `VisibleFields` pour les champs promus par embarquement.

---

➡️ [Exercices](exercices.md)
