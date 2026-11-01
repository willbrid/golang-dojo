# Leçon 6 — JSON

> JSON est le format d'échange de presque toutes les API que l'on écrira au niveau 7. Go
> fournit deux paquets : `encoding/json`, présent depuis Go 1.0 et utilisé par l'immense
> majorité du code existant, et **`encoding/json/v2`** (avec `encoding/json/jsontext`),
> disponible sans option expérimentale en Go 1.27. Cette leçon apprend le premier, parce
> qu'on le lira partout, et montre précisément ce que le second corrige.

## Objectifs

1. Encoder et décoder des structs avec les balises `json:"…"`.
2. Connaître les pièges de `encoding/json` : `null` contre `[]`, insensibilité à la casse,
   champs inconnus, nombres dans `any`.
3. Décoder un flux avec `json.Decoder` sans tout charger.
4. Personnaliser la représentation d'un type (`MarshalJSON`/`UnmarshalJSON`).
5. Savoir ce que change `encoding/json/v2`.

## Explication

### Encoder et décoder

```go
type User struct {
	ID    int64    `json:"id"`
	Name  string   `json:"name"`
	Email string   `json:"email,omitempty"`
	Tags  []string `json:"tags"`
	hash  string   // non exporté : ignoré par json
}

b, err := json.Marshal(u)            // Go → []byte
err = json.Unmarshal(b, &u)          // []byte → Go : il faut un POINTEUR
```

Les règles de correspondance :

- **Seuls les champs exportés** sont encodés : `json` passe par la réflexion (niveau 9), qui
  ne voit pas les champs non exportés. Un champ en minuscule est **silencieusement** ignoré.
- La **balise** (niveau 2, leçon 2) donne le nom JSON. Sans balise, le nom du champ Go est
  utilisé tel quel (`Name` → `"Name"`).
- `json:"-"` exclut le champ ; `json:",omitempty"` l'omet s'il est « vide » (zéro, `""`, nil,
  slice ou map de longueur 0) ; `json:",omitzero"` (Go 1.24) l'omet s'il vaut sa
  **zéro-valeur** — la seule option qui marche pour un `time.Time` ou une struct ;
  `json:",string"` encode un nombre **sous forme de chaîne**.

`json.Unmarshal(b, u)` sans `&` rend une erreur explicite : `json: Unmarshal(non-pointer main.User)`.

### Correspondance des types

| JSON | Go (dans une struct) | Go (dans un `any`) |
|---|---|---|
| `true`/`false` | `bool` | `bool` |
| nombre | `int`, `float64`… | **`float64`** |
| chaîne | `string`, `[]byte` (base64) | `string` |
| tableau | slice, tableau | `[]any` |
| objet | struct, `map[string]T` | `map[string]any` |
| `null` | zéro-valeur (ou nil pour pointeurs, slices, maps) | `nil` |

La colonne de droite contient un piège important : décodé dans un `any`, **tout nombre devient
un `float64`**. Un identifiant `12345678901234567890` devient `1.2345678901234567e+19` — les
derniers chiffres sont perdus (leçon 3 : 53 bits de mantisse). Solutions : décoder dans une
struct typée (toujours préférable), ou `decoder.UseNumber()` qui garde le nombre en texte
(`json.Number`).

Le même problème existe **de l'autre côté** : un client JavaScript lit tous les nombres en
`float64`. D'où l'option `,string` pour les grands identifiants `int64` : `"id":"1152921504606846976"`.

### Les pièges de `encoding/json` (v1)

Mesurés en Go 1.27 :

1. **Un slice nil s'encode `null`**, un slice vide `[]`. Pour un client qui fait
   `data.tags.length`, `null` est un plantage. Initialiser (`Tags: []string{}`) ou passer en v2.
2. **Le décodage est insensible à la casse** : `{"NAME": "x"}` remplit le champ `name`. Un
   attaquant peut ainsi contourner un filtre qui vérifiait la clé exacte en amont.
3. **Les champs inconnus sont ignorés** silencieusement : une faute de frappe côté client
   (`"emial"`) ne produit aucune erreur. `decoder.DisallowUnknownFields()` les refuse.
4. **Les clés dupliquées** sont acceptées, la dernière gagne.
5. **`<`, `>` et `&` sont échappés** en `<`… par défaut (prévu pour l'inclusion dans du
   HTML) : surprenant dans un journal ou un fichier.
6. Un `time.Duration` s'encode en **nanosecondes** (`3000000000` pour 3 s) — rarement ce qu'on
   veut dans une API.

### Décoder un flux : `json.Decoder`

`json.Unmarshal` prend un `[]byte` entier. Pour un `io.Reader` (corps HTTP, fichier), on utilise
un `Decoder` :

```go
dec := json.NewDecoder(io.LimitReader(r, 1<<20))  // borné, leçon 5
dec.DisallowUnknownFields()
if err := dec.Decode(&req); err != nil { … }
```

