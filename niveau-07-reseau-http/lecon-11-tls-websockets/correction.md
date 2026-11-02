# Leçon 11 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — `httptest.NewTLSServer`

`http.Get` échoue avec `tls: failed to verify certificate: x509: certificate signed by unknown
authority` : le certificat de test est signé par une AC générée par `httptest`, inconnue du système.
`srv.Client()` rend un client dont les `RootCAs` contiennent cette AC. On obtient typiquement TLS 1.3,
une suite `TLS_AES_128_GCM_SHA256` (ou ChaCha20 selon la machine) et l'échange `X25519MLKEM768`.

## E2 — Le mauvais nom

```
tls: failed to verify certificate: x509: certificate is valid for 192.0.2.1, not 127.0.0.1
```

Le client vérifie que l'hôte qu'il a **demandé** (ici l'adresse `127.0.0.1`) figure dans les noms
alternatifs (SAN) du certificat — en plus de la signature par une AC de confiance et des dates de
validité. C'est ce qui empêche un attaquant muni d'un certificat **valide pour un autre nom** de se faire
passer pour votre serveur.

## E3 — Désactiver le post-quantique

Avec `GODEBUG=tlsmlkem=0`, l'échange redevient `X25519` (classique). Avec `CurvePreferences:
[]tls.CurveID{tls.CurveP256}` côté serveur, on négocie `CurveP256`. Cas d'usage : un équipement ou un
client ancien qui échoue sur les messages de poignée de main plus gros de ML-KEM (environ 1 Kio de plus) ;
une norme qui impose des algorithmes certifiés (les variantes `SecP256r1MLKEM768` existent justement pour
combiner une courbe NIST et ML-KEM). Hors contrainte de ce type, garder le défaut.

## E4 — Certificat expiré

La poignée de main échoue : le client reçoit une erreur TLS générique (souvent `remote error: tls: bad
certificate` ou une fin de connexion), le serveur ne transmet pas le détail. Côté serveur, la raison
(`x509: certificate has expired or is not yet valid`) est écrite dans `http.Server.ErrorLog` (par défaut
la sortie d'erreur via le paquet `log`) : brancher un `*log.Logger` dédié permet de la journaliser et de la
compter. C'est souvent la première alerte d'un oubli de renouvellement.

## E5 — Server-Sent Events

```go
func events(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "text/event-stream")
	w.Header().Set("Cache-Control", "no-cache")
	rc := http.NewResponseController(w)
	t := time.NewTicker(time.Second)
	defer t.Stop()
	for i := 0; ; i++ {
		fmt.Fprintf(w, "data: tic %d\n\n", i)
		if err := rc.Flush(); err != nil {
			return
		}
		select {
		case <-t.C:
		case <-r.Context().Done():
			return
		}
	}
}
```

C'est une réponse HTTP **ordinaire**, simplement très longue et vidée au fil de l'eau : le serveur parle,
le client écoute. Pas besoin de canal bidirectionnel, donc pas de mise à niveau. Avantages : passe par
tous les proxys HTTP, reconnexion automatique par le navigateur, fonctionne en HTTP/2. Attention au
`WriteTimeout` du serveur (leçon 9), qui couperait le flux.

## Exercice intermédiaire — Deux services en mTLS

Points clés :

```go
// Côté clients : autorisation par nom commun, après la vérification TLS.
func requireCN(allowed map[string]bool, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.TLS == nil || len(r.TLS.PeerCertificates) == 0 {
			http.Error(w, "certificat client requis", http.StatusUnauthorized)
			return
		}
		if !allowed[r.TLS.PeerCertificates[0].Subject.CommonName] {
			http.Error(w, "service non autorisé", http.StatusForbidden)
			return
		}
		next.ServeHTTP(w, r)
	})
}
```

- **Où vérifier** : TLS (avec `ClientCAs`) répond à « ce certificat est-il authentique ? » ;
  l'autorisation (« ce service a-t-il le droit ? ») est une règle applicative, plus lisible dans un
  middleware, avec un 403 explicite. `VerifyPeerCertificate` couperait la connexion sans réponse HTTP.
- **Rotation sans redémarrage** :

  ```go
  var current atomic.Pointer[tls.Certificate]
  cfg := &tls.Config{
  	GetCertificate: func(*tls.ClientHelloInfo) (*tls.Certificate, error) { return current.Load(), nil },
  }
  // une goroutine recharge les fichiers PEM (tls.LoadX509KeyPair) et fait current.Store(&nouveau)
  ```

  Chaque nouvelle poignée de main lit le certificat courant ; les connexions établies continuent avec
  l'ancien jusqu'à leur fermeture. Le test de rotation : émettre un nouveau certificat, le publier, ouvrir
  une nouvelle connexion (`client.CloseIdleConnections()` d'abord) et vérifier le numéro de série reçu
  (`resp.TLS.PeerCertificates[0].SerialNumber`).

## Défi — Un serveur de discussion

La structure attendue :

```go
type hub struct {
	join    chan *client
	leave   chan *client
	publish chan []byte
	clients map[*client]bool          // n'appartient qu'à la goroutine du hub
}

type client struct {
	send chan []byte                  // file BORNÉE (par ex. 16 messages)
}

func (h *hub) run(ctx context.Context) {
	for {
		select {
		case c := <-h.join:
			h.clients[c] = true
		case c := <-h.leave:
			delete(h.clients, c)
			close(c.send)
		case msg := <-h.publish:
			for c := range h.clients {
				select {
				case c.send <- msg:
				default:                   // client trop lent : on le déconnecte (N6 L11)
					delete(h.clients, c)
					close(c.send)
				}
			}
		case <-ctx.Done():
			for c := range h.clients {
				close(c.send)
			}
			return
		}
	}
}
```

- Le hub est l'**unique** propriétaire de la map : aucun verrou (N6 L2).
- La diffusion ne bloque **jamais** : `select` + `default`. Un client lent perd sa place au lieu de geler
  la discussion de tous.
- Le hub est le seul à **fermer** `c.send` (il est son seul émetteur, N6 L3) ; la goroutine d'écriture du
  client voit la fermeture (`range c.send` se termine) et ferme la WebSocket.
- `srv.RegisterOnShutdown(cancelHub)` : à l'arrêt, le hub ferme toutes les files, donc toutes les
  connexions détournées, que `Shutdown` n'aurait pas touchées.

## Quiz

1. Confidentialité, intégrité, authentification du serveur ; pas la sécurité des extrémités, ni
   l'authentification du client (hors mTLS).
2. La signature par une AC de confiance, la période de validité, et que l'hôte demandé figure dans les
   noms du certificat.
3. Il supprime l'authentification du serveur : interception possible. Ajouter l'AC dans `RootCAs`.
4. Le serveur authentifie aussi le client par son certificat : identité forte entre services, sans
   secret partagé.
5. Un échange hybride X25519 + ML-KEM-768 (depuis 1.24, d'autres hybrides depuis 1.26), contre le
   déchiffrement futur par un ordinateur quantique d'échanges enregistrés aujourd'hui.
6. Parce qu'il combine un algorithme classique et un post-quantique : la session reste sûre si l'un des
   deux tient.
7. « Changement de protocole » : la connexion cesse d'être du HTTP.
8. Elle appartient au handler, qui doit tout gérer et la fermer ; le serveur HTTP ne s'en occupe plus.
9. Une poignée de main normalisée, un format de trames, le masquage, les pings et la fermeture propre.
10. Quand seul le serveur envoie des données au client.
