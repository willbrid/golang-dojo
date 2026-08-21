# Leçon 1 — Pointeurs

## Objectifs

1. Expliquer ce qu'est un pointeur, sans mystère et sans peur.
2. Savoir **quand** un pointeur est justifié — et quand il ne l'est pas.
3. Comprendre pourquoi Go n'a pas d'arithmétique de pointeurs et pourquoi c'est une bonne nouvelle.
4. Éviter les paniques `nil pointer dereference`.

## Explication

### Rappel général : la mémoire

Toute variable occupe une zone mémoire, qui a une **adresse** — un numéro. Un **pointeur**
est une variable qui contient l'adresse d'une autre variable, au lieu de contenir
directement une valeur.

Analogie : la valeur est le contenu d'une maison, l'adresse est le papier où est écrit
« 12 rue des Lilas ». Copier le papier est instantané, quel que soit le contenu de la
maison ; et deux personnes ayant le même papier voient les mêmes meubles.

### Syntaxe : deux opérateurs

```go
x := 42
p := &x           // & : « adresse de » → p est de type *int
fmt.Println(p)    // 0xc000012345 (une adresse)
fmt.Println(*p)   // 42 — * : « déréférencer », suivre le pointeur
*p = 100          // écrit à travers le pointeur
fmt.Println(x)    // 100 — x a changé
```

| Symbole | Nom | Effet |
|---|---|---|
| `*T` (dans un type) | type pointeur | « pointeur vers un T » |
| `&x` | opérateur adresse | donne l'adresse de `x` |
| `*p` (sur une valeur) | déréférencement | donne la valeur pointée |

Le `*` a donc deux sens selon le contexte : dans une déclaration de type il **construit**
un type pointeur ; devant une variable il **suit** le pointeur. Source de confusion
initiale, qui se dissipe vite.

### Tout est passé par valeur — les pointeurs aussi

En Go, **tous** les arguments sont copiés. Passer un pointeur copie… le pointeur (8 octets),
pas ce qu'il désigne. C'est ce qui permet à la fonction de modifier l'original :

```go
func incrementValue(n int)  { n++ }        // modifie une copie : sans effet
func incrementPointer(p *int) { *p++ }     // modifie l'original

x := 5
incrementValue(x)    // x vaut toujours 5
incrementPointer(&x) // x vaut 6
```

Go n'a **pas** de passage par référence comme le C++ (`int&`). Il n'a que le passage par
valeur — mais on peut passer la valeur d'une adresse. C'est plus simple à raisonner : il
n'y a qu'une seule règle.

### Ce qui contient déjà un pointeur

Slices, maps et channels contiennent un pointeur **en interne**. Les passer par valeur copie
leur descripteur, mais le contenu reste partagé :

```go
func modify(s []int) { s[0] = 99 }        // visible par l'appelant
func modify(m map[string]int) { m["k"]=1 } // visible par l'appelant
func grow(s []int) { s = append(s, 1) }    // PAS visible : réaffecte la copie locale
```

Conséquence pratique : **ne jamais écrire `*[]int` ni `*map[K]V`**, sauf le cas rare où
la fonction doit remplacer le slice entier de l'appelant. C'est une erreur de débutant
fréquente.

### `nil` et la panique

La zéro-valeur d'un pointeur est `nil`. Le déréférencer plante :

```go
var p *int
fmt.Println(*p)   // panic: runtime error: invalid memory address or nil pointer dereference
```

C'est la panique la plus courante en Go. Vérifier avant de déréférencer un pointeur qui peut
légitimement être `nil` :

```go
if p != nil {
	fmt.Println(*p)
}
```

Subtilité importante : appeler une **méthode** sur un pointeur `nil` ne panique pas
forcément — seul l'accès à un champ le fait. On y reviendra en leçon 3.

### `new` et les littéraux

```go
p := new(int)          // pointeur vers un int valant 0
*p = 42

u := &User{Name: "Alice"}   // littéral avec &, la forme idiomatique
v := new(User)              // équivalent mais plus rare pour les structs
```

`new(T)` alloue un `T` à zéro et retourne `*T`. En pratique, `&T{…}` est presque toujours
préféré : plus court et permet d'initialiser au passage.

### Pas d'arithmétique de pointeurs

```go
p := &arr[0]
p++          // ERREUR de compilation : impossible en Go
```

Contrairement au C, on ne peut pas se promener en mémoire avec un pointeur. Conséquence :
**pas de dépassement de tampon, pas de corruption mémoire** dans du code Go ordinaire. Le
paquet `unsafe` permet de contourner cette règle ; il ne concerne que des cas très
particuliers (interopérabilité C, sérialisation extrême) et est hors sujet ici.

### Retourner un pointeur vers une variable locale : légal

```go
func NewUser(name string) *User {
	u := User{Name: name}
	return &u        // parfaitement sûr en Go
}
```

En C, ce serait un bug grave (la variable meurt avec la pile). En Go, le compilateur détecte
que `u` **s'échappe** de la fonction et l'alloue sur le tas automatiquement. C'est
l'*escape analysis*, qu'on inspectera au niveau 8 avec `go build -gcflags="-m"`. Le
développeur n'a pas à savoir où vit une variable — le compilateur décide, et il ne se trompe
jamais sur la sûreté.

### Quand utiliser un pointeur ? La vraie question

C'est ici que se joue la qualité du code. Quatre raisons **légitimes** :

1. **La fonction doit modifier son argument.** C'est la seule raison sémantique.
2. **La struct est grosse** et la copier coûterait. « Grosse » commence vers quelques
   centaines d'octets — à mesurer, pas à supposer.
3. **La valeur doit pouvoir être absente**, et la zéro-valeur est une valeur légitime.
   `*int` distingue « pas de valeur » de « zéro » — typique en JSON ou en SQL.
