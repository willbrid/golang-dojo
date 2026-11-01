# Leçon 10 — Exercices

Code dans `mes-solutions/niveau-06/lecon-10/`.

## Exercices faciles

**E1 — Lire un rapport de race.**
Reprendre le compteur sans protection de la leçon 1 (E2). Lancer avec `-race` et annoter chaque bloc
du rapport : quelle ligne écrit, quelle ligne a écrit avant, où chaque goroutine a été créée.
Combien de rapports pour 1 000 goroutines ? Pourquoi pas 1 000 ?

**E2 — La race qui ne se produit pas.**
Écrire un programme où deux goroutines écrivent la même variable, mais l'une seulement si un
argument `-x` est passé. Lancer avec `-race` sans `-x` : que dit-il ? Qu'en conclure sur la
couverture des tests concurrents ?

**E3 — Toutes les piles.**
Écrire un programme qui s'interbloque **partiellement** (deux goroutines bloquées l'une sur l'autre,
pendant que `main` attend un `time.Sleep(time.Hour)`). Le runtime détecte-t-il l'interblocage ?
Envoyer `SIGQUIT` (Ctrl-\, ou `kill -QUIT <pid>`) et trouver les deux goroutines dans la sortie.
Quelle information indique depuis combien de temps elles attendent ?

**E4 — Le profil `goroutineleak`.**
Écrire trois fuites différentes : envoi sans lecteur, réception sur un channel jamais fermé, boucle
de service sans cas d'arrêt (bloquée sur un channel que plus personne ne référence). Le profil les
trouve-t-il toutes ? Écrire ensuite une goroutine bloquée sur un channel **encore** référencé par
une variable globale : apparaît-elle dans le profil ? Pourquoi ?

**E5 — Livelock.**
Simuler deux goroutines qui ont chacune besoin de deux verrous et utilisent `TryLock` : prendre le
premier, `TryLock` le second, et en cas d'échec tout relâcher et recommencer **immédiatement**.
Mesurer le nombre de tentatives par succès. Ajouter une attente aléatoire (`rand.N(time.Millisecond)`)
avant de recommencer : que devient ce nombre ?

---

## Exercice intermédiaire — Chasse aux bugs

Le paquet suivant contient **cinq** défauts de concurrence. Les trouver, écrire pour chacun un test
qui le révèle (`-race`, `synctest`, ou profil `goroutineleak`), puis corriger.

```go
package notifier

type Notifier struct {
	mu        sync.Mutex
	listeners map[string]chan string
	sent      int
}

func New() *Notifier { return &Notifier{listeners: map[string]chan string{}} }

func (n *Notifier) Subscribe(name string) <-chan string {
	ch := make(chan string)
	n.mu.Lock()
	n.listeners[name] = ch
	n.mu.Unlock()
	return ch
}

func (n *Notifier) Unsubscribe(name string) {
	n.mu.Lock()
	close(n.listeners[name])
	delete(n.listeners, name)
	n.mu.Unlock()
}

func (n *Notifier) Broadcast(msg string) {
	n.mu.Lock()
	defer n.mu.Unlock()
	for _, ch := range n.listeners {
		ch <- msg
	}
	n.sent++
}

func (n *Notifier) Sent() int { return n.sent }

func (n *Notifier) BroadcastAsync(msg string) {
	go n.Broadcast(msg)
}
```

*(Indices, à ne lire qu'en cas de blocage : un envoi sous verrou ; un abonné qui ne lit plus ; une
lecture sans verrou ; un double désabonnement ; des goroutines dont personne n'attend la fin.)*

---

## Défi — Un détecteur de fuites pour les tests

Écrire une fonction utilitaire de test :

```go
func VerifyNoLeaks(t *testing.T)   // à appeler en defer au début d'un test
```

1. Première version : comparer `runtime.NumGoroutine()` au début et à la fin du test, avec quelques
   réessais (les goroutines peuvent mettre un instant à finir). Quels faux positifs ? Quels faux négatifs ?
2. Seconde version : utiliser le profil `goroutineleak` et faire échouer le test s'il est non vide,
   en affichant les piles. Quels avantages ?
3. Comparer avec `testing/synctest` : dans quels cas chaque approche est-elle préférable ?
4. *(Pour information : le module `go.uber.org/goleak` est l'outil le plus utilisé pour la version 1.)*

---

## Quiz

1. Quelle est la définition exacte d'une *data race* ?
2. Pourquoi le comportement d'un programme qui contient une race est-il « indéfini » ?
3. Quelles sont les limites du détecteur de races ?
4. Quand le runtime détecte-t-il un interblocage, et quand ne le peut-il pas ?
5. Comment obtenir la pile de toutes les goroutines d'un processus en cours ?
6. Qu'est-ce qu'un *livelock*, et comment l'éviter ?
7. Qu'est-ce que la famine ? Citer deux mécanismes du runtime qui la limitent.
8. Quelles sont les quatre causes classiques de fuite de goroutines ?
9. Sur quel principe repose le profil `goroutineleak` ? Peut-il donner des faux positifs ?
10. Quelle métrique surveiller en production pour détecter une fuite ?
