# Leçon 9 — Serveur en production

> `http.ListenAndServe(":8080", mux)` suffit pour un exemple. Exposé à Internet, ce serveur tombe
> devant un seul client malveillant qui ouvre des milliers de connexions et envoie ses en-têtes un octet
> toutes les dix secondes (*slowloris*). Cette leçon rassemble ce qu'un serveur Go doit configurer avant
> d'être exposé : délais, limites de taille, en-têtes de sécurité — et la façon de se placer derrière (ou
> d'écrire) un **reverse proxy** sans se faire tromper sur l'adresse du client.

## Objectifs

1. Configurer les quatre délais de `http.Server` et savoir ce que chacun borne.
2. Limiter la taille des en-têtes et des corps, y compris le nombre de valeurs d'en-têtes (Go 1.27).
3. Ajouter les en-têtes de sécurité de base.
4. Écrire un reverse proxy avec `httputil.ReverseProxy`, et déterminer l'adresse réelle du client.

## Explication

### Les quatre délais

```go
srv := &http.Server{
	ReadHeaderTimeout: 5 * time.Second,
	ReadTimeout:       30 * time.Second,
	WriteTimeout:      60 * time.Second,
	IdleTimeout:       120 * time.Second,
}
```

| Délai | Borne | Protège contre |
|---|---|---|
| `ReadHeaderTimeout` | la lecture des en-têtes | *slowloris* : des en-têtes envoyés au compte-gouttes qui gardent des connexions ouvertes |
| `ReadTimeout` | en-têtes **et** corps | un corps envoyé au compte-gouttes |
| `WriteTimeout` | de la fin des en-têtes à la fin de la réponse | un client qui ne lit jamais ; un handler trop long |
| `IdleTimeout` | une connexion *keep-alive* inactive | l'accumulation de connexions dormantes |

Tous valent **zéro** par défaut, c'est-à-dire **aucun délai**. `go vet` ne le signale pas ; certains
linters oui (`gosec`). `ReadHeaderTimeout` est le minimum absolu.

Ces délais sont **globaux** : un `WriteTimeout` de 60 s coupe aussi un téléchargement légitime de 10
minutes. Pour une route particulière, `http.NewResponseController(w).SetWriteDeadline(…)` (leçon 2)
ajuste le délai de **cette** requête. Et le budget **métier** d'une requête reste le contexte (leçon 7).

### Les limites de taille

| Limite | Réglage | Au-delà |
|---|---|---|
| taille totale des en-têtes | `MaxHeaderBytes` (défaut 1 Mio) | 431 |
| **nombre** de valeurs d'en-têtes | `MaxHeaderValueCount` (**Go 1.27**, défaut 500) | 431 |
| taille du corps | `http.MaxBytesReader` par handler (leçon 2) | 413 |

`MaxHeaderValueCount` ferme une attaque par épuisement : des milliers de petites lignes d'en-tête,
chacune sous la limite de taille, qui coûtent au serveur une allocation chacune. Une valeur répétée sur
plusieurs lignes compte plusieurs fois ; des valeurs séparées par des virgules sur une seule ligne
comptent une fois.

### Les en-têtes de sécurité

| En-tête | Effet |
|---|---|
| `X-Content-Type-Options: nosniff` | le navigateur ne devine pas le type (un JSON exécuté comme script) |
| `Content-Security-Policy` | d'où la page peut charger scripts, styles, images — la défense principale contre le XSS |
| `X-Frame-Options: DENY` (ou CSP `frame-ancestors`) | pas d'affichage dans une iframe (*clickjacking*) |
| `Strict-Transport-Security` | le navigateur n'utilisera plus que HTTPS pour ce domaine (seulement sur une réponse HTTPS) |
| `Referrer-Policy` | ce que le navigateur envoie comme `Referer` vers d'autres sites |

Un middleware (leçon 3) les ajoute à toutes les réponses. Pour une API JSON pure, `nosniff` et une CSP
restrictive suffisent ; pour des pages HTML, la CSP demande un vrai travail de réglage.

### Derrière un reverse proxy

En production, le serveur Go est souvent derrière un répartiteur de charge ou un reverse proxy (nginx,
un *ingress* Kubernetes, un CDN) qui termine TLS. Conséquences :

