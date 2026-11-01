# Leçon 6 — Exercices

Code dans `mes-solutions/niveau-05/lecon-06/`. Installer l'outil de comparaison une fois :
`go install golang.org/x/perf/cmd/benchstat@latest`.

## Exercices faciles

**E1 — Le benchmark qui ne mesure rien.**
Écrire `func add(a, b int) int { return a + b }` et deux benchmarks : l'un avec
`for i := 0; i < b.N; i++ { add(i, 1) }`, l'autre avec `for b.Loop() { add(i, 1) }`. Comparer
les `ns/op`. Lequel est crédible ? Expliquer l'autre.

**E2 — Le bruit.**
Lancer cinq fois de suite un même benchmark avec `-count=1`. Noter l'écart entre la plus petite
et la plus grande valeur, en pourcentage. Recommencer pendant qu'une compilation lourde tourne à
côté (`go build std` en boucle). Conclusion ?

**E3 — `benchstat`.**
Reprendre `Builder` de l'exemple. Mesurer avec `-count=10` avant et après suppression du
`b.Grow(n)`. Passer les deux fichiers à `benchstat`. Recopier la sortie. Quelle métrique a la
plus petite variation (`±`) ? Pourquoi ?

**E4 — La préparation mesurée.**
Écrire un benchmark de `slices.Sort` sur un slice de 10 000 entiers aléatoires. Piège : après le
premier tour, le slice est **trié**, et l'on mesure le tri d'un slice déjà trié. Mesurer la
version naïve, puis une version correcte. Comment préparer une entrée **neuve** à chaque tour
sans mesurer la copie ? *(Lire `go doc testing.B.StopTimer` — et réfléchir à son coût.)*

**E5 — Allocations.**
Pour `fmt.Sprintf("%d", n)`, `strconv.Itoa(n)` et `strconv.AppendInt(buf[:0], int64(n), 10)`
(avec un `buf` réutilisé), mesurer `ns/op` et `allocs/op`. Classer. Dans quel contexte la
troisième forme vaut-elle sa complexité ?

---

## Exercice intermédiaire — Trancher une discussion de revue

Dans une *pull request*, un contributeur remplace :

```go
func Contains(xs []string, v string) bool { return slices.Contains(xs, v) }
```

par une version « optimisée » qui construit une `map[string]struct{}` à chaque appel, en arguant
qu'une recherche dans une map est en O(1).

1. Écrire les benchmarks des deux versions pour n = 5, 50, 500, 5 000 éléments, en cherchant un
   élément présent au milieu, puis un absent.
2. Comparer avec `benchstat`. À partir de quelle taille la map gagne-t-elle, si jamais ?
3. Ajouter une troisième version où la map est construite **une fois** et réutilisée pour 100
   recherches. Que change la conclusion ?
4. Rédiger le commentaire de revue (5 à 8 lignes), chiffres à l'appui, en suivant la méthode de
   N3 L6 : quoi, pourquoi, piste.

---

## Défi — Performances et complexité

1. Pour `Plus` (concaténation par `+=`), mesurer n = 100, 200, 400, 800, 1 600. Tracer (même à la
   main) `ns/op` en fonction de n. La courbe confirme-t-elle un coût quadratique ?
2. Mêmes mesures pour `Builder` : linéaire ?
3. Pourquoi `B/op` permet-il de conclure plus sûrement que `ns/op` ?
4. Trouver dans la bibliothèque standard un benchmark écrit avec `b.Loop` (`grep -rn "b.Loop()"
   $(go env GOROOT)/src/strings`) et un avec `b.N`. Qu'est-ce qui, dans le second, aurait été
   simplifié par `b.Loop` ?

---

## Quiz

1. Quelle est la forme d'un benchmark en Go 1.24+ ?
2. Pourquoi faut-il `-run '^$'` quand on lance les benchmarks ?
3. Que signifient les trois colonnes ajoutées par `-benchmem` ?
4. Quels sont les deux pièges de la forme `b.N` que `b.Loop` supprime ?
5. Qu'indique un résultat de 0,3 ns/op ?
6. Pourquoi une mesure unique ne prouve-t-elle rien ?
7. Que fait `benchstat`, et que signifie `~` dans sa sortie ?
8. Pourquoi mesurer plusieurs tailles d'entrée ?
9. Pourquoi `allocs/op` est-il souvent plus utile que `ns/op` ?
10. Dans quel ordre profilage, benchmark et optimisation doivent-ils venir ?
