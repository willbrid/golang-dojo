# Leçon 2 — Exercices

Code dans `mes-solutions/niveau-10/lecon-02/`, en partant du module de l'exemple.

## Exercices faciles

**E1 — Déplacer l'autorisation.**
Déplacer la vérification « un adhérent n'emprunte que pour lui-même » dans le handler. Les tests passent-
ils toujours ? Écrire ensuite une seconde porte d'entrée (une fonction `ImportBatch` qui appelle le service
pour une liste d'emprunts lus dans un fichier) : que devient la règle ?

**E2 — Retirer la transaction.**
Remplacer `WithinTx` par des appels directs au stockage, sans brouillons. Quel test échoue ? Lequel des deux
(atomicité, concurrence) aurait pu passer par chance ?

**E3 — Un champ de trop.**
Supprimer les DTO et encoder directement `Copy` en JSON. Qu'apparaît-il dans la réponse ? Ajouter à `Copy`
un champ interne `InternalNote string` : que se passe-t-il ? Quel mécanisme empêchait cela ?

**E4 — Le CRUD.**
Ajouter `GET /members/{id}` qui rend l'adhérent et son nombre d'emprunts. Faut-il une méthode de service ?
Écrire la version avec et la version sans, et choisir. Que change l'ajout ultérieur de la règle « un
adhérent ne voit que sa propre fiche, un bibliothécaire toutes » ?

**E5 — Trois étages de validation.**
Pour chacune de ces vérifications, dire à quel étage elle appartient (handler, domaine, base) et quel
statut elle produit : `copy_id` absent ; `copy_id` de 10 000 caractères ; exemplaire inexistant ;
exemplaire déjà emprunté ; deux emprunts simultanés du même exemplaire avec un stockage PostgreSQL ;
corps qui n'est pas du JSON ; adhérent suspendu.

---

## Exercice intermédiaire — Le retour, de bout en bout

Ajouter le cas d'usage `Return(ctx, who Actor, copyID string) (lateDays int, err error)` :

1. Règles : seul l'emprunteur ou un bibliothécaire peut rendre ; un exemplaire non emprunté → erreur ;
   le compteur de l'adhérent diminue ; les jours de retard sont calculés comme en leçon 1 (jour entamé).
2. Au-delà de 7 jours de retard, l'adhérent est **suspendu** (champ `SuspendedUntil`) pour autant de jours
   que de retard ; un adhérent suspendu ne peut plus emprunter. Où vit chaque règle ?
3. Route `DELETE /loans/{copy}` ; DTO de réponse `{"late_days": n, "suspended_until": "…"}` (champ absent
   si pas de suspension, `omitzero`).
4. Tests : domaine (table des retards), service (autorisation, atomicité avec `FailOn`), handler (statuts).
5. Mesurer la part de chaque couche dans le nombre de lignes. La couche service est-elle justifiée ici ?

---

## Défi — Un stockage PostgreSQL pour `Store`

1. Implémenter `Store` avec PostgreSQL (N8) : `WithinTx` ouvre une transaction, les `Repos` travaillent
   sur le `*sql.Tx` (`DBTX`, N8 L7), les lectures verrouillent (`FOR UPDATE`) dans un ordre fixe.
2. Écrire une **suite de contrat** pour `Store` (création, lecture, rollback sur erreur, concurrence) et
   l'exécuter contre `MemStore` et contre PostgreSQL.
3. Le test de concurrence de l'exemple, avec 50 adhérents qui empruntent le même exemplaire : un seul
   réussit, sur les deux implémentations.
4. Remplacer le verrou de `MemStore` (un seul pour tout le stockage) par un verrou par exemplaire. Le
   contrat passe-t-il encore ? Qu'as-tu dû garantir de plus (pense à l'adhérent, modifié en même
   temps) ?

---

## Quiz

1. Que fait un handler, et que ne fait-il pas ?
2. Quelle différence entre validation syntaxique, métier et d'intégrité, et qui fait chacune ?
3. Pourquoi l'autorisation appartient-elle au service ?
4. Pourquoi la transaction est-elle ouverte par le service, et comment sans `database/sql` ?
5. Qu'est-ce qu'un DTO, et pourquoi ne pas encoder directement les types du domaine ?
6. Pourquoi nommer les méthodes du service d'après les cas d'usage ?
7. Organisation par couche ou par fonctionnalité : avantages de la seconde ?
8. Quand un service est-il une couche vide ?
9. Pourquoi vérifier l'autorisation avant d'ouvrir la transaction ?
10. Que doit garantir un stockage en mémoire utilisé à la place d'une base dans les tests ?
