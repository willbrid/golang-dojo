# Leçon 3 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — La ligne `go` du `go.mod`

```
./main.go:3:19: generic method requires go1.27 or later (-lang was set to go1.26; check go.mod)
```

Le compilateur installé **sait** compiler la syntaxe ; il refuse parce que le module a
déclaré un contrat de langage plus ancien, et il désigne `go.mod`. C'est exactement le rôle
de la ligne `go` (niveau 2, leçon 9) : un module en `go 1.26` doit compiler avec un outil
1.26. Accepter la syntaxe le rendrait silencieusement incompatible.

## E2 — Fonction → méthode

```go
type List[T any] []T

func (l List[T]) Map[U any](f func(T) U) List[U] {
	out := make(List[U], 0, len(l))
	for _, v := range l {
		out = append(out, f(v))
	}
	return out
}

func (l List[T]) Reduce[A any](init A, f func(A, T) A) A {
	acc := init
	for _, v := range l {
		acc = f(acc, v)
	}
	return acc
}

n := List[string]{"go", "rust", "zig"}.
	Map(func(s string) int { return len(s) }).
	Reduce(0, func(acc, n int) int { return acc + n })
fmt.Println(n) // 9
```

Le paramètre `T` du récepteur reste dans la signature ; seul le paramètre **supplémentaire**
(`U`, `A`) se déclare après le nom. Remarquer le point **en fin de ligne** pour couper la
chaîne : en début de ligne, l'insertion automatique de point-virgule casserait l'expression.

## E3 — Pas d'interface

```
interface method must have no type parameters
```

`Len()` est **dans l'ensemble de méthodes** de `List[T]` : sa signature est fixée une fois
`T` connu, donc `List[int]` a une méthode `Len() int` et satisfait `Lener`. `Map` n'y est
pas : sa signature dépend d'un `U` choisi à chaque appel. Une méthode non générique d'un type
générique est une méthode **ordinaire** ; une méthode qui ajoute ses propres paramètres ne
l'est pas.

## E4 — Le `String()` fantôme

`fmt.Println(Temp(21.5))` affiche `21.5`. `fmt` cherche `String() string` par **assertion
de type** sur `fmt.Stringer`. Or une méthode générique n'est pas dans l'ensemble de méthodes :
l'assertion échoue, et `fmt` se rabat sur l'affichage par défaut du `float64`.

Aucune erreur, aucun avertissement : bug **silencieux**, le même que celui d'un `String()` à
récepteur pointeur appelé sur une valeur (niveau 2, leçon 4, E5). Correction : retirer `[T any]`,
qui ne servait à rien — `T` n'apparaissait nulle part.

Leçon générale : `go vet` ne signale pas ce cas. Un paramètre de type **inutilisé** doit
toujours alerter le relecteur.

## E5 — Valeurs et expressions de méthode

| Expression | Type |
|---|---|
| `a := l.Map[string]` | `func(func(int) string) main.List[string]` — valeur de méthode : `l` est capturé |
| `b := List[int].Map[bool]` | `func(main.List[int], func(int) bool) main.List[bool]` — expression de méthode : le récepteur devient le premier paramètre |
| `c := l.Map` | **ne compile pas** : `cannot use generic function l.Map without instantiation` |

## Exercice intermédiaire — `Config`

```go
// ErrMissing est retournée par Get quand la clé n'existe pas.
var ErrMissing = errors.New("clé absente")

type Config struct {
	values map[string]string
}

func Parse(r io.Reader) (*Config, error) {
	c := &Config{values: make(map[string]string)}
	sc := bufio.NewScanner(r)
	for n := 1; sc.Scan(); n++ {
		line := strings.TrimSpace(sc.Text())
		if line == "" || strings.HasPrefix(line, "#") {
			continue
		}
		k, v, ok := strings.Cut(line, "=")
		if !ok {
			return nil, fmt.Errorf("ligne %d : %q : « = » manquant", n, line)
		}
		k, v = strings.TrimSpace(k), strings.TrimSpace(v)
		if _, dup := c.values[k]; dup {
			return nil, fmt.Errorf("ligne %d : clé %q déjà définie", n, k)
		}
		c.values[k] = v
	}
	return c, sc.Err()
}

func (c *Config) Get[T any](key string, parse func(string) (T, error)) (T, error) {
	var zero T
	raw, ok := c.values[key]
	if !ok {
		return zero, fmt.Errorf("%q : %w", key, ErrMissing)
	}
	v, err := parse(raw)
	if err != nil {
		return zero, fmt.Errorf("%q : %w", key, err)
	}
	return v, nil
}

func (c *Config) GetOr[T any](key string, def T, parse func(string) (T, error)) (T, error) {
	v, err := c.Get(key, parse)
	if errors.Is(err, ErrMissing) {
		return def, nil
	}
	return v, err
}
```

Utilisation :

```go
port, err := cfg.Get("port", strconv.Atoi)                  // int
debug, err := cfg.Get("debug", strconv.ParseBool)           // bool
timeout, err := cfg.Get("timeout", time.ParseDuration)      // time.Duration
workers, err := cfg.GetOr("workers", 4, strconv.Atoi)       // 4 si absent
```

