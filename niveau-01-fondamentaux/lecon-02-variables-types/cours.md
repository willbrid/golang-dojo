# Leçon 2 — Variables, constantes et types primitifs

## Objectifs

1. Déclarer une variable des trois façons possibles et savoir laquelle choisir.
2. Expliquer la **zéro-valeur** et pourquoi Go n'a pas de variable « non initialisée ».
3. Choisir le bon type entier ou flottant, et comprendre pourquoi Go refuse les conversions implicites.
4. Utiliser `const` et `iota`, et comprendre les **constantes non typées**, spécificité méconnue de Go.

## Explication

### Trois façons de déclarer

```go
var x int = 42   // 1. explicite : type ET valeur
var y = 42       // 2. type déduit de la valeur
z := 42          // 3. déclaration courte — seulement DANS une fonction
var w int        // 4. sans valeur → zéro-valeur (ici 0)
```

Règle de choix, celle qu'appliquent les développeurs Go :

- **`:=` par défaut**, dans les fonctions. C'est la forme la plus courante et la plus lisible.
- **`var`** au niveau du package (obligatoire : `:=` y est interdit), ou quand on veut la
  zéro-valeur (`var buf bytes.Buffer`), ou quand le type déduit ne serait pas le bon
  (`var ratio float64 = 3` — sans quoi `3` donnerait un `int`).

### La zéro-valeur : il n'y a pas de « non initialisé » en Go

Toute variable déclarée sans valeur reçoit la **zéro-valeur** de son type. Ce n'est pas
un détail : c'est un principe de conception qui élimine toute une classe de bugs.

| Type | Zéro-valeur |
|---|---|
| tous les entiers, flottants | `0` |
| `bool` | `false` |
| `string` | `""` (chaîne vide, **pas** `nil`) |
| pointeur, slice, map, channel, fonction, interface | `nil` |
| struct | une struct dont **chaque champ** vaut sa propre zéro-valeur |

Le corollaire idiomatique : **concevoir ses types pour que la zéro-valeur soit utile**.
`var b bytes.Buffer` est immédiatement utilisable ; `var mu sync.Mutex` est un mutex
déverrouillé prêt à l'emploi. Aucun constructeur n'est nécessaire. C'est un critère de
qualité d'API auquel on reviendra au niveau 2.

#### `nil` n'est pas ce que `fmt` affiche

Piège majeur : l'affichage d'une valeur nil ne ressemble pas à `nil`.

```go
var s []int
var m map[string]int
var p *int

fmt.Printf("%v %v %v\n", s, m, p)          // []  map[]  <nil>
fmt.Println(s == nil, m == nil, p == nil)   // true true true
```

Les trois valeurs **sont** nil. Mais `%v` sur une slice ou une map imprime leur
**contenu** : une slice nil n'a aucun élément (`[]`), une map nil aucune entrée
(`map[]`). Un pointeur n'a pas de contenu — la seule chose à en dire est qu'il ne
pointe nulle part (`<nil>`).

**Ne jamais conclure « ce n'est pas nil » parce qu'on voit `[]`. Pour tester nil, on
compare à nil.**

#### Une slice nil est utilisable ; une map nil ne l'est qu'en lecture

C'est l'asymétrie qui piège tout le monde.

| | slice nil | map nil |
|---|---|---|
| `len` | `0` ✅ | `0` ✅ |
| lire / parcourir (`range`) | ✅ | ✅ |
| **ajouter un élément** | `append` ✅ **fonctionne** | **panique** ❌ |

```go
var s []int
s = append(s, 1)            // OK : append alloue et RETOURNE une nouvelle slice

var m map[string]int
fmt.Println(m["absent"])    // OK : 0 — lire ne réclame aucun stockage
m["x"] = 1                  // panic: assignment to entry in nil map
```

La lecture d'une clé absente rend la zéro-valeur du type des valeurs, table allouée ou
non : rien à allouer, tout est cohérent. L'écriture, elle, réclame une table qui
n'existe pas. Go aurait pu l'allouer en silence ; il a choisi de paniquer, parce qu'une
map nil est presque toujours une map qu'on a **oublié** d'initialiser — mieux vaut
l'échec bruyant que l'écriture dans une structure que personne ne verra jamais.

