# Leçon 3 — Exercices

Code dans `mes-solutions/niveau-03/lecon-03/`. Le `go.mod` doit contenir `go 1.27`.

## Exercices faciles

**E1 — La ligne `go` du `go.mod`.**
Écrire une méthode générique quelconque. Passer la ligne `go` du `go.mod` à `1.26` et
compiler. Recopier le message. **Quel fichier le compilateur te demande-t-il de vérifier ?**
Remettre `1.27`.

**E2 — Fonction → méthode.**
Réécrire en méthode générique sur `type List[T any] []T` :

```go
func Map[T, U any](l List[T], f func(T) U) List[U]
func Reduce[T, A any](l List[T], init A, f func(A, T) A) A
```

Puis, sur `List[string]{"go", "rust", "zig"}`, calculer **en une chaîne** la somme des
longueurs des mots. Sortie attendue : `9`.

**E3 — Pas d'interface.**
Déclarer `type Mapper interface { Map[U any](func(int) U) List[U] }`. Recopier le message.
Puis déclarer `type Lener interface { Len() int }`, ajouter à `List[T]` une méthode
**non générique** `Len`, et vérifier avec `var _ Lener = List[int]{}`. Qu'est-ce qui
distingue les deux méthodes aux yeux du compilateur ?

**E4 — Le `String()` fantôme.**
Écrire :

```go
type Temp float64
func (t Temp) String[T any]() string { return fmt.Sprintf("%.1f°C", float64(t)) }
```

Afficher `fmt.Println(Temp(21.5))`. Qu'affiche-t-on ? Pourquoi `fmt` n'a-t-il pas utilisé la
méthode ? Corriger.

**E5 — Valeurs et expressions de méthode.**
Avec la `List` de E2, prédire le type de chacune de ces expressions, puis vérifier avec `%T` :

```go
l := List[int]{1, 2}
a := l.Map[string]
b := List[int].Map[bool]
c := l.Map
```

L'une ne compile pas : laquelle, et avec quel message ?

---

## Exercice intermédiaire — `Config`, un lecteur de configuration typé

Écrire un type `Config` qui lit des lignes `clé=valeur` et rend des valeurs **typées** :

```go
type Config struct { /* map[string]string non exportée */ }

func Parse(r io.Reader) (*Config, error)

// Get rend la valeur décodée par parse. Erreur si la clé manque ou ne se décode pas.
func (c *Config) Get[T any](key string, parse func(string) (T, error)) (T, error)

// GetOr rend def si la clé est absente — mais une erreur si elle est présente et invalide.
func (c *Config) GetOr[T any](key string, def T, parse func(string) (T, error)) (T, error)
```

**Contraintes :**
1. Lignes vides et lignes commençant par `#` ignorées ; espaces autour de `=` retirés.
   *(`strings.Cut`, niveau 1, leçon 8.)*
2. Une ligne sans `=` est une erreur qui cite le **numéro de ligne** et le contenu (`%q`).
3. Une clé dupliquée est une erreur. Discuter : pourquoi ne pas simplement garder la dernière ?
4. L'erreur de `Get` doit permettre à l'appelant de distinguer **« clé absente »** de
   **« valeur invalide »** avec `errors.Is` (une sentinelle pour la première ;
   l'enveloppement `%w` de l'erreur de `parse` pour la seconde).
5. Utiliser `Get` avec `strconv.Atoi`, `strconv.ParseBool` et `time.ParseDuration`.
6. **Question de conception à trancher par écrit** : un collègue propose plutôt
   `GetInt`, `GetBool`, `GetDuration`, sans générique. Avantages et inconvénients des deux
   approches pour un **utilisateur** du paquet ? Laquelle choisirais-tu pour une bibliothèque
   publique, et pourquoi ?

Fichier de test :

```
# serveur
port = 8080
debug=true
timeout = 2s
nom = api
```

---

## Défi — `Option[T]`

```go
type Option[T any] struct {
	v  T
	ok bool
}

func Some[T any](v T) Option[T]
func None[T any]() Option[T]
func (o Option[T]) Get() (T, bool)
func (o Option[T]) OrElse(def T) T
func (o Option[T]) Map[U any](f func(T) U) Option[U]
```

1. Implémenter. Pourquoi `None` exige-t-il une instanciation explicite à l'appel (`None[int]()`) ?
2. Écrire `func (o Option[T]) String() string`. Satisfait-elle `fmt.Stringer` ? Pourquoi
   celle-ci oui, alors que le `String` d'E4 non ?
3. **La vraie question** : comparer `func Find(id int) Option[User]` et
   `func Find(id int) (User, bool)`. Lequel est idiomatique en Go, et pourquoi la
   bibliothèque standard n'a-t-elle **aucun** type `Option` ? Réponse en 5 à 10 lignes, en
   citant au moins un exemple de la stdlib qui utilise l'autre forme.

---

## Quiz

1. Depuis quelle version de Go une méthode peut-elle avoir ses propres paramètres de type ?
2. Où se placent les crochets dans la déclaration ?
3. Une méthode générique peut-elle exister sur un type non générique ?
4. Pourquoi une interface ne peut-elle pas déclarer de méthode générique ?
5. Une méthode `String[T any]() string` sera-t-elle appelée par `fmt.Println` ?
6. Que faut-il écrire pour prendre une valeur de méthode générique ?
7. Citer deux cas où une fonction générique reste préférable à une méthode générique.
8. Quel effet a l'ajout d'une méthode générique dans une bibliothèque publique sur ses utilisateurs ?
9. Que reproche-t-on à une chaîne `a.Filter(…).Map(…).Filter(…).Fold(…)` ?
