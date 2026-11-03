# Leçon 1 — Exercices

Code dans `mes-solutions/niveau-11/lecon-01/`.

## Exercices faciles

**E1 — Ton bruit.**
Mesurer `BenchmarkJoin` avec `-count 20` sur ta machine, deux fois : (a) au repos ; (b) en lançant en même
temps une compilation lourde (`go build` d'un gros projet en boucle) ou un navigateur occupé. Calculer
médiane et étendue pour chaque cas. Le test de rang conclut-il à une différence ?

**E2 — Le compilateur malin.**
Écrire `func Square(x int) int { return x * x }` et le benchmark à l'ancienne
`for i := 0; i < b.N; i++ { Square(i) }`. Que vaut `ns/op` ? Puis avec `b.Loop`. Puis en affectant le
résultat à une variable globale. Expliquer les écarts (`go test -gcflags=-m` pour l'*inlining*).

**E3 — Les tailles.**
Mesurer les trois versions de la leçon pour 10, 100, 1 000 et 10 000 éléments (`b.Run` avec des
sous-benchmarks). Comment évolue `Concat` ? Pourquoi ? Pour quelle taille `Builder` et `Join` deviennent-ils
indiscernables ?

**E4 — `Builder` préalloué.**
Ajouter `b.Grow(n)` avec la taille totale calculée d'avance dans `Builder`. Mesurer, comparer avec `Join`
par le test de rang. La différence est-elle démontrée ?

**E5 — Une p-valeur trompeuse.**
Comparer `Join` à lui-même vingt fois (vingt paires de séries de 10). Combien de comparaisons donnent
p < 0,05 ? Qu'est-ce que cela dit de la règle « p < 0,05 = différence réelle » quand on compare beaucoup de
benchmarks à la fois ?

---

## Exercice intermédiaire — Un mini `benchstat`

Transformer le comparateur de l'exemple en commande `cmpbench avant.txt apres.txt` :

1. Lire la sortie de `go test -bench` (lignes `BenchmarkNom-N  itérations  valeur unité  valeur unité…`),
   regrouper par nom de benchmark et par unité (`ns/op`, `B/op`, `allocs/op`, et les métriques
   personnalisées de `b.ReportMetric`).
2. Pour chaque benchmark et chaque unité : médiane avant, médiane après, variation en %, p-valeur, `~` si
   p > 0,05 ou si l'une des séries a moins de 5 mesures (avertissement).
3. Une ligne finale « moyenne géométrique » des variations significatives.
4. Tests : analyse d'une sortie réelle (fichier de référence), cas des égalités (allocations identiques
   partout : p doit valoir 1), séries de tailles différentes.
5. Comparer tes résultats avec ceux de `benchstat` si tu peux l'installer.

---

## Défi — Mesurer de bout en bout

Pour le service de prêts du niveau 10 (ou ton projet 3/4) :

1. Écrire un générateur de charge en Go : N goroutines, chacune envoie des requêtes en boucle pendant une
   durée donnée ; mesurer chaque durée.
2. Afficher le débit (requêtes/s) et les percentiles p50, p90, p99, p99,9 (sans histogramme : trier les
   durées).
3. Remplacer dans le service un `strings.Join` par la version `+=` (sur une liste réaliste) et mesurer :
   l'écart se voit-il de bout en bout ? À partir de quelle taille de liste ?
4. Conclure en trois lignes sur ce que le micro-benchmark annonçait et ce que la mesure de bout en bout a
   montré.

---

## Quiz

1. Que fait `b.Loop` que `for i := 0; i < b.N; i++` ne fait pas ?
2. Que signifient `ns/op`, `B/op` et `allocs/op` ? Lesquels sont déterministes ?
3. Citer trois sources de bruit dans un benchmark.
4. Pourquoi résumer par la médiane plutôt que par la moyenne ?
5. Que teste le test de Mann-Whitney, et que signifie sa p-valeur ?
6. Que conclure quand p > 0,05 ?
7. Pourquoi `strings.Join` n'alloue-t-il qu'une fois ?
8. Qu'est-ce que `benchstat`, et comment l'utilise-t-on ?
9. Qu'est-ce qu'un micro-benchmark ne dit pas ?
10. Pourquoi faut-il se méfier des p-valeurs quand on compare des dizaines de benchmarks ?
