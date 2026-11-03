# Leçon 1 — Exercices

Code dans `mes-solutions/niveau-10/lecon-01/`, en partant du module de l'exemple.

## Exercices faciles

**E1 — Tester `avant`.**
Écrire un test pour la règle « emprunt en retard » du paquet `avant`. Que faut-il faire pour simuler un
retard ? Combien de lignes, comparé à `TestBorrowRules` ? Que se passe-t-il si deux tests de ce paquet
s'exécutent en parallèle (`t.Parallel`) ?

**E2 — Lire les imports.**
Ajouter dans `loan` une fonction qui renvoie le statut HTTP d'une erreur (`func Status(err error) int`).
Que montre `go list` ? Quel paquet faut-il maintenant compiler pour tester le domaine ? Revenir en
arrière, puis tenter d'importer `memstore` depuis `loan` : quel message donne le compilateur, et
pourquoi ?

**E3 — Une deuxième porte d'entrée.**
Écrire un petit programme `cmd/emprunt` qui fait la même chose que la route HTTP depuis la ligne de
commande (`emprunt -member ada -book a`). Quelles lignes du domaine ou du service as-tu dû modifier ?

**E4 — Le retour.**
Ajouter la règle de retour : `Member.Return(bookID string, now time.Time) (lateDays int, err error)`, qui
retire l'emprunt et rend le nombre de jours de retard (0 si à l'heure). Tests du domaine d'abord, puis
service, puis route `DELETE /members/{id}/loans/{book}`.

**E5 — Quand ne pas séparer.**
Pour chacun de ces programmes, faut-il un paquet de domaine séparé ? Justifier en deux lignes : (a) un
proxy qui ajoute un en-tête et relaie les requêtes ; (b) un outil qui exporte une table PostgreSQL en
CSV chaque nuit ; (c) le calcul des pénalités de retard d'une bibliothèque, avec tarifs dégressifs et
exceptions pour les enfants ; (d) un service qui reçoit des webhooks de paiement et met à jour un
statut de commande.

---

## Exercice intermédiaire — Refactoriser sans casser

On part **uniquement** du paquet `avant`, comme d'un code hérité en production.

1. Avant de toucher au code, écrire des tests **de caractérisation** au niveau HTTP (`httptest`) qui
   figent le comportement actuel : statuts, corps, cas limites. Ils doivent passer sur `avant`.
2. Extraire pas à pas : d'abord la règle dans une fonction pure, puis le stockage derrière une interface,
   puis l'horloge. Après **chaque** étape, les tests de caractérisation passent toujours.
3. À la fin, les tests de caractérisation s'exécutent contre la nouvelle version (`httpapi.Routes`). Quels
   comportements de `avant` as-tu dû conserver alors qu'ils te semblaient discutables (par exemple le
   corps des erreurs) ? Lesquels as-tu décidé de changer, et comment le documenter ?
4. Mesurer : nombre de lignes, temps d'exécution des tests, nombre de tests possibles sans serveur.

---

## Défi — Une règle qui dépend de plusieurs agrégats

Nouvelle règle : un adhérent ne peut pas emprunter un livre **déjà emprunté** par quelqu'un d'autre, et un
livre réservé ne peut être emprunté que par la personne qui l'a réservé.

1. Où placer cette règle : dans `Member`, dans un nouveau type `Book`, dans le service ? Écrire deux
   versions et comparer leur testabilité.
2. La règle doit tenir sous concurrence (deux adhérents empruntent le même exemplaire au même instant).
   Avec le stockage en mémoire, qui garantit l'exclusion ? Et avec PostgreSQL (N8 L4) ?
3. Discuter : la règle de concurrence appartient-elle au domaine ou au stockage ? (La leçon 6 donne un
   vocabulaire pour répondre : les agrégats.)

---

## Quiz

1. Quelles sont les trois responsabilités d'un service, et pourquoi les séparer ?
2. Que ne doit jamais importer un paquet de domaine ?
3. Dans quel sens vont les dépendances ?
4. Comment lire l'architecture d'un programme Go en une commande ?
5. Pourquoi passer « maintenant » en paramètre d'une règle ?
6. Où est déclarée l'interface de stockage, et pourquoi ?
7. Que fait le service, et que ne fait-il pas ?
8. Pourquoi `memstore.Get` rend-il une copie ?
9. Citer deux cas où séparer en couches n'est pas justifié.
10. Qu'est-ce qu'un test de caractérisation, et quand l'écrire ?
