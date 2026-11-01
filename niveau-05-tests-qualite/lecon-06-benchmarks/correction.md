# Leçon 6 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé. Tous les chiffres ci-dessous ont été mesurés sur la
> machine de rédaction (Intel i5-8265U) ; les tiens seront différents — ce sont les **rapports**
> et les **tendances** qui comptent.

## E1 — Le benchmark qui ne mesure rien

```
BenchmarkN-8      1000000000    0.2828 ns/op
BenchmarkLoop-8    852537613    1.409 ns/op
```

0,28 ns, c'est **un seul cycle** d'horloge environ : impossible pour un appel de fonction suivi
d'une addition. Le compilateur a **intégré** `add` dans la boucle (*inlining*), constaté que le
résultat n'était jamais utilisé, et **supprimé** le calcul. On mesure une boucle vide. `b.Loop`
garde les appels de son corps vivants : 1,4 ns est une vraie mesure (dérisoire, mais réelle).

## E2 — Le bruit

Écart typique au repos : 5 à 20 % (18 % mesurés dans le cours, sans aucune modification). Sous
charge, les mesures deviennent inexploitables : 50 % d'écart ou plus, et des valeurs aberrantes.
Conclusion : une comparaison n'a de sens que **dans les mêmes conditions**, avec **plusieurs**
mesures, et un outil statistique pour dire si l'écart dépasse le bruit.

## E3 — `benchstat`

Forme de la sortie (les valeurs dépendent de ta machine) :

```
                     │  avant.txt  │              apres.txt               │
                     │   sec/op    │   sec/op     vs base                 │
Join/builder/n=1000-8   6.0µ ± 4%    9.5µ ± 3%   +58.33% (p=0.000 n=10)
                     │    B/op     │     B/op      vs base                │
Join/builder/n=1000-8  6.750Ki ± 0%  24.24Ki ± 0%  +259.15% (p=0.000 n=10)
                     │  allocs/op  │  allocs/op    vs base                │
Join/builder/n=1000-8   1.000 ± 0%   14.000 ± 0%  +1300.00% (p=0.000 n=10)
```

`B/op` et `allocs/op` ont une variation de **± 0 %** : ils sont **déterministes** — le même code
sur les mêmes données alloue exactement la même chose à chaque fois. Le temps, lui, dépend de la
machine, du cache, de la fréquence du processeur. C'est pourquoi les allocations sont souvent
l'indicateur le plus fiable d'une optimisation.

## E4 — La préparation mesurée

```
BenchmarkSortNaive-8   160711      8464 ns/op   ← trie un slice DÉJÀ trié
BenchmarkSortFresh-8     2164    550475 ns/op   ← copie + tri d'une entrée neuve
BenchmarkCopyOnly-8    845617      1425 ns/op   ← la copie seule
```

La version naïve est **65 fois** « plus rapide » : elle mesure le meilleur cas de l'algorithme
(pdqsort détecte une entrée triée en un passage). Version correcte :

```go
func BenchmarkSort(b *testing.B) {
	src := randomInts(10_000)
	s := make([]int, len(src))
	for b.Loop() {
		copy(s, src)        // ~1,4 µs, mesuré séparément
		slices.Sort(s)      // ~549 µs
	}
}
```

La copie est mesurée avec le tri, mais elle pèse 0,3 % du total : on l'accepte, et on le **sait**
grâce à `BenchmarkCopyOnly`. `b.StopTimer()`/`b.StartTimer()` autour de la copie semble plus
propre, mais ces appels ont eux-mêmes un coût et perturbent la mesure quand le corps est court ;
la documentation de `testing` le déconseille dans les boucles serrées.

## E5 — Allocations

```
BenchmarkSprintf-8     85.18 ns/op   16 B/op   2 allocs/op
BenchmarkItoa-8        21.88 ns/op    8 B/op   1 allocs/op
BenchmarkAppendInt-8   14.16 ns/op    0 B/op   0 allocs/op
```

