# Leçon 11 — TLS et WebSockets

> Deux sujets qui transforment une connexion : **TLS** la rend confidentielle et authentifiée, la
> **mise à niveau** (*upgrade*) la fait sortir du modèle requête-réponse de HTTP pour en faire un canal
> bidirectionnel — c'est ainsi que naissent les WebSockets. La bibliothèque standard de Go contient une
> implémentation TLS complète et moderne, y compris l'échange de clés **post-quantique**, activé par
> défaut.

## Objectifs

1. Comprendre ce que garantit TLS, et configurer un serveur et un client correctement.
2. Mettre en place le TLS **mutuel** (mTLS) entre services.
3. Savoir ce qu'apporte l'échange de clés hybride post-quantique de Go.
4. Comprendre la mise à niveau de connexion (101, `Hijack`) sur laquelle reposent les WebSockets.

## Explication

### Ce que garantit TLS

- **Confidentialité** : personne sur le chemin ne lit les données.
- **Intégrité** : personne ne les modifie sans que ce soit détecté.
- **Authentification du serveur** : le client vérifie, grâce à un **certificat** signé par une
  **autorité de certification** (AC) en laquelle il a confiance, qu'il parle bien au bon serveur (le nom
  ou l'adresse doit figurer dans le certificat).

TLS ne protège **pas** ce qui se passe aux extrémités (un serveur compromis lit tout), et il
n'authentifie pas le **client** — sauf en TLS mutuel.

### Serveur et client

```go
srv := &http.Server{Addr: ":443", Handler: h, TLSConfig: &tls.Config{MinVersion: tls.VersionTLS12}}
srv.ListenAndServeTLS("cert.pem", "key.pem")
```

Les valeurs par défaut de `crypto/tls` sont **bonnes** : suites de chiffrement sûres, versions
obsolètes désactivées, TLS 1.3 privilégié. Les règles :

- ne **jamais** écrire `InsecureSkipVerify: true` hors d'un test jetable — c'est désactiver
  l'authentification du serveur, donc permettre l'interception ;
- pour faire confiance à une AC **interne**, la fournir dans `RootCAs` (client) — pas désactiver la
  vérification ;
- ne pas fixer de liste de suites de chiffrement « à la main » sans raison : on fige des choix qui
  vieilliront ;
- les certificats publics viennent d'une AC (Let's Encrypt ; `golang.org/x/crypto/acme/autocert` les
  obtient et les renouvelle automatiquement) ; en interne, d'une PKI d'entreprise.

En test, `httptest.NewTLSServer` fournit un serveur TLS et `srv.Client()` un client qui lui fait
confiance.

### Le TLS mutuel (mTLS)

Entre services, le **client** présente aussi un certificat, que le serveur vérifie :

```go
&tls.Config{
	ClientAuth: tls.RequireAndVerifyClientCert,
	ClientCAs:  pool,   // les AC habilitées à signer les certificats clients
}
```

L'identité du client devient son certificat (`r.TLS.PeerCertificates[0]`) : pas de mot de passe ni de
jeton à faire circuler. C'est le socle des architectures « zéro confiance » et des maillages de
services (*service mesh*). La difficulté n'est pas le code mais l'**exploitation** : émettre, renouveler
et révoquer les certificats de chaque service.

### Le post-quantique, par défaut

Un ordinateur quantique suffisamment puissant casserait les échanges de clés actuels (courbes
elliptiques). Le risque est **déjà** présent pour les données à longue durée de vie : un attaquant peut
enregistrer aujourd'hui des échanges chiffrés pour les déchiffrer plus tard (*harvest now, decrypt
later*).

Depuis Go 1.24, `crypto/tls` négocie par défaut un échange **hybride** : X25519 (classique) combiné à
ML-KEM-768 (algorithme post-quantique normalisé par le NIST, FIPS 203). La session reste sûre tant que
**l'un des deux** n'est pas cassé. Go 1.26 a ajouté d'autres combinaisons hybrides
(`SecP256r1MLKEM768`, `SecP384r1MLKEM1024`). Mesuré dans l'exemple : une connexion entre deux programmes
Go 1.27 négocie `X25519MLKEM768`, sans aucune configuration. `r.TLS.CurveID` (Go 1.25) permet de le
vérifier. Seules les **signatures** des certificats restent classiques : leur migration dépend des
autorités de certification.

### La mise à niveau de connexion

