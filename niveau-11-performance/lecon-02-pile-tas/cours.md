# Leçon 2 — Pile, tas, analyse d'échappement

> En Go, on n'écrit jamais `malloc` ni `free`, et on ne choisit pas explicitement où vit une valeur. Le
> compilateur le décide : sur la **pile** de la goroutine (gratuit, libéré à la sortie de la fonction) ou sur
> le **tas** (une allocation, puis du travail pour le ramasse-miettes). Cette décision, l'**analyse
> d'échappement**, explique une grande partie des écarts de performance entre deux codes qui se
> ressemblent. Cette leçon apprend à la lire, et à savoir ce qui fait « s'échapper » une valeur.

## Objectifs

1. Comprendre la différence de coût entre pile et tas.
2. Lire la sortie de `go build -gcflags=-m`.
3. Connaître les causes d'échappement : pointeur qui survit, interface, closure, taille inconnue ou trop
   grande.
4. Vérifier par un benchmark (`allocs/op`) ce que dit le compilateur.
5. Savoir quand cela vaut la peine d'y penser.

## Explication

### Pile et tas

Chaque goroutine a sa **pile** : une zone mémoire qui grandit et rétrécit au fil des appels. Une variable
locale qui y est placée ne coûte presque rien : réserver sa place, c'est décaler un pointeur, et elle
disparaît d'elle-même au retour de la fonction. La pile d'une goroutine commence petite (quelques Kio) et
le *runtime* l'agrandit au besoin en la recopiant.

Le **tas** est partagé par tout le programme. Y allouer demande de trouver de la place (rapide en Go, mais
pas gratuit), et chaque objet alloué devra être **examiné puis récupéré** par le ramasse-miettes (leçon 3).
Le vrai coût du tas n'est donc pas l'allocation seule : c'est la pression sur le ramasse-miettes, qui ralentit
tout le programme (vu au défi de la leçon 1).

La règle du compilateur : une valeur peut rester sur la pile s'il peut **prouver** qu'aucune référence vers
elle ne survit à la fonction. Sinon, elle « s'échappe » vers le tas. C'est une analyse **à la compilation**,
prudente : dans le doute, le tas.

Ce n'est **pas** une question de syntaxe : `new(T)` et `&T{}` peuvent rester sur la pile, et une valeur
déclarée sans pointeur peut partir sur le tas. Seul compte ce que le compilateur peut prouver.

### Demander au compilateur

```sh
go build -gcflags=-m ./...        # les décisions d'échappement (et d'inlining)
go build -gcflags='-m -m' ./...   # avec les raisons, très détaillé
```

Les messages à connaître :

| Message | Sens |
|---|---|
| `x escapes to heap` | la valeur `x` est allouée sur le tas |
| `moved to heap: x` | la **variable** `x` (déclarée localement) est déplacée sur le tas |
| `x does not escape` | aucune référence ne survit ; la valeur peut rester sur la pile |
| `leaking param: p` | le paramètre `p` (un pointeur) est conservé au-delà de l'appel : ce que l'appelant passe s'échappera |
| `can inline f` | `f` est assez petite pour être recopiée dans ses appelants |

L'*inlining* compte : une fonction recopiée dans son appelant donne au compilateur une vue plus large, et une
valeur qui s'échappait d'une fonction séparée peut alors rester sur la pile de l'appelant.

### Ce qui fait s'échapper une valeur

Mesuré sur l'exemple (Go 1.27), compilateur et benchmark réunis :