- `r.RemoteAddr` est l'adresse **du proxy**, pas du client ;
- le proxy transmet l'adresse du client dans `X-Forwarded-For` (`client, proxy1, proxy2`) ;
- **n'importe quel client peut envoyer un faux `X-Forwarded-For`**.

La seule lecture sûre : ne croire l'en-tête que si la connexion vient d'un **proxy de confiance**
(liste d'adresses connues), et le lire **depuis la droite** en sautant nos propres proxys — la première
adresse qui n'est pas à nous est celle du client. Tout ce qui est plus à gauche a pu être inventé par le
client. Se tromper ici, c'est permettre de contourner les limites de débit par IP (leçon 3) ou de
falsifier les journaux.

### Écrire un reverse proxy : `httputil.ReverseProxy`

```go
proxy := &httputil.ReverseProxy{
	Rewrite: func(pr *httputil.ProxyRequest) {
		pr.SetURL(target)     // destination
		pr.SetXForwarded()    // X-Forwarded-For/Host/Proto
	},
}
```

C'est un `http.Handler` complet : il gère la transmission en flux, les en-têtes « hop-by-hop » à ne
pas transmettre, HTTP/2, les WebSockets. Utiliser `Rewrite` (Go 1.20) plutôt que l'ancien `Director` :
`Rewrite` retire d'abord les en-têtes de transfert reçus du client, ce qui évite d'en relayer de faux.

### *Open redirect* et *SSRF*

Deux failles liées aux URL :

- **redirection ouverte** : `GET /login?next=https://site-malveillant` puis `http.Redirect(w, r, next,
  …)` — n'accepter que des chemins **relatifs** internes ;
- **SSRF** (*server-side request forgery*) : un service qui va chercher une URL fournie par
  l'utilisateur (aperçu de lien, *webhook*) peut être dirigé vers `http://169.254.169.254/` (les
  métadonnées du cloud) ou vers des services internes. Valider la destination (et l'adresse **résolue**,
  pas seulement le nom).

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : prod/prod.go
// Package prod rassemble la configuration d'un serveur HTTP exposé.
package prod

import (
	"net"
	"net/http"
	"net/http/httputil"
	"net/netip"
	"net/url"
	"strings"
	"time"
)

// NewServer : les délais et limites qu'un serveur exposé à Internet doit TOUJOURS avoir.
func NewServer(addr string, h http.Handler) *http.Server {
	return &http.Server{
		Addr:                addr,
		Handler:             SecurityHeaders(h),
		ReadHeaderTimeout:   5 * time.Second,   // en-têtes complets en 5 s : contre les clients « au compte-gouttes »
		ReadTimeout:         30 * time.Second,  // en-têtes + corps
		WriteTimeout:        60 * time.Second,  // jusqu'à la fin de l'écriture de la réponse
		IdleTimeout:         120 * time.Second, // connexion keep-alive inactive
		MaxHeaderBytes:      64 << 10,          // 64 Kio d'en-têtes (défaut : 1 Mio)
		MaxHeaderValueCount: 100,               // Go 1.27 : nombre de valeurs d'en-têtes (défaut : 500)
	}
}

// SecurityHeaders ajoute les en-têtes de sécurité de base à toutes les réponses.
func SecurityHeaders(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		h := w.Header()
		h.Set("X-Content-Type-Options", "nosniff") // pas de devinette de type (leçon 2)
		h.Set("X-Frame-Options", "DENY")           // pas d'affichage dans une iframe (clickjacking)
		h.Set("Referrer-Policy", "strict-origin-when-cross-origin")
		h.Set("Content-Security-Policy", "default-src 'self'")
		if r.TLS != nil {
			h.Set("Strict-Transport-Security", "max-age=63072000; includeSubDomains")
		}
		next.ServeHTTP(w, r)
	})
}

