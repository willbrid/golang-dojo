# Leçon 7 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Rejouer la séance

Sur la machine de rédaction, le bug n° 1 a été trouvé en **0,02 s**, l'entrée minimisée étant
`"0"`. Chez toi, la durée et l'entrée exacte peuvent différer (le moteur est aléatoire), mais
l'entrée trouvée contient forcément un **chiffre** : c'est la cause. Bug n° 2 : un octet UTF-8
invalide (n'importe lequel parmi `\x80`–`\xff` seul). Bug n° 3 : un grand nombre suivi de `:` et
d'un caractère — le processus est tué faute de mémoire.

## E2 — Invariant de `Slug`

```go
var slugRe = regexp.MustCompile(`^([a-z0-9]+(-[a-z0-9]+)*)?$`)

func FuzzSlug(f *testing.F) {
	for _, s := range []string{"Bonjour le Monde", "Ça, c'est l'été!", "", "--a--"} {
		f.Add(s)
	}
	f.Fuzz(func(t *testing.T, s string) {
		got := slug.Slug(s)
		if !slugRe.MatchString(got) {
			t.Fatalf("Slug(%q) = %q : forme invalide", s, got)
		}
		if again := slug.Slug(got); again != got {
			t.Fatalf("Slug non idempotent : %q → %q → %q", s, got, again)
		}
	})
}
```

Une seule expression régulière (ancrée, N4 L2) exprime (b) et (c) ensemble. Avec le `Slug` de
N4 L1, la forme est respectée par construction (`FieldsFunc` + `Join`), et l'idempotence aussi.
Si tu as une version personnelle qui ajoute des tirets à la main, le fuzzer trouve très vite un
`"-"` final ou doublé.

## E3 — Le corpus

Avec `go test -v`, l'entrée apparaît comme un sous-test : `=== RUN FuzzRoundTrip/771e938e4458e983`.
Supprimer le fichier, c'est perdre le **test de non-régression** : si quelqu'un réintroduit le
bug, seul un nouveau fuzzing (pas lancé en CI) le retrouverait.

## E4 — Équivalence

Une version qui ne considère comme espaces que `' '`, `'\t'`, `'\n'` diverge sur les autres
espaces Unicode : `'\v'`, `'\f'`, `'\r'`, `'\u0085'` (NEL), `' '` (espace insécable), etc.
`strings.Fields` utilise `unicode.IsSpace`. Le fuzzer trouve en général `\r` ou `\v` en quelques
secondes. La cible d'équivalence est la meilleure protection d'une optimisation : on garde la
version simple comme **référence**.

## E5 — Non déterminisme

Le fuzzer finit par signaler un échec, mais la commande de rejeu **passe** neuf cent quatre-vingt-
dix-neuf fois sur mille : l'entrée enregistrée n'est pas la cause. Le moteur peut même se plaindre
d'une entrée qui « ne reproduit pas » pendant la minimisation. Conclusion : une cible doit être une
**fonction de son entrée**, sans horloge, aléatoire ni état global — les mêmes règles que pour les
golden files.

## Exercice intermédiaire — Fuzzer `duration.Parse`

```go
func FuzzParse(f *testing.F) {
	for _, s := range []string{"2h", "1h30min", "45s", "2j"} {
		f.Add(s)
	}
	f.Fuzz(func(t *testing.T, s string) {
		d, err := duration.Parse(s)
		if err == nil && d < 0 {
			t.Fatalf("Parse(%q) = %v : durée négative sans erreur", s, d)
		}
	})
}
```

Trouvé en **9 secondes** sur la machine de rédaction :

```
fuzz_test.go:16: Parse("28200000h") = -2544573h27m22.257309696s : durée négative sans erreur
```

`time.Duration` est un `int64` de nanosecondes : son maximum est d'environ **2 562 047 heures**
(292 ans). 28 200 000 h dépasse ce maximum ; `time.Duration(n) * time.Hour` déborde
**silencieusement** et le bit de signe bascule (complément à deux, N1 L2). La table de la leçon 2,
avec ses onze cas soigneusement choisis, ne contenait aucun grand nombre.

Correctif :

```go
var ErrOverflow = errors.New("durée trop grande")

// dans la boucle, avant d'ajouter :
if time.Duration(n) > (math.MaxInt64-total)/d {
	return 0, fmt.Errorf("%q : %w", s, ErrOverflow)
}
total += time.Duration(n) * d
```

Le test se fait par **division** : `n * d > max - total` est réécrit `n > (max - total) / d`, qui ne
peut pas déborder (`total` est toujours entre 0 et `max`, `d` est positif). Vérifié : 30 s de
fuzzing supplémentaires sans échec.

Pourquoi ajouter le cas **aussi** à la table : `testdata/fuzz` contient une entrée opaque
(`string("28200000h")`) dans un fichier nommé par un hash ; la table la **nomme**
(`{name: "débordement", in: "28200000h", wantErr: duration.ErrOverflow}`) et vérifie **quelle**
erreur est rendue, pas seulement l'absence de durée négative. L'un documente, l'autre garde.

Seconde propriété : une cible qui prend un `int64` (ou un `uint32` d'heures et un `uint32` de
secondes), le ramène dans `[0, 1000h]` avec un modulo, arrondit à la seconde, puis vérifie
`Parse(Format(d)) == d`. Les types d'arguments du fuzzer ne sont pas limités aux chaînes : choisir
celui qui couvre naturellement l'espace des entrées.

## Défi — Fuzzer un décodeur de la bibliothèque standard

1–2. Avec un `name` contenant de l'UTF-8 invalide, `json.Marshal` remplace chaque octet invalide
par U+FFFD (`�`) : l'aller-retour échoue. Ce n'est **pas** un bug de `encoding/json`, c'est
son comportement documenté (« invalid UTF-8 is replaced by U+FFFD ») : la propriété doit soit
exclure les entrées invalides (`utf8.ValidString`), soit comparer à la version « assainie ». C'est
exactement le bug n° 2 du cours, côté bibliothèque standard.

3. `encoding/json/v2` **refuse** l'UTF-8 invalide par défaut : `Marshal` rend une erreur au lieu
de remplacer silencieusement (l'option `jsontext.AllowInvalidUTF8(true)` restaure l'ancien
comportement). La cible doit donc accepter l'erreur dans ce cas. Encore une différence de
comportement à connaître avant de migrer (N4 L6).

## Quiz

1. `func FuzzXxx(f *testing.F)`, avec un appel à `f.Fuzz(func(t *testing.T, …))`.
2. Elle exécute la fonction sur le corpus (`f.Add` et `testdata/fuzz`), comme un test ordinaire.
3. Il garde et re-mute les entrées qui font atteindre du code jamais exécuté.
4. Pas de panique, aller-retour, idempotence, équivalence avec une référence, invariants de sortie.
5. Il la minimise, l'écrit dans `testdata/fuzz/FuzzXxx/` et affiche la commande pour la rejouer.
6. Pour que chaque bug trouvé devienne un test permanent, rejoué par `go test`.
7. Le processus de test a été tué : souvent un épuisement mémoire, une boucle infinie ou un délai
   dépassé — souvent le bug le plus grave.
8. Tout ce qui analyse ou décode des données venues de l'extérieur.
9. Pour que l'entrée enregistrée reproduise l'échec à chaque fois.
10. Une petite entrée qui fait produire au décodeur une sortie énorme ; on s'en protège par une
    limite explicite de taille décodée.
