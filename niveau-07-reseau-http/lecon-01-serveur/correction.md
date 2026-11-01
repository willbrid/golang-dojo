# Leçon 1 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Le routeur, prédit puis mesuré

| Chemin | Handler | Pourquoi |
|---|---|---|
| `/notes/new` | `new` | plus spécifique que `/notes/{id}` |
| `/notes/7` | `id` | |
| `/` | racine | `/{$}` : la racine exacte |
| `/x` | 404 | `/{$}` ne correspond **qu'à** `/` |
| `/files/a/b/c` | `files` | `{path...}` prend tout le reste |

Ajouter `"GET /{a}/x"` fait **paniquer** l'enregistrement, avec un message remarquablement précis
(mesuré) :

```
pattern "GET /{a}/x" (registered at …) conflicts with pattern "GET /files/{path...}" (registered at …):
GET /{a}/x and GET /files/{path...} both match some paths, like "/files/x".
But neither is more specific than the other.
GET /{a}/x matches "/a/x", but GET /files/{path...} doesn't.
```

Il donne un chemin ambigu (`/files/x`) et la raison. L'erreur survient **au démarrage** : un conflit
de routes ne peut pas atteindre la production.

## E2 — `HandlerFunc`

```go
type HandlerFunc func(http.ResponseWriter, *http.Request)

func (f HandlerFunc) ServeHTTP(w http.ResponseWriter, r *http.Request) { f(w, r) }

type Static string

func (s Static) ServeHTTP(w http.ResponseWriter, _ *http.Request) { io.WriteString(w, string(s)) }
```

Les fonctions (ou valeurs de méthode) pour les routes ordinaires ; un **type** quand le handler porte
une configuration ou un état propre et qu'on veut le construire, le réutiliser, le tester seul
(`http.FileServer`, un proxy, un middleware configurable).

## E3 — Une goroutine par requête

Le compteur sans verrou rend des valeurs dupliquées et un total faux, et `-race` signale la race :
chaque requête s'exécute dans sa goroutine. Avec `atomic.Int64` (`n := s.hits.Add(1)`), le total est
exact. C'est la règle du niveau 6 appliquée au HTTP : **tout handler est du code concurrent**.

## E4 — Panique dans un handler

- Avec `httptest.NewServer` : le client reçoit une **erreur** (`EOF` ou connexion fermée) — le serveur
  a récupéré la panique, journalisé `http: panic serving 127.0.0.1:…: boum` avec la pile sur la sortie
  d'erreur, et fermé **cette** connexion. Les requêtes suivantes sont servies normalement.
- Avec `httptest.NewRecorder` : la panique **remonte dans le test** et le fait échouer (ou tue le
  binaire de test). Il n'y a pas de serveur pour la récupérer : on appelle le handler directement.

La protection vient du **serveur** `net/http`, pas du handler. Un handler appelé hors serveur (dans un
worker, un test) n'est pas protégé — et le serveur ne renvoie aucune réponse propre (pas de 500) : la
leçon 3 ajoutera un middleware de récupération qui répond un vrai 500.

## E5 — Fichiers statiques

```go
//go:embed public
var publicFS embed.FS

sub, _ := fs.Sub(publicFS, "public")
mux.Handle("GET /static/", http.StripPrefix("/static/", http.FileServerFS(sub)))
```

- `/static/` affiche un **listage** du répertoire (s'il n'y a pas d'`index.html`). Pour l'empêcher :
  envelopper le `fs.FS` pour que l'ouverture d'un répertoire échoue, ou ajouter un `index.html` vide,
  ou servir seulement des fichiers connus.
- `/static/../go.mod` n'est **pas** servi : le serveur nettoie le chemin et le `fs.FS` (embarqué ou
  `os.DirFS`) ne permet pas de sortir de sa racine (N4 L7). `http.StripPrefix` est indispensable :
  sans lui, le `FileServer` chercherait `static/…` dans le système de fichiers.

## Exercice intermédiaire — Raccourcisseur d'URL

