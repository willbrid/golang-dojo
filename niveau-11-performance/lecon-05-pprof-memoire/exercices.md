# Leçon 5 — Exercices

Code dans `mes-solutions/niveau-11/lecon-05/`, en partant du paquet `prof` de l'exemple.

## Exercices faciles

**E1 — Les quatre vues de la mémoire.**
Sur le profil de `BenchmarkReport`, comparer `-sample_index=alloc_space`, `alloc_objects`, `inuse_space`,
`inuse_objects`. Corriger `Report` (`strings.Builder` et `strconv`, leçons 1 et 2), reprofiler : que deviennent
les quatre vues ?

**E2 — Sortir le calcul du verrou.**
Modifier `Cache.Get` pour que `slowSquare` soit calculé **hors** du verrou (vérifier, calculer sans verrou,
reprendre le verrou pour stocker). Mesurer le benchmark et le profil de contention. Quel nouveau risque
apparaît (deux goroutines calculent la même clé) ? Est-il grave ici ?

**E3 — Le canal bufferisé.**
Donner un buffer de 100 au canal de `Pipeline`. Que montre le profil de blocage ? Le programme est-il plus
rapide ? Pourquoi un buffer ne rend-il pas le consommateur plus rapide (N6 L3) ?

**E4 — Corriger la fuite.**
Corriger `FirstOf3` (canal bufferisé de 3). Vérifier avec le profil que `total` vaut 0 après dix appels.
Puis remettre la capture de la variable de boucle (`go func() { ch <- i }()`) **sans** corriger le canal :
que dit le profil ? Et `runtime.NumGoroutine()` ?

**E5 — La fuite de mémoire.**
Écrire un cache sans expiration (`map[string][]byte` global) qui grossit à chaque requête. Le profil
`inuse_space` désigne-t-il la ligne qui alloue ou la ligne qui retient ? Que faut-il lire pour savoir **qui
retient** la mémoire ?

---

## Exercice intermédiaire — Enquête en production

Prendre le service de la leçon 4 (exercice intermédiaire) et y introduire trois défauts : un `Rows` jamais
fermé dans une route (N8 L5) ou à défaut une goroutine qui attend un canal jamais alimenté à chaque requête ;
un verrou global tenu pendant l'encodage JSON ; un tampon de 1 Mio alloué par requête.

1. Activer sur l'adresse d'administration les profils de blocage et de contention, **échantillonnés**
   (`runtime.SetBlockProfileRate(1_000_000)`, `runtime.SetMutexProfileFraction(100)`).
2. Sous charge (le générateur de la leçon 1), collecter `heap`, `allocs`, `block`, `mutex`, `goroutine` et
   `goroutineleak` par HTTP (`go tool pprof http://…/debug/pprof/…`).
3. Pour chaque défaut, noter **quel** profil l'a révélé, et avec quelle commande `pprof`.
4. Corriger les trois, et montrer le gain (débit et p99, leçon 1).
5. Mesurer le coût des profils activés : débit avec et sans `SetBlockProfileRate`/`SetMutexProfileFraction`.

---

## Défi — Comprendre le faux négatif de `goroutineleak`

Le cours a mesuré qu'une goroutine qui envoie une variable de boucle **capturée** n'est pas détectée comme
fuie, alors qu'elle l'est quand la valeur est passée en paramètre.

1. Reproduire le constat sur un programme minimal, avec plusieurs variantes : variable capturée mais non
   issue d'une boucle ; variable de boucle capturée mais lue avant le `go` ; `for i := 0; i < n; i++`
   classique ; boucle `range` sur une slice.
2. Pour chaque variante, lire `go build -gcflags=-m` : la variable capturée est-elle déplacée sur le tas ?
   Y a-t-il une corrélation avec la détection ?
3. Formuler une hypothèse sur la cause (indice : ce que le ramasse-miettes peut atteindre depuis les
   objets du tas, et ce qu'il considère comme « une goroutine pourrait encore débloquer celle-ci »).
4. Si tu penses avoir trouvé un défaut du *runtime*, comment le signaler aux développeurs de Go (le dépôt et
   le gestionnaire de tickets officiels) avec un exemple minimal reproductible ?

---

## Quiz

1. Quel profil utiliser pour un programme lent dont le processeur est calme ?
2. Différence entre `alloc_space` et `inuse_space` ?
3. Quels profils sont désactivés par défaut, et comment les activer ?
4. À quelle pile le profil de contention attribue-t-il le temps d'attente ?
5. Pourquoi la ligne désignée dans `Cache.Get` est-elle le `return` ?
6. Pourquoi une partie du profil de blocage d'un benchmark concerne-t-elle `main.main` ?
7. Que garantit le profil `goroutineleak`, et que ne garantit-il pas ?
8. Comment exposer ces profils dans un service ?
9. Pourquoi échantillonner les profils de blocage et de contention en production ?
10. Comment savoir **qui retient** une mémoire qui grossit ?
