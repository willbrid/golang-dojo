# Leçon 3 — Exercices

Code dans `mes-solutions/niveau-07/lecon-03/`.

## Exercices faciles

**E1 — L'ordre.**
Inverser `Logging` et `Recover` dans l'exemple. Que contient le journal pour `/boom` ? Que reçoit le
client ? Écrire un test qui fige l'ordre correct.

**E2 — Le 200 implicite.**
Retirer l'interception du statut dans `Write`. Quel statut est journalisé pour `/hello` ? Pourquoi les
tests de statut côté client passent-ils quand même ?

**E3 — `Unwrap`.**
Mettre le handler `Countdown` de la leçon 2 derrière `Logging`. Retirer `Unwrap` : `rc.Flush()`
rend-il une erreur ? Laquelle (`errors.Is(err, http.ErrNotSupported)`) ? Remettre `Unwrap` et vérifier.

**E4 — Délai par requête.**
Écrire un middleware `Timeout(d)` qui donne à chaque requête un contexte avec délai. Le comparer à
`http.TimeoutHandler` de la bibliothèque standard (`go doc net/http.TimeoutHandler`) : que fait ce
dernier de plus (statut, corps) ? Pourquoi un handler qui ignore son contexte n'est-il pas arrêté par
votre version ?

**E5 — Un middleware qui refuse.**
Écrire `RequireHeader(name)` qui répond 400 si l'en-tête manque et **n'appelle pas** `next`. Le
placer avant puis après `Logging` : quelle différence dans le journal ?

---

## Exercice intermédiaire — Métriques et limite de débit

1. **`Metrics`** : compte les requêtes par `méthode + motif de route + statut`, et cumule les latences.
   Exposer un handler `GET /metrics` qui affiche les compteurs triés. *(Indice : `r.Pattern`, Go 1.23,
   donne le motif qui a servi la requête — mais seulement **après** le routage : où placer le middleware
   pour le lire ? Le motif est-il préférable au chemin `r.URL.Path` ? Pensez à `/notes/1`, `/notes/2`…)*
2. **`RateLimit`** : au plus N requêtes par seconde **par adresse IP** (`r.RemoteAddr`, découpé avec
   `net.SplitHostPort`), avec le limiteur à jetons de N6 L4 (un par IP). Au-delà : 429 avec
   `Retry-After: 1`.
3. La map des limiteurs par IP grossit sans fin : ajouter un nettoyage des IP inactives depuis 5 min.
4. Tests : `synctest` pour la limite de débit ; un test concurrent sous `-race` pour les métriques.
5. Pourquoi `r.RemoteAddr` est-il faux derrière un reverse proxy ? (Réponse complète à la leçon 9.)

---

## Défi — Compression

Écrire un middleware `Gzip` qui compresse la réponse si le client envoie `Accept-Encoding: gzip`.

1. Envelopper le `ResponseWriter` pour écrire dans un `gzip.Writer`. Fixer `Content-Encoding: gzip` et
   retirer `Content-Length` (la taille change).
2. Ajouter `Vary: Accept-Encoding` : pourquoi est-ce indispensable pour les caches ?
3. Ne pas compresser les réponses déjà compressées (images) ni les très petites (< 1 Kio) : comment le
   savoir **avant** d'avoir écrit les premiers octets ?
4. `Flush` doit vider **aussi** le `gzip.Writer`. Tester avec `Countdown`.
5. Ne pas oublier de fermer le `gzip.Writer` à la fin. Où ?

---

## Quiz

1. Quelle est la signature d'un middleware en Go ?
2. Dans `A(B(C(h)))`, qui voit la requête en premier ? la réponse en dernier ?
3. Pourquoi placer la récupération de panique à l'intérieur de la journalisation ?
4. Comment journaliser le statut d'une réponse ?
5. À quoi sert la méthode `Unwrap` d'un `ResponseWriter` enveloppé ?
6. Comment un middleware transmet-il une donnée au handler ?
7. Que fait `r.WithContext` ?
8. Qu'est-ce que `http.ErrAbortHandler` ?
9. Pourquoi borner un `X-Request-ID` reçu ?
10. Qu'arrive-t-il au corps de la requête si un middleware le lit avec `io.ReadAll` ?