`append` s'en sort parce qu'il **retourne** la nouvelle slice ; une map se modifie en
place, il n'y a pas de valeur de retour où glisser la table fraîchement créée. D'où le
`make` obligatoire.

Le bug classique, celui qu'on rencontre en vrai :

```go
type Config struct {
	Tags map[string]string   // nil tant que personne n'a appelé make()
}

var c Config
c.Tags["env"] = "prod"       // panic
```

Une struct dont la zéro-valeur contient une map n'est **pas** utilisable en écriture :
c'est le contre-exemple direct du principe « la zéro-valeur doit être utile ».

#### Pourquoi un tableau, lui, ne peut pas être nil

`var t [3]int` affiche `[0 0 0]`, jamais `nil`. La raison est structurelle :

- **Un tableau est une valeur**, et sa taille fait partie de son type — `[3]int` et
  `[4]int` sont deux types *différents*. Les trois entiers existent en mémoire dès la
  déclaration ; il n'y a aucun état « absent » à représenter.
- **Une slice est un descripteur de trois mots** : pointeur vers un tableau
  sous-jacent, longueur, capacité. Sa zéro-valeur, ce sont ces trois mots à zéro — donc
  un pointeur nil, `len` 0, `cap` 0.

Conséquence pratique, développée au niveau 2 : passer un `[3]int` à une fonction copie
les trois entiers ; passer une `[]int` copie trois mots, quelle que soit la taille des
données.

### Les types primitifs

```go
// Entiers signés            Entiers non signés
int8  int16  int32  int64    uint8  uint16  uint32  uint64
int                          uint      uintptr
// Flottants        Complexes          Autres
float32 float64     complex64/128      bool  string  byte  rune  error
```

Points à retenir :

- **`int` fait 64 bits** sur toutes les plateformes modernes (amd64, arm64), mais la
  spécification dit seulement « au moins 32 bits ». **Utiliser `int` par défaut** pour
  tout ce qui compte des choses : indices, longueurs, quantités.
- `byte` est un **alias** de `uint8`, `rune` un alias de `int32`. Un alias n'est pas un
  type distinct : c'est exactement le même type sous un autre nom, choisi pour exprimer
  l'intention (`byte` = octet de données, `rune` = point de code Unicode).
- **`float64` par défaut** pour les flottants ; `float32` seulement si la mémoire ou un
  format externe l'impose.
- **Jamais de flottant pour de l'argent.** `0.1 + 0.2 != 0.3` en binaire. Utiliser des
  entiers de centimes, ou une bibliothèque décimale.
- `uint` ne sert *pas* à dire « ce nombre est positif ». Il sert aux masques de bits et
  aux protocoles binaires. Piège classique : `for i := uint(0); i < n; i--` boucle
  éternellement car `0 - 1` donne un nombre énorme au lieu de −1.

### Pas de conversion implicite. Jamais.

```go
var i int = 3
var f float64 = i     // ERREUR : cannot use i (variable of type int) as float64
var f float64 = float64(i)  // correct : conversion explicite
```

Même `int` et `int64` sont des types **différents** qui ne se mélangent pas. C'est
verbeux, et c'est voulu : toute perte de précision ou tout dépassement devient visible
dans le code. Une conversion explicite est un endroit où l'auteur a réfléchi.

Attention : une conversion qui déborde **ne panique pas**, elle tronque silencieusement.
`int8(300)` vaut `44`. C'est l'un des rares endroits où Go laisse passer une erreur.

### Constantes, et la subtilité des constantes non typées

```go
const Pi = 3.14159            // non typée
const MaxUsers int = 1000     // typée
const (
	Prefix  = "app_"
	Version = "1.0"
)
```

Une constante est évaluée **à la compilation**. Elle ne peut contenir que des valeurs
scalaires : nombres, chaînes, booléens. `const t = time.Now()` est impossible.

La subtilité qui étonne tout le monde : une **constante non typée** n'a pas encore de
type, seulement une « sorte » (entier, flottant, chaîne…). Elle prend le type de son
contexte d'utilisation :

```go
const n = 100        // non typée
var a int     = n    // n devient int
var b float64 = n    // n devient float64  ← impossible avec une variable !
var c int8    = n    // n devient int8
```

