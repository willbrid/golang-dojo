# Module 4 — Exercices

Code dans `mes-solutions/niveau-13/module-04/`. Pour chaque enquête, rendre le **journal d'enquête** : les
observations, l'hypothèse, l'expérience qui l'a confirmée ou réfutée, la correction, le test ajouté.

## Exercices faciles

**E1 — Qui modifie cette variable ?**
Dans un programme où une variable globale `limit` est modifiée par erreur quelque part (à toi de cacher la
modification dans une fonction appelée indirectement), trouver **où**, sans lire le code, avec un point
d'observation de `delve` (`watch -w limit`). Que montre `delve` au moment où l'écriture a lieu ? Quelle
limite ont les points d'observation (nombre, taille) ?

**E2 — Un interblocage dans un vidage.**
Écrire un programme où deux goroutines prennent deux verrous dans l'ordre inverse (N6 L10), pendant que
`main` attend un signal. Le programme est figé sans planter. Obtenir un vidage mémoire du processus vivant
(`gcore` si `gdb` est installé, ou `kill -ABRT` avec `GOTRACEBACK=crash`), et trouver dans `dlv core`
quelles goroutines tiennent et attendent quels verrous.

**E3 — Corriger le pic.**
Corriger `rebuild` dans `spike` (la reconstruction hors du verrou, et un échange sous le verrou ou par
`atomic.Pointer`). Mesurer p99, p99,9 et maximum. L'enregistreur de vol se déclenche-t-il encore ? Combien de
temps `rebuild` tient-il maintenant le verrou ?

**E4 — La latence de l'ordonnanceur.**
Écrire un test où une goroutine calcule sans interruption pendant 2 s pendant que d'autres doivent répondre
toutes les 1 ms, avec `GOMAXPROCS=1`, puis 8. Le tracer (`go test -trace`) et lire
`go tool trace -pprof=sched`. Que montre-t-il ? Depuis Go 1.14, le *runtime* préempte les goroutines même
dans une boucle sans appel : le vérifier, et mesurer le retard maximal de la goroutine qui veut répondre.

**E5 — Régler le détecteur de races.**
Provoquer deux courses différentes dans un test. Comparer les rapports par défaut, avec
`GORACE=halt_on_error=1`, avec `log_path=/tmp/race`, et avec `history_size=7`. Dans quel cas un rapport
affiche-t-il `failed to restore the stack` ? Que faire alors ?

---

## Exercice intermédiaire — Une enquête complète

Ce service compte les visites par page et sert un classement. L'équipe signale quatre problèmes :
- un plantage occasionnel sous charge ;
- un classement qui contient de temps en temps une page en double ;
- des pages pourtant visitées qui n'apparaissent jamais dans le classement ;
- une mémoire qui grossit sans cesse.

```go
package top

import (
	"slices"
	"sort"
	"sync"
)

type Counter struct {
	mu     sync.Mutex
	counts map[string]int
	order  []string // pages, dans l'ordre de première visite
}

func New() *Counter { return &Counter{counts: map[string]int{}} }

func (c *Counter) Visit(page string) {
	c.mu.Lock()
	if _, ok := c.counts[page]; !ok {
		c.mu.Unlock()
		c.order = append(c.order, page)
		c.mu.Lock()
	}
	c.counts[page]++
	c.mu.Unlock()
}

// Top renvoie les n pages les plus visitées.
func (c *Counter) Top(n int) []string {
	c.mu.Lock()
	pages := c.order
	c.mu.Unlock()
	snapshot := make([]string, 0, len(pages))
	for _, p := range pages {
		snapshot = append(snapshot, p)
	}
	sort.Slice(snapshot, func(i, j int) bool { return c.counts[snapshot[i]] > c.counts[snapshot[j]] })
	return slices.Clone(snapshot[:min(n, len(snapshot))])
}

// History garde chaque classement calculé, « pour le débogage ».
var History [][]string

func (c *Counter) TopLogged(n int) []string {
	t := c.Top(n)
	History = append(History, t)
	return t
}
```

1. Écrire un test de charge qui reproduit les quatre symptômes : plusieurs goroutines qui visitent et
   demandent le classement, et un appel à `TopLogged` par requête.
2. Pour chaque symptôme : quel outil l'a révélé (`-race`, un invariant vérifié au repos, un point d'arrêt
   conditionnel, une trace, un profil mémoire) ? Rédiger le journal d'enquête.
3. Corriger, avec un test de régression pour chacun.

---

## Défi — Une boîte noire pour la production

Écrire un paquet `blackbox` qu'un service branche en trois lignes :

1. Il démarre un enregistreur de vol (fenêtre de quelques secondes, mémoire bornée).
2. Un middleware HTTP mesure chaque requête. Si l'une dépasse un seuil configurable, il prend un instantané
   de la trace et du profil de contention. Au plus un instantané par minute, pour ne pas aggraver un incident.
   L'instantané porte l'identifiant de trace de la requête lente (N12 L7).
3. Il garde les N derniers instantanés en mémoire (ou sur disque, taille bornée), avec leurs métadonnées.
4. Il les expose sur l'adresse d'administration : `GET /debug/blackbox` (liste) et
   `GET /debug/blackbox/{id}` (téléchargement).
5. Tests :
   - le seuil et la limite de fréquence, en temps virtuel (`synctest`) ;
   - un instantané téléchargé s'ouvre avec `go tool trace -pprof=sync` ;
   - le surcoût par requête, mesuré, sans et avec boîte noire.
6. Quelles données sensibles un instantané peut-il contenir ? Qui doit pouvoir le télécharger ?

---

## Quiz

1. Quelles sont les étapes d'une enquête de débogage ?
2. Quel outil pour : la valeur d'une variable, un plantage, une requête lente précise, un blocage ?
3. Pourquoi déboguer un binaire construit avec `-gcflags=all="-N -l"` ?
4. Pourquoi `dlv attach` peut-il être refusé, et que faire en production ?
5. Comment obtenir un vidage mémoire d'un programme Go qui panique ?
6. Pourquoi la ligne de la panique n'est-elle pas toujours celle de l'erreur ?
7. Qu'apporte une trace d'exécution qu'un profil CPU n'apporte pas ?
8. À quoi sert l'enregistreur de vol, et que coûte-t-il ?
9. Pourquoi le p99 ne montrait-il pas les pics de l'exemple ?
10. Que coûte `-race`, et où l'utiliser ?