| Cas | `-gcflags=-m` | Allocations |
|---|---|---|
| 1. rendre une valeur (`Point`) | — | 0 |
| 2. rendre `&Point{…}` | `&Point{...} escapes to heap` | 1 (16 o) |
| 3. recevoir un pointeur sans le garder | `p does not escape` | 0 |
| 4. ranger le pointeur dans une globale | `leaking param: p` | (ce que l'appelant passe part sur le tas) |
| 5. passer une valeur à `fmt.Sprint` (un `any`) | `p escapes to heap` | 2 |
| 6. `make([]byte, 64)`, taille constante | `does not escape` | 0 |
| 7. `make([]byte, n)`, taille variable, n ≤ 32 | `does not escape` | **0** |
| 7. `make([]byte, n)`, taille variable, n = 33 | `does not escape` | **1 (48 o)** |
| 8. closure qui capture `n` et est rendue | `moved to heap: n`, `func literal escapes` | 2 |
| 9. `var a [1 << 20]byte` (1 Mio) | `moved to heap: a` | 1 (1 Mio) |

Les causes, en résumé :

- **un pointeur qui survit** à la fonction (rendu, rangé dans une globale, dans une struct qui survit, envoyé
  dans un canal) ;
- **une interface** : convertir une valeur en `any` ou en interface la copie souvent sur le tas, car
  l'interface ne contient qu'un pointeur vers elle (les très petites valeurs et les constantes font
  exception) ; `fmt.Println(x)` en est l'exemple quotidien ;
- **une closure** qui capture une variable et survit à la fonction : la variable doit vivre aussi longtemps
  qu'elle ;
- **une taille inconnue ou trop grande** pour la pile.

Le cas 7 mérite attention, car il contredit une idée répandue (« taille variable = tas ») : le compilateur
récent réserve un petit tampon de **32 octets** sur la pile pour un `make` de taille variable qui ne
s'échappe pas, et ne passe par le tas que si la taille réelle le dépasse. Le message
`does not escape` signifie donc « pas d'échappement **structurel** », pas « jamais d'allocation ». Seul
`allocs/op` le dit à coup sûr. Le détail exact (32 octets, conditions) est un comportement du compilateur,
qui peut évoluer d'une version à l'autre : on le **mesure**, on ne l'apprend pas par cœur.

### Faut-il y penser ?

Dans 95 % du code, non : écrire du code clair, et laisser le compilateur faire. L'analyse d'échappement
devient utile **quand un profil** (leçons 4 et 5) montre qu'une fonction alloue beaucoup sur un chemin
chaud. Alors `-gcflags=-m` explique **pourquoi**, et les corrections typiques sont simples :

- rendre une valeur plutôt qu'un pointeur pour les petites structs ;
- passer un tampon à remplir (`func Encode(dst []byte, …) []byte`, sur le modèle de `strconv.AppendInt`)
  plutôt que d'en allouer un à chaque appel ;
- éviter `any` et `fmt` sur les chemins très chauds (utiliser `strconv`) ;
- préallouer (leçon 6), réutiliser (leçon 8).

## Exemple

Les neuf cas, et un benchmark qui mesure leurs allocations.

```go
// Fichier : esc.go
// Package esc : où le compilateur place-t-il chaque valeur ?
package esc

import "fmt"

type Point struct{ X, Y int }

// 1. Rendre une VALEUR : la copie part chez l'appelant ; rien ne s'échappe.
func NewValue(x, y int) Point { return Point{x, y} }

// 2. Rendre un POINTEUR vers une variable locale : elle doit survivre à la fonction → tas.
func NewPointer(x, y int) *Point { return &Point{x, y} }

// 3. Un pointeur reçu mais pas conservé : « does not escape ».
func Sum(p *Point) int { return p.X + p.Y }

// 4. Un pointeur reçu et CONSERVÉ dans une variable globale : « leaking param ».
var last *Point

func Remember(p *Point) { last = p }

// 5. Une valeur passée à une interface (ici fmt.Sprint) : souvent copiée sur le tas.
func Describe(p Point) string { return fmt.Sprint(p) }

// 6. Une slice de taille CONSTANTE et petite : sur la pile.
func SmallBuffer() int {
	buf := make([]byte, 64)
	return fill(buf)
}

// 7. Une slice de taille VARIABLE : le compilateur réserve 32 octets sur la pile ; si n les
// dépasse, le tableau est alloué sur le tas — malgré le message « does not escape ».
// (noinline : pour la démonstration, empêche le compilateur de voir n comme une constante.)
//
//go:noinline
func DynamicBuffer(n int) int {
	buf := make([]byte, n)
	return fill(buf)
}

func fill(buf []byte) int {
	s := 0
	for i := range buf {
		buf[i] = byte(i)
		s += int(buf[i])
	}
	return s
}

// 8. Une closure qui capture une variable et survit à la fonction : la variable part sur le tas.
func Counter() func() int {
	n := 0
	return func() int { n++; return n }
}

// 9. Un tableau trop grand pour la pile (implicite) : tas.
func Big() int {
	var a [1 << 20]byte // 1 Mio
	a[0] = 1
	return len(a)
}
```

```go
// Fichier : esc_test.go
package esc

import "testing"

var (
	sinkP *Point
	sinkV Point
	sinkI int
	sinkS string
	sinkF func() int
)

func BenchmarkEscape(b *testing.B) {
	p := Point{1, 2}
	cases := []struct {
		name string
		fn   func()
	}{
		{"1-valeur", func() { sinkV = NewValue(1, 2) }},
		{"2-pointeur", func() { sinkP = NewPointer(1, 2) }},
		{"3-Sum", func() { sinkI = Sum(&p) }},
		{"5-interface", func() { sinkS = Describe(p) }},
		{"6-petit-buffer", func() { sinkI = SmallBuffer() }},
		{"7-buffer-variable-32", func() { sinkI = DynamicBuffer(32) }},
		{"7-buffer-variable-33", func() { sinkI = DynamicBuffer(33) }},
		{"8-closure", func() { sinkF = Counter() }},
		{"9-gros-tableau", func() { sinkI = Big() }},
	}
	for _, c := range cases {
		b.Run(c.name, func(b *testing.B) {
			for b.Loop() {
				c.fn()
			}
		})
	}
}
```

`go build -gcflags=-m .` (lignes d'*inlining* retirées) :

```
# esc
./esc.go:12:43: &Point{...} escapes to heap
./esc.go:15:10: p does not escape
./esc.go:20:15: leaking param: p
./esc.go:23:50: ... argument does not escape
./esc.go:23:51: p escapes to heap
./esc.go:27:13: make([]byte, 64) does not escape
./esc.go:37:13: make([]byte, n) does not escape
./esc.go:41:11: buf does not escape
./esc.go:52:2: moved to heap: n
./esc.go:53:9: func literal escapes to heap
./esc.go:58:6: moved to heap: a
```

`go test -bench . -benchmem` (extrait, temps indicatifs) :

```
BenchmarkEscape/1-valeur                   1.8 ns/op         0 B/op   0 allocs/op
BenchmarkEscape/2-pointeur                15.5 ns/op        16 B/op   1 allocs/op
BenchmarkEscape/3-Sum                      1.8 ns/op         0 B/op   0 allocs/op
BenchmarkEscape/5-interface              225.7 ns/op        21 B/op   2 allocs/op
BenchmarkEscape/6-petit-buffer            23.3 ns/op         0 B/op   0 allocs/op
BenchmarkEscape/7-buffer-variable-32      17.7 ns/op         0 B/op   0 allocs/op
BenchmarkEscape/7-buffer-variable-33      53.5 ns/op        48 B/op   1 allocs/op
BenchmarkEscape/8-closure                 32.3 ns/op        24 B/op   2 allocs/op
BenchmarkEscape/9-gros-tableau        154723   ns/op   1048577 B/op   1 allocs/op
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `NewValue` contre `NewPointer` | Même contenu ; le pointeur rendu oblige le tas (1 allocation, ×9 en temps ici). |
| `Sum(p *Point)` | Un pointeur reçu n'implique pas d'allocation : `p does not escape`. |
| `Remember` | `leaking param` : ce que l'appelant passera partira sur le tas. |
| `Describe` → `fmt.Sprint(p)` | La conversion en `any` copie `p` sur le tas, et `fmt` alloue la chaîne : 2 allocations, 225 ns. |
| `DynamicBuffer(32)` / `(33)` | Même code, même message du compilateur ; une allocation de plus au-delà de 32 octets. |
| `//go:noinline` | Pour la démonstration seulement : sans lui, l'appel serait *inliné* avec une taille constante. |
| `Counter` | La variable capturée `n` vit aussi longtemps que la closure : sur le tas. |
| `Big` | 1 Mio ne tient pas sur une pile qu'on veut petite : tas. |
| Globales `sink…` | Empêchent l'élimination des résultats ; le benchmark mesure bien un appel. |

## Erreurs fréquentes

1. **Croire que `new` ou `&` signifient « tas »**, et `var x T` « pile ».
2. **Lire `does not escape` comme « aucune allocation »** (cas 7).
3. **Rendre des pointeurs vers de petites structs** « pour éviter une copie », et payer une allocation à la
   place.
4. **`fmt` sur un chemin très chaud**.
5. **Optimiser l'échappement sans profil** : du code plus obscur pour un gain nul.
6. **Se fier à une règle apprise** sur une version ancienne du compilateur au lieu de mesurer.

## Bonnes pratiques Go

- Code clair d'abord ; `-gcflags=-m` quand un profil désigne une fonction qui alloue.
- Vérifier toute hypothèse par `allocs/op`.
- Petites structs rendues par valeur ; fonctions `Append…(dst, …)` pour les tampons.
- `strconv` plutôt que `fmt` pour les conversions fréquentes.
- Relire les décisions du compilateur après une mise à jour de Go : elles évoluent.

## Ce que je dois retenir

- Pile : quasi gratuite, libérée au retour. Tas : une allocation, puis du travail pour le ramasse-miettes.
- Le compilateur décide (analyse d'échappement) ; dans le doute, le tas.
- Causes : pointeur qui survit, interface, closure capturée, taille trop grande ou inconnue.
- `go build -gcflags=-m` dit pourquoi ; `allocs/op` dit combien.
- N'y penser que sur un chemin chaud désigné par un profil.

---

➡️ [Exercices](exercices.md)
