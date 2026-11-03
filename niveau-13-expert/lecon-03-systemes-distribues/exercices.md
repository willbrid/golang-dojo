# Module 3 — Exercices

Code dans `mes-solutions/niveau-13/module-03/`. Les simulations sont **déterministes** : une graine fixée, un
temps virtuel, aucun `time.Sleep`. Un résultat surprenant se rejoue à l'identique.

## Exercices faciles

**E1 — Détecter les écritures concurrentes.**
Remplacer « le dernier écrit gagne » par des **vecteurs de versions**. Chaque réplique tient un compteur par
réplique, et une écriture porte le vecteur qu'elle a lu. Écrire `Compare(a, b)` qui renvoie « avant »,
« après », « égal » ou « concurrent ». Sur le scénario de `LostWrites`, combien d'écritures sont détectées
comme **concurrentes** au lieu d'être perdues en silence ? Que doit faire le système d'un conflit détecté ?

**E2 — Lire ses propres écritures.**
Avec N=3, W=1, R=1 (66,6 % de lectures périmées), le client retient la **version** de sa dernière écriture et
n'accepte qu'une réplique qui l'a. Si la réplique tirée est en retard, il en essaie une autre, puis attend.
Mesurer :
- la proportion de lectures servies au premier essai ;
- le nombre moyen d'essais ;
- le temps d'attente au p99.

Comparer avec W=R=2.

**E3 — Le quorum et la latence de queue.**
Les répliques répondent en 2 ms, sauf une fois sur cent où une réplique met 200 ms. Mesurer le p50 et le p99 du
temps d'une écriture pour W = 1, 2 et 3 sur N = 3. Pourquoi W = 2 protège-t-il de la queue de **chaque**
réplique ? Relier au module N11 L7 (la page qui attend le plus lent de ses services).

**E4 — Réessayer sans dupliquer.**
Simuler un serveur qui traite les requêtes, mais dont la **réponse** se perd une fois sur dix. Le client
réessaie après un délai. Sur 10 000 virements, combien sont exécutés deux fois, sans puis avec une clé
d'idempotence enregistrée côté serveur ? Que doit-on enregistrer, et combien de temps le garder ?

**E5 — Le bail et l'horloge du client.**
Le client calcule l'échéance de son bail avec **sa propre** horloge, qui avance 10 % trop lentement. Le
service de verrou, lui, a une horloge juste. Pendant combien de temps le client se croit-il encore titulaire
d'un bail que le service a déjà réattribué ? Quelle marge de sécurité faut-il prendre ? Pourquoi le jeton de
clôture reste-t-il nécessaire même avec une marge ?

---

## Exercice intermédiaire — Un verrou à bail sur PostgreSQL

Plusieurs processus se disputent un rôle unique (par exemple, le « planificateur » qui lance des tâches
périodiques). La base PostgreSQL du niveau 8 sert d'arbitre.

1. Une table `leases(name text primary key, holder text, expires_at timestamptz, token bigint)`.
   - `Acquire` prend le bail s'il est libre ou expiré, et incrémente le jeton, dans **une** instruction SQL
     atomique.
   - `Renew` prolonge le bail seulement si l'on en est toujours titulaire.
   - L'heure utilisée est **celle de la base** (`now()`), jamais celle du processus.
2. La ressource protégée est une autre table, dans laquelle chaque écriture porte le jeton. Une écriture dont
   le jeton est inférieur au plus grand déjà vu est refusée, dans la même instruction.
3. Lancer 3 processus (ou goroutines avec chacune sa connexion) qui tentent d'acquérir le bail toutes les
   100 ms et écrivent tant qu'ils le tiennent. Injecter des **pauses** aléatoires de 0 à 500 ms dans le
   titulaire (bail de 300 ms).
4. Vérifier :
   - qu'à aucun moment deux écritures **acceptées** ne portent des jetons en ordre décroissant ;
   - que les écritures d'un titulaire mis en pause sont refusées.

   Compter les écritures refusées. Que se passerait-il sans la vérification du jeton ?
5. Comparer, en une page, avec les verrous consultatifs de PostgreSQL (`pg_try_advisory_lock`, N8 L6) :
   ce qui est plus simple, et ce qu'on perd (indice : que devient le verrou si la **connexion** reste
   ouverte mais que le processus est figé ?).

---

## Défi — Chercher le scénario qui casse

Un petit protocole d'élection : chaque nœud se proclame leader s'il n'a pas reçu de battement de cœur
(*heartbeat*) depuis 300 ms. Le leader envoie un battement toutes les 100 ms. Les messages ont un retard
aléatoire et peuvent être perdus.

1. Écrire une simulation à événements discrets de 3 nœuds, avec une graine, un réseau qui retarde (0 à
   50 ms), perd (paramétrable) et peut **partitionner** pendant un intervalle.
2. Propriété : à aucun instant **deux** leaders n'écrivent en même temps sur la ressource partagée. Lancer
   10 000 graines aléatoires avec des pannes aléatoires, et trouver une graine qui viole la propriété.
3. Réduire le scénario : la plus petite configuration (pertes, partition, durée) qui la viole encore, à
   rejouer à l'identique dans un test.
4. Corriger, par exemple avec des mandats numérotés et un jeton de clôture vérifié par la ressource. Vérifier
   que 10 000 graines ne trouvent plus de violation.
5. Expliquer pourquoi le correctif ne garantit pas qu'il n'y a **jamais** deux nœuds qui se croient leaders,
   et pourquoi ce n'est pas grave.

---

## Quiz

1. Quelles sont les trois issues d'un appel distant, et qu'en déduire pour les réessais ?
2. Pourquoi « le dernier écrit gagne » perd-il des écritures ?
3. Que garantit une horloge de Lamport, et que ne garantit-elle pas ?
4. Pourquoi W + R > N garantit-il de lire la dernière écriture ?
5. Que coûte W = N ? Et W = 1 ?
6. Qu'est-ce que la cohérence à terme ?
7. Quel choix une partition impose-t-elle ?
8. Sur quelle propriété des majorités reposent les algorithmes de consensus ?
9. Pourquoi un verrou à bail ne suffit-il pas, et que fait le jeton de clôture ?
10. Comment tester un protocole réparti ?
