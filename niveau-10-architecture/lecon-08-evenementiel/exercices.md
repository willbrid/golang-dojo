# Leçon 8 — Exercices

Code dans `mes-solutions/niveau-10/lecon-08/`, en partant du module de l'exemple. PostgreSQL et
`DATABASE_URL` requis.

## Exercices faciles

**E1 — La double écriture.**
Écrire `BorrowNaive` qui valide l'emprunt puis appelle `bus.Publish` directement. Simuler une panne entre
les deux (une fonction `crash()` injectée qui rend une erreur juste avant `Publish`). Compter emprunts et
événements publiés. Inverser l'ordre (publier puis valider, avec une transaction qui échoue) : que
compte-t-on ?

**E2 — Retrouver le *livelock*.**
Remettre la première version du relais (tout annuler à chaque échec de publication) et lancer le test avec
un délai court (`-timeout 10s`). Que montre la trace des goroutines ? Avec `failEvery: 11` et des lots de
10, le test se termine-t-il ? Pourquoi ?

**E3 — Le consommateur naïf.**
Retirer la table `processed` et incrémenter les statistiques à chaque message reçu. Que vaut `total` ? Le
résultat est-il stable d'une exécution à l'autre ?

**E4 — Sans `SKIP LOCKED`.**
Retirer `SKIP LOCKED` (garder `FOR UPDATE`). Les trois relais fonctionnent-ils encore ? Mesurer la durée du
test avec et sans. Que font les relais pendant ce temps ?

**E5 — L'âge de l'*outbox*.**
Écrire la requête qui donne le nombre d'événements en attente et l'âge du plus ancien (ajouter une
colonne `created_at`). Pourquoi est-ce la métrique la plus importante de ce dispositif ?

---

## Exercice intermédiaire — Ordre par adhérent et messages empoisonnés

1. Ajouter à chaque événement un numéro de séquence **par adhérent** (`seq`, attribué dans la transaction
   de `Borrow` : comment, sans mise à jour perdue ?).
2. Le consommateur ne doit **jamais** appliquer l'événement `seq = n` d'un adhérent avant `n-1` : s'il reçoit
   `n` trop tôt, il le met de côté (ou le refuse, pour qu'il soit relivré plus tard). Tester avec un
   courtier qui **mélange** l'ordre des messages.
3. Un message dont le JSON est invalide : après 3 tentatives, il est déplacé dans une table `dead_letters`
   avec l'erreur, et le consommateur continue. Tester.
4. Mesurer : avec ce mécanisme, combien de messages sont mis de côté ou relivrés, pour 200 événements et un
   courtier qui mélange ?

---

## Défi — Un relais qui tourne en permanence

1. Transformer `RelayOnce` en `Run(ctx)` : boucle tant que `ctx` n'est pas annulé ; quand l'*outbox* est
   vide, attendre (200 ms) avant de relire ; en cas d'erreur du courtier, attendre de plus en plus
   longtemps (leçon 9).
2. Réveil immédiat au lieu d'attendre : utiliser `LISTEN`/`NOTIFY` de PostgreSQL (`pg_notify` dans la
   transaction de `Borrow`, `pgx` natif côté relais : `conn.WaitForNotification`). Mesurer la latence entre
   `Borrow` et la publication, avec et sans `NOTIFY`.
3. Purge : supprimer par lots les événements publiés depuis plus de 7 jours, sans verrouiller longtemps
   l'*outbox*.
4. Arrêt gracieux : à l'annulation, le relais termine le lot en cours et rend la main ; tester qu'aucun
   événement n'est perdu ni laissé verrouillé.

---

## Quiz

1. Quelle différence entre une file et une publication/abonnement ?
2. Quelle différence entre un événement et une commande ?
3. Que garantissent « au plus une fois » et « au moins une fois » ?
4. Pourquoi la livraison « exactement une fois » n'existe-t-elle pas, et qu'obtient-on à la place ?
5. Qu'est-ce que le problème de la double écriture ?
6. Comment l'*outbox* transactionnelle le résout-elle ?
7. À quoi sert `FOR UPDATE SKIP LOCKED` dans un relais ?
8. Comment rendre un consommateur idempotent ?
9. Pourquoi un relais qui annule tout son lot à chaque échec peut-il ne jamais progresser ?
10. Que faire d'un message qu'on ne pourra jamais traiter ?