HTTP est requête-réponse : le serveur ne peut pas parler le premier. Pour un canal bidirectionnel
(messagerie, collaboration en temps réel, jeux), le client demande à **changer de protocole** sur la
connexion existante :

```
GET /chat HTTP/1.1
Connection: Upgrade
Upgrade: websocket
```

Le serveur répond `101 Switching Protocols`, et à partir de là, la connexion TCP n'est plus du HTTP.
En Go, le handler **détourne** la connexion :

```go
conn, brw, err := http.NewResponseController(w).Hijack()
```

Après `Hijack`, le serveur HTTP n'est plus responsable de rien : ni délais, ni fermeture, ni arrêt
gracieux (`Shutdown` ignore les connexions détournées, leçon 7). Côté client, après un 101, `resp.Body`
implémente aussi `io.Writer` : c'est un canal dans les deux sens.

### Les WebSockets

Le protocole WebSocket (RFC 6455) ajoute à ce mécanisme une poignée de main avec un en-tête de
contrôle, un **format de trames** (texte, binaire, ping, fermeture), le masquage des données du
client, et la gestion propre de la fermeture. La bibliothèque standard ne l'implémente pas ; on utilise
une bibliothèque, par exemple `github.com/coder/websocket` (maintenue, compatible `context`) ou
`github.com/gorilla/websocket`. Les règles de la leçon 9 s'y appliquent : vérifier l'**origine**
(`Origin`) pour éviter qu'un site tiers ouvre une WebSocket avec les cookies de l'utilisateur, borner la
taille des messages, poser des délais, envoyer des *pings*.

Si le serveur seul doit pousser des données (notifications, progression), les **Server-Sent Events**
suffisent : une réponse HTTP ordinaire en `text/event-stream`, streamée avec `Flush` (leçon 2), sans
mise à niveau.

## Exemple

```go
// Fichier : go.mod
module exemple

go 1.27
```

```go
// Fichier : secure/pki.go
// Package secure montre TLS mutuel et la mise à niveau d'une connexion HTTP.
package secure

import (
	"crypto/ecdsa"
	"crypto/elliptic"
	"crypto/rand"
	"crypto/tls"
	"crypto/x509"
	"crypto/x509/pkix"
	"math/big"
	"net"
	"time"
)

// PKI de TEST : une autorité et des certificats signés par elle. En production, les
// certificats viennent d'une vraie autorité (Let's Encrypt, PKI interne), jamais du code.
type PKI struct {
	CA     *x509.Certificate
	caKey  *ecdsa.PrivateKey
	CAPool *x509.CertPool
}

func NewPKI() (*PKI, error) {
	key, err := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
	if err != nil {
		return nil, err
	}
	tmpl := &x509.Certificate{
		SerialNumber:          big.NewInt(1),
		Subject:               pkix.Name{CommonName: "AC de test"},
		NotBefore:             time.Now().Add(-time.Hour),
		NotAfter:              time.Now().Add(24 * time.Hour),
		IsCA:                  true,
		KeyUsage:              x509.KeyUsageCertSign,
		BasicConstraintsValid: true,
	}
	der, err := x509.CreateCertificate(rand.Reader, tmpl, tmpl, &key.PublicKey, key)
	if err != nil {
		return nil, err
	}
	ca, err := x509.ParseCertificate(der)
	if err != nil {
		return nil, err
	}
	pool := x509.NewCertPool()
	pool.AddCert(ca)
	return &PKI{CA: ca, caKey: key, CAPool: pool}, nil
}

// Issue délivre un certificat pour un serveur (ip non nulle) ou un client (cn seulement).
func (p *PKI) Issue(cn string, ip net.IP, usage x509.ExtKeyUsage) (tls.Certificate, error) {
	key, err := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
	if err != nil {
		return tls.Certificate{}, err
	}
	serial, _ := rand.Int(rand.Reader, big.NewInt(1<<62))
	tmpl := &x509.Certificate{
		SerialNumber: serial,
		Subject:      pkix.Name{CommonName: cn},
		NotBefore:    time.Now().Add(-time.Hour),
		NotAfter:     time.Now().Add(time.Hour),
		KeyUsage:     x509.KeyUsageDigitalSignature,
		ExtKeyUsage:  []x509.ExtKeyUsage{usage},
	}
	if ip != nil {
		tmpl.IPAddresses = []net.IP{ip} // le client vérifie que l'adresse jointe figure dans le certificat
	}
	der, err := x509.CreateCertificate(rand.Reader, tmpl, p.CA, &key.PublicKey, p.caKey)
	if err != nil {
		return tls.Certificate{}, err
	}
	return tls.Certificate{Certificate: [][]byte{der}, PrivateKey: key}, nil
}

// ServerConfig : TLS 1.3 minimum, certificat client EXIGÉ et vérifié par l'AC (mTLS).
func ServerConfig(cert tls.Certificate, clientCAs *x509.CertPool) *tls.Config {
	return &tls.Config{
		Certificates: []tls.Certificate{cert},
		MinVersion:   tls.VersionTLS13,
		ClientAuth:   tls.RequireAndVerifyClientCert,
		ClientCAs:    clientCAs,
	}
}
```

