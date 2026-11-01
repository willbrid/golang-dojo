# Leçon 6 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Lire les sources

Réponse personnelle. Correspondances attendues pour les sections les plus courantes :

| Section de *CodeReviewComments* | Où on l'a vue |
|---|---|
| *Error Strings* | N1 L10 — minuscules, sans ponctuation |
| *Handle Errors* / *Indent Error Flow* | N1 L10, N1 L4 (early return) |
| *Interfaces* | N2 L4 — côté consommateur, pas d'interface prématurée |
| *Receiver Names* / *Receiver Type* | N2 L3 |
| *Initialisms* | cette leçon |
| *Package Names* | N2 L8 |
| *Doc Comments* | N2 L8 |
| *Contexts* | N6 L8 (à venir) |
| *Copying* | fil rouge « Copié ou partagé ? » |
| *Synchronous Functions* | N6 (à venir) : préférer des fonctions synchrones, laisser l'appelant ajouter la concurrence |

## E2 — Renommer

| Avant | Après | Pourquoi |
|---|---|---|
| `package stringutils` | `package slug` (ou le nom de ce qu'il fait vraiment) | un paquet « utils » est un tiroir fourre-tout |
| `IStorage` | `Storage` (ou mieux, plus petit : `Getter`) | pas de préfixe `I` en Go |
| `func (self *Cache) GetSize() int` | `func (c *Cache) Len() int` | récepteur court ; pas de `Get` ; `Len` est le nom conventionnel d'une taille |
| `HttpClientWrapper` | `Client` (dans un paquet qui dit le reste) | acronyme `HTTP` ; « Wrapper » ne dit rien de l'usage |
| `userIdList` | `userIDs` | acronyme ; le pluriel dit déjà « liste » |
| `(c *Config) LoadConfigFromFile(configFilePath string)` | `(c *Config) Load(path string)` | bégaiement ×3 ; et accepter un `io.Reader` serait encore mieux |
| `models.ModelUser` | `user.User` ou `models.User` | bégaiement |
| `MAX_RETRIES` | `maxRetries` (ou `MaxRetries` si exporté) | pas de SNAKE_CASE en Go : la visibilité passe par la majuscule |

## E3 — La ligne d'œil

```go
func process(path string) (int, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return 0, err
	}
	if len(data) == 0 {
		return 0, fmt.Errorf("%s : fichier vide", path)
	}
	return bytes.Count(data, []byte("\n")), nil
}
```

Remarques restantes dans l'original, hors structure :

1. `os.Open` + `io.ReadAll` → `os.ReadFile` fait les deux et ferme le fichier.
2. `"Empty file"` → majuscule interdite, et le message ne dit pas **quel** fichier.
3. Le `data, err :=` interne **masque** le `err` externe (niveau 1, leçon 9) — sans bug ici,
   mais l'analyseur `shadow` (`golang.org/x/tools/.../shadow`, branché via `go vet -vettool`) le signalerait.
4. Les erreurs remontées sans contexte : `os.Open` en a déjà (le chemin y figure), donc ici
   c'est acceptable — **savoir le justifier** fait partie de la réponse.
5. Le nom `process` ne dit rien : `countLines`.
6. Charger tout le fichier pour compter des lignes : un `bufio.Scanner` borne la mémoire
   (projet 1, contrainte 5).

## E4 — Commentaires

```go
// Package stats fournit des statistiques descriptives sur des slices numériques.
//
// Toutes les fonctions acceptent les types définis par l'utilisateur
// (type Celsius float64) et rendent ErrEmpty sur une entrée vide.
package stats

// Median rend la médiane de xs. Pour un nombre pair d'éléments, c'est la
// moyenne des deux valeurs centrales. xs n'est pas modifié.
func Median[T Number](xs []T) (float64, error)
```

Le test : un lecteur qui ne voit **que** `go doc` sait-il quoi passer, ce qu'il recevra, et ce
qui peut échouer ? La phrase « xs n'est pas modifié » répond à une question que tout
utilisateur se pose devant un calcul de médiane.

## E5 — `bool` en paramètre

1. **Struct d'options** :
   ```go
   type ExportOptions struct {
   	Overwrite bool
   	Compress  bool
   }
   Export("out.csv", ExportOptions{Overwrite: true})
   ```
2. **Options fonctionnelles** (niveau 1, leçon 11) :
   ```go
   Export("out.csv", WithOverwrite())
   ```

Si une troisième option arrive, **les deux** s'étendent sans casser les appelants existants —
c'est ce qui les distingue des `bool` positionnels. La struct est plus simple, se documente
d'un coup d'œil et a une zéro-valeur lisible ; les options fonctionnelles se justifient pour
une API publique avec beaucoup d'options et des valeurs par défaut non nulles. Pour un usage
interne : la struct.

## Exercice intermédiaire — Revue de code

**Bloquant**

1. **`Cache` est une map nil** : `Cache[parts[0]] = …` panique (`assignment to entry in nil
   map`) au premier `Load`. *Piste :* supprimer l'état global (voir 6) ; à défaut, l'initialiser.
2. **`parts[1]` panique** sur toute ligne sans `=` — y compris la **ligne vide** que produit
   presque toujours un fichier terminé par `\n`. *Piste :* `strings.Cut`, et décider quoi faire
   d'une ligne invalide (erreur avec numéro de ligne).
3. **`Load` a un récepteur valeur** : `cm.data = make(…)` modifie une **copie**. Après
   `NewConfigManager`, `c.data` est nil et `GetValue` ne trouve jamais rien. *Piste :*
   récepteur pointeur — et le même pour toutes les méthodes du type.
4. **`panic(err)`** sur un fichier absent : un cas **attendu** fait planter le programme
   appelant. *Piste :* `Load() error`, et `NewConfigManager` rend `(*ConfigManager, error)`.
5. Une valeur contenant `=` (`url=http://x?a=b`) est tronquée par `strings.Split`. *Piste :*
   `strings.Cut` coupe au **premier** `=`.

**Important**

6. **État global mutable** `Cache`, exporté, écrit par chaque instance : deux configurations
   s'écrasent mutuellement, et ce n'est pas testable. À retirer.
7. **`GetValue` affiche** un avertissement sur la sortie standard : une bibliothèque n'écrit
   jamais sur `os.Stdout`. Et `""` comme valeur d'absence est ambigu (une clé peut valoir
   `""`). *Piste :* `Value(key string) (string, bool)`.
