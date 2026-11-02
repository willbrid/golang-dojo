# Leçon 9 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Slowloris

Sans `ReadHeaderTimeout`, les 500 connexions restent ouvertes indéfiniment, chacune avec sa goroutine
(500 goroutines de plus, chacune avec un buffer de lecture). Le serveur répond encore à `curl` (Go
n'a pas de limite de connexions par défaut : c'est la mémoire et les descripteurs de fichiers qui
finiront par manquer — `ulimit -n`). Un serveur à pool de threads fixe serait déjà inaccessible. Avec
`ReadHeaderTimeout: 5s`, chaque connexion est coupée 5 secondes après son ouverture si ses en-têtes ne
sont pas complets : l'attaquant doit rouvrir sans cesse, et le serveur ne garde jamais plus de quelques
secondes de connexions lentes.

## E2 — Le `WriteTimeout` qui coupe

Le client reçoit **environ 3 lignes** puis la connexion est coupée (le délai court depuis la fin de la
lecture des en-têtes). Le handler, lui, continue de s'exécuter : ses écritures échouent. Correction
locale :

```go
rc := http.NewResponseController(w)
rc.SetWriteDeadline(time.Now().Add(30 * time.Second))   // pour cette requête seulement
```

Si le `ResponseWriter` est enveloppé par un middleware, il faut `Unwrap` (leçon 3).

## E3 — 431

Sur **une seule ligne** séparée par des virgules, les 20 valeurs **passent** : la documentation précise
que des valeurs séparées par des virgules sur une seule ligne d'en-tête comptent **une** fois, alors que
des lignes répétées comptent chacune. La limite vise le nombre de **lignes** analysées (le coût réel
pour le serveur) ; la taille totale reste bornée par `MaxHeaderBytes`.

## E4 — Falsifier son IP

Le serveur voit `RemoteAddr: 127.0.0.1:…` et `X-Forwarded-For: 1.2.3.4` — la valeur inventée par
`curl`. Une limite de débit fondée sur cet en-tête se contourne en envoyant une valeur **différente à
chaque requête** : chaque requête semble venir d'un nouveau client. D'où `ClientIP` avec liste de proxys
de confiance.

## E5 — Redirection ouverte

```go
func safeNext(next string) string {
	if !strings.HasPrefix(next, "/") || strings.HasPrefix(next, "//") || strings.HasPrefix(next, "/\\") {
		return "/"
	}
	return next
}
```

`//site.invalid/x` est une URL **relative au protocole** : le navigateur la comprend comme
`https://site.invalid/x`. Et certains navigateurs normalisent `/\site` en `//site`. Les quatre cas :
`/compte` → accepté ; `//site`, `https://site`, `/\site` → `/`. Une alternative plus robuste : analyser
avec `url.Parse` et exiger `u.Scheme == "" && u.Host == ""`, puis vérifier le premier caractère du chemin.

## Exercice intermédiaire — Une passerelle d'API

Points clés :

```go
proxy := &httputil.ReverseProxy{
	Rewrite: func(pr *httputil.ProxyRequest) {
		pr.SetURL(target)
		pr.SetXForwarded()
		pr.Out.Header.Set("X-Request-ID", mw.RequestID(pr.In.Context()))   // transmis à l'origine
	},
	ModifyResponse: func(resp *http.Response) error {
		resp.Header.Del("Server")
		resp.Header.Del("X-Powered-By")
		return nil
	},
	ErrorHandler: func(w http.ResponseWriter, r *http.Request, err error) {
		writeProblem(w, http.StatusBadGateway, "/problems/upstream-unavailable", "Service indisponible")
		// err (connexion refusée, délai…) va au journal, pas au client (leçon 5)
	},
}
```

- `pr.In` est la requête reçue, `pr.Out` celle qui part : on lit le contexte de l'une, on écrit les
  en-têtes de l'autre.
- `ErrorHandler` remplace la page d'erreur par défaut (502 en texte brut) : la passerelle garde un format
  d'erreur unique.
- Le routage : un `ServeMux` avec `"/users/"` et `"/orders/"` vers deux `ReverseProxy`.
- Test du 502 : un `httptest.NewServer` qu'on **ferme** avant la requête.

## Défi — SSRF

1. Avec `{"url": "http://127.0.0.1:<port>/admin"}`, le service lit la page d'administration interne et
   en rend le titre à l'attaquant.
2. Contournements de la défense « par l'URL » : un domaine dont l'enregistrement DNS pointe vers
   `127.0.0.1` ; une URL publique qui **redirige** vers `http://127.0.0.1/…` (le client suit les
   redirections) ; des écritures exotiques d'adresses (`http://2130706433/`, `http://[::ffff:127.0.0.1]/`).
3. La défense au moment de la connexion :

   ```go
   dialer := &net.Dialer{
   	Timeout: 5 * time.Second,
   	Control: func(network, address string, _ syscall.RawConn) error {
   		host, _, _ := net.SplitHostPort(address)
   		ip, err := netip.ParseAddr(host)
   		if err != nil {
   			return err
   		}
   		ip = ip.Unmap()
   		if ip.IsLoopback() || ip.IsPrivate() || ip.IsLinkLocalUnicast() || ip.IsUnspecified() {
   			return fmt.Errorf("destination interdite : %s", ip)
   		}
   		return nil
   	},
   }
   client := &http.Client{Transport: &http.Transport{DialContext: dialer.DialContext}, Timeout: 10 * time.Second}
   ```

   `Control` s'exécute **après** la résolution DNS, sur l'adresse **réellement** utilisée, et pour
   **chaque** connexion — y compris celles des redirections. Aucun nom, aucune redirection, aucune
   écriture exotique ne passe : on vérifie ce qui est réellement contacté. (`169.254.169.254`, les
   métadonnées du cloud, est une adresse *link-local* : refusée.)
4. Limites restantes : `Client.Timeout`, `io.LimitReader` sur le corps (1 Mio), `CheckRedirect` qui
   limite le nombre de redirections (ex. 3), et seulement `http`/`https`.

## Quiz

1. `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`, `IdleTimeout` ; zéro, c'est-à-dire aucun délai.
2. Les clients qui envoient leurs en-têtes au compte-gouttes pour garder des connexions ouvertes.
3. Non : il coupe l'écriture vers le client ; le handler continue (c'est le contexte qui l'arrête).
4. Le nombre de valeurs (lignes) d'en-têtes d'une requête ; Go 1.27 ; 431.
5. `X-Content-Type-Options: nosniff`.
6. Sur HTTP, un attaquant pourrait l'injecter ou le retirer ; la norme impose de l'ignorer hors HTTPS.
7. N'importe quel client peut l'envoyer avec une valeur inventée.
8. Les valeurs de droite sont ajoutées par nos proxys ; celles de gauche peuvent venir du client.
9. `Rewrite` retire d'abord les en-têtes de transfert reçus et sépare requête entrante et sortante.
10. Une attaque qui fait émettre au serveur des requêtes vers des destinations internes choisies par
    l'attaquant.
