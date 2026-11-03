# Module 7 — Exercices

Code dans `mes-solutions/niveau-13/module-07/`. Chaque réponse s'appuie sur une **mesure** : un tableau de
l'exemple, refait avec ta modification. Les scénarios de surcharge tournent en temps virtuel (`synctest`).

## Exercices faciles

**E1 — FIFO ou LIFO sous surcharge.**
Ajouter au serveur d'`overload` une option `LIFO` : sans abandon des requêtes expirées et avec une file de
1 000 places, les travailleurs prennent la requête la **plus récente**. Sur le scénario de l'exemple, sans
réessai, comparer le taux de succès par fenêtre en FIFO et en LIFO. Pourquoi LIFO sert-il mieux pendant la
pointe ? Qui paie ce choix ? (Indice : un canal est FIFO ; il faut une autre structure, protégée par un verrou,
et un moyen de réveiller les travailleurs.)

**E2 — Des priorités.**
Les requêtes portent une priorité : `Critical` (20 % du trafic) ou `BestEffort`. Quand la file dépasse la
moitié de sa capacité, le serveur refuse les requêtes `BestEffort`. Mesurer, pendant la pointe, le taux de
succès de chaque classe, avec et sans cette règle.

**E3 — Propager le délai.**
Le générateur de charge envoie son délai restant dans un en-tête `Request-Timeout` (en millisecondes). Le
serveur en fait un contexte (`context.WithTimeout`). Avant d'attendre une connexion à la base, il vérifie le
délai, et n'attend pas une connexion plus longtemps qu'il ne reste de temps. Mesurer, sans contrôle
d'admission, le débit utile à 1 000 et 1 200 req/s, avant et après. Compter les requêtes traitées pour rien
(la réponse arrive après le délai du client) dans les deux cas.

**E4 — Le contrôle d'admission qui ne voit rien.**
Remplacer le pool de connexions par un calcul d'environ 1 ms par requête (par exemple 4 500 tours de
`sha256.Sum256`, à calibrer par un benchmark), et lancer le serveur sur **un** cœur (`GOMAXPROCS=1 taskset -c 0`).
Refaire le test par paliers avec et sans `-limit 4`. Le contrôle d'admission refuse-t-il des requêtes ?
Prendre une trace d'exécution (N13 M4) d'une seconde de surcharge : où attendent les requêtes ? Proposer, et
mesurer si possible, une protection qui fonctionne.

**E5 — Injecter des pannes.**
Écrire un `http.RoundTripper` `Chaos` qui enveloppe un autre `RoundTripper`. Avec des probabilités réglables,
il ajoute un retard, ou renvoie une erreur sans appeler le serveur. Un service `page` appelle un service
`reco` (des recommandations) avec un délai de 50 ms, et affiche la page **sans** recommandations s'il
échoue. Écrire les tests :
- `reco` lent dans 30 % des cas : la page répond toujours en moins de 60 ms ;
- `reco` en panne : la page répond, sans recommandations, et un compteur de repli augmente ;
- les tests sont déterministes (une graine pour `Chaos`).

---

## Exercice intermédiaire — Une limite de concurrence adaptative

La limite d'admission de l'exemple (8) a été choisie après un test par paliers. Mais la capacité change : la
base est partagée, un index disparaît, une instance voisine tombe. Une limite **adaptative** se règle seule.

1. Écrire `adaptive.Limiter`, un contrôle d'admission dont la limite suit l'algorithme AIMD (comme TCP) :
   - à chaque requête terminée en moins de `Target` (par exemple 20 ms), la limite augmente de `1/limite`,
     soit d'environ 1 par « fenêtre » de requêtes ;
   - à chaque requête plus lente que `Target`, la limite est multipliée par 0,9, au plus une fois par
     `Target` ;
   - une requête qui trouve la limite atteinte est refusée tout de suite (503) ;
   - la limite reste entre `Min` et `Max`.
2. Le serveur de l'exemple démarre avec 4 connexions à la base. Au bout de 10 s, il passe à 16, puis à 2 au
   bout de 20 s (une goroutine prend ou rend des places dans le pool). Charge constante, en boucle ouverte, à
   2 000 req/s.
3. Tracer, par seconde, la limite, le débit utile, les refus et le p99. Comparer avec une limite fixe à 8.
4. Quels inconvénients ? Que se passe-t-il si `Target` est trop bas, ou trop haut ? Si une seule requête sur
   cent est lente pour une autre raison que la charge ?

---

## Défi — La panne métastable, en vrai

Reproduire l'expérience d'`overload` avec le vrai serveur HTTP et un vrai réseau local :

1. Le générateur de charge réessaie (3 réessais, attente exponentielle avec *jitter*, délai d'une seconde par
   tentative), en boucle ouverte. Il affiche, seconde par seconde, le débit demandé, le nombre de
   **tentatives** par seconde et le taux de succès des requêtes.
2. Scénario : 500 req/s pendant 10 s (environ 70 % de la capacité), 1 200 req/s pendant 5 s, puis 500 req/s
   pendant 30 s.
3. Montrer que le serveur **sans** contrôle d'admission ne se rétablit pas, ou très lentement, après la pointe.
   Expliquer avec la loi de Little ce qui maintient la file.
4. Ajouter le contrôle d'admission (`-limit 8`). Les clients réessaient-ils les 503 ? Mesurer avec et sans
   budget de réessais : combien de tentatives par seconde pendant et après la pointe ?
5. Trouver un réglage où le contrôle d'admission **seul** ne suffit pas, et où le budget est nécessaire
   (indice : quand un refus coûte cher, par exemple une poignée de main TLS ou une authentification avant le
   contrôle).

---

## Quiz

1. Quelle différence entre débit et débit utile ?
2. Quels sont les deux ingrédients d'une panne métastable ? Citer trois effets d'entretien.
3. Pourquoi corriger le déclencheur ne suffit-il pas ?
4. Pourquoi une file sans limite est-elle dangereuse, et comment dimensionner une file bornée ?
5. Que vérifier avant de commencer le travail d'une requête ?
6. Pourquoi l'attente exponentielle ne suffit-elle pas, et que fait un budget de réessais ?
7. Pourquoi un contrôle d'admission dans le gestionnaire ne protège-t-il pas un service limité par le
   processeur ?
8. Pourquoi servir en LIFO sous surcharge ?
9. Comment mesurer une capacité, et quelle marge prévoir ?
10. Pourquoi ajouter des instances peut-il ne pas augmenter la capacité ?
