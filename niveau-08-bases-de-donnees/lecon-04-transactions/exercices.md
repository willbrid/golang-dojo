# Leçon 4 — Exercices

Code dans `mes-solutions/niveau-08/lecon-04/`. Tous les tests concurrents se lancent avec `-race`.

## Exercices faciles

**E1 — La mise à jour perdue, chez toi.**
Reproduire le tableau du cours : 100 retraits concurrents de 1 sur un solde de 1 000, en version naïve,
`FOR UPDATE`, `REPEATABLE READ` et `SERIALIZABLE`. Ajouter une cinquième variante : l'`UPDATE` atomique
`… SET balance = balance - 1 WHERE id = 1 AND balance >= 1`. Lancer chaque variante cinq fois : le
nombre de retraits perdus de la version naïve est-il stable ? Qu'en conclure sur la capacité d'un test
à attraper ce bug ?

**E2 — `db` au lieu de `tx`.**
Dans `Transfer`, remplacer le second `tx.ExecContext` par `db.ExecContext`. Que se passe-t-il pour un
virement vers le compte 99 (le débit est-il annulé) ? Puis, avec `db.SetMaxOpenConns(1)` et un contexte
de 2 s : que se passe-t-il pour **n'importe quel** virement, et pourquoi ?

**E3 — La transaction oubliée.**
Écrire une fonction qui ouvre une transaction, fait un `UPDATE`, puis rend une erreur **sans**
`Rollback` (pas de `defer`). L'appeler, puis, depuis `psql`, essayer de modifier la même ligne. Que
voit-on dans `pg_stat_activity` ? Combien de temps la ligne reste-t-elle bloquée ? Régler
`idle_in_transaction_session_timeout` à 5 s pour la session (`SET`) et observer.

**E4 — L'interblocage.**
Reproduire les 20 virements croisés **sans** `ORDER BY id`, avec un `pg_sleep(0.01)` entre le débit et
le crédit. Compter les `40P01`. Puis ajouter `ORDER BY id FOR UPDATE` : combien en reste-t-il ? Pourquoi
le `pg_sleep` rend-il le problème reproductible ?

**E5 — Ce que voit `REPEATABLE READ`.**
Deux transactions : T1 (`REPEATABLE READ`) lit `count(*)` d'une table ; T2 insère une ligne et valide ;
T1 relit `count(*)`. Même expérience en `READ COMMITTED`. Utiliser deux `*sql.Tx` dans le même
programme et noter les deux comptes de T1 dans chaque cas.

---

## Exercice intermédiaire — Réessayer correctement

1. Écrire `func WithRetryTx(ctx context.Context, db *sql.DB, opts *sql.TxOptions, fn func(*sql.Tx) error) error`
   qui rejoue **toute** la transaction sur `40001` et `40P01`, cinq tentatives au plus, avec une attente
   aléatoire entre 1 et 2ᵗ ms (t = numéro de tentative), en respectant `ctx`.
2. L'utiliser avec la version naïve du retrait (lecture puis écriture) en `SERIALIZABLE`, 100 retraits
   concurrents. Invariant à vérifier : **aucun retrait perdu** (solde final = 1 000 − nombre de retraits
   réussis). Mesurer, avec 5, 10 et 20 tentatives au plus : combien de retraits aboutissent, combien
   d'exécutions de `fn` au total ? Comparer avec la version `FOR UPDATE`. Qu'en conclure pour une ligne
   très disputée ?
3. Écrire un test qui vérifie que `fn` peut être appelée **plusieurs fois** : un compteur `atomic` dans
   `fn` dépasse 100. Que se passerait-il si `fn` envoyait un e-mail ?
4. L'erreur finale, après épuisement des tentatives, doit permettre `errors.As(err, &pgErr)` **et**
   indiquer le nombre de tentatives.

---

## Défi — Réserver une place

Un système de réservation : `events(id, capacity)` et `bookings(event_id, user_id, UNIQUE(event_id, user_id))`.
Règle : jamais plus de réservations que `capacity` ; un utilisateur ne réserve qu'une fois.

1. Écrire `Book(ctx, db, eventID, userID) error` en **trois** variantes : (a) `SELECT count(*)` puis
   `INSERT` en `READ COMMITTED` ; (b) la même chose avec `SELECT … FROM events … FOR UPDATE` d'abord ;
   (c) `SERIALIZABLE` + réessai.
2. Test de charge : un événement de capacité 50, 500 utilisateurs différents réservent en même temps.
   Combien de réservations pour chaque variante ? (La variante (a) dépasse-t-elle 50 ?)
3. Le doublon (même utilisateur, deux requêtes simultanées) : quelle erreur produit la contrainte
   `UNIQUE` ? La traduire en `ErrAlreadyBooked` (→ 409).
4. Une quatrième variante sans transaction explicite, en une seule instruction
   (`INSERT … SELECT … WHERE (SELECT count(*) …) < capacity`). Est-elle correcte en `READ COMMITTED` ?
   Le prouver par le test de charge, puis expliquer le résultat.

---

## Quiz

1. Pourquoi toutes les instructions d'une transaction doivent-elles passer par `tx` ?
2. Que fait `tx.Rollback()` après un `Commit` réussi ?
3. Que se passe-t-il si le contexte est annulé pendant une transaction ?
4. Une transaction en `READ COMMITTED` empêche-t-elle la mise à jour perdue ?
5. Trois façons d'éviter la mise à jour perdue ?
6. Que signifie le code `40001`, et que faut-il faire ?
7. Pourquoi rejouer toute la transaction et pas seulement l'instruction qui a échoué ?
8. Comment éviter les interblocages entre virements croisés ?
9. Que signifie `idle in transaction` dans `pg_stat_activity` ?
10. Pourquoi ne pas appeler un service HTTP dans une transaction ?
