# Leçon 8 — `context`

> Les leçons 4 et 7 ont buté deux fois sur la même limite : un délai dépassé **abandonne
> l'attente** mais **n'arrête pas le travail**. Un client HTTP qui se déconnecte, un utilisateur
> qui annule, un budget de temps épuisé : dans tous ces cas, il faut que **toute la chaîne
> d'appels** — et toutes les goroutines qu'elle a lancées — apprenne qu'il faut s'arrêter. C'est le
> rôle de `context.Context`, présent dans presque toute signature de fonction d'un service Go.

## Objectifs

1. Créer des contextes annulables, avec délai, avec cause.
2. Écrire une fonction qui **respecte** l'annulation, et propager le contexte.
3. Connaître les règles d'usage : premier paramètre, jamais stocké, `cancel` toujours appelé.
4. Utiliser les valeurs de contexte **avec parcimonie**.

## Explication

### Ce qu'est un contexte

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}     // fermé quand le contexte est annulé ou expiré
	Err() error                // nil, context.Canceled ou context.DeadlineExceeded
	Value(key any) any
}
```

Le cœur est `Done()` : un channel **fermé** à l'annulation — exactement le `quit` de la leçon 4.
Une fermeture est vue par **toutes** les goroutines qui l'écoutent. `context` n'invente rien de
nouveau ; il standardise ce patron et le rend **composable**.

### Créer des contextes

On part toujours d'une racine, puis on **dérive** :

| Fonction | Annulé quand |
|---|---|
| `context.Background()` | jamais — la racine, dans `main`, les tests, l'initialisation |
| `context.TODO()` | jamais — racine provisoire : « je n'ai pas encore de contexte à passer ici » |
| `context.WithCancel(parent)` | on appelle `cancel()`, ou le parent est annulé |
| `context.WithTimeout(parent, d)` | après `d`, ou `cancel()`, ou le parent |
| `context.WithDeadline(parent, t)` | à l'instant `t`, ou… |
| `context.WithCancelCause(parent)` | `cancel(err)` — avec une **cause** lisible par `context.Cause(ctx)` |
| `context.WithTimeoutCause(parent, d, err)` | après `d`, avec la cause `err` |
| `context.WithoutCancel(parent)` | jamais, mais garde les **valeurs** du parent (pour un travail qui doit survivre à la requête, comme un journal d'audit) |

Les contextes forment un **arbre** : annuler un parent annule tous ses descendants ; un enfant
**ne peut pas** survivre à son parent (un `WithTimeout(parent, time.Hour)` est annulé dès que le
parent l'est). Un enfant peut seulement **raccourcir** le délai de son parent, jamais l'allonger.

### Toujours appeler `cancel`

```go
ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
defer cancel()
```

Les fonctions `With…` rendent une fonction `cancel` qui libère les ressources du contexte (sa
minuterie, son lien avec le parent). Ne pas l'appeler est une **fuite**, jusqu'à l'expiration — ou
pour toujours avec `WithCancel`. `go vet` le signale (`lostcancel`) :

```
the cancel function returned by context.WithTimeout should be called, not discarded, to avoid a context leak
```

`defer cancel()` juste après la création, même si le délai doit expirer de lui-même.

### Respecter l'annulation

Recevoir un `ctx` crée une **obligation** : s'arrêter quand il est annulé. Concrètement :

- passer `ctx` à toutes les fonctions appelées qui en acceptent un (`http.NewRequestWithContext`,
  `db.QueryContext`… niveaux 7 et 8) — c'est la **propagation** ;
- dans une attente, ajouter un cas `<-ctx.Done()` au `select` ;
- dans une longue boucle de calcul, tester régulièrement `ctx.Err() != nil` ;
- rendre une erreur qui **enveloppe** `ctx.Err()` (ou `context.Cause(ctx)`), pour que l'appelant
  puisse tester `errors.Is(err, context.DeadlineExceeded)`.

```go
select {
case res := <-work:
	return res, nil
case <-ctx.Done():
	return zero, ctx.Err()
}
```

Un contexte ne **tue** rien : il **demande** l'arrêt. Une fonction qui ignore son `ctx` continue
jusqu'au bout, et l'annulation ne sert à rien. C'est une coopération.

### La cause

`ctx.Err()` ne dit que `context canceled` ou `context deadline exceeded`. Avec `WithCancelCause`
et `context.Cause(ctx)` (Go 1.20), on sait **pourquoi** : `"arrêt du service"`, `"client
déconnecté"`, `"budget de 1ms dépassé"`. Pour un délai, `Cause` rend `DeadlineExceeded` si aucune
cause n'a été fournie : envelopper `context.Cause(ctx)` avec `%w` donne donc toujours une erreur
testable avec `errors.Is`.

