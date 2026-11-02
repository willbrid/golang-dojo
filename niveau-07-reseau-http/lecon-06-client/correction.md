# Leçon 6 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Le serveur qui ne répond jamais

Avec `http.Get`, la goroutine bloque **indéfiniment** (jusqu'à la fermeture du serveur de test). Avec
`Timeout: 500 * time.Millisecond`, l'erreur ressemble à :

```
Get "http://127.0.0.1:…": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

`os.IsTimeout(err)` est **vrai** (l'erreur implémente `Timeout() bool`), et `errors.Is(err,
context.DeadlineExceeded)` l'est aussi dans les versions récentes. Pour un code portable, préférer
`errors.As(err, &netErr)` avec un `net.Error` et `netErr.Timeout()`.

## E2 — Réutilisation des connexions

Avec un `http.Client` neuf par requête (chacun avec son propre `Transport`) : **20 connexions** pour
20 requêtes, même en fermant les corps. Le pool appartient au `Transport` ; un client neuf (avec
`&http.Transport{}`) n'en hérite pas. Note : `&http.Client{}` sans `Transport` utilise
`http.DefaultTransport`, partagé — la réutilisation fonctionne alors, mais on hérite aussi de sa
configuration par défaut.

## E3 — Un 500 n'est pas une erreur

`err` vaut `nil` et `resp.StatusCode` vaut 500.

```go
func checkStatus(resp *http.Response) error {
	if resp.StatusCode >= 200 && resp.StatusCode < 300 {
		return nil
	}
	b, _ := io.ReadAll(io.LimitReader(resp.Body, 200))
	return &StatusError{Code: resp.StatusCode, Body: string(b)}
}
```

## E4 — Encoder une URL

```go
u := url.URL{Scheme: "https", Host: "api.exemple.fr", Path: "/search"}
q := url.Values{}
q.Set("q", "café & thé")
q.Add("tag", "a")
q.Add("tag", "b")
u.RawQuery = q.Encode()
fmt.Println(u.String())
// https://api.exemple.fr/search?q=caf%C3%A9+%26+th%C3%A9&tag=a&tag=b
```

Concaténé brut, le `&` de « café & thé » serait lu comme un **séparateur de paramètres** : le serveur
recevrait `q=café ` et un paramètre ` thé` sans valeur. Un attaquant qui contrôle la valeur peut ainsi
**injecter** des paramètres (`q=x&admin=true`). `url.Values.Encode` trie aussi les clés (sortie
déterministe).

## E5 — `Retry-After`

```go
func retryAfter(h http.Header, now time.Time) (time.Duration, bool) {
	v := h.Get("Retry-After")
	if v == "" {
		return 0, false
	}
	var d time.Duration
	if secs, err := strconv.Atoi(v); err == nil && secs >= 0 {
		d = time.Duration(secs) * time.Second
	} else if t, err := http.ParseTime(v); err == nil {
		d = t.Sub(now)
	} else {
		return 0, false
	}
	return min(max(d, 0), 30*time.Second), true
}
```

Il faut garder les en-têtes de la réponse dans la `StatusError` (ou retourner la durée avec l'erreur)
pour que la boucle de réessai les consulte. Le plafond protège d'un serveur qui demanderait d'attendre
une heure.

## Exercice intermédiaire — Un client pour l'API de tâches

Les deux points délicats :

```go
type APIError struct {
	Status int
	Type   string
	Title  string
}

func (e *APIError) Error() string { return fmt.Sprintf("%d %s", e.Status, e.Title) }

var ErrNotFound = errors.New("introuvable")

// Is permet errors.Is(err, ErrNotFound) sur n'importe quelle APIError 404.
func (e *APIError) Is(target error) bool { return target == ErrNotFound && e.Status == http.StatusNotFound }
```

```go
func (c *Client) List(ctx context.Context) iter.Seq2[Task, error] {
	return func(yield func(Task, error) bool) {
		cursor := ""
		for {
			page, err := c.page(ctx, cursor)
			if err != nil {
				yield(Task{}, err)
				return
			}
			for _, t := range page.Items {
				if !yield(t, nil) {
					return                     // break du consommateur : pas de page suivante
				}
			}
			if page.NextCursor == "" {
				return
			}
			cursor = page.NextCursor
		}
	}
}
```

- La méthode `Is` sur un type d'erreur (N2 L6) permet aux appelants de raisonner par **sentinelles**
  sans connaître le statut HTTP.
- L'itérateur charge **une page à la fois** ; un `break` après 3 tâches ne déclenche aucune requête de
  plus — à vérifier dans le test en comptant les appels au serveur.
- Tester contre le **vrai** handler (`httptest.NewServer(tasks.New())`) garantit que client et serveur
  s'accordent sur le contrat. C'est un test d'intégration léger, rapide, sans réseau externe.

## Défi — Le `Transport` sous la loupe

1. `DialContext` (avec un `net.Dialer{Timeout}` : connexion TCP), `TLSHandshakeTimeout` (poignée de
   main TLS), `ResponseHeaderTimeout` (attente des en-têtes après l'envoi de la requête),
   `ExpectContinueTimeout`, `IdleConnTimeout` (durée de vie d'une connexion inactive dans le pool). Aucun
   ne borne la **lecture du corps** : c'est le rôle de `Client.Timeout` ou du contexte.
2. ```go
   trace := &httptrace.ClientTrace{
   	DNSDone:              func(httptrace.DNSDoneInfo) { log("DNS") },
   	ConnectDone:          func(_, _ string, err error) { log("connexion") },
   	GotConn:              func(i httptrace.GotConnInfo) { log(fmt.Sprint("réutilisée=", i.Reused)) },
   	GotFirstResponseByte: func() { log("premier octet") },
   }
   req = req.WithContext(httptrace.WithClientTrace(req.Context(), trace))
   ```
3. Avec `MaxIdleConnsPerHost: 2`, la première vague ouvre ~50 connexions ; à la fin, le pool n'en garde
   que **2** et ferme les autres. La seconde vague en rouvre ~48. Avec 50, le pool garde les 50, et la
   seconde vague n'ouvre rien. La valeur par défaut convient à un navigateur, pas à un service qui
   parle beaucoup à un même hôte.

## Quiz

1. Il n'a aucun délai : un serveur distant muet bloque la goroutine pour toujours.
2. Toute la tentative : connexion, envoi, en-têtes, lecture du corps.
3. Pour profiter du pool de connexions de son `Transport` ; il est sûr en concurrence.
4. Non : seulement pour un problème de transport ; il faut tester `StatusCode`.
5. La connexion n'est jamais rendue au pool : nouvelles connexions à chaque requête, fuites de sockets.
6. `Close` draine lui-même le corps (asynchrone, jusqu'à une limite) : la lecture complète manuelle
   n'est plus nécessaire dans la plupart des cas.
7. Erreurs réseau (méthodes idempotentes), 429, 502/503/504, éventuellement 500 ; jamais les autres 4xx
   ni une annulation.
8. Pour que des clients qui ont échoué ensemble ne réessaient pas ensemble.
9. Il n'est pas idempotent : un réessai après une réponse perdue crée un doublon.
10. Avec `url.Values` (et `url.URL`), ou `url.QueryEscape` pour une valeur isolée.
