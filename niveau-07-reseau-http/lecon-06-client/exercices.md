# Leçon 6 — Exercices

Code dans `mes-solutions/niveau-07/lecon-06/`.

## Exercices faciles

**E1 — Le serveur qui ne répond jamais.**
Avec `httptest.NewServer`, un handler qui attend `r.Context().Done()` sans jamais répondre. L'appeler
avec `http.Get` dans une goroutine : combien de temps bloque-t-elle ? Puis avec un client
`Timeout: 500 * time.Millisecond` : quelle erreur ? Tester `errors.Is(err, context.DeadlineExceeded)`
et `os.IsTimeout(err)` — lesquels sont vrais ?

**E2 — Réutilisation des connexions.**
Reproduire la mesure du cours (`srv.Config.ConnState` pour compter les connexions). Ajouter le cas
« un `http.Client` neuf par requête » : combien de connexions ?

**E3 — Un 500 n'est pas une erreur.**
Appeler un serveur qui répond 500 avec `client.Get`. Afficher `err` et `resp.StatusCode`. Écrire la
fonction `checkStatus(resp *http.Response) error` qui rend une `*StatusError` avec les 200 premiers
octets du corps.

**E4 — Encoder une URL.**
Construire `https://api.exemple.fr/search?q=café & thé&tag=a&tag=b` correctement avec `url.URL` et
`url.Values`. Afficher le résultat. Que donnerait une simple concaténation de chaînes, et que recevrait
le serveur ?

**E5 — `Retry-After`.**
Étendre le client de l'exemple : sur un 429 avec `Retry-After: 2`, attendre 2 secondes (et non le
*backoff*). Gérer les deux formes de l'en-tête (secondes ou date HTTP — `http.ParseTime`). Plafonner
l'attente à 30 secondes.

---

## Exercice intermédiaire — Un client pour l'API de tâches

Écrire un paquet `taskclient` pour l'API de la leçon 4 :

```go
func New(baseURL string, opts ...Option) *Client
func (c *Client) Create(ctx context.Context, title string) (Task, error)
func (c *Client) Get(ctx context.Context, id int) (Task, error)
func (c *Client) List(ctx context.Context) iter.Seq2[Task, error]   // suit les curseurs (N3 L5)
func (c *Client) SetDone(ctx context.Context, id int, done bool) (Task, error)
```

**Contraintes :**
1. Options fonctionnelles (N1 L11) : délai, nombre de réessais, `*http.Client` fourni par l'appelant.
2. Les erreurs *problem details* (leçon 5) sont décodées en une erreur typée `*APIError{Status, Type,
   Title}` ; un 404 se teste avec `errors.Is(err, taskclient.ErrNotFound)` (implémenter une méthode
   `Is` sur `*APIError`).
3. `Create` (un `POST`) n'est **pas** réessayé… sauf si l'API acceptait une clé d'idempotence :
   l'ajouter en option.
4. `List` est un **itérateur** qui charge les pages au fur et à mesure et s'arrête si le consommateur
   fait `break` (sans charger la page suivante).
5. Tests contre le **vrai** handler de la leçon 4 servi par `httptest.NewServer`.

---

## Défi — Le `Transport` sous la loupe

1. Lire `go doc net/http.Transport`. Lister les délais qu'il propose (connexion, TLS, en-têtes de
   réponse, inactivité) et dire ce que chacun borne.
2. Utiliser `net/http/httptrace` pour afficher, pour une requête, les instants de : résolution DNS,
   connexion, connexion réutilisée ou non, premier octet de réponse.
3. Avec `MaxIdleConnsPerHost` à sa valeur par défaut (2), lancer 50 requêtes **concurrentes** vers le
   même hôte, deux fois de suite. Combien de connexions la seconde vague ouvre-t-elle ? Recommencer
   avec `MaxIdleConnsPerHost: 50`. Expliquer.

---

## Quiz

1. Pourquoi ne jamais utiliser `http.DefaultClient` en production ?
2. Que borne `Client.Timeout` ?
3. Pourquoi réutiliser un même `http.Client` ?
4. `client.Do` rend-il une erreur pour un statut 500 ?
5. Que se passe-t-il si on ne ferme pas `resp.Body` ?
6. Qu'est-ce qui a changé en Go 1.27 concernant la fermeture du corps ?
7. Quelles erreurs et quels statuts peut-on réessayer ?
8. Pourquoi ajouter du *jitter* au *backoff* ?
9. Pourquoi ne pas réessayer un `POST` sans précaution ?
10. Comment encoder correctement des paramètres d'URL ?