// ClientIP rend l'adresse du client. X-Forwarded-For n'est cru QUE si la connexion vient
// d'un proxy de confiance — sinon n'importe quel client pourrait l'inventer.
func ClientIP(r *http.Request, trusted []netip.Prefix) netip.Addr {
	host, _, _ := net.SplitHostPort(r.RemoteAddr)
	peer, err := netip.ParseAddr(host)
	if err != nil {
		return netip.Addr{}
	}
	if !isTrusted(peer, trusted) {
		return peer
	}
	// X-Forwarded-For : "client, proxy1, proxy2". On remonte depuis la DROITE
	// et on s'arrête à la première adresse qui n'est pas un de nos proxys.
	parts := strings.Split(r.Header.Get("X-Forwarded-For"), ",")
	for i := len(parts) - 1; i >= 0; i-- {
		a, err := netip.ParseAddr(strings.TrimSpace(parts[i]))
		if err != nil {
			return peer
		}
		if !isTrusted(a, trusted) {
			return a
		}
	}
	return peer
}

func isTrusted(a netip.Addr, trusted []netip.Prefix) bool {
	for _, p := range trusted {
		if p.Contains(a) {
			return true
		}
	}
	return false
}

// NewProxy est un reverse proxy vers target, qui transmet l'adresse du client.
func NewProxy(target *url.URL) *httputil.ReverseProxy {
	return &httputil.ReverseProxy{
		Rewrite: func(pr *httputil.ProxyRequest) {
			pr.SetURL(target)  // schéma, hôte, chemin de base
			pr.SetXForwarded() // X-Forwarded-For/Host/Proto
		},
	}
}
```

```go
// Fichier : prod/prod_test.go
package prod

import (
	"bufio"
	"fmt"
	"io"
	"net"
	"net/http"
	"net/http/httptest"
	"net/netip"
	"net/url"
	"strings"
	"testing"
	"time"
)

func TestHeaderValueCount(t *testing.T) {
	srv := httptest.NewUnstartedServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {}))
	srv.Config.MaxHeaderValueCount = 10
	srv.Start()
	defer srv.Close()
	req, _ := http.NewRequest("GET", srv.URL, nil)
	for i := range 20 {
		req.Header.Add("X-Bourrage", fmt.Sprint(i)) // 20 lignes d'en-tête
	}
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		t.Fatal(err)
	}
	resp.Body.Close()
	if resp.StatusCode != http.StatusRequestHeaderFieldsTooLarge {
		t.Errorf("statut = %d, attendu 431", resp.StatusCode)
	}
}

func TestSlowHeaders(t *testing.T) {
	srv := httptest.NewUnstartedServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {}))
	srv.Config.ReadHeaderTimeout = 200 * time.Millisecond
	srv.Start()
	defer srv.Close()
	conn, err := net.Dial("tcp", srv.Listener.Addr().String())
	if err != nil {
		t.Fatal(err)
	}
	defer conn.Close()
	fmt.Fprint(conn, "GET / HTTP/1.1\r\nHost: x\r\n") // en-têtes JAMAIS terminés (pas de ligne vide)
	start := time.Now()
	conn.SetReadDeadline(time.Now().Add(5 * time.Second))
	_, err = bufio.NewReader(conn).ReadString('\n') // le serveur doit couper la connexion
	if err == nil {
		t.Fatal("le serveur a répondu au lieu de couper")
	}
	if d := time.Since(start); d > 2*time.Second {
		t.Errorf("connexion coupée après %v : ReadHeaderTimeout non appliqué", d)
	}
}

func TestClientIP(t *testing.T) {
	trusted := []netip.Prefix{netip.MustParsePrefix("10.0.0.0/8")}
	tests := []struct {
		name, remote, xff, want string
	}{
		{"direct", "203.0.113.5:4000", "", "203.0.113.5"},
		{"XFF ignoré sans proxy de confiance", "203.0.113.5:4000", "1.2.3.4", "203.0.113.5"},
		{"derrière notre proxy", "10.0.0.2:4000", "198.51.100.7", "198.51.100.7"},
		{"usurpation à gauche ignorée", "10.0.0.2:4000", "6.6.6.6, 198.51.100.7", "198.51.100.7"},
		{"deux proxys internes", "10.0.0.2:4000", "198.51.100.7, 10.0.0.9", "198.51.100.7"},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			r := httptest.NewRequest("GET", "/", nil)
			r.RemoteAddr = tt.remote
			if tt.xff != "" {
				r.Header.Set("X-Forwarded-For", tt.xff)
			}
			if got := ClientIP(r, trusted); got.String() != tt.want {
				t.Errorf("= %s, attendu %s", got, tt.want)
			}
		})
	}
}

