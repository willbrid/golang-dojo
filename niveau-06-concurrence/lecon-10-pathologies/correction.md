# Leçon 10 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Lire un rapport de race

Chaque bloc `WARNING: DATA RACE` contient : l'accès courant (`Write at … by goroutine N`, avec sa
pile), l'accès en conflit (`Previous write at …` ou `Previous read at …`), puis
`Goroutine N (running) created at:` et la même chose pour l'autre goroutine. On obtient **quelques**
rapports seulement, pas 1 000 : le détecteur ne signale qu'une fois chaque **paire de lignes de code**
en conflit (dédoublonnage par pile). Une race signalée une fois concerne en général toutes les
exécutions du même code.

## E2 — La race qui ne se produit pas

Sans `-x`, `-race` ne dit **rien** : il n'observe que les accès effectivement exécutés. Un détecteur
dynamique ne prouve pas l'absence de race ; il trouve celles que les tests **déclenchent**. D'où :
des tests qui exercent réellement la concurrence (beaucoup de goroutines, `-count`, `-cpu=1,4,8`),
et la relecture avec la question « qui d'autre accède à cette variable ? ».

## E3 — Toutes les piles

Le runtime **ne** détecte **pas** l'interblocage : `main` dort, elle n'est pas « bloquée pour
toujours » au sens du détecteur. Après `SIGQUIT`, la sortie contient toutes les goroutines, dont :

```
goroutine 6 [sync.Mutex.Lock, 2 minutes]:
…
goroutine 7 [sync.Mutex.Lock, 2 minutes]:
…
```

