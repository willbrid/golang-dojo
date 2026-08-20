# Leçon 9 — Exercices

Code dans `mes-solutions/niveau-02/lecon-09/`.

## Exercices faciles

**E1 — Anatomie.**
Créer un module, ajouter deux dépendances, lancer `go mod tidy`. Puis commenter **chaque
ligne** de `go.mod` et **chaque ligne** de `go.sum` en expliquant ce qu'elle signifie et
pourquoi il y a deux lignes par module.

**E2 — `go mod why`.**
Ajouter une dépendance qui en tire elle-même plusieurs autres (par exemple un client HTTP ou
un logger structuré). Lancer `go list -m all`, compter les modules, puis `go mod why` sur trois
dépendances indirectes. Laquelle a amené le plus de code ?

**E3 — Retirer une dépendance.**
Supprimer l'import d'une dépendance dans le code **sans** lancer `go mod tidy`. Le projet
compile-t-il ? `go.mod` a-t-il changé ? Puis lancer `go mod tidy` et observer le diff.
Conclure sur le rôle exact de cette commande.

**E4 — Version du langage.**
Créer un module en `go 1.21`, y écrire une boucle capturant sa variable dans une closure,
observer le résultat. Passer la ligne à `go 1.27`, relancer **sans rien changer d'autre**.
Expliquer.

**E5 — `go install` vs dépendance.**
Installer `goimports` avec `go install …@latest`. Vérifier que `go.mod` n'a **pas** changé.
Où le binaire a-t-il atterri ? Puis déclarer un outil avec la directive `tool` de `go.mod` et
l'appeler avec `go tool`. Comparer les deux approches : laquelle pour quel besoin ?

---

## Exercice intermédiaire — publier un module

Publier réellement un petit module réutilisable sur GitHub.

**Contenu :** une bibliothèque `slugify` (reprendre l'exercice du niveau 1, leçon 8) avec une
API publique minimale : `Slugify(s string) string` et `SlugifyWithSep(s, sep string) string`.

**Contraintes :**
1. Dépôt public dédié, chemin de module = URL du dépôt.
2. **Zéro dépendance externe.** Justifier ce choix dans le README.
3. Un `README.md` avec exemple d'usage, un `LICENSE`, un commentaire `// Package slugify …`.
4. Documentation complète sur chaque identifiant exporté. Vérifier le rendu avec `go doc -all .`.
5. Publier en **`v0.1.0`** d'abord. Expliquer dans le README pourquoi pas `v1.0.0`.
6. Depuis un **autre** module, faire `go get` de la version publiée et l'utiliser. Vérifier le contenu de `go.sum`.
7. Publier ensuite une `v0.2.0` avec un changement **cassant** (renommer une fonction). Vérifier que le module consommateur continue de compiler tant qu'il ne met pas à jour. Expliquer pourquoi — c'est MVS en action.
8. Publier enfin une `v1.0.0`, puis tenter un changement cassant : que faudrait-il faire, exactement ?

*Cet exercice est le seul de la formation qui produit quelque chose de **public et
permanent**. Une version publiée ne s'efface pas : c'est aussi la leçon.*

---

## Défi

**a) MVS à la main.**
Étant donné ce graphe, déterminer la version retenue pour chaque module **sans lancer Go**,
puis vérifier :
```
A (mon module) → B v1.2.0, C v1.5.0
B v1.2.0       → D v1.1.0
C v1.5.0       → D v1.3.0, E v2.0.0
E v2.0.0       → D v1.2.0
```
Puis répondre : que se passe-t-il si l'on ajoute `D v1.0.0` en dépendance directe de A ?
Et si `E` passait en `v3.0.0` ?
Enfin, comparer avec ce que ferait npm sur le même graphe, et dire quel modèle produit le
moins de surprises.

**b) Migration en v2.**
Prendre le module publié à l'exercice intermédiaire et le faire passer en `v2` avec une
rupture d'API. Effectuer la migration complète : ligne `module`, étiquette, chemins d'import.
Puis écrire un programme qui utilise **v1 et v2 simultanément**. Expliquer pourquoi c'est
possible en Go et ce que cela change pour une migration progressive dans une grande base de
code.
Comparer les deux stratégies possibles pour v2 : sous-répertoire `/v2` ou branche dédiée.

**c) Chaîne d'approvisionnement.**
Lancer `govulncheck ./...` sur un projet ayant quelques dépendances. Puis répondre par écrit :
comment Go se protège-t-il d'une substitution de code en amont ? Que se passerait-il si un
auteur forçait la réécriture d'une étiquette git déjà publiée ?
Quel rôle jouent `GONOSUMDB`, `GOPRIVATE` et `GONOSUMCHECK` — et lequel de ces trois n'existe
pas ? *(Vérifier dans `go help environment` plutôt que de deviner.)*

---

## Quiz

1. Quelle est la différence entre un module, un package et un dépôt ?
2. Que signifie exactement la ligne `go 1.27` dans `go.mod` ?
3. À quoi sert `go.sum` ? Faut-il le committer ?
4. Pourquoi y a-t-il deux lignes par module dans `go.sum` ?
5. Que veut dire `// indirect` ?
6. Qu'impose Go à partir de la version majeure 2 ?
7. Deux versions majeures d'un même module peuvent-elles coexister dans un binaire ?
8. Qu'est-ce que MVS, et en quoi diffère-t-il de la résolution de npm ?
9. Pourquoi Go n'a-t-il pas de fichier de verrouillage ?
10. Que fait `go mod tidy` ? Quand le lancer ?
11. Pourquoi ne faut-il jamais committer un `replace` local ?
12. Une version publiée peut-elle être supprimée ?
