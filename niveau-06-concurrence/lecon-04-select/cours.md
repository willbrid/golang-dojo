# Leçon 4 — `select`

> Avec un seul channel, une goroutine ne peut faire qu'une chose : attendre **ce** channel. Un
> serveur réel attend plusieurs choses à la fois : une requête, un délai, un ordre d'arrêt, une
> tâche périodique. `select` est l'instruction qui attend **le premier** de plusieurs événements.
> C'est elle qui transforme les channels en outil de conception.

## Objectifs

1. Attendre plusieurs opérations de channel avec `select`.
2. Poser un délai (`time.After`) et faire une opération non bloquante (`default`).
3. Désactiver un cas avec un channel nil.
4. Écrire une **boucle de service** : `for` + `select` + un signal d'arrêt.

## Explication

### La forme

```go
select {
case v := <-a:
	// a a fourni une valeur
case b <- x:
	// l'envoi sur b a réussi
case v, ok := <-c:
	// réception avec détection de fermeture
}
```

`select` évalue toutes les opérations de ses `case`, puis :

- si **une seule** est prête, il l'exécute ;
- si **plusieurs** sont prêtes, il en choisit une **au hasard** (uniformément) — pas la première
  écrite. Ce hasard est voulu : il empêche qu'un channel très actif affame les autres ;
- si **aucune** n'est prête, il **bloque** jusqu'à ce que l'une le devienne — sauf s'il y a un `default`.

`select {}` sans aucun cas bloque pour toujours.

### Un délai

```go
select {
case v := <-res:
	return v, nil
case <-time.After(2 * time.Second):
	return "", ErrTimeout
}
```

`time.After(d)` rend un channel qui recevra une valeur après `d`. Le premier arrivé gagne. Depuis
Go 1.23, la minuterie créée par `time.After` est récupérée par le ramasse-miettes dès qu'elle n'est
plus référencée, même si elle n'a pas expiré : l'ancien conseil « ne pas utiliser `time.After` dans
une boucle » ne vaut plus pour la mémoire. Dans une boucle **serrée**, un `time.NewTimer` réutilisé
reste plus économe.

Attention : abandonner l'attente ne **stoppe pas** le travail. La goroutine qui calcule continue,
et si elle envoie sur un channel non bufferisé que plus personne ne lit, elle fuit (leçon 3) — d'où
le channel de capacité 1 dans l'exemple. Pour vraiment **annuler** le travail, il faut `context`
(leçon 8).

### Une opération non bloquante : `default`

```go
select {
case events <- e:
	// envoyé
default:
	// buffer plein : on n'attend pas, on abandonne l'événement (ou on compte une perte)
}
```

Le cas `default` s'exécute si aucun autre n'est prêt **immédiatement**. C'est la seule façon
correcte de « tenter » un envoi ou une réception — bien mieux que `if len(ch) < cap(ch)`, qui est
périmé aussitôt lu (leçon 3). Usage typique : des métriques ou des journaux qu'on préfère perdre
plutôt que de ralentir le chemin principal.

Piège : un `for { select { … default: } }` sans rien de bloquant tourne à **100 % du CPU** (attente
active). Si la boucle doit attendre, pas de `default`.

### Désactiver un cas : le channel nil

Une opération sur un channel nil bloque pour toujours (leçon 2) ; dans un `select`, son cas n'est
donc **jamais choisi**. On s'en sert pour désactiver dynamiquement un cas : quand un channel est
fermé, on met la variable à `nil`, et `select` ne le regarde plus.

```go
for a != nil || b != nil {
	select {
	case v, ok := <-a:
		if !ok { a = nil; continue }
		…
	case v, ok := <-b:
		if !ok { b = nil; continue }
		…
	}
}
```

Sans ce `a = nil`, un channel fermé serait **toujours prêt** (il rend immédiatement la zéro-valeur),
et la boucle tournerait à vide en le choisissant sans cesse.

### La boucle de service

La structure la plus répandue d'une goroutine de longue durée :

```go
for {
	select {
	case req := <-requests:
		handle(req)
	case <-ticker.C:
		periodic()
	case <-quit:
		cleanup()
		return
	}
}
```

Une goroutine, **un** état qu'elle seule possède, et des événements qui arrivent par channels.
`quit` est en général un `chan struct{}` **fermé** pour donner l'ordre d'arrêt : une fermeture est
reçue par **tous** les récepteurs, alors qu'un envoi n'en réveille qu'un. La leçon 8 remplacera ce
`quit` par `ctx.Done()`, qui fonctionne exactement ainsi.

`time.NewTicker(d)` envoie une valeur toutes les `d` sur `ticker.C`. Il faut l'arrêter
(`defer ticker.Stop()`) quand on n'en a plus besoin.

## Exemple