`AppendInt` dans un buffer réutilisé n'alloue **rien**. Il vaut sa complexité dans une boucle
**chaude** qui produit beaucoup de texte : un encodeur (JSON, journal, protocole réseau) qui
écrit des millions de nombres par seconde. Partout ailleurs, `strconv.Itoa` est plus lisible et
la différence (7 ns) n'a aucune importance. `fmt.Sprintf` reste le bon choix pour du **formatage**
(largeur, remplissage, plusieurs valeurs).

## Exercice intermédiaire — Trancher une discussion de revue

Mesures, élément présent au milieu :

| n | `slices.Contains` | map construite à chaque appel |
|---|---|---|
| 5 | 11 ns, 0 alloc | 88 ns, 0 alloc |
| 50 | 78 ns, 0 alloc | 1 259 ns, 3 allocs |
| 500 | 517 ns, 0 alloc | 10 722 ns, 3 allocs |
| 5 000 | 4 875 ns, 0 alloc | 124 033 ns, 17 allocs |

La map **ne gagne jamais** : elle est 8 à 25 fois plus lente à toutes les tailles. L'argument
« O(1) contre O(n) » oublie que **construire** la map coûte O(n) — avec des allocations et du
hachage pour chaque élément — à chaque appel. On remplace un parcours par… un parcours plus cher,
suivi d'une recherche.

Avec une map construite **une fois** et 100 recherches, le calcul change : la construction est
amortie, chaque recherche coûte quelques dizaines de nanosecondes quel que soit n, et la map gagne
dès que n dépasse quelques dizaines. La bonne question n'était pas « map ou slice ? » mais
« combien de recherches par construction ? ».

Commentaire de revue :

> Merci pour la proposition. Mesuré avec `-count=10` et `benchstat` (sortie jointe), la version
> map est 8× à 25× **plus lente** pour n = 5 à 5 000, et alloue (jusqu'à 17 allocations par
> appel) : la construction de la map à chaque appel est elle-même en O(n), et bien plus chère
> que le parcours. Je propose de garder `slices.Contains`. Si un appelant fait beaucoup de
> recherches sur la même liste, construire un ensemble **une fois** côté appelant serait la
> bonne optimisation — avec un benchmark de ce cas d'usage précis.

## Défi — Performances et complexité

1. Pour `Plus`, quand n double, le temps est multiplié par ~3,5 à 4 et `B/op` par ~4 : courbe
   quadratique, comme mesuré au niveau 4 (leçon 1, E4) avec `time.Since`.
2. `Builder` : temps et octets doublent avec n — linéaire.
3. `B/op` est déterministe : sa progression (×4 quand n double) se lit sans ambiguïté sur deux
   mesures, alors que le temps, bruité, demande des répétitions et des statistiques.
4. Dans le source de la stdlib, les benchmarks récents utilisent `b.Loop` ; les anciens écrivent
   `for i := 0; i < b.N; i++` précédé de la préparation et d'un `b.ResetTimer()`, avec parfois une
   variable globale « puits » (`var sink string`) pour empêcher l'élimination du résultat. `b.Loop`
   supprime le `ResetTimer` **et** le puits.

## Quiz

1. `func BenchmarkXxx(b *testing.B) { for b.Loop() { … } }`.
2. Pour ne pas lancer les tests en même temps (aucun nom ne correspond à l'expression vide).
3. Temps par opération, octets alloués par opération, nombre d'allocations par opération.
4. La préparation mesurée (oubli de `ResetTimer`) et l'élimination du calcul par le compilateur.
5. Que le compilateur a supprimé le code mesuré : on mesure une boucle vide.
6. Parce que le bruit de mesure (10–20 %) dépasse souvent l'effet qu'on croit observer.
7. Il compare statistiquement des séries de mesures ; `~` signifie qu'aucune différence
   significative n'est démontrée.
8. Parce que les complexités se révèlent avec la taille : une version plus lente sur petites
   entrées peut être bien plus rapide sur grandes.
9. Parce qu'il est déterministe et qu'il explique souvent le temps (allocation + ramasse-miettes).
10. Profiler le programme réel, puis écrire un benchmark de la partie coûteuse, puis optimiser en
    mesurant avant/après.
