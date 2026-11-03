# Leçon 3 — Exercices

Code dans `mes-solutions/niveau-11/lecon-03/`, en partant du programme de l'exemple.

## Exercices faciles

**E1 — Lire `gctrace`.**
Lancer `GODEBUG=gctrace=1 ./gcload 2> trace.txt`. Pour les cycles 5 à 10, relever le tas vivant et
l'objectif. Vérifier la formule objectif = vivant × (1 + GOGC/100). Que valent les pauses comparées au temps
de marquage concurrent ?

**E2 — Moins de pointeurs.**
Remplacer `live []*node` par `live []node` (des valeurs, plus des pointeurs), et `next *node` par
`next int32` (un indice). Le flux d'objets éphémères reste identique. Mesurer cycles, pic et part du GC.
Pourquoi le marquage est-il moins cher ?

**E3 — Moins d'allocations.**
Garder `live []*node`, mais au lieu d'allouer un nouveau `node` à chaque tour, réutiliser l'objet remplacé
(on recopie les nouvelles valeurs dans `*live[i]`). Mesurer. Qu'est-ce qui a disparu dans les métriques ?

**E4 — La limite dans un conteneur.**
Lancer l'exemple dans un conteneur limité à 100 Mio (`docker run --memory=100m` avec l'image `golang`, ou
`systemd-run --scope -p MemoryMax=100M`) : (a) sans `GOMEMLIMIT` et `GOGC=400` ; (b) avec
`GOMEMLIMIT=85MiB`. Que se passe-t-il dans chaque cas ?

**E5 — `runtime.GC()`.**
Ajouter un `runtime.GC()` toutes les 100 000 itérations « pour garder la mémoire basse ». Mesurer. Dans quel
cas réel un appel explicite peut-il se justifier ?

---

## Exercice intermédiaire — Un tableau de bord du GC

Écrire un paquet `gcstats` qui, toutes les secondes (horloge injectée pour les tests), lit dans
`runtime/metrics` :

- le tas vivant (`/gc/heap/live:bytes`) et l'objectif (`/gc/heap/goal:bytes`) ;
- le nombre de cycles (`/gc/cycles/total:gc-cycles`) ;
- la part du processeur consacrée au GC (deux lectures de `/cpu/classes/gc/total:cpu-seconds` et
  `/cpu/classes/total:cpu-seconds`) ;
- la distribution des pauses (`/sched/pauses/total/gc:seconds`, un histogramme : en extraire le maximum et
  le p99).

1. Exposer ces valeurs en texte sur `/debug/gc` (adresse d'administration, N10 L10).
2. Écrire une alerte (une ligne de journal) quand la part du GC dépasse 25 % pendant 10 secondes.
3. Tester avec une charge qui alloue peu, puis beaucoup.
4. Pourquoi `runtime/metrics` plutôt que `runtime.ReadMemStats` pour une lecture périodique ?

---

## Défi — Trouver le bon réglage

Un service tourne dans des conteneurs de 512 Mio ; son tas vivant varie entre 120 et 200 Mio selon l'heure ;
il est limité par le processeur aux heures de pointe.

1. Écrire une charge de test qui reproduit ce profil (tas vivant qui varie, allocations soutenues).
2. Mesurer, pour `GOGC` ∈ {100, 200, 400, off} et `GOMEMLIMIT` ∈ {aucune, 400 Mio, 450 Mio} : part du GC,
   mémoire maximale, et marge restante avant 512 Mio.
3. Choisir un réglage et le justifier en cinq lignes. Que se passe-t-il, avec ton réglage, si une fuite fait
   monter le tas vivant à 480 Mio ?
4. Comparer avec la construction `GOEXPERIMENT=nogreenteagc` sur **cette** charge.

---

## Quiz

1. Que signifie « marquage tricolore » ?
2. Pourquoi faut-il une barrière d'écriture pendant le marquage ?
3. Le GC de Go déplace-t-il les objets ? Est-il générationnel ?
4. Quel est le coût principal du GC : les pauses ou le processeur ?
5. Quand un cycle démarre-t-il avec `GOGC=100` ?
6. Que fait `GOMEMLIMIT`, et pourquoi ne pas la fixer au ras du tas vivant ?
7. Que signifient les trois nombres `67->73->40 MB` d'une ligne `gctrace` ?
8. Quels sont les deux leviers du code pour réduire le coût du GC ?
9. Quelles nouveautés du *runtime* sont activées par défaut en Go 1.27 ?
10. Comment mesurer l'effet d'une expérience du *runtime* sur son programme ?
