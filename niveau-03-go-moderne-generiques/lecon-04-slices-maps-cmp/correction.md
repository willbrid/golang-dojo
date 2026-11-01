# Leçon 4 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Ce que voit l'ancien slice

Sortie mesurée (Go 1.27) :

```
[10 40 50] [10 40 50 0 0] [10 40]
[10 99 40 50] [10 99 40 50 0] 4 5
[1 2 3 10 99 40 50] [10 99 40 50 0] 7 10
```

Ligne par ligne :

1. `Delete(s, 1, 3)` décale `40 50` vers la gauche **dans le tableau de `s`**, puis met à
   zéro les deux dernières cases. `s` (longueur 5) voit donc `[10 40 50 0 0]`. `a` (longueur
   2) voit `[10 40]` : son `20` a été écrasé par le `40` décalé — `a` n'a rien demandé et a
   changé quand même.
2. `Insert(t, 1, 99)` : `t` a une longueur 3 et une capacité 5, il reste de la place.
   L'insertion se fait **dans le même tableau** ; `s` voit `[10 99 40 50 0]`.
3. `Insert(u, 0, 1, 2, 3)` : 4 + 3 = 7 > capacité 5. `Insert` alloue un **nouveau** tableau
   (capacité 10 ici). À partir de là, `v` est indépendant : `s` n'a pas bougé entre les lignes
   2 et 3.

La preuve par `cap` : tant que `cap(u)` reste 5 (celle de `s`), le tableau est partagé ; le
saut à 10 signe la réallocation. C'est exactement la règle d'`append`, qu'`Insert` utilise
en interne.

## E2 — Remplacer `sort`

```go
slices.SortFunc(people, func(a, b Person) int { return cmp.Compare(a.Age, b.Age) })
slices.Sort(names)
i, found := slices.BinarySearch(sorted, 42)
```

`sort.Slice` reçoit une fonction `func(i, j int) bool` qui travaille sur des **indices** dans
un slice capturé par la closure, et répond « strictement inférieur ? ». `slices.SortFunc`
reçoit `func(a, b T) int` qui travaille sur des **valeurs** et répond par un signe à trois
issues. Conséquences : pas de closure sur le slice (moins d'erreurs d'indice), et un
comparateur réutilisable pour `BinarySearchFunc`, `MaxFunc`, `CompareFunc`…

`slices.BinarySearch` rend en plus un booléen `found`, alors que `sort.SearchInts` obligeait
à vérifier `i < len(s) && s[i] == 42` soi-même.

## E3 — Le comparateur qui déborde

```
a - b      : [1 9223372036854775807 -9223372036854775808 -1]
cmp.Compare: [-9223372036854775808 -1 1 9223372036854775807]
```

Avec la soustraction, le tri est absurde. Deux comparaisons donnent un signe faux :

| Comparaison | Valeur mathématique | Valeur calculée en `int` | Conclusion du tri |
|---|---|---|---|
| `1 - math.MinInt` | +9 223 372 036 854 775 809 | **−9 223 372 036 854 775 807** | « 1 < MinInt » ✗ |
| `math.MinInt - math.MaxInt` | −18 446 744 073 709 551 615 | **+1** | « MinInt > MaxInt » ✗ |

Le résultat ne tient pas sur 64 bits : on garde les 64 bits de poids faible, et le bit de
signe bascule (complément à deux, niveau 1, leçon 2). Remarque : écrire ces calculs avec des
**constantes** ne compile pas (`constant … overflows int`) — il faut des variables pour voir le
débordement, comme toujours.

L'ordre exact obtenu dépend de l'algorithme de tri interne : il peut changer d'une version de
Go à l'autre. Un comparateur incohérent ne donne pas un résultat faux **stable**, il donne un
résultat **quelconque**.

## E4 — Dédoublonner

```go
func Unique[S ~[]E, E cmp.Ordered](s S) S {
	u := slices.Clone(s)
	slices.Sort(u)
	return slices.Compact(u)
}
```

`fmt.Printf("%T", Unique(Tags{"b", "a", "b"}))` affiche `main.Tags` : le `S ~[]E` a préservé
le type défini. Avec une signature `func Unique[E cmp.Ordered](s []E) []E`, on obtiendrait un
`[]string`, et l'appelant perdrait les méthodes éventuelles de `Tags`.

Le `Clone` est indispensable : `Sort` et `Compact` modifient en place.

## E5 — `maps.Clone` est superficiel

- `c["a"][0] = 99` → `m["a"]` vaut aussi `[99 2]` : les deux maps ont deux entrées distinctes
  qui contiennent **le même slice** (même pointeur vers le même tableau).
- `c["b"] = …` → `m` n'a pas de clé `"b"` : les **tables** sont distinctes.

```go
func DeepClone[K comparable, V any](m map[K][]V) map[K][]V {
	if m == nil {
		return nil
	}
	out := make(map[K][]V, len(m))
	for k, v := range m {
		out[k] = slices.Clone(v)
	}
	return out
}
```

Une copie « profonde » générique pour **tout** type n'existe pas en Go, et c'est voulu : la
profondeur dépend de la structure. Chaque niveau de pointeur se clone explicitement.

## Exercice intermédiaire — Classement d'un tournoi