### Les règles d'usage

Elles viennent de la documentation du paquet et de *Go Code Review Comments* :

1. **`ctx` est le premier paramètre**, nommé `ctx` : `func Fetch(ctx context.Context, id string)`.
2. **Ne jamais stocker un contexte dans une struct** : il appartient à **un appel**, pas à un objet
   qui vit plus longtemps. Exception tolérée : une struct qui *représente* une opération (une requête).
3. **Ne jamais passer `nil`** : `context.TODO()` si l'on n'a pas encore de contexte.
4. **Une fonction qui fait des E/S ou peut être longue accepte un `ctx`.** Une fonction de calcul
   pur et rapide n'en a pas besoin.
5. Dans un test : `t.Context()` (Go 1.24), annulé automatiquement à la fin du test (N5 L1).

### Les valeurs de contexte, avec parcimonie

`context.WithValue(ctx, key, val)` attache une valeur à la chaîne d'appels. La tentation est d'y
mettre tout ce qu'on n'a pas envie de passer en paramètre. La règle :

> **Seulement des données « de requête » qui traversent les couches** — identifiant de requête,
> identité authentifiée, informations de traçage. **Jamais** des paramètres de fonction, des
> dépendances (base de données, logger) ni des options.

Une valeur de contexte est invisible dans la signature, non typée à la lecture, et absente si
quelqu'un a oublié de la mettre. La clé doit être d'un **type non exporté** propre au paquet
(`type requestIDKey struct{}`), sinon deux paquets qui utilisent la clé `"id"` s'écrasent. Et on
fournit des fonctions typées pour la lire et l'écrire.

## Exemple

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"time"
)

// slowQuery simule un appel à une base : il dure d, sauf si ctx est annulé avant.
// C'est le comportement attendu de TOUTE fonction qui reçoit un context.
func slowQuery(ctx context.Context, name string, d time.Duration) (string, error) {
	select {
	case <-time.After(d):
		return "résultat de " + name, nil
	case <-ctx.Done():
		return "", fmt.Errorf("%s : %w", name, context.Cause(ctx))
	}
}

// process enchaîne des étapes ; chacune reçoit le MÊME ctx, propagé.
func process(ctx context.Context, steps []time.Duration) error {
	for i, d := range steps {
		if _, err := slowQuery(ctx, fmt.Sprint("étape ", i+1), d); err != nil {
			return err
		}
	}
	return nil
}

// worker est une boucle de service qui s'arrête quand ctx est annulé (remplace le quit de la leçon 4).
func worker(ctx context.Context, jobs <-chan int, done chan<- int) {
	n := 0
	defer func() { done <- n }()
	for {
		select {
		case <-jobs:
			n++
		case <-ctx.Done():
			return
		}
	}
}

// Une clé de contexte : type non exporté, pour qu'aucun autre paquet ne puisse la heurter.
type requestIDKey struct{}

func withRequestID(ctx context.Context, id string) context.Context {
	return context.WithValue(ctx, requestIDKey{}, id)
}

func requestID(ctx context.Context) string {
	id, _ := ctx.Value(requestIDKey{}).(string)
	return id
}

