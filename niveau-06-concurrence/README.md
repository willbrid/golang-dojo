# Niveau 6 — Concurrence et coordination

**Objectif :** raisonner juste sur des programmes concurrents — pas seulement les faire
« marcher » sur sa machine. C'est le cœur de Go, et le niveau le plus long du parcours,
volontairement : beaucoup d'exercices, presque tous à lancer avec `go test -race`.

**Prérequis :** niveaux 1 à 5. En particulier :
- les **closures** (N1 L11) : une goroutine lancée sur une closure capture des variables ;
- les **pointeurs** et le tableau *Copié ou partagé ?* (N2 L1) : la concurrence pose la même
  question, avec en plus « par **qui**, et **quand** ? » ;
- `defer` (N2 L7) : pour déverrouiller et pour fermer ;
- les **tests** (niveau 5) : le détecteur de *data races* s'utilise avec `go test -race`, et
  `testing/synctest` permet de tester le temps sans attendre.

> **Ordre des leçons.** D'abord le modèle (goroutines, ordonnanceur), puis la communication
> (channels, `select`), puis la synchronisation par mémoire partagée (`WaitGroup`, `Mutex`,
> `atomic`), puis l'annulation (`context`), et seulement ensuite les patrons qui combinent le
> tout. Les deux dernières leçons traitent de ce qui casse : pathologies, surcharge, paniques.

## Leçons

| # | Leçon | Ce que je saurai faire |
|---|---|---|
| 1 | [Le modèle de concurrence](lecon-01-modele/) | Lancer des goroutines, comprendre l'ordonnanceur, distinguer concurrence et parallélisme |
| 2 | [Channels non bufferisés](lecon-02-channels/) | Communiquer et synchroniser par rendez-vous, typer la direction d'un channel |
| 3 | [Channels bufferisés](lecon-03-channels-bufferises/) | Dimensionner un buffer, fermer correctement, savoir qui ferme |
| 4 | [`select`](lecon-04-select/) | Multiplexer, poser des délais, écrire une boucle de service |
| 5 | [`sync.WaitGroup`](lecon-05-waitgroup/) | Attendre un groupe de goroutines sans fuite ni course |
| 6 | [`Mutex` et `RWMutex`](lecon-06-mutex/) | Protéger un état partagé avec la bonne granularité |
| 7 | [`atomic`, `Once`, `Pool`](lecon-07-atomic-once-pool/) | Utiliser les primitives fines quand elles se justifient |
| 8 | [`context`](lecon-08-context/) | Annuler, poser des délais, propager à travers les appels |
| 9 | [Patrons](lecon-09-patrons/) | Worker pool, pipeline, fan-out/fan-in, `errgroup` |
| 10 | [Pathologies](lecon-10-pathologies/) | Diagnostiquer races, interblocages, fuites de goroutines |
| 11 | [Backpressure](lecon-11-backpressure/) | Borner les files, rejeter proprement, garder la latence sous contrôle |
| 12 | [Panique et goroutines](lecon-12-panique/) | Empêcher qu'une goroutine fasse tomber le programme, superviser |

## Évaluation de fin de niveau

Le **[projet 5 — serveur concurrent](../projets/projet-05-serveur-concurrent/)** : un serveur
TCP qui mobilise goroutines, channels, `context`, arrêt gracieux et backpressure.

## Règle du niveau

**Tout code concurrent de ce niveau se teste avec `go test -race`.** Un programme qui produit la
bonne sortie mais que le détecteur signale est **faux**, même s'il « marche » mille fois de suite.
