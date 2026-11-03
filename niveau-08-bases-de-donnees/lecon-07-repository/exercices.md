# Leçon 7 — Exercices

Code dans `mes-solutions/niveau-08/lecon-07/`, en partant du module de l'exemple.

## Exercices faciles

**E1 — L'interface du mauvais côté.**
Déplacer `Repository` dans le paquet `postgres`, et faire dépendre `book.Service` de
`postgres.Repository`. Que doit maintenant importer `book` ? Que se passe-t-il si `postgres` importe
aussi `book` pour les types et les erreurs ? Que devient le test du service avec l'implémentation
mémoire ?

**E2 — Le contrat attrape un menteur.**
Introduire, une à une, trois « petites » différences dans `memory.Repo` : (a) `Get` rend
`errors.New("not found")` au lieu d'envelopper `book.ErrNotFound` ; (b) `ListByAuthor` ne trie pas ;
(c) `Create` accepte les ISBN en double. Pour chacune, quel sous-test du contrat échoue ? Lequel des
trois bugs aurait été invisible sans la suite de contrat, parce que les tests du service utilisent la
version mémoire ?

**E3 — Une transaction par test.**
Écrire une variante de `newTestDB` qui ouvre une transaction et rend `postgres.New(tx)`, avec
`tx.Rollback()` dans `t.Cleanup`. Le contrat passe-t-il ? Ajouter un sous-test qui, **après** l'échec
d'un doublon, crée un autre livre dans le même *repository*. Que se passe-t-il avec cette stratégie, et
pas avec le schéma par test ? (Code SQLSTATE à identifier.)

**E4 — Le nettoyage qui n'a pas lieu.**
Remettre `t.Context()` dans le `Cleanup` de `newTestDB`, lancer les tests trois fois, et compter les
schémas `test_%` restants. Ajouter la vérification d'erreur : quel message obtient-on ?

**E5 — Faire évoluer le contrat.**
Ajouter `Delete(ctx, id int64) error` à `book.Repository` (avec `ErrNotFound` si l'identifiant n'existe
pas). Écrire d'abord les sous-tests du contrat, les voir échouer (et même ne pas compiler) pour les deux
implémentations, puis implémenter. Combien de fichiers a-t-il fallu toucher ?

---

## Exercice intermédiaire — Une opération atomique sur deux *repositories*

On ajoute des avis : `reviews(book_id, rating, comment)`, avec une clé étrangère vers `books` et
`ON DELETE CASCADE`.

1. Un cas d'usage du domaine : `Service.AddWithFirstReview(ctx, book, rating, comment)` crée le livre
   **et** son premier avis, tout ou rien. Si la note est invalide (hors 1–5), **aucun** livre ne doit
   être créé.
2. Ajouter au domaine un `ReviewRepository`, et une interface `Store` avec
   `WithinTx(ctx, func(Books Repository, Reviews ReviewRepository) error) error` (ou une variante
   de ton choix).
3. Implémenter `Store` pour PostgreSQL (une transaction, deux *repositories* construits sur le même
   `*sql.Tx`) et pour la mémoire (un verrou, et une **annulation** des modifications en cas d'erreur).
4. Un sous-test de contrat : un avis invalide après un livre valide → `ListByAuthor` ne rend rien.
   Les deux implémentations doivent le passer.
5. Quelle est la difficulté de l'annulation en mémoire, et comment la résoudre simplement ?

---

## Défi — Une base par test, depuis un modèle

1. Dans un `TestMain` (N5 L1), créer **une fois** une base modèle `bookshelf_template` et y appliquer
   les migrations (leçon 6).
2. Chaque test crée sa base avec `CREATE DATABASE test_xxx TEMPLATE bookshelf_template`, s'y connecte,
   et la supprime à la fin (`DROP DATABASE … WITH (FORCE)`).
3. Mesurer, pour 50 tests parallèles, la durée totale avec cette stratégie et avec le schéma par test
   (en appliquant les migrations dans chaque schéma), pour 20, 60 et 120 tables. Laquelle est la plus
   rapide ? Formuler une hypothèse sur ce que coûte chaque stratégie, et sur les cas où la base modèle
   reprendrait l'avantage.
4. Que se passe-t-il si une connexion reste ouverte sur la base modèle pendant qu'un test la copie ?
   Comment s'en protéger ?
5. Le modèle ne doit être reconstruit que si les migrations ont changé : utiliser les empreintes de la
   leçon 6 pour le nommer (`bookshelf_template_<empreinte>`).

---

## Quiz

1. Quel est le rôle d'un *repository* ?
2. Pourquoi l'interface du stockage est-elle déclarée dans le paquet du domaine ?
3. Quels sont les signes d'une abstraction de stockage trop générale ?
4. À quoi sert l'interface `DBTX` ?
5. Qu'est-ce qu'une suite de contrat, et pourquoi est-elle indispensable quand il existe une
   implémentation en mémoire ?
6. Pourquoi ne pas tester un *repository* avec des *mocks* de `database/sql` ?
7. Quatre façons d'isoler des tests d'intégration, et la limite de chacune ?
8. Pourquoi un test d'intégration doit-il se sauter, et non échouer, sans `DATABASE_URL` ?
9. Que vaut `t.Context()` pendant l'exécution des fonctions de `t.Cleanup` ?
10. Pourquoi `clock_timestamp()` plutôt que `now()` pour `created_at` ?
