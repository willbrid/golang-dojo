# Leçon 6 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Ce qui sort

Mesuré en Go 1.27 :

```
v1 : {"a":0,"d":null,"f":null,"g":null,"h":"0001-01-01T00:00:00Z","j":"0"}
v2 : {"a":0,"b":0,"d":[],"f":{},"g":null,"h":"0001-01-01T00:00:00Z","j":"0"}
```

| Champ | v1 | v2 | Explication |
|---|---|---|---|
| `a` | `0` | `0` | |
| `b,omitempty` | omis | **`0`** | en v2, `omitempty` omet une valeur **JSON** vide (`""`, `null`, `[]`, `{}`), pas un zéro Go. Pour omettre `0` en v2 : `omitzero` |
| `c` | absent | absent | non exporté |
| `d` (slice nil) | `null` | `[]` | |
| `e,omitempty` | omis | omis | v1 : longueur 0 ; v2 : `[]` est vide |
| `f` (map nil) | `null` | `{}` | |
| `g` (pointeur nil) | `null` | `null` | |
| `h,omitempty` (`time.Time`) | **présent** | **présent** | une struct n'est jamais « vide » pour v1 ; en v2 elle s'encode en chaîne non vide |
| `i,omitzero` | omis | omis | la bonne option pour un `time.Time` |
| `j,string` | `"0"` | `"0"` | |
| `K` (`-`) | absent | absent | |

La ligne `b` est la surprise : **le sens de `omitempty` a changé entre v1 et v2**. C'est
exactement le genre de détail qui rend une migration risquée (défi).

## E2 — La casse

Les deux donnent `c` sur cette entrée — mais pour des raisons différentes : en v1, les **trois**
clés correspondent au champ (insensible à la casse) et la dernière gagne ; en v2, seule `"name"`
correspond.

Le contournement : `{"NAME":"admin"}` ne contient pas `"name":`, donc passe le filtre naïf, et
v1 remplit quand même le champ `Name` (mesuré : `b` pour `{"NAME":"b"}`). Un filtre de
sécurité ne doit **jamais** inspecter le texte brut d'un format qu'un autre composant interprète
différemment : il doit décoder avec le **même** décodeur, puis vérifier la valeur.

## E3 — Les nombres dans `any`

```
int64(m["id"].(float64))  → 1234567890123456768   ✗ (21 de trop)
UseNumber + Int64()       → 1234567890123456789   ✓
```

Avec une struct `struct{ ID int64 \`json:"id"\` }`, le décodeur lit directement un `int64` :
correct, et sans assertion de type. La struct typée est presque toujours la bonne réponse.

## E4 — JSON Lines

**On ne peut pas continuer** avec un seul `json.Decoder` après une erreur de syntaxe :
mesuré, chaque `Decode` suivant rend **la même erreur** (`invalid character 'b' looking for
beginning of object key string`), indéfiniment. Le décodeur ne sait pas où « reprendre ».

La solution sépare le découpage en lignes (`bufio.Scanner`) du décodage (`json.Unmarshal` par
ligne) :

```go
counts := map[string]int{}
sc := bufio.NewScanner(os.Stdin)
for n := 1; sc.Scan(); n++ {
	var e struct {
		Level string `json:"level"`
		Msg   string `json:"msg"`
	}
	if err := json.Unmarshal(sc.Bytes(), &e); err != nil {
		fmt.Fprintf(os.Stderr, "ligne %d : %v\n", n, err)
		continue
	}
	counts[e.Level]++
}
if err := sc.Err(); err != nil { … }
for _, lvl := range slices.Sorted(maps.Keys(counts)) {
	fmt.Println(lvl, counts[lvl])
}
```

