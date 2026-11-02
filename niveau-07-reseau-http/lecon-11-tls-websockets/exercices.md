# Leçon 11 — Exercices

Code dans `mes-solutions/niveau-07/lecon-11/`.

## Exercices faciles

**E1 — `httptest.NewTLSServer`.**
Démarrer un serveur TLS de test. L'appeler avec `http.Get` : quelle erreur ? Puis avec `srv.Client()` :
pourquoi ça marche ? Afficher `resp.TLS.Version`, `resp.TLS.CipherSuite` (avec `tls.CipherSuiteName`)
et `resp.TLS.CurveID`.

**E2 — Le mauvais nom.**
Émettre (avec la PKI de l'exemple) un certificat serveur pour `192.0.2.1` au lieu de `127.0.0.1`.
Recopier l'erreur du client. Que vérifie-t-il exactement ?

**E3 — Désactiver le post-quantique.**
Relancer `TestMutualTLS` avec `GODEBUG=tlsmlkem=0`. Quel échange de clés est négocié ? Puis forcer
`CurvePreferences: []tls.CurveID{tls.CurveP256}` côté serveur. Dans quel contexte voudrait-on le faire
(pensez à du matériel ou à des normes imposées) ?

**E4 — Certificat expiré.**
Émettre un certificat client dont `NotAfter` est dans le passé. Que se passe-t-il à la connexion ? Côté
serveur, comment journaliser **pourquoi** un client a été refusé ? *(Indice : `http.Server.ErrorLog`.)*

**E5 — Server-Sent Events.**
Écrire `GET /events` qui envoie une ligne `data: …\n\n` par seconde avec `Content-Type:
text/event-stream` et `Flush`, et une page HTML minimale qui l'affiche avec `new EventSource("/events")`.
Pourquoi n'a-t-on besoin d'aucune mise à niveau de connexion ?

---

## Exercice intermédiaire — Deux services en mTLS

Deux services : `facturation` appelle `clients` (`GET /v1/customers/{id}`) en mTLS.

1. Une PKI de test générée au démarrage (ou des fichiers PEM générés une fois avec un petit outil
   `cmd/mkcerts` qui utilise `crypto/x509` et `encoding/pem`).
2. `clients` n'accepte que les certificats signés par l'AC **et** dont le nom commun figure dans une liste
   d'autorisation (`facturation` oui, `marketing` non) : où faire cette vérification ? (Indice :
   `tls.Config.VerifyPeerCertificate` ou dans un middleware avec `r.TLS.PeerCertificates`.)
3. Le client (`facturation`) réutilise un `http.Client` configuré une fois (leçon 6), avec délais.
4. Rechargement des certificats sans redémarrage : utiliser `tls.Config.GetCertificate` (côté serveur) et
   `GetClientCertificate` (côté client) qui lisent un certificat protégé par `atomic.Pointer` (N6 L7),
   mis à jour par une goroutine qui surveille les fichiers.
5. Tests : appel autorisé, certificat inconnu refusé, nom commun non autorisé refusé (403), rotation du
   certificat pendant que le service tourne.

---

## Défi — Un serveur de discussion

Écrire un salon de discussion avec une bibliothèque WebSocket (`go get github.com/coder/websocket`) :

1. `GET /ws` met à niveau la connexion après avoir vérifié l'origine.
2. Chaque client a une goroutine de lecture et une goroutine d'écriture ; un « hub » central (une
   goroutine propriétaire, N6 L2) diffuse les messages.
3. Un client **lent** ne doit pas bloquer les autres : file d'envoi bornée par client, et déconnexion du
   client si elle déborde (N6 L11).
4. Messages limités à 4 Kio, *ping* toutes les 30 s, déconnexion après 60 s sans réponse.
5. Arrêt gracieux : `Shutdown` n'arrête pas les WebSockets — les fermer proprement depuis le hub avec
   `RegisterOnShutdown`.
6. Tests : deux clients échangent des messages ; un client qui ne lit plus est déconnecté sans bloquer
   l'autre ; aucune goroutine ne reste après l'arrêt.

---

## Quiz

1. Que garantit TLS ? Que ne garantit-il pas ?
2. Que vérifie un client TLS sur le certificat du serveur ?
3. Pourquoi `InsecureSkipVerify` est-il dangereux ? Comment faire confiance à une AC interne ?
4. Qu'apporte le TLS mutuel ?
5. Quel échange de clés Go négocie-t-il par défaut, et contre quelle menace ?
6. Pourquoi parle-t-on d'échange « hybride » ?
7. Que signifie le statut 101 ?
8. Que devient la connexion après un `Hijack` ? Qui la ferme ?
9. Qu'ajoute le protocole WebSocket à la simple mise à niveau ?
10. Quand préférer les Server-Sent Events aux WebSockets ?
