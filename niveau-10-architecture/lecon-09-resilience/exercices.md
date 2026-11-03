# Leçon 9 — Exercices

Code dans `mes-solutions/niveau-10/lecon-09/`, en partant du paquet `resil` de l'exemple.

## Exercices faciles

**E1 — Le troupeau, en vrai.**
Un serveur `httptest` qui répond 503 pendant ses 500 premières millisecondes, puis 200, et qui compte les
requêtes reçues par tranche de 50 ms. 200 clients appellent au même instant, avec 5 tentatives :
(a) réessais immédiats ; (b) attente fixe de 100 ms ; (c) exponentielle sans aléa ; (d) *full jitter*.
Tracer (en texte) le nombre de requêtes par tranche pour chaque stratégie. Laquelle laisse le serveur
« respirer » ?

**E2 — Le budget.**
Appeler `Retry` avec 5 tentatives et un contexte de 300 ms, sur une opération qui met 100 ms à échouer.
Combien de tentatives ont lieu ? Que rend `Retry` ? Ajouter un délai **par tentative** de 50 ms : que
change-t-il ?

**E3 — `Retry-After`.**
Étendre `Retry` : si l'erreur porte une durée imposée par le serveur (`429` ou `503` avec `Retry-After`,
représentée par un type `RetryAfter{Wait time.Duration}`), attendre cette durée au lieu du calcul
exponentiel — sauf si elle dépasse le budget restant (`ctx.Deadline()`), auquel cas abandonner tout de
suite.

**E4 — Un `POST` réessayé.**
Sans le middleware `Idempotent`, un client envoie `POST /loans`, le serveur crée l'emprunt mais la
connexion est coupée avant la réponse (simuler avec un `http.Hijacker` qui ferme la connexion, N7 L11), et
le client réessaie. Combien d'emprunts ? Avec le middleware et une clé : combien ?

**E5 — Le disjoncteur dans le temps.**
Avec l'horloge simulée, écrire le test qui vérifie : 3 échecs → ouvert ; à 29 s, refus ; à 30 s, un essai ;
essai échoué → ouvert de nouveau pour 30 s ; à 61 s, essai réussi → fermé ; un échec isolé ensuite ne
rouvre pas (le compteur est revenu à 0).

---

## Exercice intermédiaire — Un client résilient complet

Écrire `type CatalogClient` qui appelle `GET /works/{isbn}` d'un service externe, avec :

1. un délai par tentative (200 ms) et le respect du contexte de l'appelant ;
2. 3 tentatives sur erreurs réseau, délais, `502`, `503`, `504` et `429` (avec `Retry-After`) ; aucune sur
   `4xx` ;
3. un disjoncteur (5 échecs, 10 s) **autour** des réessais (pourquoi autour et pas à l'intérieur ?) ;
4. une cloison : au plus 10 appels simultanés vers ce service ; au-delà, erreur immédiate `ErrBusy` (ou
   attente bornée : choisir) ;
5. des compteurs (sûrs en concurrence) : appels, réessais, refus du disjoncteur, refus de la cloison ;
6. des tests avec `httptest` couvrant chaque cas, sous `-race`, sans attente réelle de plus d'une seconde
   au total (horloge et attentes injectées).

---

## Défi — Des clés d'idempotence pour plusieurs instances

1. Stocker les clés dans PostgreSQL : `idempotency(key text PRIMARY KEY, status int, body bytea,
   created_at timestamptz, completed boolean)`. La première requête **insère** la clé (`INSERT … ON CONFLICT
   DO NOTHING`) : si l'insertion réussit, elle exécute ; sinon, elle attend que `completed` soit vrai
   (interrogation courte avec délai), puis rejoue la réponse.
2. Deux instances du service (deux `httptest.Server` sur la même base) reçoivent la même clé en même
   temps : une seule exécution (vérifier par un compteur en base).
3. Que faire si l'instance qui exécute **tombe** avant de marquer `completed` ? Proposer une expiration
   (« clé en cours depuis plus de 30 s : considérée abandonnée ») et en discuter le risque.
4. Lier la clé au client (une clé identique envoyée par deux clients différents n'est pas un rejeu), et la
   faire expirer au bout de 24 h (purge par lots, leçon 8).
5. Pourquoi l'exécution de l'opération **et** l'enregistrement de la réponse devraient-ils être dans la même
   transaction quand l'opération elle-même est une écriture en base ?

---

## Quiz

1. Pourquoi les délais sont-ils le premier outil de résilience ?
2. Quelles deux conditions faut-il réunir pour réessayer ?
3. Quelles erreurs ne faut-il jamais réessayer ?
4. Pourquoi une attente exponentielle, et pourquoi y ajouter de l'aléa ?
5. Qu'est-ce que le *full jitter* ?
6. Décrire les trois états d'un disjoncteur et leurs transitions.
7. Que gagne l'appelant quand le disjoncteur est ouvert ?
8. Comment une clé d'idempotence rend-elle un `POST` réessayable ?
9. Que doit faire le serveur quand la même clé arrive pendant que la première requête s'exécute encore ?
10. Pourquoi ne pas réessayer à chaque niveau d'une chaîne d'appels ?