Points de correction :

- **`GetOr` appelle `Get`** : une méthode générique peut en appeler une autre, `T` est
  inféré depuis `parse`.
- **Clé absente vs invalide** : `errors.Is(err, ErrMissing)` pour l'une ; pour l'autre,
  l'erreur de `strconv` est enveloppée, donc `errors.As(err, &numErr)` avec un
  `*strconv.NumError` fonctionne aussi.
- **Clé dupliquée** : garder silencieusement la dernière est la source d'un bug classique —
  quelqu'un ajoute `port = 9090` en bas du fichier, l'ancienne valeur en haut reste, et on
  débogue la mauvaise ligne. Refuser est plus sûr ; c'est au minimum un choix à documenter.
- `GetOr` ne masque **pas** une valeur invalide : `port = abc` doit échouer, pas tomber
  silencieusement sur la valeur par défaut.

**Question de conception.**

| | `Get[T]` générique | `GetInt`, `GetBool`, … |
|---|---|---|
| Types couverts | **tous**, y compris ceux de l'utilisateur (`net.IP`, un `enum`…) | ceux prévus par l'auteur |
| Appel | il faut fournir `parse` | le plus simple possible |
| Découvrabilité dans `go doc` | une méthode | une méthode par type, auto-documentée |
| Version minimale de Go | **1.27** | n'importe laquelle |
| Cohérence des messages d'erreur | dépend de chaque `parse` | uniforme |

Pour une bibliothèque publique **aujourd'hui**, la réponse raisonnable est souvent : les
deux. `GetInt`/`GetBool`/`GetDuration` couvrent 90 % des usages, et une fonction générique
`Get` reste disponible pour le reste. Et la version minimale compte : imposer Go 1.27 en
août 2026, c'est exclure les utilisateurs encore en 1.25 ou 1.26, que l'équipe Go supporte
encore. Il n'y a pas de bonne réponse unique ; il y a une réponse **argumentée**.

## Défi — `Option[T]`

```go
type Option[T any] struct {
	v  T
	ok bool
}

func Some[T any](v T) Option[T] { return Option[T]{v: v, ok: true} }
func None[T any]() Option[T]    { return Option[T]{} }

func (o Option[T]) Get() (T, bool) { return o.v, o.ok }

func (o Option[T]) OrElse(def T) T {
	if !o.ok {
		return def
	}
	return o.v
}

func (o Option[T]) Map[U any](f func(T) U) Option[U] {
	if !o.ok {
		return None[U]()
	}
	return Some(f(o.v))
}

func (o Option[T]) String() string {
	if !o.ok {
		return "None"
	}
	return fmt.Sprintf("Some(%v)", o.v)
}
```

1. `None` n'a aucun paramètre : `T` n'apparaît que dans le résultat, l'inférence n'a rien à
   unifier (leçon 1). D'où `None[int]()`.
2. **Oui**, ce `String()` satisfait `fmt.Stringer` : ce n'est pas une méthode générique, c'est
   une méthode ordinaire d'un type générique. Une fois `T` fixé, `Option[int]` a une méthode
   `String() string` dans son ensemble de méthodes. C'est la distinction d'E3.
3. **`(User, bool)` est idiomatique.** Le comma-ok est partout dans la stdlib et dans le
   langage lui-même : `v, ok := m[k]`, `v, ok := x.(T)`, `v, ok := <-ch` (niveau 6),
   `os.LookupEnv`, `strings.Cut`, `sync.Map.Load`. Un type `Option` ferait coexister deux
   conventions pour la même idée, sans rien apporter que Go n'ait déjà : le compilateur oblige
   à récupérer le `bool` (on ne peut pas ignorer un résultat sans l'écrire `_`), et la
   zéro-valeur de `User` est une valeur définie, pas un pointeur nul dangereux. `Option` est
   la bonne idée dans un langage où l'absence est `null` ; en Go, l'absence a déjà sa forme.
   Ajouter un `Option` dans une base de code Go, c'est forcer chaque lecteur à apprendre une
   seconde convention.

## Quiz

1. Go 1.27.
2. Après le nom de la méthode : `func (r R) Nom[U any](…)`.
3. Oui.
4. Parce que les interfaces sont résolues à l'exécution via une table de méthodes construite
   à la compilation, qui devrait contenir une entrée par instanciation possible — une infinité
   non connue à la compilation de chaque paquet.
5. Non : elle n'est pas dans l'ensemble de méthodes, l'assertion de `fmt` sur `Stringer` échoue.
6. Une instanciation explicite : `x.M[int]`.
7. Une opération symétrique (`Union(a, b)`) ; une opération qui doit marcher sur des types
   ordinaires (`[]T`, `map[K]V`) et pas seulement sur un type défini.
8. Elle relève la version minimale de Go à 1.27 pour tous les utilisateurs.
9. Une allocation par maillon, pas de variable intermédiaire à inspecter, pas d'arrêt anticipé,
   et une lisibilité qui se dégrade vite.
