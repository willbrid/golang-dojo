# Leçon 3 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — L'ordre

Avec `Recover` à l'**extérieur** de `Logging` : la panique traverse `Logging` sans qu'il ait le temps
d'écrire sa ligne — **aucune** entrée de journal pour `/boom`. Le client reçoit bien le 500 (`Recover`
répond), mais la requête est invisible dans les journaux d'accès : précisément celle qu'on voudrait
voir. Test qui fige l'ordre : celui de l'exemple, qui vérifie la ligne `req-1 GET /boom 500 …`.

## E2 — Le 200 implicite

Le statut journalisé pour `/hello` devient **0** : le handler écrit avec `fmt.Fprintf` sans jamais
appeler `WriteHeader`. Les tests côté client passent quand même parce que le **vrai**
`ResponseWriter` envoie bien un 200 implicite ; seule la copie du middleware est fausse. D'où
l'importance de tester le **journal** lui-même.

## E3 — `Unwrap`

Sans `Unwrap`, `rc.Flush()` rend une erreur `http.ErrNotSupported` (« feature not supported ») :
`ResponseController` ne trouve ni méthode `Flush` sur l'enveloppe, ni moyen de descendre jusqu'au
`ResponseWriter` d'origine. Le compte à rebours n'arrive au client qu'à la fin (ou le handler s'arrête,
selon qu'il teste l'erreur). Avec `Unwrap`, tout fonctionne : c'est **une ligne** à ne jamais oublier.

## E4 — Délai par requête

```go
func Timeout(d time.Duration) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			ctx, cancel := context.WithTimeout(r.Context(), d)
			defer cancel()
			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}
```

Cette version ne fait que **demander** l'arrêt : un handler qui ignore `ctx` continue jusqu'au bout
(N6 L8). `http.TimeoutHandler` fait plus : il exécute le handler dans une goroutine avec une réponse
**tamponnée**, et si le délai expire, il répond lui-même **503** avec un corps fixé, et rejette les
écritures ultérieures du handler (`http.ErrHandlerTimeout`). Le client obtient une réponse à temps
**même si** le handler ignore son contexte — au prix d'une goroutine qui continue de tourner pour rien,
et de l'impossibilité de streamer.

## E5 — Un middleware qui refuse

Placé **avant** `Logging` (plus extérieur), il refuse sans que la requête soit journalisée. Placé
**après** (plus intérieur), le 400 apparaît dans le journal. En général on veut journaliser les
refus — ils signalent des clients mal configurés ou des attaques.

## Exercice intermédiaire — Métriques et limite de débit

**Motif de route.** `r.Pattern` n'est renseigné qu'**après** le routage, dans la requête que le routeur
passe au handler. Un middleware placé **autour** du routeur voit la requête **avant** : `r.Pattern` y
est vide. Solutions : lire le motif après `next.ServeHTTP` grâce à un objet partagé (le middleware place
un pointeur dans le contexte, le handler — ou un mini-middleware par route — y écrit le motif) ; ou
envelopper **chaque** route au moment de l'enregistrement. Le motif est préférable au chemin : avec
`r.URL.Path`, chaque identifiant crée une série distincte (`/notes/1`, `/notes/2`…) — l'explosion de
**cardinalité** qui fait tomber les systèmes de métriques (niveau 12).

```go
type limiters struct {
	mu   sync.Mutex
	byIP map[string]*entry // protégé par mu
	rate int
}

type entry struct {
	l        *Limiter          // N6 L4
	lastSeen time.Time
}

func RateLimit(ls *limiters, now func() time.Time) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			ip, _, err := net.SplitHostPort(r.RemoteAddr)
			if err != nil {
				ip = r.RemoteAddr
			}
			if !ls.get(ip, now()).TryAcquire() {
				w.Header().Set("Retry-After", "1")
				http.Error(w, "trop de requêtes", http.StatusTooManyRequests)
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}
```

- Le nettoyage parcourt la map sous verrou toutes les minutes et supprime (et **arrête**, `Stop`) les
  limiteurs inactifs depuis 5 min. Sans `Stop`, chaque limiteur de N6 L4 garde sa goroutine de recharge :
  une fuite par IP vue. Un limiteur sans goroutine (`golang.org/x/time/rate`) évite ce piège.
- Derrière un reverse proxy, `r.RemoteAddr` est l'adresse **du proxy** : tous les clients partagent la
  même limite. Il faut lire l'en-tête `X-Forwarded-For` — mais seulement s'il vient d'un proxy de
  confiance, sinon n'importe quel client le falsifie (leçon 9).

## Défi — Compression

```go
type gzipWriter struct {
	http.ResponseWriter
	gz *gzip.Writer
}

func (g *gzipWriter) Write(p []byte) (int, error) { return g.gz.Write(p) }
func (g *gzipWriter) Unwrap() http.ResponseWriter   { return g.ResponseWriter }
func (g *gzipWriter) Flush() error {
	if err := g.gz.Flush(); err != nil {           // vider le compresseur…
		return err
	}
	return http.NewResponseController(g.ResponseWriter).Flush()   // …puis la connexion
}

func Gzip(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Add("Vary", "Accept-Encoding")
		if !strings.Contains(r.Header.Get("Accept-Encoding"), "gzip") {
			next.ServeHTTP(w, r)
			return
		}
		w.Header().Set("Content-Encoding", "gzip")
		w.Header().Del("Content-Length")
		gz := gzip.NewWriter(w)
		defer gz.Close()                              // écrit la fin du flux gzip
		next.ServeHTTP(&gzipWriter{ResponseWriter: w, gz: gz}, r)
	})
}
```

- `Vary: Accept-Encoding` dit aux caches que la réponse dépend de cet en-tête : sans lui, un cache
  pourrait servir la version compressée à un client qui ne sait pas la lire.
- Décider **avant** d'écrire : impossible de connaître la taille ou le type de la réponse avant les
  premiers octets. Les implémentations sérieuses **retardent** la décision : elles tamponnent les
  premiers octets (jusqu'à 1 Kio ou jusqu'au `WriteHeader`), regardent `Content-Type` et la taille, puis
  choisissent. C'est nettement plus complexe que cette version — d'où l'intérêt d'une bibliothèque
  éprouvée en production.
- `Flush` doit vider **le compresseur** d'abord, sinon les données restent dans son buffer interne.
- `gz.Close()` en `defer` dans le middleware : il écrit le pied du flux gzip après la fin du handler.

## Quiz

1. `func(http.Handler) http.Handler`.
2. A dans les deux cas.
3. Pour que la panique, convertie en 500, soit vue et journalisée par la journalisation.
4. En enveloppant le `ResponseWriter` dans un type qui intercepte `WriteHeader` (et `Write` pour le 200
   implicite).
5. À permettre à `http.ResponseController` d'atteindre le `ResponseWriter` d'origine (Flush, délais).
6. Par le contexte de la requête, avec une clé de type non exporté.
7. Elle rend une copie superficielle de la requête avec un autre contexte.
8. Une panique volontaire qui signifie « abandonner la réponse » ; elle doit être relancée.
9. C'est une entrée client : sans borne, elle peut remplir les journaux ou servir d'injection.
10. Il est consommé : le handler lit un corps vide, sauf si le middleware le remplace par une copie
    (`io.NopCloser(bytes.NewReader(b))`).
