# Leçon 10 — Exercices

Code dans `mes-solutions/niveau-07/lecon-10/`.

## Exercices faciles

**E1 — Les frontières perdues.**
Un client envoie deux `Write` de 5 octets (`"hello"` et `"world"`) sans pause. Un serveur fait un seul
`conn.Read(buf)` avec un buffer de 100 octets. Que lit-il, le plus souvent, en local ? Puis le client
attend 100 ms entre les deux `Write` : que se passe-t-il ? Conclure.

**E2 — `ReadFull`.**
Avec le lecteur « un octet à la fois » du cours, lire un en-tête de 5 octets avec `r.Read(hdr[:])`, puis
avec `io.ReadFull`. Combien d'octets chaque méthode obtient-elle ?

**E3 — Le délai d'inactivité.**
Avec le serveur de l'exemple (`idle` de 2 s), ouvrir une connexion sans rien envoyer. Que se passe-t-il
au bout de 2 s ? Quelle erreur le client obtient-il sur son prochain `Read` ? Tester avec
`errors.Is(err, os.ErrDeadlineExceeded)` côté serveur, et avec `io.EOF` côté client.

**E4 — Écho UDP.**
Écrire un serveur UDP qui renvoie chaque datagramme en majuscules, et un client qui en envoie trois.
Envoyer un datagramme de 2 000 octets à un serveur qui lit dans un buffer de 1 500 : que reçoit-il ?
Quelle erreur ?

**E5 — DNS avec délai.**
Résoudre `go.dev` avec `net.Resolver.LookupHost` et un contexte de 2 s. Puis un nom qui n'existe pas :
quelle erreur ? Tester `errors.As(err, &dnsErr)` avec `*net.DNSError` et lire `IsNotFound`.

---

## Exercice intermédiaire — Le client du projet 5

Écrire un paquet `kvclient` pour le serveur `kvd` du projet 5 (protocole texte, une commande par ligne) :

```go
func Dial(ctx context.Context, addr string) (*Client, error)
func (c *Client) Get(ctx context.Context, key string) (string, bool, error)
func (c *Client) Set(ctx context.Context, key, value string) error
func (c *Client) Incr(ctx context.Context, key string) (int64, error)
func (c *Client) Close() error
```

**Contraintes :**
1. Le contexte de chaque appel devient l'échéance de la connexion (`ctx.Deadline()` → `SetDeadline`) ;
   une annulation interrompt l'appel en cours.
2. Un `Client` est sûr pour un usage concurrent : les commandes d'une même connexion sont **sérialisées**
   (un verrou), sinon les réponses se mélangent. Pourquoi ?
3. Une valeur contenant `\n` est refusée côté client (elle casserait le framing par lignes).
4. Une **réserve de connexions** (*pool*) : `kvclient.Pool` avec N connexions réutilisées. Inspirez-vous
   de ce que fait le `Transport` HTTP (leçon 6).
5. Tests contre le serveur du projet 5 (ou un faux serveur sur `127.0.0.1:0`).

---

## Défi — Un protocole versionné

Le protocole de l'exemple doit évoluer : ajouter un identifiant de requête (pour pouvoir envoyer plusieurs
requêtes sans attendre les réponses — le *pipelining*) et des drapeaux.

1. Proposer un nouveau format d'en-tête. Comment un serveur reconnaît-il un client de l'ancienne version ?
   (Pistes : un octet de version en tête, un message de négociation à l'ouverture.)
2. Implémenter le *pipelining* : le client envoie 100 requêtes d'affilée, puis lit les 100 réponses, en les
   associant par identifiant. Mesurer le gain par rapport à un aller-retour par requête, sur `127.0.0.1`
   puis en simulant 10 ms de latence (une attente dans le serveur).
3. Fuzzer le nouveau décodeur.

---

## Quiz

1. TCP conserve-t-il les frontières entre deux `Write` ?
2. Quelles sont les deux grandes méthodes de framing ?
3. Différence entre `io.EOF` et `io.ErrUnexpectedEOF` rendus par `io.ReadFull` ?
4. Pourquoi borner la longueur annoncée avant d'allouer ?
5. Quel ordre d'octets utilisent les protocoles réseau ?
6. Que fait `SetDeadline`, et comment obtenir un délai d'inactivité ?
7. Comment débloquer un `Accept` ou un `Read` en cours lors d'un arrêt ?
8. En quoi UDP diffère-t-il de TCP ?
9. Que se passe-t-il si un datagramme UDP dépasse le buffer de lecture ?
10. Pourquoi passer un contexte à la résolution DNS ?