Attention à la limite de 64 Kio par ligne du `Scanner` (`sc.Buffer` pour l'augmenter).

## E5 — `Unmarshaler`

```go
type Level int

const (
	Debug Level = iota
	Info
	Warn
	Error
)

var levelNames = [...]string{"debug", "info", "warn", "error"}

func (l Level) MarshalText() ([]byte, error) {
	if l < 0 || int(l) >= len(levelNames) {
		return nil, fmt.Errorf("niveau invalide %d", int(l))
	}
	return []byte(levelNames[l]), nil
}

func (l *Level) UnmarshalText(b []byte) error {
	i := slices.Index(levelNames[:], string(b))
	if i < 0 {
		return fmt.Errorf("niveau inconnu %q", b)
	}
	*l = Level(i)
	return nil
}
```

Ce que `TextMarshaler` apporte de plus :

- `json` l'utilise **et** l'entoure de guillemets lui-même : pas de `json.Marshal` interne.
- Le type devient utilisable comme **clé de map** : `map[Level]int{Info: 3}` → `{"info":3}`.
  Avec `MarshalJSON` seul, les clés seraient les entiers `"1"`.
- D'autres paquets l'utilisent aussi (`encoding/xml`, `flag.TextVar`, `log/slog`) : une seule
  méthode sert partout.

## Exercice intermédiaire — Décodage strict

Les points clés :

```go
const maxBody = 64 << 10

func DecodeOrder(r io.Reader) (Order, error) {
	dec := json.NewDecoder(io.LimitReader(r, maxBody+1))
	dec.DisallowUnknownFields()

	var raw orderJSON
	if err := dec.Decode(&raw); err != nil {
		return Order{}, fmt.Errorf("corps invalide : %w", err)
	}
	// Contrainte 3 : rien d'autre qu'éventuellement des espaces après l'objet.
	if _, err := dec.Token(); !errors.Is(err, io.EOF) {
		return Order{}, errors.New("corps invalide : données après l'objet JSON")
	}
	if dec.InputOffset() > maxBody {
		return Order{}, fmt.Errorf("corps trop volumineux (> %d octets)", maxBody)
	}
	return raw.validate()
}

type orderJSON struct {
	CustomerID string     `json:"customer_id"`
	Items      []itemJSON `json:"items"`
	Delivery   struct {
		Mode string    `json:"mode"`
		Slot time.Time `json:"slot"`
	} `json:"delivery"`
	Coupon *string `json:"coupon"` // nil = absent ou null ; "" = chaîne vide
}

type itemJSON struct {
	SKU       string `json:"sku"`
	Quantity  int    `json:"quantity"`
	UnitPrice string `json:"unit_price"`
}

func (o orderJSON) validate() (Order, error) {
	var errs []error
	if len(o.Items) == 0 {
		errs = append(errs, errors.New("items : au moins un article"))
	}
	for i, it := range o.Items {
		if it.Quantity < 1 || it.Quantity > 100 {
			errs = append(errs, fmt.Errorf("items[%d].quantity : %d hors de [1, 100]", i, it.Quantity))
		}
		if _, err := parseCents(it.UnitPrice); err != nil {
			errs = append(errs, fmt.Errorf("items[%d].unit_price : %w", i, err))
		}
	}
	switch o.Delivery.Mode {
	case "standard", "express":
	default:
		errs = append(errs, fmt.Errorf("delivery.mode : %q inconnu", o.Delivery.Mode))
	}
	if err := errors.Join(errs...); err != nil {
		return Order{}, err
	}
	return o.toOrder(), nil
}
```

Points de correction :

- **Deux types** : `orderJSON` décrit le **format d'échange**, `Order` le **modèle** validé.
  Les séparer évite qu'une `Order` invalide puisse exister dans le programme — et découple le
  domaine du format JSON (niveau 10).
- **Contrainte 3** : après le `Decode`, `dec.Token()` doit rendre `io.EOF`. Mesuré :
  `{"a":1} garbage` → `invalid character 'g' looking for beginning of value`. Sans ce test, le
  reste du corps est ignoré sans bruit.
- **Taille** : `LimitReader(r, max+1)` puis comparer l'offset, même astuce que `ReadAtMost`
  (leçon 5). En HTTP, `http.MaxBytesReader` (niveau 7) le fait mieux.
- **Coupon** : `*string`. `nil` pour absent **ou** `null` (v1 ne distingue pas les deux), un
  pointeur vers `""` pour la chaîne vide. Si l'API devait distinguer absent de `null`, il faudrait
  un type personnalisé avec `UnmarshalJSON` (qui n'est appelé que si la clé est présente).
- **`parseCents`** : découper sur `.` avec `strings.Cut`, refuser plus de deux décimales, parser
  les deux parties avec `strconv.Atoi`. Jamais `strconv.ParseFloat` (leçon 3).
- **Chemins JSON** dans les messages : l'utilisateur de l'API sait exactement quoi corriger.

## Défi — Migration v1 → v2

1–2. Différences et risques :

| Différence | Casse un client ? |
|---|---|
| slice nil `null` → `[]` | rarement — sauf un client qui teste `=== null` pour « non renseigné » |
| `omitempty` ne retire plus les `0`/`false` | **oui** si un client teste la présence de la clé |
| clés de map non triées | **oui** pour les tests de non-régression qui comparent du texte, les signatures, les caches par contenu |
| décodage sensible à la casse | **oui** : un client qui envoyait `"Name"` voit son champ ignoré **sans erreur** |
| clés dupliquées refusées | oui pour un client mal écrit (qui fonctionnait par chance) |
| `<>&` non échappés | non pour un client JSON ; oui si la sortie était insérée telle quelle dans du HTML |
| `time.Duration` refusé | **erreur côté serveur** à l'encodage : à corriger avant tout |

3. `json.DefaultOptionsV1()` (dans `encoding/json`) rend un jeu d'options qui reproduit v1.
   Mesuré : `jsonv2.Marshal(v, json.DefaultOptionsV1())` produit **exactement** la sortie v1.

4. Stratégie : migrer d'abord le **code** vers l'API v2 en passant `DefaultOptionsV1()` (sortie
   inchangée, tests de non-régression au vert) ; puis activer les comportements v2 **un par un**,
   en commençant par ceux qui ne cassent rien (rejet des doublons à l'entrée), en annonçant les
   autres dans le journal des changements ; réserver les changements visibles (`null`→`[]`,
   casse) à une **nouvelle version** de l'API (`/v2/…`, niveau 7, leçon 4).

## Quiz

1. `json` utilise la réflexion, qui n'accède pas aux champs non exportés d'un autre paquet.
2. `omitempty` (v1) omet zéro, `""`, nil, longueur 0 — mais jamais une struct ; `omitzero` omet
   toute zéro-valeur, structs et `time.Time` compris.
3. `float64` : précision perdue au-delà de 2⁵³.
4. Encode le nombre (ou booléen) sous forme de chaîne JSON, et l'accepte ainsi au décodage.
5. `decoder.DisallowUnknownFields()` ; `jsonv2.RejectUnknownMembers(true)`.
6. `null` en v1, `[]` en v2.
7. Il doit modifier la valeur pointée ; avec un récepteur valeur, il modifierait une copie.
8. Il lit un `io.Reader` directement (pas de copie complète en mémoire), gère les suites de
   valeurs, et se combine avec `LimitReader`.
9. Slice nil → `[]` ; sensibilité à la casse ; doublons refusés ; pas d'échappement HTML ; clés
   non triées ; `Duration` sans représentation par défaut ; sens de `omitempty`.
10. Chaque différence de comportement par défaut est visible par les clients ; certaines les
    cassent silencieusement.
