# Leçon 2 — Exercices

Code dans `mes-solutions/niveau-09/lecon-02/`, en partant du chargeur de l'exemple.

## Exercices faciles

**E1 — Lister les balises.**
Écrire `func Tags(v any, key string) map[string]string` qui rend, pour chaque champ exporté (champs
promus compris, `VisibleFields`), la valeur de la balise `key`. L'appliquer avec `"json"` à une struct
qui embarque une autre struct. Qu'est-ce qui change si l'on utilise `NumField`/`Field` au lieu de
`VisibleFields` ?

**E2 — Les balises cassées.**
Écrire trois balises mal formées différentes (espace après `:`, virgule entre les paires, guillemets
simples). Que rend `Tag.Get` pour chacune ? Lesquelles `go vet` signale-t-il ? Et une clé en double
(`env:"A" env:"B"`) ?

**E3 — La troncature.**
Remplacer `ParseInt(raw, 10, f.Type().Bits())` par `ParseInt(raw, 10, 64)`. Que vaut `Workers` avec
`WORKERS=300` ? Corriger d'une autre façon, avec `f.OverflowInt(n)`.

**E4 — Les flottants et les non signés.**
Ajouter `Ratio float64` (`RATIO`) et `Port uint16` (`PORT`). Que doit rendre `PORT=-1` ? `PORT=70000` ?
`RATIO=NaN` (est-ce une valeur raisonnable pour une configuration ?) ?

**E5 — Qui a demandé ce champ ?**
Ajouter une option `env:"TOKEN,secret"` : la valeur est chargée normalement, mais une méthode
`func Describe(cfg any) string`, qui affiche la configuration effective au démarrage, remplace les champs
`secret` par `****`. Pourquoi ce besoin est-il réel en production (N7 L9) ?

---

## Exercice intermédiaire — Déléguer aux types

1. Si le type d'un champ (ou son pointeur) implémente `encoding.TextUnmarshaler`, `Load` lui délègue
   l'analyse **avant** tout autre traitement. Le tester avec `net/netip.Addr` (qui l'implémente) et un
   type maison `LogLevel` (`debug`, `info`, `warn`, `error`). Attention : c'est en général `*T` qui
   implémente l'interface ; comment l'appeler à partir d'un champ `f` de type `T` ? (`f.Addr()`.)
2. Supprimer le cas particulier de `time.Duration` : peut-on s'en passer avec cette délégation ?
   (Vérifier si `time.Duration` implémente `TextUnmarshaler`.)
3. Ajouter un champ embarqué : `type Config struct { Common; … }`. Ses champs doivent être chargés
   **sans** préfixe.
4. Un champ pointeur `*DBConfig` : l'allouer seulement si au moins une de ses variables est définie.
5. Table de tests : toutes les sortes, toutes les erreurs, le message joint complet.

---

## Défi — Un mini `encoding/csv` → struct

Écrire `func DecodeCSV[T any](r io.Reader) ([]T, error)` qui associe les colonnes d'un CSV (première ligne
= en-tête) aux champs de `T` selon la balise `csv:"nom"` :

1. Analyse du type **une seule fois** par appel : construire au départ une table « colonne → chemin du
   champ (`Index`) + fonction d'analyse », puis l'appliquer à chaque ligne sans réflexion sur les
   balises.
2. Colonnes inconnues ignorées ; champs `csv:",required"` absents de l'en-tête → erreur avant toute ligne.
3. Erreurs avec ligne et colonne : `ligne 7, colonne "year" : …`.
4. Un cache global des tables par `reflect.Type` (`sync.Map`), et un benchmark (N5 L6) qui compare
   avec et sans cache pour 10 000 lignes.
5. Comparer avec une fonction écrite à la main pour **un** type précis : écart de performance ? Quand le
   générique-réflexif vaut-il la peine ?

---

## Quiz

1. Quelle différence entre `t.Field(i)` et `v.Field(i)` ?
2. Quelle est la syntaxe d'une balise de struct ?
3. Que rend `Tag.Get` pour une clé absente, et en quoi `Lookup` est-il différent ?
4. Que se passe-t-il quand une balise est mal formée ? Qui le détecte ?
5. Pourquoi passer `f.Type().Bits()` à `strconv.ParseInt` ?
6. Pourquoi tester `time.Duration` avant la sorte `Int64` ?
7. Comment un champ embarqué apparaît-il dans `t.Field(i)`, et que rend `VisibleFields` ?
8. Pourquoi rendre toutes les erreurs de configuration ensemble ?
9. Pourquoi les encodeurs mettent-ils en cache l'analyse d'un type ?
10. Que doit vérifier une fonction réflexive qui remplit une struct, avant tout ?
