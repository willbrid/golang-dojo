# Leçon 7 — Exercices

Code dans `mes-solutions/niveau-07/lecon-07/`.

## Exercices faciles

**E1 — La chaîne coupée.**
Dans l'exemple, remplacer `r.Context()` par `context.Background()`. Relancer
`TestClientDisconnectCancelsStore` : que se passe-t-il ? Combien de temps la « base » travaille-t-elle
pour rien ?

**E2 — Observer l'abandon.**
Écrire un handler qui affiche (dans un journal injecté) `context.Cause(r.Context())` quand il voit
l'annulation. Le déclencher avec `curl --max-time 1` sur une requête de 5 s. Quelle cause ?

**E3 — Le budget partagé.**
Un handler avec un budget de 1 s appelle successivement trois dépendances simulées de 400 ms chacune.
Laquelle est interrompue ? Quelle erreur rend-elle ? Quel statut reçoit le client ?

**E4 — Le mauvais contexte d'arrêt.**
Dans `Run`, passer `ctx` (le contexte de signal) à `Shutdown` au lieu d'un contexte neuf. Relancer
`TestGracefulShutdown` : la requête en cours se termine-t-elle ? Que rend `Shutdown` ?

**E5 — Les goroutines de fond.**
Ajouter au service une goroutine qui purge un cache toutes les secondes. S'assurer qu'elle s'arrête avec
le même contexte que le serveur, et que `main` attend sa fin avant de sortir. Le vérifier avec
`runtime.NumGoroutine()` ou le profil `goroutineleak` (N6 L10).

---

## Exercice intermédiaire — Une passerelle

Écrire un service `gateway` qui expose `GET /v1/dashboard` et, pour chaque requête, appelle **en
parallèle** deux services distants (simulés par des `httptest.NewServer`) : `profile` et `orders`.

**Contraintes :**
1. Budget total de 800 ms par requête, propagé aux deux appels sortants (`NewRequestWithContext`).
2. `errgroup.WithContext` (N6 L9) : si l'un échoue, l'autre est annulé.
3. Si `orders` dépasse son budget mais que `profile` a répondu, renvoyer un tableau de bord **partiel**
   (`"orders": null` et un champ `"degraded": true`) plutôt qu'une erreur. *(Indice : un budget séparé et
   plus court pour `orders`, et ne pas le faire échouer le groupe.)*
4. Si le client de la passerelle se déconnecte, les deux appels sortants sont annulés : le vérifier en
   test côté services distants (comme `TestClientDisconnectCancelsStore`).
5. Arrêt gracieux avec `signal.NotifyContext`.

---

## Défi — Arrêt en deux temps

Simuler le comportement attendu derrière un répartiteur de charge :

1. Ajouter `GET /readyz` qui répond 200 tant que le service accepte du trafic.
2. À la réception du signal : `/readyz` passe immédiatement à 503 ; le service **continue de servir**
   normalement pendant 5 secondes (le temps que le répartiteur le retire) ; puis `Shutdown` avec un délai
   de 20 s.
3. Écrire un test (avec des durées réduites) qui vérifie la séquence : `/readyz` à 503, requêtes
   ordinaires toujours servies, puis refus des connexions.
4. Pourquoi ne pas simplement appeler `Shutdown` immédiatement ? Que verraient les clients pendant les
   quelques secondes où le répartiteur envoie encore du trafic ?

---

## Quiz

1. Quand le contexte d'une requête est-il annulé ?
2. Pourquoi ne jamais créer `context.Background()` dans un handler ?
3. Comment fixer un budget par requête ?
4. Que se passe-t-il pour le budget quand une première dépendance a consommé une partie du temps ?
5. Comment répondre à `DeadlineExceeded` ? à `Canceled` ?
6. Que fait `srv.Shutdown` ?
7. Pourquoi passer un contexte neuf à `Shutdown` ?
8. Que rend `ListenAndServe` après un `Shutdown` ?
9. Comment transformer `SIGTERM` en annulation de contexte ?
10. Que ne gère pas `Shutdown` automatiquement ?
