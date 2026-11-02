# Leçon 9 — Exercices

Code dans `mes-solutions/niveau-07/lecon-09/`.

## Exercices faciles

**E1 — Slowloris.**
Lancer un serveur **sans** `ReadHeaderTimeout`. Écrire un petit client qui ouvre 500 connexions et
envoie une ligne d'en-tête toutes les 5 secondes sans jamais terminer. Le serveur répond-il encore à un
`curl` normal ? Combien de goroutines a-t-il (`/debug/pprof/goroutine` ou un compteur) ? Recommencer avec
`ReadHeaderTimeout: 5 * time.Second`.

**E2 — Le `WriteTimeout` qui coupe.**
Un handler streame 10 lignes à une seconde d'intervalle, derrière un serveur avec `WriteTimeout: 3 *
time.Second`. Combien de lignes le client reçoit-il ? Corriger **pour cette route seulement** avec
`http.NewResponseController(w).SetWriteDeadline`.

**E3 — 431.**
Reproduire `TestHeaderValueCount`, puis envoyer les 20 valeurs sur **une seule** ligne séparées par des
virgules. Le serveur les refuse-t-il ? Pourquoi ? Que dit la documentation de `MaxHeaderValueCount` ?

**E4 — Falsifier son IP.**
Écrire un handler qui affiche `r.RemoteAddr` et `r.Header.Get("X-Forwarded-For")`. Avec `curl -H
'X-Forwarded-For: 1.2.3.4'`, que voit le serveur ? Si une limite de débit utilisait naïvement cet
en-tête, comment la contourner ?

**E5 — Redirection ouverte.**
Écrire `GET /login?next=…` qui redirige après connexion. Montrer qu'avec `next=https://exemple.invalid`
l'utilisateur est envoyé ailleurs. Corriger : n'accepter qu'un chemin qui commence par `/` mais **pas**
par `//` (pourquoi ?). Tester les cas `/compte`, `//site`, `https://site`, `/\site`.

---

## Exercice intermédiaire — Une passerelle d'API

Écrire `gateway` : un reverse proxy devant deux services (`/users/` → service A, `/orders/` → service B),
qui :

1. Utilise `NewServer` (délais et limites).
2. Ajoute les en-têtes de sécurité et un identifiant de requête (leçon 3) **transmis** au service d'origine
   (`X-Request-ID`).
3. Limite le débit par IP client, en déterminant l'IP avec `ClientIP` (la passerelle étant elle-même
   derrière un répartiteur `10.0.0.0/8`).
4. Retire de la réponse les en-têtes qui trahissent l'infrastructure (`Server`, `X-Powered-By`) avec
   `ReverseProxy.ModifyResponse`.
5. Si un service d'origine est en panne, répond un *problem details* 502 (et non la page d'erreur par
   défaut) avec `ReverseProxy.ErrorHandler`.
6. Tests : chaque service simulé par `httptest.NewServer` ; vérifier routage, en-têtes transmis et
   retirés, erreur 502.

---

## Défi — SSRF

Un service propose `POST /preview` avec `{"url": "…"}` : il télécharge la page et en rend le titre.

1. Montrer qu'on peut lui faire lire `http://127.0.0.1:<port>/admin` (un service interne simulé).
2. Une première défense : refuser les noms d'hôtes `localhost` et les adresses privées **dans l'URL**.
   La contourner avec un nom de domaine qui **résout** vers `127.0.0.1`, ou avec une redirection.
3. La défense correcte : vérifier l'adresse **au moment de la connexion**, avec un
   `net.Dialer{Control: …}` dans le `Transport` qui refuse les adresses privées, de bouclage et de
   métadonnées (`netip.Addr.IsPrivate`, `IsLoopback`, `IsLinkLocalUnicast`…). Pourquoi cette défense
   résiste-t-elle aux redirections et au DNS ?
4. Ajouter les autres limites nécessaires : délai, taille maximale de la page lue, nombre de redirections.

---

## Quiz

1. Quels sont les quatre délais de `http.Server` ? Que valent-ils par défaut ?
2. Contre quoi protège `ReadHeaderTimeout` ?
3. `WriteTimeout` arrête-t-il le handler ?
4. Que limite `MaxHeaderValueCount` ? Depuis quelle version ? Quel statut au-delà ?
5. Quel en-tête empêche le navigateur de deviner le type d'une réponse ?
6. Pourquoi n'envoyer HSTS que sur HTTPS ?
7. Pourquoi ne pas croire `X-Forwarded-For` sans condition ?
8. Pourquoi lire `X-Forwarded-For` depuis la droite ?
9. Pourquoi préférer `Rewrite` à `Director` dans `ReverseProxy` ?
10. Qu'est-ce qu'une SSRF ?