4. **Le type contient un `sync.Mutex`** ou tout autre élément qui ne doit pas être copié.
   Copier un mutex verrouillé est un bug ; `go vet` le détecte.

Trois raisons **illégitimes**, fréquentes chez les débutants :

- « C'est plus rapide » — souvent faux : une petite struct sur la pile est plus rapide
  qu'une allocation sur le tas suivie d'indirections. **Mesurer.**
- « Comme ça je peux retourner nil » — cela transforme chaque accès en risque de panique.
  Une valeur plus une erreur, ou un `(T, bool)`, est presque toujours meilleur.
- « J'ai vu ça dans un exemple » — la stdlib utilise massivement des valeurs :
  `time.Time`, `strings.Builder` (par valeur, mais non copiable après usage), `net.IP`.

Le réflexe idiomatique : **commencer par une valeur, passer au pointeur si un besoin réel
apparaît**.

## Exemple

```go
package main

import "fmt"

type Config struct {
	Host    string
	Port    int
	Timeout *int // pointeur : distingue « non spécifié » de « 0 »
}

// setDefaults modifie la config en place : le pointeur est justifié par la sémantique.
func setDefaults(c *Config) {
	if c == nil {
		return // défense : une méthode publique ne doit pas paniquer sur nil
	}
	if c.Host == "" {
		c.Host = "localhost"
	}
	if c.Port == 0 {
		c.Port = 8080
	}
	if c.Timeout == nil {
		d := 30
		c.Timeout = &d // 30 par défaut, mais 0 explicite reste possible
	}
}

// describe ne modifie rien : elle prend une VALEUR. Config est petite, la copie est gratuite.
func describe(c Config) string {
	t := "non défini"
	if c.Timeout != nil {
		t = fmt.Sprintf("%ds", *c.Timeout)
	}
	return fmt.Sprintf("%s:%d (timeout %s)", c.Host, c.Port, t)
}

func main() {
	var c1 Config
	setDefaults(&c1)
	fmt.Println(describe(c1))

	zero := 0
	c2 := Config{Host: "api.local", Timeout: &zero} // timeout explicitement nul
	setDefaults(&c2)
	fmt.Println(describe(c2)) // timeout 0s, PAS 30s — c'est tout l'intérêt du pointeur

	// Deux variables, une seule donnée
	a := 10
	p := &a
	q := p
	*q = 20
	fmt.Println(a, *p, *q) // 20 20 20
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `Timeout *int` | Le seul cas où un pointeur sur un scalaire se justifie : distinguer « absent » de « zéro ». Très courant en JSON et en SQL. |
| `if c == nil { return }` | Une fonction publique qui prend un pointeur devrait gérer `nil` plutôt que paniquer. |
| `d := 30; c.Timeout = &d` | On ne peut pas écrire `&30` : les littéraux ne sont pas adressables. Détail syntaxique agaçant, à connaître. |
| `describe(c Config)` | Prend une **valeur** : elle ne modifie rien, et `Config` est petite. Choisir un pointeur ici serait du bruit. |
| `c2` avec `Timeout: &zero` | Démontre le besoin : sans pointeur, `setDefaults` écraserait le 0 explicite par 30. |
| `q := p; *q = 20` | Copier un pointeur copie l'adresse ; les deux désignent la même donnée. |

## Erreurs fréquentes

1. **Déréférencer un pointeur `nil`** → la panique n°1 en Go.
2. **`*[]T` ou `*map[K]V`** : redondant, slices et maps partagent déjà leur contenu.
3. **`&30`** ou `&someFunc()` : les littéraux et les résultats d'appel ne sont pas adressables.
4. **Pointeur « pour la performance » sans mesure** : souvent contre-productif.
5. **Prendre l'adresse d'une entrée de map** : `&m["k"]` ne compile pas (niveau 1, leçon 7).
6. **Copier une struct contenant un `sync.Mutex`** : `go vet` le signale, et c'est un vrai bug.
7. **Retourner `*T, error` avec le pointeur non nil ET l'erreur non nil** : contrat ambigu.
8. **Croire qu'il faut un pointeur pour modifier un slice** : `s[0] = 1` suffit.

## Bonnes pratiques Go

- **Valeur par défaut, pointeur quand c'est nécessaire.** L'inverse conduit à du code fragile.
- Documenter quand une fonction modifie son argument — ou l'exprimer dans son nom.
- Ne pas mélanger récepteurs valeur et pointeur sur un même type (leçon 3).
- Vérifier `nil` à la frontière publique d'un paquet, pas à chaque ligne interne.
- `&T{…}` plutôt que `new(T)`.
- Ne pas exposer de pointeurs vers l'état interne d'une structure : l'appelant pourrait le
  corrompre. Retourner une copie (voir l'exercice du `Ring` en leçon 3).
- `unsafe` : jamais, sauf raison documentée et validée en revue.

## Ce que je dois retenir

- Un pointeur contient une **adresse** ; `&` la prend, `*` la suit.
- **Tout est passé par valeur** en Go, y compris les pointeurs.
- Slices, maps et channels **partagent déjà** leur contenu : pas de pointeur dessus.
- Déréférencer `nil` panique ; c'est l'erreur d'exécution la plus fréquente.
- **Pas d'arithmétique de pointeurs** : la mémoire ne peut pas être corrompue.
- Retourner l'adresse d'une locale est **sûr** : le compilateur gère l'échappement.
- Quatre raisons d'utiliser un pointeur : muter, grosse struct, valeur optionnelle, non copiable.

🧵 **Fil rouge :** cette leçon ajoute une ligne au tableau *[Copié ou partagé ?](../../ressources/copie-ou-partage.md)* — c'est le moment de le relire.

➡️ [Exercices](exercices.md)