8. **`io/ioutil`** est déprécié depuis Go 1.16 → `os.ReadFile`.
9. **Paquet `helpers`** : nom fourre-tout. **`ConfigManager`** : « Manager » ne dit rien ;
   `config.Config` ou `config.File`.
10. Accepter un **`io.Reader`** plutôt qu'un chemin rendrait le code testable sans fichier.
11. Aucun commentaire de documentation.
12. Espaces autour de `=` et lignes de commentaire non gérés — à documenter au minimum.

**Nit**

13. `ok == true` → `ok`. `else` après `return`. `GetValue` → `Value`. Imports non triés
    (`gofmt` les aurait réordonnés). Le message `"Warning: key " + key + " not found!"` ne
    respecte aucune convention, mais il doit de toute façon disparaître (7).

**Appréciation globale** : « Merci pour la contribution — le besoin est réel et la structure
générale (un type, un constructeur, une méthode de lecture) est la bonne. Les points
bloquants tiennent surtout au récepteur valeur de `Load` et à la gestion des lignes mal
formées ; une fois `Load` converti en `func (c *Config) Load(r io.Reader) error`, la plupart
des autres remarques se résolvent d'elles-mêmes. »

Remarquer : la revue ne **réécrit** pas le code. Elle donne assez d'information pour que
l'auteur le fasse — et apprenne quelque chose.

## Défi — Relire la bibliothèque standard

Exemple de réponse sur `strings/builder.go` :

1. Conventions appliquées : commentaire de type commençant par son nom (`// A Builder is
   used to efficiently build a string…`) ; **zéro-valeur utile** (`// The zero value is ready
   to use.`) ; récepteur d'une lettre `b` dans toutes les méthodes, toutes pointeur.
2. Surprise : le champ `addr *Builder` et la méthode `copyCheck` — le `Builder` **détecte
   qu'il a été copié** et panique (`strings: illegal use of non-zero Builder copied by
   value`). Copier un `Builder` non vide partagerait son buffer interne : exactement le piège
   « mi-copié, mi-partagé » du fil rouge, rendu impossible par une vérification à l'exécution.
3. `String()` utilise `unsafe.String` pour éviter une copie du buffer — c'est sûr **uniquement**
   parce que le `Builder` n'écrit jamais dans un octet déjà produit. Optimisation légitime dans
   la stdlib, à ne jamais imiter dans du code applicatif.

## Quiz

1. *Effective Go* et *Go Code Review Comments*.
2. `gofmt` décide de tout ; un code non formaté n'est même pas relu.
3. `userID`, `httpURL`, `jsonAPI`.
4. Bégaiement : à l'usage on lit `user.UserService` ; `user.Service` suffit.
5. Par le nom de l'identifiant documenté.
6. Minuscule, sans ponctuation finale, avec le contexte (et la valeur en cause).
7. Une erreur est soit journalisée (et absorbée), soit retournée (et le niveau supérieur
   décide) ; faire les deux la fait apparaître plusieurs fois dans les journaux.
8. Une valeur déclarée sans initialisation qui fonctionne : `bytes.Buffer`, `strings.Builder`,
   `sync.Mutex`, un slice nil pour `append`.
9. Remarque mineure, non bloquante.
10. Bugs, puis conception, puis conventions, puis détails (`nit:`).
