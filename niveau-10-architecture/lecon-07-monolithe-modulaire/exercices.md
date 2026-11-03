# Leçon 7 — Exercices

Code dans `mes-solutions/niveau-10/lecon-07/`, en partant du module de l'exemple.

## Exercices faciles

**E1 — La frontière `internal`.**
Depuis `lending`, importer `exemple/catalog/internal/store`. Relever le message. Puis déplacer
`internal/` d'un niveau (`exemple/internal/store`) : qui peut maintenant l'importer ? Qu'en conclure sur
l'emplacement des `internal/` dans un monolithe modulaire ?

**E2 — Le test d'architecture.**
Écrire un test qui échoue si un paquet de `lending/…` importe `catalog` **autrement** que par l'API
publique `exemple/catalog` (par exemple `exemple/catalog/httpapi`), et si `catalog/…` importe quoi que ce
soit de `lending`. Le faire échouer exprès, puis le remettre au vert.

**E3 — Le prix du réseau, en vrai.**
Reprendre le benchmark avec un délai réseau simulé de 1 ms dans le handler HTTP (`time.Sleep`). Le reçu
fait un appel au catalogue : combien de reçus par seconde peut produire **une** goroutine ? Et si chaque
reçu demandait les titres de 20 livres, un par un ?

**E4 — Les pannes.**
Avec le client HTTP de l'exemple, que se passe-t-il si le catalogue répond en 30 secondes ? Ajouter un
délai par appel (`context.WithTimeout`) côté `lending`, puis un délai sur le `http.Client`. Lequel des
deux faut-il, et pourquoi pas seulement l'autre ?

**E5 — Les tables partagées.**
On te propose : « pour le reçu, une simple jointure `loans JOIN works` suffit, pas besoin d'appeler le
catalogue ». Lister trois conséquences pour l'évolution des deux modules. Proposer deux alternatives.

---

## Exercice intermédiaire — Un appel par lot

Le module de prêt affiche la liste des emprunts d'un adhérent avec leurs titres (jusqu'à 20).

1. Ajouter à l'API du catalogue `Titles(ctx, isbns []string) (map[string]string, error)` et son
   équivalent HTTP (`GET /works?isbn=…&isbn=…`).
2. Le module de prêt déclare une interface qui contient seulement ce dont il a besoin.
3. Mesurer, en mémoire puis à travers HTTP (avec le délai de 1 ms de E3), 20 appels `Title` contre un appel
   `Titles`. Qu'en conclure sur la conception d'une API **destinée** à passer par le réseau ?
4. Quel est le comportement attendu si 3 des 20 ISBN sont inconnus : erreur globale, ou résultat partiel ?
   Décider, documenter, tester sur les deux assemblages.

---

## Défi — Extraire le catalogue

1. Écrire `cmd/catalog` (le catalogue seul, servi en HTTP) et `cmd/biblio` (le monolithe sans le catalogue,
   qui l'appelle par `catalogclient`). Une variable de configuration choisit l'assemblage du monolithe :
   `CATALOG_URL` vide → module en mémoire ; renseignée → client HTTP.
2. Le client ajoute : délai par appel, deux réessais avec attente aléatoire sur les erreurs réseau et les
   503 (leçon 9 en avant-première), et un cache de 5 minutes des titres.
3. Faire tourner les deux processus, puis arrêter le catalogue : que voit l'utilisateur du monolithe ?
   Avec le cache ? Proposer un comportement dégradé acceptable (afficher l'ISBN au lieu du titre).
4. Rédiger en dix lignes ce que cette extraction a coûté (code, exploitation, nouveaux modes de panne), et
   ce qu'elle a apporté. Était-elle justifiée pour ce module ?

---

## Quiz

1. Qu'est-ce qu'un monolithe modulaire ?
2. Que garantit le répertoire `internal/`, et que ne garantit-il pas ?
3. Pourquoi un module ne doit-il pas lire les tables d'un autre ?
4. Comment vérifier les règles de dépendance que `internal/` ne couvre pas ?
5. Quels coûts ajoute une frontière réseau, au-delà de la latence ?
6. Citer trois raisons valables d'extraire un service, et deux mauvaises.
7. Comment un monolithe modulaire prépare-t-il une extraction ?
8. Qu'est-ce qu'un « monolithe distribué » ?
9. Pourquoi une API destinée au réseau doit-elle proposer des opérations par lot ?
10. Que change l'extraction d'un service pour les transactions ?