Un `Decoder` lit aussi une **suite** de valeurs JSON (format *JSON Lines* : une valeur par
ligne) en appelant `Decode` en boucle jusqu'à `io.EOF`. Symétriquement, `json.NewEncoder(w)`
écrit directement dans un `io.Writer` (avec un `\n` après chaque valeur).

### Personnaliser : `MarshalJSON` et `UnmarshalJSON`

Un type qui implémente `json.Marshaler` (`MarshalJSON() ([]byte, error)`) ou
`json.Unmarshaler` (`UnmarshalJSON([]byte) error`) contrôle sa représentation. Le paquet le
détecte par **assertion d'interface**, comme `fmt` avec `Stringer`.

```go
// Duration s'échange sous forme lisible : "1h30m".
type Duration time.Duration

func (d Duration) MarshalJSON() ([]byte, error) {
	return json.Marshal(time.Duration(d).String())
}

func (d *Duration) UnmarshalJSON(b []byte) error {
	var s string
	if err := json.Unmarshal(b, &s); err != nil {
		return err
	}
	v, err := time.ParseDuration(s)
	if err != nil {
		return err
	}
	*d = Duration(v)
	return nil
}
```

Deux règles :

- `MarshalJSON` sur **valeur**, `UnmarshalJSON` sur **pointeur** (il doit modifier la cible).
  C'est l'une des rares exceptions admises à la règle « un seul type de récepteur »
  (niveau 2, leçon 3), et la stdlib la pratique (`time.Time`).
- **Réutiliser** `json.Marshal`/`Unmarshal` à l'intérieur pour produire et lire du JSON valide :
  jamais de concaténation de chaînes.

Pour du texte, `encoding.TextMarshaler` (`MarshalText`) suffit souvent : `json` l'utilise
aussi, et le type devient utilisable comme **clé de map** JSON.

### `encoding/json/v2` : ce qui change

Le paquet v2 corrige les défauts de v1 **par défaut**. Comparaison mesurée en Go 1.27 :

| Situation | `encoding/json` | `encoding/json/v2` |
|---|---|---|
| slice/map nil | `null` | `[]` / `{}` |
| `{"NAME": …}` pour un champ `name` | accepté | **ignoré** (sensible à la casse) |
| clé dupliquée | dernière gagne | **erreur** `duplicate object member name` |
| `<`, `>`, `&` | échappés `<` | laissés tels quels |
| ordre des clés d'une map | trié | **non trié** (option `Deterministic(true)`) |
| `time.Duration` | nanosecondes | **erreur** : pas de représentation par défaut |
| champs inconnus | ignorés | ignorés (option `RejectUnknownMembers(true)`) |
| API flux | `Decoder`/`Encoder` | `UnmarshalRead(r, &v)`, `MarshalWrite(w, v)` |
| options | méthodes du `Decoder` | arguments variadiques `opts ...Options` |
| erreurs | `cannot unmarshal string into Go struct field User.age` | ajoute le chemin JSON : `within "/age"` |

```go
import jsonv2 "encoding/json/v2"

err := jsonv2.UnmarshalRead(r, &req, jsonv2.RejectUnknownMembers(true))
```

Faut-il migrer ? Pour du code **neuf** dont on contrôle les deux bouts : v2 est plus sûr.
Pour une API **existante**, chaque ligne du tableau est un changement visible par les
clients (`null` devient `[]`, une clé en majuscules cesse de marcher) : migrer est une décision
de compatibilité, à prendre en connaissance de cause (niveau 13, leçon 6). Dans ce cours, les
exemples utilisent v1 parce que c'est ce qu'on rencontre partout.

## Exemple

