# Leçon 4 — Exercices

Code dans `mes-solutions/niveau-12/lecon-04/`, en partant du projet de l'exemple. Les exercices qui
demandent GitHub se font dans un dépôt **personnel de test**, jamais dans un dépôt partagé.

## Exercices faciles

**E1 — Rendre le script plus utile.**
`ci.sh` s'arrête au premier échec. Ajouter une option `-k` (*keep going*) qui exécute tous les contrôles,
affiche à la fin la liste de ceux qui ont échoué, et sort avec un code non nul s'il y en a. Pourquoi le
comportement par défaut doit-il rester l'arrêt au premier échec en CI ?

**E2 — Le test qui dépend de l'ordre.**
Écrire deux tests qui partagent une variable de paquet : le second ne passe que si le premier s'est exécuté
avant. Lancer `go test -count=1 ./...` plusieurs fois, puis `go test -shuffle=on -count=1 ./...` plusieurs
fois. Comment rejouer exactement un ordre qui a échoué ? Corriger les tests.

**E3 — La couverture comme garde-fou.**
Ajouter à `ci.sh` une étape qui lit `cover.out` (`go tool cover -func=cover.out`) et échoue si la couverture
totale descend sous un seuil. Est-ce une bonne idée de fixer 80 % pour tout le dépôt ? Quelle alternative ?

**E4 — L'injection.**
Dans un dépôt de test, écrire un workflow déclenché par `pull_request` qui affiche le titre de la *pull
request* de façon **sûre**. Le faire valider par `actionlint`. Puis écrire la version dangereuse, et
expliquer, sans l'exécuter, ce que ferait un titre comme `x"; echo PWNED; echo "`.

**E5 — Épingler et mettre à jour.**
Retrouver l'empreinte de commit d'une version d'action (`git ls-remote https://github.com/actions/checkout
refs/tags/v7.0.1`). Attention aux étiquettes annotées : laquelle des deux lignes (`…` ou `…^{}`) faut-il
prendre ? Comment être prévenu des nouvelles versions des actions épinglées (Dependabot, `.github/dependabot.yml`
avec l'écosystème `github-actions`) ?

---

## Exercice intermédiaire — La CI du service de prêts

Pour le service de prêts du niveau 10 (ou ton projet 4/7), dans un dépôt de test personnel :

1. Un `ci.sh` avec tous les contrôles du cours, et `go vet` pour chaque étiquette de build utilisée.
2. Un workflow `ci.yml`. Le job `integration` démarre PostgreSQL en service et lance la suite de contrat du
   dépôt (N8 L7) avec `-tags integration`. Il échoue, et ne saute pas les tests, si la base n'est pas là.
3. Pousser une branche avec une course de données volontaire, ouvrir une *pull request* : la CI doit échouer
   sur `-race`. Puis la corriger.
4. Protéger `main` dans les réglages du dépôt : *pull request* obligatoire, jobs `checks` et `integration`
   requis.
5. Mesurer la durée de la CI, avec puis sans le cache de `setup-go` (`cache: false`). Quelle part revient à
   chaque job ?

---

## Défi — La publication de bout en bout

1. Dans le dépôt de test, ajouter `release.yml`. Pousser l'étiquette `v0.1.0-rc.1`, puis `v0.1.0`. Vérifier :
   - la préversion est marquée comme telle ;
   - la publication contient les binaires et `SHA256SUMS` ;
   - l'image `ghcr.io/…:0.1.0` existe pour amd64 et arm64 (`docker buildx imagetools inspect`).
2. Télécharger le binaire Linux publié et vérifier sa somme. Le reconstruire localement avec `build.sh
   v0.1.0` dans l'image `golang` de la version du `go.mod` (leçon 2, défi) : la somme est-elle identique ?
   Sinon, trouver la différence avec `go version -m`. Indice : le dépôt cloné en CI et le tien sont-ils dans
   le même état Git ?
3. Ajouter une **attestation de provenance** à l'image (entrée `provenance` de `docker/build-push-action`) et
   la lire. Que contient-elle ?
4. Rédiger la liste des droits dont chaque job a besoin, et vérifier que le workflow n'en donne pas plus.

---

## Quiz

1. Quels contrôles une CI Go doit-elle lancer, et dans quel ordre ?
2. Pourquoi lancer les tests avec `-race` en CI, même s'ils passent sans ?
3. Que fait `-shuffle=on` ?
4. Pourquoi mettre les contrôles dans un script du dépôt plutôt que dans le workflow ?
5. Comment la CI choisit-elle la version de Go ?
6. Comment lancer des tests d'intégration avec une base de données en CI ?
7. Pourquoi épingler une action par empreinte de commit ?
8. Que risque-t-on avec `run: echo "${{ github.event.pull_request.title }}"` ? Comment faire ?
9. Pourquoi déclarer `permissions` ?
10. Que doit faire un workflow de publication avant de publier ?