```go
type Match struct {
	Home, Away           string
	HomeGoals, AwayGoals int
}

type Stats struct {
	Team                      string
	Played, Won, Drawn, Lost  int
	GoalsFor, GoalsAgainst    int
}

func (s *Stats) Diff() int   { return s.GoalsFor - s.GoalsAgainst }
func (s *Stats) Points() int { return 3*s.Won + s.Drawn }

func (s *Stats) record(gf, ga int) {
	s.Played++
	s.GoalsFor += gf
	s.GoalsAgainst += ga
	switch {
	case gf > ga:
		s.Won++
	case gf == ga:
		s.Drawn++
	default:
		s.Lost++
	}
}

func Standings(matches []Match) ([]*Stats, error) {
	table := map[string]*Stats{}
	get := func(team string) *Stats {
		s, ok := table[team]
		if !ok {
			s = &Stats{Team: team}
			table[team] = s
		}
		return s
	}
	for i, m := range matches {
		if m.Home == m.Away {
			return nil, fmt.Errorf("match %d : %q joue contre elle-même", i+1, m.Home)
		}
		get(m.Home).record(m.HomeGoals, m.AwayGoals)
		get(m.Away).record(m.AwayGoals, m.HomeGoals)
	}

	out := make([]*Stats, 0, len(table))
	for _, s := range table {
		out = append(out, s)
	}
	slices.SortFunc(out, func(a, b *Stats) int {
		return cmp.Or(
			cmp.Compare(b.Points(), a.Points()),
			cmp.Compare(b.Diff(), a.Diff()),
			cmp.Compare(b.GoalsFor, a.GoalsFor),
			cmp.Compare(a.Team, b.Team),
		)
	})
	return out, nil
}
```

Points de correction :

- **Déterminisme (contrainte 3).** La boucle sur la map produit un ordre aléatoire ; c'est le
  **dernier critère** (le nom, unique) qui rend l'ordre total. Sans lui, deux équipes à égalité
  parfaite sortiraient dans un ordre qui change d'une exécution à l'autre — et le test des 20
  exécutions afficherait plusieurs sorties. C'est la difficulté D6 du projet 1, résolue par
  la même idée.
- **Trois premiers critères décroissants, le dernier croissant** : arguments `(b, a)` puis `(a, b)`.
- **`get` en closure** : elle capture `table` et évite de répéter le « créer si absent ».
- **Le signe (contrainte 5).** `%+d` affiche `+6` et `-3`… mais aussi **`+0`**. Si l'on veut
  `0` sans signe, il faut un cas particulier. Lire la spécification de sortie jusqu'au bout,
  y compris le cas zéro.
- **Pourquoi `*Stats` dans la map** : avec `map[string]Stats`, `table[t].Won++` ne compile pas
  (une valeur de map n'est pas adressable — niveau 2, leçon 3).

## Défi — `GroupBy` et `TopN`

```go
func GroupBy[S ~[]E, E any, K comparable](s S, key func(E) K) map[K]S {
	out := make(map[K]S)
	for _, v := range s {
		k := key(v)
		out[k] = append(out[k], v)
	}
	return out
}

// Version 1 : O(n log n).
func TopNSort[S ~[]E, E any](s S, n int, cmp func(a, b E) int) S {
	if n <= 0 {
		return S{}
	}
	c := slices.Clone(s)
	slices.SortFunc(c, func(a, b E) int { return cmp(b, a) }) // décroissant
	return c[:min(n, len(c))]
}

// Version 2 : O(len(s) · n) dans le pire cas, O(len(s) · log n) comparaisons.
func TopN[S ~[]E, E any](s S, n int, cmp func(a, b E) int) S {
	if n <= 0 {
		return S{}
	}
	best := make(S, 0, n+1)
	desc := func(a, b E) int { return cmp(b, a) }
	for _, v := range s {
		if len(best) == n && cmp(v, best[n-1]) <= 0 {
			continue // pas meilleur que le plus petit des n retenus
		}
		i, _ := slices.BinarySearchFunc(best, v, desc)
		best = slices.Insert(best, i, v)
		if len(best) > n {
			best = best[:n]
		}
	}
	return best
}
```

- `GroupBy` garde l'ordre d'origine parce qu'`append` ajoute en fin et qu'on parcourt `s` dans
  l'ordre.
- `TopNSort` retourne `c[:n]` : le reste du tableau cloné reste en mémoire tant que le résultat
  vit. Pour un gros slice, `slices.Clip` ou un `Clone` du résultat le libère.
- `TopN` : le test `cmp(v, best[n-1]) <= 0` élimine l'immense majorité des éléments en **une**
  comparaison. Sur un million d'entiers aléatoires et `n = 10`, la machine de rédaction mesure
  environ 170 ms contre 3 ms, un facteur ~60. La mesure dépend de la machine : c'est **ta**
  mesure qui compte, pas ce chiffre.
- `min(n, len(c))` : la fonction prédéclarée `min` (Go 1.21).

*La structure de données idéale pour ce problème est un tas (`container/heap`) — hors
programme ici, mais c'est la réponse qu'on attendrait en entretien.*

## Quiz

1. Pour **préserver le type défini** de l'appelant : `slices.Clone(ids)` rend un `IDs`, pas un `[]int`.
2. Elles sont mises à **zéro**, pour que le ramasse-miettes puisse libérer ce qu'elles
   référençaient.
3. Négatif si `a` avant `b`, zéro si équivalents, positif si `a` après `b`.
4. La soustraction peut déborder et inverser le signe.
5. `cmp.Or(cmp.Compare(…), cmp.Compare(…), cmp.Compare(…))`.
6. `SortStableFunc` garde l'ordre relatif des éléments équivalents. Inutile quand le dernier
   critère rend l'ordre total (aucun élément équivalent).
7. Rien : `[1 2 1]`. Aucun doublon n'est adjacent.
8. La position d'insertion qui garderait le slice trié, et `false`.
9. Comparaison d'octets : `'Z'` vaut 90, `'a'` vaut 97.
10. Superficielle. `maps.Clone(nil)` rend `nil`.