La durée entre crochets (affichée au-delà d'une minute d'attente) est l'indice décisif : une goroutine
bloquée depuis 2 minutes sur un verrou, dans un service où une section critique dure des
microsecondes, est presque sûrement interbloquée. Les deux piles montrent quel verrou chacune tient
(la fonction qui l'a pris) et lequel elle attend.

## E4 — Le profil `goroutineleak`

Mesuré :

```
goroutineleak profile: total 3
#	…	main.sendLeak.func1+0x1d	main.go:7
#	…	main.recvLeak.func1+0x18	main.go:8
#	…	main.serviceLeak.func1+0x64	main.go:9
```

Les trois fuites sont trouvées. La goroutine bloquée sur le channel **global** n'apparaît **pas** : une
variable globale reste atteignable, donc n'importe quel code pourrait encore envoyer dessus — le
runtime ne peut pas **prouver** que la goroutine est perdue. C'est la contrepartie de l'absence de
faux positifs : le profil ne signale que les fuites **certaines**. Une goroutine qui attend sur un
channel global oublié reste un problème… que seuls `NumGoroutine` et le profil `goroutine` montrent.

## E5 — Livelock

Sans attente, sur plusieurs cœurs, les deux goroutines peuvent échouer en cadence des centaines, voire
des milliers de fois par succès, en consommant du CPU. Avec une attente aléatoire, elles se
désynchronisent : le nombre de tentatives par succès tombe à un ou deux. C'est le même principe que le
*backoff* avec *jitter* des clients réseau (niveau 10) : l'aléa casse la symétrie.

## Exercice intermédiaire — Chasse aux bugs

| # | Défaut | Révélé par | Correction |
|---|---|---|---|
| 1 | `Broadcast` **envoie sous verrou** sur des channels non bufferisés : un abonné qui ne lit pas bloque `Broadcast` **en tenant le verrou** ; `Subscribe`/`Unsubscribe` bloquent alors aussi | `synctest` (goroutines bloquées) ou interblocage dans un test où l'abonné lit après s'être désabonné | copier la liste des channels sous verrou, envoyer **hors** verrou, et ne jamais bloquer : `select` avec `default` (message perdu, compté) ou avec un délai |
| 2 | un abonné qui cesse de lire bloque tout le monde | idem | channels bufferisés + envoi non bloquant, ou une goroutine par abonné |
| 3 | `Sent()` lit `n.sent` **sans verrou** | `-race` | prendre le verrou (ou `atomic.Int64`) |
| 4 | `Unsubscribe` deux fois : `close` d'un channel nil (la map rend nil pour une clé absente) → **panique** | test de double désabonnement | vérifier la présence (`ch, ok := …`) avant de fermer |
| 5 | `BroadcastAsync` lance des goroutines que personne n'attend : fuites si elles bloquent, et aucun moyen d'attendre qu'un message soit parti avant `Close` | profil `goroutineleak` ou `synctest` | un `WaitGroup` interne et une méthode `Close` qui attend ; ou supprimer l'API asynchrone et laisser l'appelant lancer `go n.Broadcast(…)` s'il le souhaite |

Une version corrigée de `Broadcast` :

```go
func (n *Notifier) Broadcast(msg string) (dropped int) {
	n.mu.Lock()
	targets := make([]chan string, 0, len(n.listeners))
	for _, ch := range n.listeners {
		targets = append(targets, ch)
	}
	n.mu.Unlock()

	for _, ch := range targets {
		select {
		case ch <- msg:            // channels créés avec une capacité (par ex. 16) dans Subscribe
		default:
			dropped++              // abonné trop lent : on ne bloque personne
		}
	}
	n.sentCount.Add(1)
	return dropped
}
```

Mais **attention** : envoyer hors verrou rouvre une course avec `Unsubscribe`, qui peut **fermer** un
channel pendant qu'on envoie dessus → `send on closed channel`. La règle de la leçon 3 (« seul
l'émetteur ferme ») est violée par la conception elle-même : `Unsubscribe` ferme un channel sur lequel
`Broadcast` écrit. Correction propre : `Unsubscribe` ne ferme pas ; il retire l'abonné de la map **et**
signale la fin par un second channel `done` propre à l'abonné, que le récepteur écoute. Ce bug-là est
le plus instructif des cinq : il ne se voit qu'en corrigeant les autres.

## Défi — Un détecteur de fuites pour les tests

1. **Version `NumGoroutine`** : faux positifs quand une goroutine légitime de la bibliothèque (une
   minuterie, un *keep-alive* HTTP) démarre pendant le test ; faux négatifs quand une goroutine fuie
   est compensée par une autre qui se termine. Les réessais avec une petite attente réduisent les faux
   positifs dus aux goroutines en train de finir.
2. **Version `goroutineleak`** :

   ```go
   func VerifyNoLeaks(t *testing.T) {
   	t.Helper()
   	t.Cleanup(func() {
   		var buf bytes.Buffer
   		p := pprof.Lookup("goroutineleak")
   		p.WriteTo(&buf, 1)
   		if p.Count() > 0 {
   			t.Errorf("goroutines fuies :\n%s", buf.String())
   		}
   	})
   }
   ```

   Avantages : aucun faux positif (seules les fuites prouvées sont signalées), et la pile exacte de
   chaque fuite. Limite : il ne voit pas une goroutine bloquée sur un objet encore atteignable (E4).
3. **`synctest`** est préférable quand le code utilise le temps (il le virtualise) et qu'on peut
   enfermer tout le scénario dans une bulle ; il détecte **toute** goroutine restante, atteignable ou
   non. Le profil convient à des tests d'intégration plus larges, hors bulle, et fonctionne aussi en
   production.

## Quiz

1. Deux accès concurrents à la même variable, dont au moins une écriture, non ordonnés par une
   synchronisation.
2. Le modèle mémoire ne garantit rien en présence de race : compilateur et processeur peuvent
   réordonner les accès, le résultat n'est pas défini.
3. Il ne trouve que les races exécutées pendant l'observation, ralentit et alourdit le programme, et
   ignore les erreurs de logique.
4. Quand toutes les goroutines sont bloquées ; pas quand d'autres goroutines vivent encore.
5. `SIGQUIT` (Ctrl-\), ou le profil `goroutine` (`debug=2`), éventuellement via `net/http/pprof`.
6. Des goroutines actives qui ne progressent pas parce qu'elles se gênent ; de l'aléa dans les réessais.
7. Une goroutine jamais servie ; préemption, mode famine du `Mutex`, choix aléatoire de `select`,
   `RWMutex` qui bloque les nouveaux lecteurs quand un écrivain attend.
8. Envoi sans lecteur, réception sur un channel jamais fermé, boucle sans cas d'arrêt, étage qui
   n'écoute pas `ctx`.
9. Le ramasse-miettes prouve qu'une goroutine est bloquée sur un objet que plus aucune autre goroutine
   ne peut atteindre ; pas de faux positifs (mais des faux négatifs possibles).
10. Le nombre de goroutines (`runtime.NumGoroutine()`) dans le temps.