func TestProxy(t *testing.T) {
	backend := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "%s %s xff=%q", r.Method, r.URL.Path, r.Header.Get("X-Forwarded-For"))
	}))
	defer backend.Close()
	u, _ := url.Parse(backend.URL + "/api")
	front := httptest.NewServer(SecurityHeaders(NewProxy(u)))
	defer front.Close()

	resp, err := http.Get(front.URL + "/users")
	if err != nil {
		t.Fatal(err)
	}
	defer resp.Body.Close()
	b, _ := io.ReadAll(resp.Body)
	if got := string(b); got != `GET /api/users xff="127.0.0.1"` {
		t.Errorf("corps = %s", got)
	}
	if resp.Header.Get("X-Content-Type-Options") != "nosniff" || !strings.Contains(resp.Header.Get("Content-Security-Policy"), "self") {
		t.Errorf("en-têtes de sécurité absents : %v", resp.Header)
	}
}
```

`go test -race -v ./prod` : les quatre tests passent. `TestSlowHeaders` montre une connexion coupée
après 200 ms au lieu de rester ouverte indéfiniment ; `TestHeaderValueCount` obtient bien un **431** au
vingtième en-tête avec une limite de 10.

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `NewServer` | Une fonction unique qui fixe délais et limites : aucun serveur du projet n'est créé sans. |
| `MaxHeaderValueCount: 100` | Nouveauté 1.27 ; le test vérifie le 431. |
| `TestSlowHeaders` avec `net.Dial` | Un client « slowloris » minimal : des en-têtes jamais terminés. Le serveur coupe au bout de `ReadHeaderTimeout`. |
| `SecurityHeaders` | HSTS seulement si la requête est arrivée en TLS (`r.TLS != nil`). |
| `ClientIP` lit `X-Forwarded-For` depuis la droite | Le cas « usurpation à gauche » montre qu'un `6.6.6.6` ajouté par le client est ignoré. |
| `netip.Prefix` | Le paquet `net/netip` (Go 1.18) : des adresses comparables, sans allocation. |
| `ReverseProxy` avec `Rewrite` + `SetXForwarded` | Le serveur d'origine voit le chemin réécrit (`/api/users`) et l'adresse du client. |
| `SecurityHeaders(NewProxy(u))` | Un proxy est un handler comme un autre : on le compose avec des middlewares. |

## Erreurs fréquentes

1. **`http.ListenAndServe` sans délais** exposé à Internet.
2. **`WriteTimeout` trop court** qui coupe les téléchargements, ou trop long pour être utile — et la
   croyance qu'il annule le handler (il ne fait que couper l'écriture).
3. **Corps non bornés.**
4. **Croire `X-Forwarded-For`** sans vérifier que la connexion vient d'un proxy de confiance.
5. **Prendre la première adresse** (la plus à gauche) de `X-Forwarded-For`.
6. **HSTS sur HTTP**, ou une CSP copiée sans être testée (pages cassées).
7. **`Director` au lieu de `Rewrite`** dans un reverse proxy.
8. **Rediriger vers une URL fournie** par l'utilisateur.

## Bonnes pratiques Go

- Une fonction `NewServer` avec les quatre délais et les limites.
- `MaxBytesReader` dans chaque handler qui lit un corps.
- Un middleware d'en-têtes de sécurité.
- Une liste explicite de proxys de confiance ; lecture de `X-Forwarded-For` depuis la droite.
- `httputil.ReverseProxy` avec `Rewrite`.
- Des tests qui simulent les attaques (client lent, en-têtes en masse, en-têtes falsifiés).

## Ce que je dois retenir

- `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`, `IdleTimeout` : zéro par défaut = aucun délai.
- `MaxHeaderBytes`, `MaxHeaderValueCount` (1.27, défaut 500) → 431 ; corps borné → 413.
- En-têtes de sécurité : `nosniff`, CSP, `X-Frame-Options`, HSTS (en HTTPS).
- `X-Forwarded-For` : seulement depuis un proxy de confiance, lu depuis la droite.
- `httputil.ReverseProxy` + `Rewrite` + `SetXForwarded`.

---

➡️ [Exercices](exercices.md)
