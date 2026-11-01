# Leçon 2 — Exercices

Code dans `mes-solutions/niveau-07/lecon-02/`. Tout avec `httptest`.

## Exercices faciles

**E1 — Trop tard.**
Un handler écrit `fmt.Fprint(w, "<p>bonjour</p>")`, **puis** fixe un en-tête `X-Trop-Tard` et appelle
`w.WriteHeader(418)`. Avec un `httptest.NewServer` et un vrai client : quel statut reçoit-on ? L'en-tête
est-il présent ? Quel `Content-Type` ? Que voit-on dans la sortie d'erreur du serveur ?

**E2 — Le `return` oublié.**
Écrire un handler qui appelle `http.Error(w, "x", 400)` **sans** `return`, puis continue et écrit un
JSON. Que reçoit le client (statut et corps) ? Écrire le test qui l'aurait détecté.

**E3 — Paramètres de requête.**
Écrire `GET /search?q=…&page=…&tag=…&tag=…` : `q` obligatoire (1 à 100 caractères), `page` entier ≥ 1
(défaut 1), `tag` répétable (au plus 5). Chaque violation donne un 400 avec un message qui nomme le
paramètre. Table de tests.

**E4 — 413.**
Envoyer un corps de 2 Mio à un handler qui utilise `MaxBytesReader(w, r.Body, 1<<20)`. Vérifier le 413.
Puis remplacer par `io.LimitReader(r.Body, 1<<20)` : que se passe-t-il ? Pourquoi est-ce pire ?

**E5 — Le `Content-Type` deviné.**
Écrire trois réponses sans `Content-Type` : un JSON, un HTML, un PNG (les octets d'un vrai fichier).
Quel type le serveur annonce-t-il pour chacune ? Lequel est faux ?

---

## Exercice intermédiaire — Export CSV en streaming

`GET /export?from=AAAA-MM-JJ&to=AAAA-MM-JJ` exporte des « ventes » (générées à la volée, une par
minute de la période) au format CSV.

**Contraintes :**
1. Paramètres validés avec `time.Parse` (N4 L4) : `from` < `to`, période ≤ 31 jours, sinon 400.
2. `Content-Type: text/csv`, `Content-Disposition: attachment; filename="ventes.csv"`.
3. Écriture avec `encoding/csv` directement dans `w`, **vidée** toutes les 1 000 lignes : la mémoire du
   serveur reste constante quelle que soit la période.
4. Si le client se déconnecte, l'export s'arrête (`r.Context()`).
5. Une erreur **pendant** l'écriture (après les premières lignes) ne peut plus changer le statut :
   comment le client peut-il savoir que l'export est incomplet ? Proposer une solution (indice : une
   ligne finale de contrôle, ou un *trailer* HTTP — `go doc net/http.ResponseWriter`, chercher « Trailer »).
6. Tests : statuts d'erreur, nombre de lignes, en-têtes, et un test qui annule le contexte au milieu.

---

## Défi — Requêtes conditionnelles

Implémenter le cache HTTP côté serveur pour `GET /articles/{id}` :

1. Calculer un `ETag` (hash du contenu, entre guillemets) et l'envoyer.
2. Si la requête contient `If-None-Match` avec le même `ETag`, répondre **304 Not Modified sans corps**.
3. Ajouter `Cache-Control: max-age=60`.
4. Lire la doc de `http.ServeContent` : que fait-il de tout cela automatiquement (y compris `Range`) ?
   Réécrire avec lui et comparer.

---

## Quiz

1. Comment lire un paramètre de chemin ? un paramètre de requête ? un en-tête ?
2. Pourquoi `MaxBytesReader` plutôt que `io.LimitReader` pour un corps de requête ?
3. Faut-il fermer `r.Body` dans un handler ?
4. Dans quel ordre écrire en-têtes, statut et corps ?
5. Que se passe-t-il si on appelle `Write` sans `WriteHeader` ?
6. Que se passe-t-il après un second `WriteHeader` ?
7. Différence entre 400 et 422 ? entre 401 et 403 ?
8. Pourquoi toujours fixer `Content-Type` ?
9. Comment envoyer une réponse au fil de l'eau ?
10. Pourquoi faut-il `return` après `http.Error` ?
