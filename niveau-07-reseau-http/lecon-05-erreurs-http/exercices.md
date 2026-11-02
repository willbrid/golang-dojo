# Leçon 5 — Exercices

Code dans `mes-solutions/niveau-07/lecon-05/`.

## Exercices faciles

**E1 — La fuite.**
Remplacer le `default` de `toProblem` par `Detail: err.Error()`. Quelle information sort pour le
compte « panne » ? Lister trois usages qu'un attaquant pourrait en faire. Revenir à la version sûre.

**E2 — Comparer des messages.**
Réécrire `toProblem` avec `strings.Contains(err.Error(), "introuvable")`. Modifier ensuite le message de
`ErrNotFound` (« compte inexistant »). Quels tests cassent, et lesquels **auraient dû** casser mais
passent ?

**E3 — L'identifiant de requête.**
Combiner avec le middleware `WithRequestID` de la leçon 3 : ajouter l'identifiant au `Problem` (champ
d'extension `request_id`) et au journal. Pourquoi est-ce la seule information « interne » qu'on peut
donner au client ?

**E4 — 404 ou 403.**
Ajouter une notion de propriétaire au compte et un en-tête `X-User` (authentification simulée). Un
utilisateur qui demande le compte d'un autre reçoit… 403 ou 404 ? Implémenter les deux variantes et
écrire, en trois lignes, quand choisir chacune.

**E5 — Erreur après écriture.**
Un handler encode une longue liste en JSON directement dans `w` et rencontre une erreur au milieu
(simulée). Il rend cette erreur. Que reçoit le client ? Que dit le journal du serveur ? Corriger en
encodant d'abord dans un `bytes.Buffer`. Quel est le coût de cette correction pour de très grosses
réponses ?

---

## Exercice intermédiaire — Un modèle d'erreur pour l'API de bibliothèque

Reprendre l'API de bibliothèque (leçon 4, exercice intermédiaire) :

1. Extraire un paquet de domaine `library` sans aucune dépendance à `net/http`, avec ses sentinelles
   (`ErrBookNotFound`, `ErrAlreadyLoaned`…) et un `*ValidationError`.
2. Un paquet `httpapi` avec `HandlerFunc`, `Adapt` et une **table** de correspondance déclarative :

   ```go
   var mapping = []struct {
   	err    error
   	status int
   	typ    string
   }{ … }
   ```

   parcourue avec `errors.Is`. Que faire des types (`errors.As`) qui ne rentrent pas dans cette table ?
3. Toutes les routes passent par `Adapt`. Aucun `http.Error` ne reste dans le code.
4. Un test qui parcourt **toutes** les sentinelles du domaine et vérifie qu'aucune ne tombe dans le cas
   500 par oubli. *(Indice : une liste des sentinelles exportée pour les tests, ou un test qui échoue
   quand on ajoute une sentinelle sans l'ajouter à la table.)*
5. Le client reçoit `application/problem+json` partout, y compris pour les 404 et 405 **du routeur**.
   Comment intercepter ces réponses-là, qui ne passent pas par vos handlers ?

---

## Défi — Des erreurs pour les humains et pour les machines

1. Ajouter au `Problem` un champ `code` stable et court (`INSUFFICIENT_FUNDS`) en plus du `type`.
   Quel intérêt pour un client mobile qui doit traduire le message dans 12 langues ?
2. Localiser `title` selon l'en-tête `Accept-Language` (fr, en), sans toucher au domaine.
3. Pour les erreurs de validation, adopter la convention de pointeurs JSON (RFC 6901 : `/items/3/quantity`)
   pour désigner le champ fautif. Écrire la validation d'une commande à plusieurs lignes qui produit ces
   pointeurs.

---

## Quiz

1. Pourquoi le domaine ne doit-il pas connaître les statuts HTTP ?
2. Pourquoi traduire les erreurs à un seul endroit ?
3. Que doit rendre la traduction pour une erreur inconnue ?
4. Quel avantage à des handlers qui rendent `error` ?
5. Quels sont les membres d'un *problem details* ? Sur lequel les clients doivent-ils se brancher ?
6. Quel type de média utiliser ?
7. Quelles informations ne doivent jamais sortir vers le client ?
8. Faut-il journaliser les erreurs 4xx ?
9. Quand répondre 404 plutôt que 403 ?
10. Pourquoi un handler ne doit-il rien avoir écrit avant de rendre une erreur ?
