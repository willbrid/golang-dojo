# Leçon 7 — Exercices

Code dans `mes-solutions/niveau-11/lecon-07/`, en partant du paquet `lat` de l'exemple.

## Exercices faciles

**E1 — L'amplification par la queue.**
Une page appelle n services en parallèle et attend la réponse la plus lente. La latence de chaque service suit
la distribution simulée par `Simulate(…, ρ = 0.7, …)`. Tirer au hasard n durées dans cette distribution,
prendre le maximum, recommencer 100 000 fois. Donner le p50 et le p99 de la page pour n = 1, 10, 100.
Comparer le p50 de la page avec le p99 d'un seul service.

**E2 — Little pour dimensionner.**
Un service reçoit 2 000 requêtes/s ; chacune garde une connexion à la base pendant 5 ms en moyenne.
(a) Combien de connexions sont occupées en moyenne ? (b) Avec `Simulate` (λ = 2 par ms, μ = 0,2 par ms et
par worker, file commune), trouver le plus petit nombre de connexions c tel que le temps de séjour au p99
reste sous 20 ms, s'il existe. Que vaut alors le taux d'occupation de chaque connexion ?

**E3 — La variabilité.**
Ajouter à `Simulate` une option de durée de service **constante** (toujours 1/μ). Comparer, à ρ = 0,9 et un
worker, le temps d'**attente** moyen (séjour moins service) avec des durées constantes et exponentielles.
Quel rapport trouve-t-on ? Qu'en déduire pour un service dont quelques requêtes sont 100 fois plus longues que
les autres ?

**E4 — Corriger la boucle fermée.**
Modifier `Closed` pour que chaque worker suive un calendrier (une requête toutes les 8 ms) et mesure la durée
depuis l'instant **prévu**. Relancer le test de la panne. La mesure se rapproche-t-elle de la boucle
ouverte ? Que reste-t-il de différent (indice : combien de requêtes le serveur reçoit-il **pendant** la
panne) ?

**E5 — Les percentiles ne se moyennent pas.**
Simuler 10 serveurs : 9 à ρ = 0,5 et un à ρ = 0,95, chacun recevant le même nombre de requêtes. Comparer la
moyenne des 10 p99 avec le p99 de l'ensemble des durées. Lequel décrit ce que voient les clients ?

---

## Exercice intermédiaire — Un test de charge par paliers

Un handler qui simule un service à capacité limitée : un sémaphore de 4 places (un canal bufferisé, N6 L3),
et un traitement de 2 ms en moyenne, de durée aléatoire (`rand.ExpFloat64`). Sa capacité théorique est donc
de 2 000 requêtes/s.

1. Avec `Open`, envoyer 30 s de charge à 50 %, 70 %, 90 %, 100 % et 110 % de la capacité. Relever à
   chaque palier le débit obtenu, le p50, le p99, et le nombre de requêtes en cours (Little).
2. Comparer avec `Simulate(…, c = 4, …)` aux mêmes taux. Où la mesure s'écarte-t-elle de la simulation, et
   pourquoi (temps de réseau, ordonnancement, précision de `time.Sleep`) ?
3. Ajouter un contrôle d'admission (N6 L11) : au-delà de 16 requêtes en attente, répondre 503 tout de suite.
   Refaire le palier à 110 % : que deviennent le p99 des requêtes acceptées et le taux de refus ?
4. Écrire en cinq lignes la capacité utile de ce service et la règle d'admission que tu recommandes.

---

## Défi — L'effondrement par les relances

Les clients d'un service ont un délai de 50 ms ; au-delà, ils abandonnent et **réessaient** aussitôt. La
requête abandonnée reste dans la file du serveur et sera tout de même traitée (le serveur ne le sait pas).

1. Étendre la simulation : un worker, service moyen 1 ms, arrivées à λ = 0,8 par ms. Toute requête dont le
   séjour dépasserait 50 ms produit une nouvelle arrivée 50 ms après la sienne. Mesurer le débit utile
   (réponses reçues dans les délais) et le débit total traité.
2. Ajouter un pic : λ = 1,5 pendant 200 ms, puis retour à 0,8. Le système revient-il à son régime normal ?
   Tracer la longueur de la file dans le temps.
3. Trouver deux remèdes et les mesurer. Pistes : le serveur jette les requêtes déjà trop vieilles quand il les
   prend (`ctx.Err()`, N6 L11) ; les relances avec attente exponentielle et un budget de relances (N10 L9) ;
   une file bornée.
4. Expliquer pourquoi ce système a **deux** régimes stables pour le même λ = 0,8, et pourquoi on parle
   d'un état « métastable ».

---

## Quiz

1. Pourquoi décrire une latence par des percentiles plutôt que par une moyenne ?
2. Une page appelle 50 services ; chacun dépasse son p99 une fois sur cent. Quelle proportion de pages est
   touchée ?
3. Énoncer la loi de Little. Que suppose-t-elle sur les distributions ?
4. Un pool de connexions : 500 requêtes/s, 20 ms de connexion chacune. Combien de connexions en moyenne ?
5. Comment le temps de séjour d'une file à un worker évolue-t-il avec le taux d'occupation ?
6. Pourquoi une file commune vaut-elle mieux qu'une file par worker ?
7. Qu'est-ce que l'omission coordonnée ? Pourquoi un générateur en boucle fermée y est-il sujet ?
8. Depuis quel instant faut-il mesurer la latence dans un générateur en boucle ouverte ?
9. Que mesure le débit d'un générateur en boucle fermée ?
10. Pourquoi ne faut-il pas moyenner des p99 ?