```go
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"strings"
	"time"
)

// Duration s'échange en texte lisible ("1h30m") au lieu de nanosecondes.
type Duration time.Duration

func (d Duration) MarshalJSON() ([]byte, error) {
	return json.Marshal(time.Duration(d).String())
}

func (d *Duration) UnmarshalJSON(b []byte) error {
	var s string
	if err := json.Unmarshal(b, &s); err != nil {
		return fmt.Errorf("durée : chaîne attendue : %w", err)
	}
	v, err := time.ParseDuration(s)
	if err != nil {
		return err
	}
	*d = Duration(v)
	return nil
}

type Job struct {
	ID       int64     `json:"id,string"`
	Name     string    `json:"name"`
	Timeout  Duration  `json:"timeout"`
	Tags     []string  `json:"tags"`
	Disabled bool      `json:"disabled,omitempty"`
	LastRun  time.Time `json:"last_run,omitzero"`
}

// decodeJobs lit une suite de Job au format JSON Lines, en refusant les champs inconnus.
func decodeJobs(r io.Reader) ([]Job, error) {
	dec := json.NewDecoder(r)
	dec.DisallowUnknownFields()
	var jobs []Job
	for i := 1; ; i++ {
		var j Job
		err := dec.Decode(&j)
		if errors.Is(err, io.EOF) {
			return jobs, nil
		}
		if err != nil {
			return jobs, fmt.Errorf("job %d : %w", i, err)
		}
		if j.Tags == nil {
			j.Tags = []string{} // un client recevra [] et non null
		}
		jobs = append(jobs, j)
	}
}

const input = `{"id":"9007199254740993","name":"backup","timeout":"1h30m","tags":["nuit"]}
{"id":"2","name":"report","timeout":"45s"}
{"id":"3","name":"purge","timeout":"10m","timout":"5m"}`

func main() {
	jobs, err := decodeJobs(strings.NewReader(input))
	fmt.Println("erreur :", err)

	out, _ := json.MarshalIndent(jobs, "", "  ")
	fmt.Println(string(out))

	var generic map[string]any
	json.Unmarshal([]byte(`{"id": 9007199254740993}`), &generic)
	fmt.Printf("dans un any : %T %.0f\n", generic["id"], generic["id"])
}
```

Sortie :

```
erreur : job 3 : json: unknown field "timout"
[
  {
    "id": "9007199254740993",
    "name": "backup",
    "timeout": "1h30m0s",
    "tags": [
      "nuit"
    ]
  },
  {
    "id": "2",
    "name": "report",
    "timeout": "45s",
    "tags": []
  }
]
dans un any : float64 9007199254740992
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `type Duration time.Duration` | Un type défini pour **porter** la représentation JSON ; l'arithmétique reste disponible par conversion. |
| `MarshalJSON` valeur / `UnmarshalJSON` pointeur | Le second doit modifier la cible. |
| `json.Marshal(…String())` | Produit une chaîne JSON correctement échappée ; jamais `"\"" + s + "\""`. |
| `,string` sur `ID` | `9007199254740993` dépasse 2⁵³ : transporté en texte, il arrive intact. |
| `omitzero` sur `LastRun` | Avec `omitempty`, un `time.Time` zéro serait encodé `"0001-01-01T00:00:00Z"`. |
| `Decode` en boucle jusqu'à `io.EOF` | JSON Lines : une valeur après l'autre, mémoire constante. |
| `errors.Is(err, io.EOF)` | La fin normale du flux, distinguée des vraies erreurs. |
| `"timout"` | La faute de frappe est **détectée** grâce à `DisallowUnknownFields`. Sans lui, le job 3 aurait été accepté avec un `timeout` nul. |
| `jobs` partiel + erreur | On rend ce qui a été décodé avec l'erreur : l'appelant choisit. |
| `9007199254740992` | Le même nombre, décodé dans un `any`, a perdu une unité. |

## Erreurs fréquentes

1. **Champ non exporté** → silencieusement absent du JSON.
2. **`Unmarshal` sans `&`**.
3. **Nombres dans `map[string]any`** → `float64`, précision perdue.
4. **`null` au lieu de `[]`** pour un slice nil.
5. **Ne pas refuser les champs inconnus** dans une API : les fautes de frappe des clients passent.
6. **`io.ReadAll` puis `Unmarshal`** sur un corps non borné.
7. **`omitempty` sur un `time.Time` ou une struct** : sans effet ; il faut `omitzero`.
8. **`UnmarshalJSON` avec récepteur valeur** : la modification est perdue.
9. **Construire du JSON par concaténation** : injection et échappement cassés.
10. **Migrer vers v2 une API publique** sans mesurer les différences visibles.

## Bonnes pratiques Go

- Décoder dans des **structs typées**, pas dans `map[string]any`.
- Balises en `snake_case` ou `camelCase` selon la convention de l'API — **une** convention.
- `json.NewDecoder` + `io.LimitReader` + `DisallowUnknownFields` pour les entrées externes.
- `,string` pour les identifiants `int64` destinés à du JavaScript.
- Slices initialisés quand le contrat de l'API promet un tableau.
- Les types métier portent leur propre représentation (`MarshalJSON`/`MarshalText`).
- `encoding/json/v2` pour du code neuf aux deux bouts maîtrisés.

## Ce que je dois retenir

- Seuls les champs **exportés** ; la balise donne le nom ; `omitempty`, `omitzero`, `string`, `-`.
- Dans un `any`, tout nombre est un `float64`.
- v1 : slice nil → `null`, décodage insensible à la casse, champs inconnus ignorés.
- `json.Decoder` pour les flux, `Decode` en boucle jusqu'à `io.EOF`.
- `MarshalJSON` valeur, `UnmarshalJSON` pointeur.
- `encoding/json/v2` (1.27) : défauts plus sûrs, mais changements visibles.

---

➡️ [Exercices](exercices.md)
