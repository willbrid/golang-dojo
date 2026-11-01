# Leçon 6 — Exercices

Code dans `mes-solutions/niveau-04/lecon-06/`.

## Exercices faciles

**E1 — Ce qui sort.**
Prédire, puis vérifier, le JSON produit par :

```go
type T struct {
	A int               `json:"a"`
	B int               `json:"b,omitempty"`
	c int
	D []int             `json:"d"`
	E []int             `json:"e,omitempty"`
	F map[string]int    `json:"f"`
	G *int              `json:"g"`
	H time.Time         `json:"h,omitempty"`
	I time.Time         `json:"i,omitzero"`
	J int64             `json:"j,string"`
	K string            `json:"-"`
}
json.Marshal(T{c: 1, K: "x"})
```

Puis refaire avec `encoding/json/v2`. Lister les différences.

**E2 — La casse.**
Décoder `{"Name":"a","NAME":"b","name":"c"}` dans `struct{ Name string \`json:"name"\` }` avec
v1, puis avec v2. Quelle valeur obtient-on dans chaque cas ? Imaginer un filtre de sécurité
écrit en amont qui vérifie `strings.Contains(body, "\"name\":")`. Comment le contourner en v1 ?

**E3 — Les nombres dans `any`.**
Décoder `{"id": 1234567890123456789}` dans un `map[string]any`, afficher l'identifiant avec
`%d` après conversion en `int64`. Est-ce le bon nombre ? Corriger avec `UseNumber`, puis avec
une struct typée.

**E4 — JSON Lines.**
Écrire un programme qui lit sur l'entrée standard des objets `{"level":…,"msg":…}`, un par
ligne, et affiche le nombre de messages par niveau. Une ligne invalide est signalée sur
`os.Stderr` avec son **numéro**, et le traitement continue. *(Piège : avec un seul
`json.Decoder`, peut-on continuer après une erreur de syntaxe ? Mesurer.)*

**E5 — `Unmarshaler`.**
Écrire un type `Level` (`Debug`, `Info`, `Warn`, `Error`) qui s'encode en JSON sous forme de
chaîne minuscule (`"warn"`) et refuse au décodage toute valeur inconnue, avec un message qui la
cite. Utiliser `MarshalText`/`UnmarshalText` plutôt que les versions JSON : qu'est-ce que ça
apporte de plus ? *(Essayer `map[Level]int`.)*

---

## Exercice intermédiaire — Décodage strict d'une requête d'API

Une API reçoit des commandes :

```json
{
  "customer_id": "C-1042",
  "items": [{"sku": "KB-01", "quantity": 2, "unit_price": "49.90"}],
  "delivery": {"mode": "express", "slot": "2026-11-03T14:00:00+01:00"},
  "coupon": null
}
```

Écrire `DecodeOrder(r io.Reader) (Order, error)`.

**Contraintes :**
1. Corps limité à 64 Kio ; au-delà, une erreur explicite.
2. Champs inconnus refusés.
3. **Un seul** objet JSON : `{…}{…}` ou `{…} garbage` doivent être refusés. *(Indice : après le
   `Decode`, que doit-il rester dans le flux ?)*
4. `unit_price` est une **chaîne** décimale convertie en `Cents` (leçon 3) — refuser plus de deux
   décimales.
5. `quantity` entre 1 et 100 ; au moins un article ; `mode` dans `standard`/`express`.
6. `coupon` peut être absent, `null` ou une chaîne : distinguer « absent ou null » d'une chaîne
   vide. Quel type Go utiliser ?
7. Toutes les erreurs de **validation** (pas de syntaxe) sont collectées et rendues ensemble
   (`errors.Join`), chacune avec son chemin JSON : `items[0].quantity : 0 hors de [1, 100]`.
8. Écrire 8 cas de test sous forme de tableau dans le `main`.

---

## Défi — Migration v1 → v2

On maintient une API publique encodée avec `encoding/json`. Un collègue propose de passer à v2.

1. Écrire un programme qui encode la **même** valeur (choisie pour exposer toutes les
   différences du tableau du cours) avec v1 et v2, et affiche un diff.
2. Pour chaque différence, dire si elle peut **casser un client existant**, et comment.
3. Trouver les options v2 qui reproduisent le comportement v1 (`go doc encoding/json/v2`,
   `go doc encoding/json` — chercher `DefaultOptionsV1`). Écrire l'appel v2 qui produit
   exactement la sortie v1.
4. Proposer une stratégie de migration en 5 à 10 lignes.

---

## Quiz

1. Pourquoi un champ en minuscule n'apparaît-il pas dans le JSON ?
2. Différence entre `omitempty` et `omitzero` ?
3. En quel type Go un nombre JSON est-il décodé dans un `any` ? Quel risque ?
4. Que fait `,string` ?
5. Comment refuser les champs inconnus avec v1 ? Avec v2 ?
6. Comment encode-t-on un slice nil en v1 ? En v2 ?
7. Pourquoi `UnmarshalJSON` doit-il avoir un récepteur pointeur ?
8. Pourquoi utiliser `json.Decoder` plutôt que `io.ReadAll` + `json.Unmarshal` ?
9. Citer trois différences de comportement par défaut entre v1 et v2.
10. Pourquoi la migration d'une API vers v2 est-elle une décision de compatibilité ?
