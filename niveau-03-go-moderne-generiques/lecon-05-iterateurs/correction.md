# Leçon 5 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Le corps devient `yield`

```go
func Range(lo, hi int) iter.Seq[int] {
	return func(yield func(int) bool) {
		fmt.Println("début")
		for i := lo; i < hi; i++ {
			if !yield(i) {
				fmt.Println("fin")
				return
			}
		}
		fmt.Println("fin")
	}
}
```

Sortie :

```
début
0
1
fin
```

- `début` s'affiche **au premier tour** de boucle, pas à l'appel de `Range(0, 10)` : l'appel
  ne fait que fabriquer une fonction.
- `0` et `1` sont affichés **par le corps de la boucle**, exécuté *à l'intérieur* de `yield`.
- Au tour `i == 2`, le `break` fait rendre `false` à `yield` ; l'itérateur passe dans sa
  branche d'arrêt et affiche `fin`.

Tout s'exécute dans la même pile d'appels : la boucle appelle l'itérateur, qui appelle le
corps. C'est la meilleure façon de comprendre « le corps devient `yield` ».

## E2 — Le contrat violé

Sans `break` : **rien d'anormal**. `yield` rend toujours `true`, l'itérateur l'ignore sans
conséquence. Avec `break` :

```
panic: runtime error: range function continued iteration after function for loop body returned false
```

Le runtime ne peut détecter que ce qu'il observe : un appel à `yield` **après** que celui-ci
a rendu `false`. Tant que le consommateur ne s'arrête jamais, le bug est invisible. C'est un
bug **latent** — il passe tous les essais qui parcourent la collection en entier, et éclate
chez le premier utilisateur qui fait `break`. D'où le réflexe `if !yield(v) { return }`
systématique, même quand « personne ne fera de break ».

## E3 — Clés triées

```go
keys := slices.Sorted(maps.Keys(m))

vals := slices.SortedFunc(maps.Values(m), func(a, b int) int { return cmp.Compare(b, a) })
```

## E4 — `Seq2`

```go
func Enumerate[V any](seq iter.Seq[V]) iter.Seq2[int, V] {
	return func(yield func(int, V) bool) {
		i := 0
		for v := range seq {
			if !yield(i, v) {
				return
			}
			i++
		}
	}
}
```

`strings.FieldsSeq` ignore les espaces multiples et ceux des bords, comme `strings.Fields`.

## E5 — Usage unique

Second parcours : **zéro** mot. Le `bufio.Scanner` a lu le flux jusqu'au bout ; un
`io.Reader` ne se rembobine pas. L'itérateur n'a aucun moyen de « recommencer », et aucune
erreur n'est signalée.

L'utilisateur ne peut le savoir que par la **documentation** : le commentaire de `Words` doit
le dire (« usage unique : le flux est consommé »). Une API honnête peut aussi rendre ce
caractère visible dans le nom du constructeur ou faire prendre un `io.Reader` à la méthode
elle-même, pour qu'il soit évident qu'un second appel exige un second flux.

## Exercice intermédiaire — `logscan`

```go
// Entries itère sur les entrées du journal. Usage unique : r est consommé.
// Une ligne mal formée produit (Entry{}, err) puis l'itération continue ;
// une erreur de lecture du flux est produite une dernière fois et termine l'itération.
func Entries(r io.Reader) iter.Seq2[Entry, error] {
	return func(yield func(Entry, error) bool) {
		sc := bufio.NewScanner(r)
		for n := 1; sc.Scan(); n++ {
			e, err := parseLine(sc.Text())
			if err != nil {
				err = fmt.Errorf("ligne %d : %w", n, err)
			}
			if !yield(e, err) {
				return
			}
		}
		if err := sc.Err(); err != nil {
			yield(Entry{}, fmt.Errorf("lecture : %w", err))
		}
	}
}

func parseLine(line string) (Entry, error) {
	f := strings.Fields(line)
	if len(f) < 4 {
		return Entry{}, fmt.Errorf("%q : 4 champs attendus, %d trouvés", line, len(f))
	}
	return Entry{
		Time:    f[0],
		Level:   f[1],
		Service: f[2],
		Message: strings.Join(f[3:], " "),
	}, nil
}

func ByLevel(seq iter.Seq2[Entry, error], level string) iter.Seq2[Entry, error] {
	return func(yield func(Entry, error) bool) {
		for e, err := range seq {
			if err != nil || e.Level == level {
				if !yield(e, err) {
					return
				}
			}
		}
	}
}
```

Côté `main` :

```go
counts := map[string]int{}
bad, errorsSeen := 0, 0
for e, err := range Entries(os.Stdin) {
	if err != nil {
		bad++
		fmt.Fprintln(os.Stderr, err)
		continue
	}
	counts[e.Level]++
	if e.Level == "ERROR" {
		fmt.Println(e.Time, e.Service, e.Message)
		errorsSeen++
		if errorsSeen == 5 {
			break
		}
	}
}
for _, lvl := range slices.Sorted(maps.Keys(counts)) {
	fmt.Printf("%-5s %d\n", lvl, counts[lvl])
}
fmt.Println("mal formées :", bad)
```

