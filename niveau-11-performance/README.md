# Niveau 11 — Performance

**Objectif :** mesurer avant d'optimiser, comprendre ce que coûte réellement un programme Go (allocations,
ramasse-miettes, mémoire, contention), trouver le goulot avec les profils, et n'optimiser que ce qui
compte.

**Prérequis :** niveaux 1 à 10. En particulier :
- les **benchmarks** (N5 L6) : `b.Loop`, `-benchmem`, `-count` ;
- les **pointeurs** et les slices (N1 L6, N2 L1) : la leçon 2 explique où vivent les valeurs ;
- la **concurrence** (niveau 6) : la contention et le profil `goroutineleak` (leçon 5) ;
- l'**observabilité** (N10 L10) : percentiles et histogrammes, repris à la leçon 7.

> **Ordre des leçons.** Savoir mesurer d'abord (1). Puis le modèle de mémoire d'exécution : pile, tas
> et analyse d'échappement (2), ramasse-miettes (3). Puis les outils qui disent **où** va le temps et la
> mémoire (4, 5). Enfin les structures de données (6), le comportement d'un service sous charge (7), et les
> optimisations qui valent la peine (8).

## Leçons

| # | Leçon | Ce que je saurai faire |
|---|---|---|
| 1 | [Mesurer](lecon-01-mesurer/) | Écrire des benchmarks fiables, quantifier le bruit, comparer deux versions statistiquement |
| 2 | [Pile, tas, analyse d'échappement](lecon-02-pile-tas/) | Lire `-gcflags=-m`, savoir pourquoi une valeur part sur le tas |
| 3 | [Le ramasse-miettes](lecon-03-ramasse-miettes/) | Comprendre le GC de Go, régler `GOGC` et `GOMEMLIMIT`, lire les traces du GC |
| 4 | [pprof — CPU](lecon-04-pprof-cpu/) | Collecter un profil CPU, le lire, trouver le goulot |
| 5 | [pprof — mémoire, blocage, verrous](lecon-05-pprof-memoire/) | Profils d'allocation, de blocage, de contention et de fuite de goroutines |
| 6 | [Slices, maps et cache](lecon-06-slices-maps-cache/) | Préallouer, choisir sa structure, tenir compte de la localité mémoire |
| 7 | [Latence et débit](lecon-07-latence-debit/) | Percentiles, loi de Little, files d'attente, tests de charge |
| 8 | [Optimisations qui valent le coup](lecon-08-optimisations/) | Réduire les allocations, réutiliser, et savoir quand s'arrêter |

## Règle du niveau

**Pas d'optimisation sans mesure avant et après.** Une modification « pour la performance » qui n'est
pas accompagnée d'un benchmark ou d'un profil qui la justifie n'est pas une optimisation : c'est une
complication.
