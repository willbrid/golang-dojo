# Leçon 6 — Exercices

Code dans `mes-solutions/niveau-10/lecon-06/`, en partant du module de l'exemple.

## Exercices faciles

**E1 — Contourner.**
Depuis le paquet `lending_test`, essayer : `ada.loans = nil`, `lending.CopyID{"x"}`, et
`lending.CopyID{}`. Lesquels compilent ? La valeur zéro `CopyID{}` est-elle une faille ? Que se passe-t-il
si on l'emprunte ? Proposer une parade dans `Borrow`.

**E2 — Le langage.**
Un bibliothécaire te dit : « Un adhérent qui a rendu un livre en retard il y a moins d'un mois ne peut pas
réserver de nouveauté. » Repérer les mots du métier, et écrire la signature de la méthode et le nom de
l'erreur. À quel agrégat la règle appartient-elle ? Quelles informations lui manquent aujourd'hui ?

**E3 — Payer.**
Ajouter `Member.PayFines(amount Money) (rest Money, err error)` : on ne peut pas payer un montant négatif
ni plus que ce qui est dû ; un paiement enregistre l'événement `PénalitéPayée`. Tests depuis
`lending_test`.

**E4 — L'égalité.**
Deux `Money` de même montant sont-ils égaux avec `==` ? Deux `*Member` de même identifiant ? Pourquoi
cette différence est-elle exactement celle entre valeur et entité ?

**E5 — La taille de l'agrégat.**
Pour chacune de ces règles, dire si elle tient dans l'agrégat `Member`, demande un autre agrégat, ou relie
deux agrégats (et alors : transaction commune ou cohérence à terme ?) : (a) 3 emprunts au plus ; (b) un
exemplaire n'a qu'un emprunteur ; (c) un adhérent est notifié 3 jours avant l'échéance ; (d) la
bibliothèque ne prête pas plus de 1 000 exemplaires en même temps ; (e) un exemplaire réservé n'est prêté
qu'à la personne qui l'a réservé.

---

## Exercice intermédiaire — Persister un agrégat

L'agrégat a des champs non exportés : comment un *repository* d'un **autre** paquet peut-il l'enregistrer
et le reconstruire ?

1. Comparer trois solutions : (a) le *repository* dans le paquet `lending` ; (b) des méthodes
   `Snapshot() MemberState` / `RestoreMember(MemberState) (*Member, error)` avec un type d'état exporté ;
   (c) rendre les champs exportés. Pour chacune : qu'est-ce qui empêche encore un contournement des
   invariants ?
2. Implémenter (b) : `RestoreMember` **revérifie** les invariants (un état stocké invalide est refusé avec
   une erreur claire).
3. Un *repository* en mémoire qui stocke des `MemberState` (et non des `*Member`, pourquoi ?), avec un
   numéro de **version** pour détecter une modification concurrente (verrou optimiste : l'enregistrement
   échoue si la version a changé depuis le chargement).
4. Un service `Borrow(ctx, memberID, copy)` : charger, appeler l'agrégat, enregistrer avec la version,
   récupérer les événements. Test : deux emprunts concurrents du même adhérent qui partent de la même
   version → un seul réussit, l'autre reçoit `ErrConcurrentModification` (et peut réessayer).

---

## Défi — Deux contextes bornés

Ajouter un contexte `catalog` (œuvres : ISBN, titre, auteurs) à côté de `lending` (exemplaires, emprunts).

1. Aucun des deux paquets n'importe l'autre. Le prêt connaît l'œuvre d'un exemplaire par son ISBN
   seulement.
2. Pour afficher « Ada a emprunté *La Peste* », qui assemble l'information ? (Un paquet de lecture qui
   importe les deux, côté présentation.)
3. Le catalogue retire une œuvre. Que doit-il se passer pour les exemplaires en prêt ? Proposer un
   événement `ŒuvreRetirée` publié par le catalogue et une réaction du contexte prêt, en cohérence à terme.
4. Écrire un test d'architecture (leçon 7 en avant-première, avec `go list`) qui échoue si `lending`
   importe `catalog` ou l'inverse.

---

## Quiz

1. Qu'est-ce que le langage ubiquitaire, et pourquoi compte-t-il dans le code ?
2. Différence entre objet valeur et entité ?
3. Comment rendre un objet valeur valide par construction en Go ?
4. Qu'est-ce qu'un agrégat, et qu'est-ce que sa racine ?
5. Pourquoi un agrégat est-il l'unité de transaction ?
6. Comment Go protège-t-il les invariants d'un agrégat ?
7. Pourquoi référencer un autre agrégat par identifiant ?
8. Qu'est-ce qu'un événement de domaine, et qui le publie ?
9. Qu'est-ce qu'un contexte borné ? Donner un exemple dans la bibliothèque.
10. Quand le DDD n'est-il pas utile ?