```go
type Shortener struct {
	mu     sync.Mutex
	links  map[string]*link // protégé par mu
	newCode func() string
	mux    *http.ServeMux
}

type link struct {
	target string
	hits   atomic.Int64
}

func (s *Shortener) create(w http.ResponseWriter, r *http.Request) {
	b, err := io.ReadAll(io.LimitReader(r.Body, 2048))
	if err != nil {
		http.Error(w, "corps illisible", http.StatusBadRequest)
		return
	}
	u, err := url.Parse(strings.TrimSpace(string(b)))
	if err != nil || (u.Scheme != "http" && u.Scheme != "https") || u.Host == "" {
		http.Error(w, "URL invalide : schéma http(s) et hôte requis", http.StatusBadRequest)
		return
	}
	s.mu.Lock()
	code := s.newCode()
	for _, taken := s.links[code]; taken; _, taken = s.links[code] {
		code = s.newCode()                     // collision : on retire
	}
	s.links[code] = &link{target: u.String()}
	s.mu.Unlock()
	w.Header().Set("Location", "/l/"+code)
	w.WriteHeader(http.StatusCreated)
	io.WriteString(w, code)
}

func (s *Shortener) follow(w http.ResponseWriter, r *http.Request) {
	s.mu.Lock()
	l, ok := s.links[r.PathValue("code")]
	s.mu.Unlock()
	if !ok {
		http.NotFound(w, r)
		return
	}
	l.hits.Add(1)                              // atomique : pas besoin du verrou global
	http.Redirect(w, r, l.target, http.StatusFound)
}
```

- `http.Redirect` pose l'en-tête `Location` et le statut ; pour un 302, le client suit le lien.
- Le compteur par lien est **atomique** : le verrou global ne protège que la map. `-race` silencieux
  sous 1 000 redirections concurrentes, total exact.
- Générateur injecté : en test, une fonction qui rend `"aaaaaa"`, puis `"aaaaaa"` à nouveau, puis
  `"bbbbbb"` permet de tester la **collision** de façon déterministe.
- Validation stricte : `javascript:alert(1)` ou `ftp://…` sont refusés — sinon le service servirait de
  redirecteur vers n'importe quoi (*open redirect*, leçon 9).

## Défi — Lire le code du routeur

1. Règles non vues : un motif avec **hôte** est prioritaire ; `GET` correspond aussi à `HEAD` ; un
   motif qui finit par `/` correspond à tout le sous-arbre ; les chemins sont nettoyés (`//`, `/./`,
   `/../`) avec une **redirection** 301 vers la forme propre ; les segments sont comparés **décodés**
   (`%2F` ne coupe pas un segment).
2. `GODEBUG=httpmuxgo121=1` restaure le routeur d'avant Go 1.22 : utile seulement pour un programme
   ancien dont des motifs contenant des accolades ou des méthodes avaient un autre sens.
3. ```go
   mux.HandleFunc("api.local/v1/", h("hôte"))
   mux.HandleFunc("/v1/", h("sans hôte"))
   req := httptest.NewRequest("GET", "http://api.local/v1/x", nil)
   // → "hôte"
   ```
4. `r.PathValue("idd")` (faute de frappe) rend `""` sans erreur. Parades : un test par route qui vérifie
   l'identifiant reçu ; ou des constantes partagées entre l'enregistrement et la lecture. Le
   compilateur ne peut pas vérifier la cohérence entre une chaîne de motif et un nom de joker.

## Quiz

1. `ServeHTTP(http.ResponseWriter, *http.Request)`.
2. À utiliser une fonction ordinaire comme `Handler`.
3. Restreindre à une méthode et capturer un segment de chemin, lu par `r.PathValue("id")`.
4. Il répond 405 avec un en-tête `Allow`.
5. `{$}` : fin exacte du chemin (la racine seule pour `/{$}`) ; `{path...}` : tout le reste du chemin.
6. Le plus spécifique.
7. L'enregistrement panique, avec un message qui donne un chemin ambigu.
8. Dans la goroutine de sa connexion : les handlers sont concurrents, l'état partagé doit être protégé.
9. `NewRecorder` appelle le handler directement, sans réseau ; `NewServer` démarre un vrai serveur
   local accessible par son URL.
10. Il n'a aucun délai (lecture, écriture, inactivité) : un client lent ou malveillant peut garder des
    connexions ouvertes indéfiniment (leçon 9).