Points de correction :

- **Mémoire bornée** : une ligne à la fois. Attention à la limite par défaut de
  `bufio.Scanner` (64 Kio par ligne) — difficulté D5 du projet 1. Une ligne plus longue fait
  échouer `Scan` avec `bufio.ErrTooLong`, que notre conception remonte proprement comme
  erreur de lecture finale.
- **Le dernier `yield` sans `if`** : c'est le dernier appel, il n'y a rien à faire ensuite —
  ignorer le résultat est sans risque **parce qu'aucun `yield` ne suit**. C'est la seule
  exception tolérable au réflexe.
- **Arrêt au 5ᵉ `ERROR`** : le `break` fait rendre `false` à `yield`, `Entries` fait `return`,
  et `sc.Scan()` n'est plus jamais appelé. Le compteur le prouve : sur un fichier de 1 000
  lignes dont le 5ᵉ `ERROR` est ligne 40, `Entries` a lu 40 lignes.
- **Question de conception (style `Err()`)** : `func Entries(r io.Reader) *Scanner` avec
  `All() iter.Seq[Entry]` et `Err() error`. Mais `Err()` ne rapporte qu'**une** erreur, à la
  fin : les lignes mal formées devraient soit arrêter l'itération (contraire à l'énoncé), soit
  être collectées dans un `[]error` exposé par une autre méthode (`Malformed()`), soit être
  silencieusement sautées. **Conclusion** : `Seq2[V, error]` convient quand des erreurs
  **non fatales** jalonnent l'itération ; le style `Err()` convient quand seule une erreur
  **fatale** est possible (cas de `bufio.Scanner`).

## Défi — Fusionner deux itérateurs triés

1. Un `range` **cède le contrôle** à l'itérateur jusqu'à la fin de la boucle. Pour fusionner,
   il faut avancer tantôt `a`, tantôt `b`, selon la comparaison — donc **tirer** une valeur de
   l'un sans avancer l'autre. Deux boucles imbriquées parcourraient `b` en entier pour chaque
   élément de `a` ; deux boucles successives épuiseraient `a` avant de toucher `b`. Il faut
   des itérateurs **pull**.

2. ```go
   func Merge[V cmp.Ordered](a, b iter.Seq[V]) iter.Seq[V] {
   	return func(yield func(V) bool) {
   		nextA, stopA := iter.Pull(a)
   		defer stopA()
   		nextB, stopB := iter.Pull(b)
   		defer stopB()

   		va, okA := nextA()
   		vb, okB := nextB()
   		for okA || okB {
   			if !okB || (okA && cmp.Compare(va, vb) <= 0) {
   				if !yield(va) {
   					return
   				}
   				va, okA = nextA()
   			} else {
   				if !yield(vb) {
   					return
   				}
   				vb, okB = nextB()
   			}
   		}
   	}
   }
   ```

   Les `stop()` sont en `defer` **à l'intérieur** de la fonction itérateur, juste après chaque
   `Pull` : si le consommateur fait `break`, `yield` rend `false`, on fait `return`, et les deux
   `defer` libèrent les itérateurs sous-jacents. Les placer dans `Merge` lui-même (hors de la
   fonction rendue) serait faux : ils s'exécuteraient avant même le premier tour.

3. `[1 2 3 4 9 10 11]`.

4. `[0 0 3 5 6 9 10 12 15 15]`. Ça fonctionne avec des itérateurs **infinis** parce que
   `Merge` ne tire qu'une valeur à la fois de chaque côté. Aucune version à base de slices ne
   pourrait le faire.

## Quiz

1. `type Seq[V any] func(yield func(V) bool)`.
2. La fonction `yield` passée à l'itérateur ; `continue` y devient `return true`, `break`
   devient `return false`.
3. S'arrêter et ne plus appeler `yield`. Sinon : panique à l'exécution.
4. Un `Seq` ne produit qu'une valeur par tour ; deux variables exigent un `Seq2`.
5. `slices.Sorted(maps.Keys(m))`.
6. Aucun slice intermédiaire alloué, et arrêt possible avant la fin.
7. `Seq2[V, error]` (tester `err` à chaque tour, sinon on l'ignore) ; méthode `Err()` après la
   boucle (l'oublier rend l'erreur silencieuse).
8. Un itérateur qui ne peut être parcouru qu'une fois parce qu'il consomme une ressource :
   un itérateur sur un `io.Reader`, un `bufio.Scanner`.
9. Convertir un itérateur push en fonctions `next`/`stop` pour tirer les valeurs une à une ;
   il faut **toujours** appeler `stop`.
10. Quand la collection est petite, déjà en mémoire, et sera parcourue plusieurs fois ou
    indexée.