```go
// Fichier : secure/upgrade.go
package secure

import (
	"bufio"
	"fmt"
	"io"
	"net/http"
	"strings"
)

// Upgrade bascule la connexion HTTP vers un protocole maison « shout/1 » :
// après le 101, chaque ligne reçue est renvoyée en majuscules. C'est le mécanisme
// qu'utilise WebSocket (avec, en plus, sa poignée de main et son format de trames).
func Upgrade(w http.ResponseWriter, r *http.Request) {
	if !strings.EqualFold(r.Header.Get("Upgrade"), "shout/1") {
		http.Error(w, "Upgrade: shout/1 attendu", http.StatusUpgradeRequired)
		return
	}
	conn, brw, err := http.NewResponseController(w).Hijack() // on reprend la connexion TCP brute
	if err != nil {
		http.Error(w, "détournement impossible", http.StatusInternalServerError)
		return
	}
	defer conn.Close() // après Hijack, c'est NOUS qui fermons : le serveur HTTP ne gère plus rien
	fmt.Fprint(brw, "HTTP/1.1 101 Switching Protocols\r\nConnection: Upgrade\r\nUpgrade: shout/1\r\n\r\n")
	brw.Flush()
	for {
		line, err := brw.ReadString('\n')
		if err != nil {
			return
		}
		brw.WriteString(strings.ToUpper(line))
		if err := brw.Flush(); err != nil {
			return
		}
	}
}

// Shout est le client : après le 101, resp.Body est un io.ReadWriteCloser sur la connexion.
func Shout(c *http.Client, url string, lines []string) ([]string, error) {
	req, _ := http.NewRequest("GET", url, nil)
	req.Header.Set("Connection", "Upgrade")
	req.Header.Set("Upgrade", "shout/1")
	resp, err := c.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusSwitchingProtocols {
		return nil, fmt.Errorf("statut %d", resp.StatusCode)
	}
	rw := resp.Body.(io.ReadWriteCloser)
	r := bufio.NewReader(rw)
	var out []string
	for _, l := range lines {
		if _, err := io.WriteString(rw, l+"\n"); err != nil {
			return out, err
		}
		reply, err := r.ReadString('\n')
		if err != nil {
			return out, err
		}
		out = append(out, strings.TrimSuffix(reply, "\n"))
	}
	return out, nil
}
```

```go
// Fichier : secure/secure_test.go
package secure

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"net"
	"net/http"
	"net/http/httptest"
	"slices"
	"testing"
)

func TestMutualTLS(t *testing.T) {
	pki, err := NewPKI()
	if err != nil {
		t.Fatal(err)
	}
	serverCert, _ := pki.Issue("serveur", net.ParseIP("127.0.0.1"), x509.ExtKeyUsageServerAuth)
	clientCert, _ := pki.Issue("service-facturation", nil, x509.ExtKeyUsageClientAuth)

	srv := httptest.NewUnstartedServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Avec mTLS, l'identité du client est son certificat, vérifié par TLS.
		fmt.Fprintf(w, "bonjour %s (TLS 1.%d, échange %v)", r.TLS.PeerCertificates[0].Subject.CommonName,
			r.TLS.Version-tls.VersionTLS10, r.TLS.CurveID)
	}))
	srv.TLS = ServerConfig(serverCert, pki.CAPool)
	srv.StartTLS()
	defer srv.Close()

	client := func(certs ...tls.Certificate) *http.Client {
		return &http.Client{Transport: &http.Transport{TLSClientConfig: &tls.Config{
			RootCAs:      pki.CAPool, // on fait confiance à NOTRE AC, pas à celles du système
			Certificates: certs,
		}}}
	}

	resp, err := client(clientCert).Get(srv.URL)
	if err != nil {
		t.Fatal(err)
	}
	var body [200]byte
	n, _ := resp.Body.Read(body[:])
	resp.Body.Close()
	t.Logf("%s", body[:n])
	if want := "bonjour service-facturation (TLS 1.3, échange X25519MLKEM768)"; string(body[:n]) != want {
		t.Errorf("= %q\nattendu %q", body[:n], want)
	}

	if _, err := client().Get(srv.URL); err == nil { // sans certificat client
		t.Error("connexion acceptée sans certificat client")
	}
	if _, err := http.Get(srv.URL); err == nil { // client ordinaire : notre AC lui est inconnue
		t.Error("un client qui ne connaît pas l'AC a fait confiance au serveur")
	}
}

func TestUpgrade(t *testing.T) {
	srv := httptest.NewServer(http.HandlerFunc(Upgrade))
	defer srv.Close()
	got, err := Shout(srv.Client(), srv.URL, []string{"bonjour", "fin"})
	if err != nil || !slices.Equal(got, []string{"BONJOUR", "FIN"}) {
		t.Errorf("= %q, %v", got, err)
	}
	resp, _ := http.Get(srv.URL) // sans en-tête Upgrade
	resp.Body.Close()
	if resp.StatusCode != http.StatusUpgradeRequired {
		t.Errorf("statut = %d, attendu 426", resp.StatusCode)
	}
}
```

