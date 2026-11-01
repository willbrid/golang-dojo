# Leçon 1 — Exercices

Code dans `mes-solutions/niveau-07/lecon-01/`. Chaque handler a un test `httptest`.

## Exercices faciles

**E1 — Le routeur, prédit puis mesuré.**
Enregistrer `"GET /notes/{id}"`, `"GET /notes/new"`, `"/{$}"` et `"GET /files/{path...}"`. Prédire le
handler choisi pour `/notes/new`, `/notes/7`, `/`, `/x`, `/files/a/b/c`, puis vérifier avec
`httptest`. Ajouter `"GET /{a}/x"` : que se passe-t-il, et que dit le message ?

**E2 — `HandlerFunc`.**
Sans regarder le cours, écrire le type `HandlerFunc` et sa méthode. Puis écrire un `Handler` qui
**n'est pas** une fonction : un type `Static string` qui répond toujours sa propre valeur. Lequel des
deux styles préférer, et quand ?

**E3 — Une goroutine par requête.**
Écrire un handler qui incrémente un compteur **sans** verrou et répond sa valeur. Lancer 1 000
requêtes concurrentes contre un `httptest.NewServer` (avec un `WaitGroup` côté client). `-race` ?
Corriger avec `atomic.Int64` (N6 L7).

**E4 — Panique dans un handler.**
Un handler qui panique. Avec `httptest.NewServer`, que reçoit le client (`err` de `http.Get`) ? Le
serveur continue-t-il de répondre aux requêtes suivantes ? Que voit-on sur la sortie d'erreur ? Et avec
`httptest.NewRecorder` (appel direct) : que se passe-t-il ? Pourquoi la différence ?

**E5 — Fichiers statiques.**
Servir un répertoire avec `http.FileServerFS` et un `embed.FS` (N4 L7) sous `/static/`. Que renvoie
`/static/` (le répertoire lui-même) ? Comment empêcher le listage de répertoire ? Un chemin comme
`/static/../go.mod` est-il servi ?

---

## Exercice intermédiaire — Un service de raccourcissement d'URL

```
POST /links            corps : l'URL longue      → 201, Location: /l/{code}, corps : le code
GET  /l/{code}                                   → 302 vers l'URL longue, ou 404
GET  /links/{code}/stats                         → nombre de redirections (texte)
DELETE /links/{code}                             → 204
```

**Contraintes :**
1. Un type `Shortener` avec ses dépendances : un générateur de codes **injecté** (pour des tests
   déterministes, N5 L3) et l'état protégé par un verrou.
2. L'URL est validée avec `net/url.Parse` : schéma `http` ou `https` obligatoire, hôte non vide.
3. Les codes font 6 caractères base62 ; en production, générés avec `crypto/rand` (N4 L3) ; une
   collision est détectée et un autre code est tiré.
4. Le compteur de redirections est exact sous charge (test concurrent sous `-race`).
5. Tests `httptest` en table pour chaque route, cas d'erreur compris (méthode, code inconnu, URL invalide).
6. Un `cmd/shortener/main.go` qui lance le service sur `127.0.0.1:8080`.

---

## Défi — Lire le code du routeur

1. Lire `go doc net/http.ServeMux` en entier, puis la section « Patterns » : quelles règles de
   précédence n'ont pas été vues dans le cours ?
2. Quelle variable `GODEBUG` restaure l'ancien comportement du routeur (d'avant 1.22) ? Dans quel cas
   en aurait-on besoin ?
3. Écrire un test qui démontre qu'un motif avec hôte (`"api.local/v1/"`) l'emporte sur un motif sans
   hôte.
4. `r.PathValue` sur un joker qui n'existe pas dans le motif : que rend-il ? Comment éviter cette
   faute de frappe silencieuse ? *(Indice : des constantes, ou un test par route.)*

---

## Quiz

1. Quelle est l'unique méthode de `http.Handler` ?
2. À quoi sert `http.HandlerFunc` ?
3. Que permet un motif `"GET /notes/{id}"` depuis Go 1.22 ?
4. Que fait le routeur quand la route existe pour une autre méthode ?
5. Que signifie `{$}` ? Et `{path...}` ?
6. Quel motif gagne quand deux correspondent ?
7. Que se passe-t-il si deux motifs se chevauchent sans que l'un soit plus spécifique ?
8. Dans quelle goroutine un handler s'exécute-t-il ? Quelle conséquence ?
9. Différence entre `httptest.NewRecorder` et `httptest.NewServer` ?
10. Pourquoi ne pas utiliser `http.ListenAndServe` tel quel en production ?