Mieux : les constantes non typées ont une **précision arbitraire** à la compilation.

```go
const Huge = 1 << 100        // parfaitement légal
const Small = Huge >> 98     // vaut 4, tient dans un int
fmt.Println(Small)           // OK
fmt.Println(Huge)            // ERREUR : constant overflows int
```

C'est la raison pour laquelle `math.MaxInt64` ou `time.Second` s'utilisent si naturellement.

### `iota` : les énumérations du pauvre

Go n'a pas d'`enum`. `iota` est un compteur remis à 0 à chaque bloc `const` et incrémenté
à chaque ligne :

```go
type Weekday int

const (
	Sunday Weekday = iota // 0
	Monday                // 1  (l'expression précédente est répétée implicitement)
	Tuesday               // 2
)

// Avec une expression :
const (
	_  = iota             // on jette 0
	KB = 1 << (10 * iota) // 1 << 10 = 1024
	MB                    // 1 << 20
	GB                    // 1 << 30
)
```

Limite importante : `Weekday(42)` est parfaitement valide pour le compilateur. Un type
`iota` ne garantit **pas** que la valeur fait partie de l'énumération — il faudra valider
soi-même. On y reviendra au niveau 2.

## Exemple

```go
package main

import "fmt"

type Severity int

const (
	Debug Severity = iota
	Info
	Warning
	Error
)

// severityName rend Severity lisible.
// C'est une FONCTION et non une méthode : les méthodes arrivent au niveau 2.
// On verra alors qu'une méthode String() serait appelée automatiquement par fmt.
func severityName(s Severity) string {
	switch s {
	case Debug:
		return "DEBUG"
	case Info:
		return "INFO"
	case Warning:
		return "WARN"
	case Error:
		return "ERROR"
	default:
		return fmt.Sprintf("Severity(%d)", int(s))
	}
}

func main() {
	const maxRetries = 3        // non typée : utilisable comme int, float64…
	var attempt int             // zéro-valeur : 0
	level := Warning            // type déduit : Severity
	ratio := float64(attempt) / float64(maxRetries) // conversions explicites obligatoires

	fmt.Printf("niveau=%s tentative=%d/%d ratio=%.2f\n", severityName(level), attempt, maxRetries, ratio)
	fmt.Printf("%d %q %T\n", level, severityName(level), level)
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `type Severity int` | Crée un **type distinct** basé sur `int`. `Severity` et `int` ne sont pas interchangeables sans conversion : c'est ce qui rend l'énumération utile. |
| `Debug Severity = iota` | Le type et l'expression se répètent implicitement sur les lignes suivantes. |
| `default:` dans `severityName` | Traite les valeurs hors énumération. Sans lui, `Severity(42)` renverrait `""`, ce qui masquerait le bug. |
| `float64(attempt)` | Conversion explicite. Diviser deux `int` donnerait une **division entière** : `0/3 = 0`, pas `0.0`. Piège fréquent. |
| `%v` `%d` `%q` `%T` | Format par défaut · entier décimal · **littéral cité** · **type** de la valeur. `%T` est l'outil de débogage n°1 du débutant. Sur `%q`, voir l'encadré ci-dessous : il ne veut pas dire « entre guillemets » pour tout le monde. |
| `%d` sur `level` | Affiche `2` : sans méthode `String()`, `fmt` ne sait rien du sens de la valeur. C'est précisément ce que le niveau 2 corrigera. |

### `%q` et les *format errors* de `fmt`

`%q` n'a pas le même sens selon le type, et n'en a aucun sur certains :

| Opérande | `%q` donne | Lecture |
|---|---|---|
| `string` | `"texte"` | guillemets **doubles** |
| entier / rune | `'A'`, `'\x00'` | guillemets **simples** : « ce nombre vu comme un caractère » |
| slice, tableau, map, struct | le verbe s'applique **à chaque élément** | `[3]int` nul → `['\x00' '\x00' '\x00']` |
| `[]byte` | `"texte"` | **exception** : traité comme une chaîne, pas élément par élément |
| `float64`, `bool`, pointeur | `%!q(float64=0)` | aucune règle ne s'applique |

Cette dernière ligne est le format standard des **erreurs de formatage** :
`%!verbe(type=valeur)`. Le `%!` signale l'anomalie, puis vient le verbe fautif, puis le
type et la valeur reçus.

**`fmt` ne panique jamais sur un mauvais format** — il écrit le diagnostic *dans la
sortie*. C'est délibéré : un appel de journalisation ne doit pas faire tomber le
programme qu'il journalise. Mieux vaut une ligne de log étrange qu'un service à terre.

Le corollaire : puisque `fmt` ne vous arrêtera pas à l'exécution, il faut un outil qui
le fasse **avant**. C'est le rôle de `go vet`, qui signale `%q` sur un `float64` comme
il signale `%d` sur une `string`. Un programme peut compiler, s'exécuter, ne pas
paniquer — et être faux à chaque ligne. **`go vet` fait partie du cycle de
développement, pas des finitions.**

## Erreurs fréquentes

1. **`:=` hors d'une fonction** → `syntax error: non-declaration statement outside function body`. Au niveau du package, `var` est obligatoire.
2. **`:=` qui redéclare par accident** dans un bloc imbriqué :
   ```go
   err := doA()
   if cond {
       err := doB()   // ← NOUVELLE variable, l'externe n'est pas modifiée
       _ = err
   }
   ```
   C'est le *shadowing*. `go vet` ne le voit pas toujours ; `golangci-lint` avec `shadow` si.
3. **Division entière involontaire** : `1/2` vaut `0`. Convertir *avant* de diviser.
4. **Comparer des flottants avec `==`** : `0.1+0.2 == 0.3` est faux. Comparer à une tolérance près.
5. **Dépassement silencieux** : `int8(200)` vaut `-56`, sans aucune alerte.
6. **Croire que `var s string` vaut `nil`.** Une chaîne vaut `""`. Seuls pointeurs, slices, maps, channels, fonctions et interfaces valent `nil`.
7. **Utiliser `uint` pour « un nombre positif ».** Source de boucles infinies. `int` avec une validation est préférable.
8. **Écrire dans une map nil** → `panic: assignment to entry in nil map`. La lire est pourtant licite. Toute map doit passer par `make` (ou un littéral) avant la première écriture — y compris quand elle est un champ de struct.
9. **Croire qu'une slice n'est pas nil parce que `%v` affiche `[]`.** L'affichage montre le contenu, pas l'identité de la valeur. Tester avec `== nil`.

## Bonnes pratiques Go

- **`int`, `float64`, `string`** par défaut. Ne choisir une taille précise que si un
  format externe, une contrainte mémoire mesurée ou un protocole l'impose.
- Concevoir les types pour que **la zéro-valeur soit utilisable**.
- Constantes **non typées** par défaut : elles sont plus souples. Ne typer que pour créer
  une énumération ou contraindre volontairement.
- Grouper les constantes liées dans un bloc `const` unique.
- Un type d'énumération mérite une représentation textuelle ; au niveau 2, ce sera une
  méthode `String()`, que l'outil `stringer` sait générer.
- `_` (le « trou noir ») pour ignorer explicitement une valeur : `_, err := f()`.

## Ce que je dois retenir

- Trois formes de déclaration ; `:=` dans les fonctions, `var` au niveau du package.
- **La zéro-valeur existe toujours** : pas de variable non initialisée en Go.
- **`nil` ne s'affiche pas `nil`** pour une slice (`[]`) ni pour une map (`map[]`) : `%v`
  montre le contenu, pas l'identité. Tester avec `== nil`.
- Une **slice nil** s'utilise telle quelle (`len`, `range`, `append`) ; une **map nil**
  se lit mais **panique à l'écriture** — `make` avant d'écrire.
- **`go vet` avant de rendre**, toujours : `fmt` accepte des formats absurdes et écrit
  `%!verbe(type=valeur)` dans la sortie plutôt que de paniquer.
- **Aucune conversion implicite**, même entre `int` et `int64`.
- Une conversion qui déborde **tronque en silence** — c'est au développeur de vérifier.
- Les **constantes non typées** ont une précision arbitraire et s'adaptent au contexte.
- `iota` fabrique des énumérations, mais **ne valide rien**.
- Jamais de flottant pour de l'argent.

➡️ [Exercices](exercices.md)