Sortie de `go test -race -v ./secure` (extrait) :

```
secure_test.go:45: bonjour service-facturation (TLS 1.3, échange X25519MLKEM768)
--- PASS: TestMutualTLS (0.20s)
--- PASS: TestUpgrade (0.00s)
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `NewPKI` / `Issue` | Une autorité et des certificats fabriqués **pour les tests** avec `crypto/x509`. En production, ils viennent d'une vraie AC. |
| `IPAddresses: 127.0.0.1` | Le client vérifie que l'adresse jointe figure dans le certificat du serveur. |
| `ExtKeyUsage` serveur / client | Un certificat client ne peut pas servir à se faire passer pour un serveur, et inversement. |
| `RequireAndVerifyClientCert` + `ClientCAs` | mTLS : sans certificat signé par l'AC, la poignée de main échoue — le test le vérifie. |
| `RootCAs: pki.CAPool` | Le client fait confiance à **notre** AC, sans désactiver la vérification. |
| `http.Get` échoue | Un client ordinaire ne connaît pas notre AC : il refuse le serveur. C'est le comportement voulu. |
| `r.TLS.CurveID` → `X25519MLKEM768` | L'échange hybride post-quantique, négocié par défaut. |
| `Hijack` puis `101` écrit à la main | Après le détournement, on parle directement sur la connexion TCP. |
| `resp.Body.(io.ReadWriteCloser)` | Côté client, après un 101, le corps est un canal bidirectionnel. |
| 426 Upgrade Required | Réponse correcte quand le protocole demandé manque. |

## Erreurs fréquentes

1. **`InsecureSkipVerify: true`** « pour que ça marche ».
2. **Suites de chiffrement ou versions** figées à la main sans raison.
3. **Certificat sans le bon nom/IP** (SAN) : erreurs de vérification incomprises.
4. **mTLS sans plan de renouvellement** : tout tombe le jour où les certificats expirent.
5. **Clés privées dans le dépôt Git.**
6. **Oublier qu'après `Hijack`, on gère tout** : délais, fermeture, arrêt.
7. **WebSocket sans vérification de l'`Origin`.**
8. **WebSocket là où des Server-Sent Events suffisent.**

## Bonnes pratiques Go

- Garder les valeurs par défaut de `crypto/tls` ; au plus `MinVersion: tls.VersionTLS12` ou `13`.
- AC internes dans `RootCAs` / `ClientCAs`, jamais de vérification désactivée.
- mTLS entre services, avec rotation automatique des certificats.
- Tests avec `httptest.NewTLSServer` ou une PKI de test générée en mémoire.
- Une bibliothèque éprouvée pour les WebSockets ; délais, tailles et origines contrôlés.

## Ce que je dois retenir

- TLS : confidentialité, intégrité, authentification du serveur (du client en mTLS).
- Ne jamais désactiver la vérification ; fournir ses AC.
- mTLS : `RequireAndVerifyClientCert` + `ClientCAs` ; l'identité est le certificat.
- Go négocie par défaut un échange hybride post-quantique (X25519MLKEM768 depuis 1.24).
- Mise à niveau : `Upgrade` → `101` → `Hijack` ; WebSocket = ce mécanisme + un format de trames.

---

➡️ [Exercices](exercices.md)
