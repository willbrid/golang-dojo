# Leçon 10 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Les frontières perdues

En local et sans pause, le serveur lit le plus souvent `"helloworld"` d'un coup : les deux écritures
sont arrivées avant sa lecture, et TCP les livre comme un flux continu. Avec 100 ms de pause, il lit
`"hello"` puis, au `Read` suivant, `"world"`. Le **même** programme donne deux résultats selon le
moment : on ne peut **jamais** déduire une frontière de message d'un `Read`. D'où le framing.

## E2 — `ReadFull`

`r.Read(hdr[:])` obtient **1** octet (le lecteur n'en livre jamais plus) ; `io.ReadFull` boucle jusqu'à
en avoir **5**. Sur un vrai réseau, le comportement de `Read` dépend des paquets reçus : il marche
souvent, et échoue dès que l'en-tête arrive coupé entre deux paquets.

## E3 — Le délai d'inactivité

Au bout de 2 s, le `Read` du serveur échoue avec une erreur pour laquelle `errors.Is(err,
os.ErrDeadlineExceeded)` est vrai ; `handle` rend la main et ferme la connexion. Côté client, le
prochain `Read` rend `io.EOF` (fermeture propre par le pair). Un client qui veut garder la connexion doit
envoyer quelque chose avant l'échéance (un message de maintien de connexion, un *ping*).

## E4 — Écho UDP

```go
pc, _ := net.ListenPacket("udp", "127.0.0.1:0")
buf := make([]byte, 1500)
for {
	n, addr, err := pc.ReadFrom(buf)
	if err != nil {
		return
	}
	pc.WriteTo(bytes.ToUpper(buf[:n]), addr)
}
```

Un datagramme de 2 000 octets lu dans un buffer de 1 500 : sur Linux, `ReadFrom` rend **1 500 octets
sans erreur** — les 500 derniers sont perdus silencieusement. Deux parades : un buffer de 64 Kio (la
taille maximale d'un datagramme UDP), ou un protocole qui limite lui-même la taille des messages et la
vérifie.

## E5 — DNS avec délai

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()
_, err := net.DefaultResolver.LookupHost(ctx, "nexiste-pas.invalid")
var dnsErr *net.DNSError
if errors.As(err, &dnsErr) && dnsErr.IsNotFound {
	fmt.Println("nom inconnu :", dnsErr.Name)
}
```

Le domaine `.invalid` est réservé et ne résout jamais : on obtient un `*net.DNSError` avec `IsNotFound`
vrai. Un délai expiré donne un `*net.DNSError` avec `IsTimeout` vrai.

## Exercice intermédiaire — Le client du projet 5

```go
type Client struct {
	mu   sync.Mutex      // une commande à la fois sur la connexion
	conn net.Conn
	r    *bufio.Reader
	w    *bufio.Writer
}

func (c *Client) do(ctx context.Context, line string) (string, error) {
	if strings.ContainsAny(line, "\r\n") {
		return "", errors.New("retour à la ligne interdit dans une commande")
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if dl, ok := ctx.Deadline(); ok {
		c.conn.SetDeadline(dl)
	} else {
		c.conn.SetDeadline(time.Time{})              // aucune échéance
	}
	stop := context.AfterFunc(ctx, func() { c.conn.SetDeadline(time.Now()) })   // annulation → interruption
	defer stop()
	if _, err := c.w.WriteString(line + "\n"); err != nil {
		return "", err
	}
	if err := c.w.Flush(); err != nil {
		return "", err
	}
	resp, err := c.r.ReadString('\n')
	if err != nil {
		return "", err
	}
	return strings.TrimSuffix(resp, "\n"), nil
}
```

- **Pourquoi sérialiser** : sur une connexion, les réponses arrivent dans l'ordre des commandes. Deux
  goroutines qui écrivent chacune sa commande puis lisent « la » réponse peuvent lire celle de l'autre.
  Le verrou fait de l'aller-retour une opération atomique.
- **Annulation** : `SetDeadline(time.Now())` interrompt immédiatement le `Read` en cours. Après une
  annulation au milieu d'un échange, l'état de la connexion est incertain (la réponse arrivera peut-être
  plus tard et serait lue comme réponse à la commande **suivante**) : il faut **fermer** cette connexion
  et ne pas la remettre dans le pool.
- **Pool** : un channel bufferisé de `*Client` (N6 L5 : le sémaphore) ; `Get` prend une connexion libre
  ou en ouvre une si le nombre total est sous N ; `Put` la rend, sauf si elle est en erreur. C'est le
  principe du pool de `database/sql` (niveau 8) et du `Transport` HTTP.

## Défi — Un protocole versionné

1. Un en-tête `[version 1 octet][type 1][id u32][drapeaux 1][longueur u32]`. La version en **premier**
   octet permet au serveur de choisir le décodeur avant de lire le reste. Pour une compatibilité
   ascendante sans octet de version préexistant, une **négociation** à l'ouverture (le client annonce
   les versions qu'il connaît, le serveur répond celle retenue) est plus souple.
2. Avec le *pipelining*, 100 requêtes ne coûtent plus 100 allers-retours mais à peu près **un** : sur
   `127.0.0.1`, le gain est modeste (la latence est minuscule) ; avec 10 ms simulées par aller-retour,
   on passe d'environ 1 s à quelques dizaines de millisecondes. Les réponses peuvent revenir dans le
   désordre si le serveur traite en parallèle : l'identifiant permet de les associer (une
   `map[uint32]chan Message` côté client).
3. Le fuzzing vérifie surtout les nouvelles bornes : version inconnue, identifiant, drapeaux réservés non
   nuls refusés.

## Quiz

1. Non : c'est un flux d'octets.
2. Un délimiteur (ligne) ou un préfixe de longueur.
3. `io.EOF` : rien n'a été lu (fin propre) ; `io.ErrUnexpectedEOF` : flux coupé au milieu.
4. Sinon n'importe quel pair fait allouer des gigaoctets avec quelques octets d'en-tête.
5. Le gros-boutiste (ordre réseau).
6. Elle fixe une échéance absolue pour les lectures/écritures ; on la repousse avant chaque lecture.
7. En fermant le listener ou la connexion (ou avec une échéance immédiate), par exemple via
   `context.AfterFunc`.
8. Datagrammes indépendants, sans connexion, sans garantie de livraison ni d'ordre, frontières conservées.
9. Il est tronqué, souvent sans erreur.
10. Une résolution lente bloquerait sinon le `Dial` sans limite.