func main() {
	// 1. Un délai global pour toute une opération : 3 étapes de 40 ms dans un budget de 100 ms.
	ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
	defer cancel()
	err := process(ctx, []time.Duration{40 * time.Millisecond, 40 * time.Millisecond, 40 * time.Millisecond})
	fmt.Println("process :", err)
	fmt.Println("délai dépassé ?", errors.Is(err, context.DeadlineExceeded))

	// 2. Annulation explicite, avec une cause.
	ctx2, cancel2 := context.WithCancelCause(context.Background())
	jobs, done := make(chan int), make(chan int)
	go worker(ctx2, jobs, done)
	for i := range 5 {
		jobs <- i
	}
	cancel2(errors.New("arrêt du service"))
	fmt.Println("worker arrêté après", <-done, "tâches ; cause :", context.Cause(ctx2))

	// 3. Un enfant ne survit jamais à son parent.
	parent, cancelParent := context.WithCancel(context.Background())
	child, cancelChild := context.WithTimeout(parent, time.Hour)
	defer cancelChild()
	cancelParent()
	fmt.Println("enfant :", child.Err())

	// 4. Une valeur de requête, avec une clé typée.
	rctx := withRequestID(context.Background(), "req-42")
	fmt.Printf("identifiant : %q, absent : %q\n", requestID(rctx), requestID(context.Background()))
}
```

Sortie :

```
process : étape 3 : context deadline exceeded
délai dépassé ? true
worker arrêté après 5 tâches ; cause : arrêt du service
enfant : context canceled
identifiant : "req-42", absent : ""
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `slowQuery` : `select` sur le travail **et** sur `ctx.Done()` | La fonction respecte l'annulation : elle s'arrête au plus tôt au lieu de finir ses 40 ms. |
| Erreur enveloppant `context.Cause(ctx)` | L'appelant peut tester `errors.Is(err, context.DeadlineExceeded)`, et le message dit quelle étape a été interrompue. |
| Budget de 100 ms pour 3 × 40 ms | Les étapes 1 et 2 réussissent (80 ms), l'étape 3 est interrompue à 100 ms. Un seul délai couvre toute l'opération, quel que soit le nombre d'étapes. |
| `process` passe **le même** `ctx` à chaque étape | La propagation : le délai global s'applique à tout ce qui est appelé en dessous. |
| `worker` écoute `ctx.Done()` | La boucle de service de la leçon 4, où `quit` est remplacé par le contexte. |
| `cancel2(errors.New("arrêt du service"))` | La cause est récupérable par `context.Cause` ; `ctx2.Err()` ne dirait que `context canceled`. |
| L'enfant d'une heure annulé avec son parent | L'arbre des contextes : on ne peut pas survivre à son parent. |
| `defer cancel()` / `defer cancelChild()` | Toujours, même quand le délai expirera tout seul. |
| `requestIDKey struct{}` | Clé de type non exporté ; lecteur typé `requestID(ctx)` qui rend `""` si absent. |

## Erreurs fréquentes

1. **Ne pas appeler `cancel`** (`go vet` : `lostcancel`).
2. **Recevoir un `ctx` et l'ignorer** : l'annulation ne sert à rien.
3. **Créer un `context.Background()` au milieu de la chaîne** au lieu de propager celui reçu : la
   chaîne d'annulation est coupée.
4. **Stocker un contexte dans une struct** de longue durée.
5. **Passer `nil`** comme contexte.
6. **Mettre des dépendances ou des paramètres** dans les valeurs de contexte.
7. **Une clé de valeur de type `string`** : collisions entre paquets.
8. **Comparer `err == context.DeadlineExceeded`** alors que l'erreur est enveloppée : `errors.Is`.
9. **Un travail qui doit survivre à la requête** (audit, envoi différé) lancé avec le contexte de la
   requête : il est annulé avec elle. `context.WithoutCancel`.

## Bonnes pratiques Go

- `ctx context.Context` en premier paramètre de toute fonction d'E/S ou potentiellement longue.
- `ctx, cancel := context.With…(ctx, …)` suivi de `defer cancel()`.
- Propager le contexte reçu ; ne créer de racine qu'à l'entrée du programme (`main`, test, serveur).
- `select` avec `<-ctx.Done()` dans toute attente.
- `WithCancelCause` / `Cause` pour des erreurs d'annulation explicites.
- Valeurs de contexte : données de requête seulement, clés de type non exporté, accesseurs typés.

## Ce que je dois retenir

- `ctx.Done()` est un channel fermé à l'annulation ; `ctx.Err()` dit pourquoi (`Canceled`/`DeadlineExceeded`).
- Contextes en **arbre** : annuler un parent annule les descendants.
- `WithCancel`, `WithTimeout`, `WithDeadline`, `…Cause` ; **toujours** `defer cancel()`.
- Un contexte **demande** l'arrêt ; la fonction doit écouter `Done()`.
- Premier paramètre, jamais stocké, jamais nil, toujours propagé.
- `WithValue` : rarement, avec une clé typée non exportée.

---

➡️ [Exercices](exercices.md)
