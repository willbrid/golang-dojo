# Leçon 1 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — `main` n'attend pas

Sans attente, on obtient le plus souvent **zéro** ligne, parfois une ou deux : `main` se termine
avant que l'ordonnanceur ait exécuté les goroutines. Corrigé :

```go
var wg sync.WaitGroup
for i := range 5 {
	wg.Go(func() { fmt.Println("goroutine", i) })
}
wg.Wait()
```

Cinq lignes à chaque fois, dans un ordre **qui change**. Il n'y a rien à corriger : sans
synchronisation, l'ordre n'existe pas.

## E2 — La race, mesurée

Résultats typiques : `422979`, `487211`, `499500`, `461840`… Le résultat juste apparaît parfois —
c'est ce qui rend ce bug dangereux. `total += i` est en réalité trois opérations (lire, ajouter,
écrire) : deux goroutines peuvent lire la même ancienne valeur, et l'une des deux additions est
perdue. Le rapport commence par :

```
==================
WARNING: DATA RACE
Write at 0x00c00011c018 by goroutine 100018:
```

suivi de la pile de l'écriture, puis de l'accès concurrent (`Previous write at … by goroutine …`)
et des endroits où les deux goroutines ont été créées. Correction sans verrou : une case par
goroutine.

```go
parts := make([]int, 1000)
var wg sync.WaitGroup
for i := range 1000 {
	wg.Go(func() { parts[i] = i })
}
wg.Wait()
total := 0
for _, p := range parts {
	total += p
}
```

Des goroutines qui écrivent dans des cases **différentes** d'un même slice ne sont pas en
conflit : chaque case est une variable distincte. (Les leçons 6 et 7 montreront `Mutex` et
`atomic` pour le cas où l'on ne peut pas séparer.)

## E3 — Combien coûte une goroutine ?

Mesuré sur la machine de rédaction pour 100 000 goroutines bloquées :

| Mesure | Valeur |
|---|---|
| pile par goroutine (`StackInuse` / n) | **2 048 octets** |
| mémoire totale demandée au système par goroutine (`Sys` / n) | ~2,7 Kio |
| temps de création (avec la synchronisation de l'exercice) | ~235 ms pour 100 000, soit ~2,3 µs chacune |

`ulimit -s` affiche généralement `8192` (Kio) : la pile par défaut d'un thread est de **8 Mio**,
4 000 fois plus. 100 000 threads réserveraient 800 Gio d'espace d'adressage ; 100 000 goroutines
occupent environ 270 Mio. C'est toute la différence.

## E4 — Concurrence sans parallélisme

Le gain suit approximativement `GOMAXPROCS` (×1, ×~1,8, ×~3,2), plafonné par le nombre de cœurs
physiques. Le résultat reste **correct** dans tous les cas parce que le programme est bien
synchronisé : chaque goroutine écrit sa case, et la lecture n'a lieu qu'après `Wait`. Un programme
correct l'est quel que soit le degré de parallélisme ; un programme dont le résultat **change** avec
`GOMAXPROCS` a un bug de synchronisation. (Lancer ses tests avec `GOMAXPROCS=1` et avec le maximum
est d'ailleurs une bonne pratique pour en débusquer.)

## E5 — Les arguments évalués au `go`

`argument : 1` est **déterministe** : `x` est évalué au moment du `go`, avant `x = 2`. `capture :`
peut afficher `1` ou `2` : la closure lit la **variable** `x` au moment où elle s'exécute. Et
surtout, `-race` signale une **data race** : la goroutine principale écrit `x` pendant que la
seconde goroutine le lit, sans synchronisation. La version « argument » est le moyen le plus simple
de donner à une goroutine une **copie** stable d'une valeur.

## Exercice intermédiaire — `wordstat` en parallèle

```go
type result struct {
	stats Stats
	err   error
}

results := make([]result, len(files))     // une case par fichier
var wg sync.WaitGroup
for i, path := range files {
	wg.Go(func() {
		results[i].stats, results[i].err = countFile(path)
	})
}
wg.Wait()

var total Stats
code := 0
for i, r := range results {               // ordre des arguments, pas d'arrivée
	if r.err != nil {
		fmt.Fprintf(stderr, "%s : %v\n", files[i], r.err)
		code = 1
		continue
	}
	total.Merge(r.stats)
}
```

- La sortie est identique à la version séquentielle parce que l'ordre de **traitement** (parallèle)
  est découplé de l'ordre de **présentation** (la boucle finale sur `results`).
- **Calcul ou disque ?** Lancer deux fois de suite : la seconde fois, les fichiers sont dans le
  cache du système, et si le gain augmente nettement, c'est que la première exécution était limitée
  par le disque. `time` (commande shell) aide aussi : si `user` est bien supérieur à `real`, plusieurs
  cœurs calculaient ; s'il est proche, le programme attendait.
- **10 000 fichiers** : 10 000 fichiers ouverts en même temps dépassent souvent la limite de
  descripteurs (`too many open files`, `ulimit -n` vaut souvent 1 024). Il faut **borner** le nombre
  de goroutines actives : un *worker pool* ou un sémaphore (leçon 9).

## Défi — Le déséquilibre de charge

1. Avec 8 tranches contiguës, la dernière (`[2 625 000, 3 000 000)`) est la plus lente : tester un
   nombre `x` coûte jusqu'à √x divisions, et ses nombres sont les plus grands. Les premières
   tranches finissent bien avant, et leurs cœurs restent **inactifs** pendant que la dernière
   termine.
2. Découpage **entrelacé** : la goroutine `k` traite tous les `x` tels que `x % parts == k` (ou des
   blocs de 1 000 nombres distribués en tourniquet). Chaque goroutine reçoit alors un mélange de
   petits et de grands nombres.
3. On approche du ×5,5 avec 8 goroutines seulement.
4. Beaucoup de petites tâches laissent l'**ordonnanceur** équilibrer (vol de travail) sans
   connaître la forme du coût ; un découpage savant suppose de connaître cette forme, et casse dès
   qu'elle change. Le coût d'une goroutine étant minuscule, la solution simple est généralement la
   bonne — tant que chaque tâche reste nettement plus coûteuse que sa création.

## Quiz

1. Elle exécute `f(x)` dans une nouvelle goroutine et continue sans attendre ; `x` est évalué
   immédiatement, dans la goroutine appelante.
2. Une goroutine démarre avec une pile de quelques kilo-octets qui grandit à la demande, et ses
   changements de contexte sont gérés par le runtime ; un thread réserve ~8 Mio et passe par le noyau.
3. G : goroutine ; M : thread du système ; P : contexte d'exécution, au nombre de `GOMAXPROCS`.
4. L'ordonnanceur la met en attente et exécute une autre goroutine sur le même P ; elle reprend
   quand les données arrivent.
5. Le nombre de goroutines qui exécutent du Go simultanément ; depuis Go 1.25, la limite de CPU du
   conteneur si elle est inférieure au nombre de cœurs.
6. La concurrence structure en tâches indépendantes ; le parallélisme exécute réellement en même
   temps sur plusieurs cœurs.
7. Elles sont arrêtées net, sans exécuter leurs `defer`.
8. C'est un pari sur une durée, pas une synchronisation : trop court, il échoue ; trop long, il
   gaspille du temps.
9. Deux accès concurrents à la même variable, dont au moins une écriture, sans synchronisation ; se
   détecte avec `-race`.
10. Quand et comment cette goroutine se termine-t-elle ?
