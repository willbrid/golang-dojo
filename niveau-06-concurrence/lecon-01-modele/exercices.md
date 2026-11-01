# Leçon 1 — Exercices

Code dans `mes-solutions/niveau-06/lecon-01/`. **Tous** les programmes de ce niveau se lancent
aussi avec `-race` (`go run -race .`, `go test -race ./...`).

## Exercices faciles

**E1 — `main` n'attend pas.**
Écrire un `main` qui lance 5 goroutines affichant chacune son numéro, puis se termine. Lancer
10 fois (`for i in $(seq 10); do go run .; echo ---; done`). Combien de lignes à chaque fois ?
Corriger avec un `sync.WaitGroup` et `wg.Go`. L'ordre est-il stable ?

**E2 — La race, mesurée.**
Faire calculer `0 + 1 + … + 999` par 1 000 goroutines qui ajoutent chacune `i` à une variable
`total` partagée. Afficher le résultat cinq fois de suite : est-il toujours `499500` ? Lancer avec
`-race` et recopier les **trois premières lignes** du rapport. Corriger **sans** verrou (indice :
l'exemple du cours).

**E3 — Combien coûte une goroutine ?**
Lancer 100 000 goroutines qui restent bloquées (sur un channel jamais alimenté — anticipation de
la leçon 2 : `stop := make(chan struct{})` puis `<-stop` dans la goroutine, `close(stop)` à la
fin). Mesurer avec `runtime.ReadMemStats` (champ `StackInuse`) la pile moyenne par goroutine, et
le temps de création. Comparer avec la pile par défaut d'un thread sur ton système (`ulimit -s`).

**E4 — Concurrence sans parallélisme.**
Relancer l'exemple du cours avec `GOMAXPROCS=1`, `2`, `4`. Tracer le gain en fonction de
`GOMAXPROCS`. Pourquoi le programme reste-t-il **correct** (même nombre de premiers) dans tous les
cas ?

**E5 — Les arguments évalués au `go`.**
Prédire, puis vérifier :

```go
x := 1
var wg sync.WaitGroup
wg.Add(2)
go func(v int) { defer wg.Done(); fmt.Println("argument :", v) }(x)
go func() { defer wg.Done(); fmt.Println("capture :", x) }()
x = 2
wg.Wait()
```

Le second affichage est-il déterministe ? Que dit `-race` ?

---

## Exercice intermédiaire — `wordstat` en parallèle

Reprendre le projet 1 : traiter les fichiers passés en arguments **en parallèle**, une goroutine
par fichier.

**Contraintes :**
1. Chaque goroutine produit **ses propres** statistiques ; la fusion se fait **après** `Wait`, dans
   la goroutine principale. Aucune donnée partagée en écriture.
2. La sortie est **identique** à la version séquentielle (ordre des erreurs compris — les erreurs
   sont affichées dans l'ordre des arguments, pas dans l'ordre d'arrivée).
3. `go run -race .` silencieux sur 20 fichiers.
4. Mesurer sur 20 gros fichiers : quel gain ? Le programme est-il limité par le **calcul** ou par
   la **lecture disque** ? Comment le vérifier ?
5. Que se passe-t-il avec 10 000 fichiers ? (Penser aux descripteurs de fichiers ouverts
   simultanément, `ulimit -n`.) Proposer une limite — la leçon 9 donnera le patron propre.

---

## Défi — Le déséquilibre de charge

Dans l'exemple du cours, 64 goroutines font mieux que 8, alors que la machine n'a que 8 cœurs
logiques.

1. Instrumenter `parallel` pour mesurer la **durée de chaque goroutine** (avec 8 tranches). Quelle
   est la plus lente ? Pourquoi ?
2. Proposer un découpage qui équilibre la charge **sans** augmenter le nombre de goroutines.
   (Piste : les tranches n'ont pas besoin d'être contiguës.)
3. Mesurer. Atteins-tu le ×5,5 de la version à 64 goroutines ?
4. Conclusion en trois lignes : pourquoi « beaucoup de petites tâches » est-il souvent plus simple
   qu'un découpage savant ?

---

## Quiz

1. Que fait `go f(x)` ? Quand `x` est-il évalué ?
2. Pourquoi peut-on lancer des centaines de milliers de goroutines, mais pas de threads ?
3. Que désignent G, M et P ?
4. Que se passe-t-il quand une goroutine bloque sur une lecture réseau ?
5. Qu'est-ce que `GOMAXPROCS`, et quelle est sa valeur par défaut depuis Go 1.25 dans un conteneur ?
6. Différence entre concurrence et parallélisme ?
7. Que deviennent les goroutines quand `main` se termine ?
8. Pourquoi `time.Sleep` n'est-il pas un moyen d'attendre une goroutine ?
9. Qu'est-ce qu'une *data race* ? Comment la détecter ?
10. Quelle question doit-on se poser avant chaque `go` ?