```go
package main

import (
	"errors"
	"fmt"
	"time"
)

var ErrTimeout = errors.New("délai dépassé")

// fetch simule un appel lent. Le résultat part sur un channel de capacité 1 :
// si l'appelant abandonne, la goroutine peut quand même terminer (leçon 3).
func fetch(d time.Duration, v string) <-chan string {
	ch := make(chan string, 1)
	go func() {
		time.Sleep(d)
		ch <- v
	}()
	return ch
}

// withTimeout attend un résultat, ou abandonne après limit.
func withTimeout(res <-chan string, limit time.Duration) (string, error) {
	select {
	case v := <-res:
		return v, nil
	case <-time.After(limit):
		return "", ErrTimeout
	}
}

// tryPublish n'attend jamais : si le buffer est plein, l'événement est abandonné.
func tryPublish(events chan<- string, e string) bool {
	select {
	case events <- e:
		return true
	default:
		return false
	}
}

// mergeTwo lit a et b jusqu'à ce que les DEUX soient fermés.
// Un channel fermé est remplacé par nil : son cas ne sera plus jamais choisi.
func mergeTwo(a, b <-chan int) []int {
	var out []int
	for a != nil || b != nil {
		select {
		case v, ok := <-a:
			if !ok {
				a = nil
				continue
			}
			out = append(out, v)
		case v, ok := <-b:
			if !ok {
				b = nil
				continue
			}
			out = append(out, v)
		}
	}
	return out
}

func main() {
	// 1. Délai.
	for _, d := range []time.Duration{10 * time.Millisecond, 300 * time.Millisecond} {
		v, err := withTimeout(fetch(d, fmt.Sprint("réponse en ", d)), 100*time.Millisecond)
		fmt.Printf("%q %v\n", v, err)
	}

	// 2. Envoi non bloquant.
	events := make(chan string, 2)
	for i := range 4 {
		fmt.Printf("événement %d publié : %v\n", i, tryPublish(events, fmt.Sprint("e", i)))
	}

	// 3. Fusion jusqu'à fermeture des deux sources.
	a, b := make(chan int), make(chan int)
	go func() { defer close(a); a <- 1; a <- 2 }()
	go func() { defer close(b); b <- 10 }()
	got := mergeTwo(a, b)
	fmt.Println(len(got), "valeurs reçues")

	// 4. Boucle de service : un ticker, un channel de travail, un signal d'arrêt.
	work := make(chan string)
	quit := make(chan struct{})
	done := make(chan struct{})
	go func() {
		defer close(done)
		tick := time.NewTicker(40 * time.Millisecond)
		defer tick.Stop()
		processed := 0
		for {
			select {
			case w := <-work:
				processed++
				_ = w
			case <-tick.C:
				// tâche périodique (ici : rien ; en vrai, un vidage de cache, une métrique…)
			case <-quit:
				fmt.Println("arrêt après", processed, "tâches")
				return
			}
		}
	}()
	for i := range 3 {
		work <- fmt.Sprint("tâche ", i)
	}
	close(quit)
	<-done
}
```

Sortie :

```
"réponse en 10ms" <nil>
"" délai dépassé
événement 0 publié : true
événement 1 publié : true
événement 2 publié : false
événement 3 publié : false
3 valeurs reçues
arrêt après 3 tâches
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `fetch` rend un channel de capacité 1 | Quand `withTimeout` abandonne, la goroutine de `fetch` peut quand même déposer son résultat et se terminer : pas de fuite. |
| `withTimeout` | Le patron du délai en cinq lignes. Le travail lent **continue** en arrière-plan : on a cessé d'attendre, pas annulé. |
| `tryPublish` avec `default` | Les deux premiers envois remplissent le buffer, les suivants sont abandonnés sans bloquer. |
| `mergeTwo` et `a = nil` | Un channel fermé désactivé ; la boucle se termine quand les deux le sont. On n'affiche que le **nombre** de valeurs : leur **ordre** dépend du hasard de `select`. |
| La boucle de service | Trois sources d'événements, une seule goroutine : `processed` n'a pas besoin de verrou. |
| `defer tick.Stop()` | Le ticker est arrêté quand la boucle se termine. |
| `close(quit)` puis `<-done` | L'ordre d'arrêt, puis l'attente que la goroutine ait **vraiment** fini (son affichage compris) avant que `main` se termine. |
| `arrêt après 3 tâches` | Chaque envoi sur `work` (non bufferisé) est un rendez-vous : les trois tâches sont reçues **avant** que `close(quit)` ne s'exécute. |

## Erreurs fréquentes

1. **Croire que `select` choisit le premier cas** quand plusieurs sont prêts.
2. **Un `default` dans une boucle d'attente** : 100 % de CPU.
3. **Un channel fermé non désactivé** dans un `select` en boucle : il est choisi sans cesse.
4. **Croire que le délai annule le travail.**
5. **Un résultat sur un channel non bufferisé** derrière un délai : fuite de la goroutine.
6. **Oublier `ticker.Stop()`.**
7. **Arrêter une boucle par un envoi** sur `quit` quand plusieurs goroutines l'écoutent : une seule
   est réveillée. Fermer.
8. **Ne pas attendre la fin** de la goroutine après lui avoir demandé de s'arrêter.

## Bonnes pratiques Go

- `for` + `select` + cas d'arrêt : la structure de toute goroutine de longue durée.
- Arrêter par **fermeture** d'un channel (puis `ctx.Done()`, leçon 8).
- `default` seulement pour une tentative ponctuelle, jamais pour attendre.
- Mettre à `nil` un channel fermé qu'on continue de surveiller.
- Chaque délai doit dire ce qu'il advient du travail abandonné.

## Ce que je dois retenir

- `select` attend le premier cas prêt ; plusieurs prêts → choix **aléatoire**.
- `case <-time.After(d)` : un délai.
- `default` : opération non bloquante.
- Un channel nil désactive son cas ; un channel fermé est toujours prêt.
- Boucle de service : `for { select { …; case <-quit: return } }`.
- Abandonner une attente n'arrête pas le travail : il faut `context`.

---

➡️ [Exercices](exercices.md)
