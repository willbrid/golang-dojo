# Leçon 4 — Exercices

Code dans `mes-solutions/niveau-07/lecon-04/`.

## Exercices faciles

**E1 — Critiquer une API.**
Voici les routes d'une API existante. Pour chacune, dire ce qui ne va pas et proposer la version REST :

```
GET  /getAllUsers
POST /user/create
GET  /user?id=42
POST /user/42/delete
POST /updateUserEmail        corps : {"id": 42, "email": "…"}
GET  /users/42/orders/7/items/3/reviews
```

**E2 — Idempotence.**
Pour chaque opération, dire si elle est idempotente telle qu'écrite, et sinon comment la rendre
idempotente : (a) `PUT /tasks/42` avec la tâche complète ; (b) `POST /accounts/7/deposits` avec un
montant ; (c) `PATCH /counters/9` avec `{"increment": 1}` ; (d) `PATCH /tasks/42` avec `{"done": true}`.

**E3 — Absent ou zéro.**
Remplacer les pointeurs du `PATCH` de l'exemple par des types valeur. Envoyer `{"title":"x"}` :
qu'arrive-t-il au champ `done` d'une tâche terminée ? Écrire le test qui révèle le problème.

**E4 — Pagination instable.**
Implémenter une version de `list` par **décalage** (`?offset=…&limit=…`). Écrire un test : lire la page
1 (limite 2), supprimer la tâche 1, lire la page 2. Quel élément a été sauté ? Refaire le même scénario
avec le curseur.

**E5 — Curseur opaque.**
Encoder le curseur en base64 (`encoding/base64.RawURLEncoding`) pour qu'il ne soit plus un simple
identifiant. Pourquoi est-ce préférable pour l'évolution de l'API, même si ce n'est pas une
protection de sécurité ?

---

## Exercice intermédiaire — Une API de bibliothèque

Concevoir **puis** implémenter une API de gestion de livres et d'emprunts.

**Étape 1 — sur papier, avant tout code :** un tableau des routes (méthode, chemin, corps, statuts de
succès et d'erreur) pour : livres (CRUD), auteurs (lecture), emprunts (emprunter, rendre, lister les
emprunts en cours d'un membre). Décider comment modéliser « rendre un livre ». Justifier chaque choix
en une ligne.

**Étape 2 — implémentation :**
1. Toutes les routes sous `/v1/`, motifs 1.22+.
2. Emprunter un livre déjà emprunté → 409 ; rendre un livre non emprunté → 409.
3. `GET /v1/books?author=…&available=true&limit=…&cursor=…` : filtres en liste blanche, pagination par
   curseur.
4. `POST /v1/loans` accepte un en-tête `Idempotency-Key` : un même emprunt renvoyé avec la même clé
   rend la même réponse sans créer de second emprunt.
5. Un test « conversation » qui couvre le parcours complet, cas d'erreur compris.

---

## Défi — Faire évoluer sans casser

L'API de l'exemple doit évoluer :

1. Ajouter un champ `due` (date d'échéance, facultative). Est-ce un changement compatible ? Que doivent
   faire les clients existants ?
2. Le métier demande de renommer `done` en `status` (valeurs `todo`, `doing`, `done`). Proposer une
   stratégie : nouvelle version `/v2/` ? Les deux champs en parallèle pendant une période ? Écrire le
   code qui sert `/v1/` et `/v2/` à partir du **même** stockage.
3. Ajouter aux réponses `/v1/` les en-têtes `Deprecation` et `Sunset` (chercher les RFC
   correspondantes). Que doit en faire un client bien écrit ?

---

## Quiz

1. Pourquoi des noms au pluriel et pas de verbe dans les URL ?
2. Que doit rendre un `POST` de création ?
3. Quelles méthodes sont idempotentes ? Pourquoi cette propriété compte-t-elle ?
4. Pourquoi un second `DELETE` doit-il réussir ?
5. Comment un serveur rend-il un `POST` idempotent ?
6. Comment distinguer en Go un champ absent d'un champ à zéro dans un `PATCH` ?
7. Quels sont les deux défauts de la pagination par décalage ?
8. Pourquoi envelopper une liste dans un objet JSON ?
9. Quels changements sont compatibles dans une version d'API ?
10. Versionnement par chemin ou par en-tête : avantages de chacun ?
