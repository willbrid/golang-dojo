# Leçon 8 — Exercices

Code dans `mes-solutions/niveau-12/lecon-08/`. Les exemples de secrets sont **factices** : on n'utilise
jamais un vrai secret pour un exercice.

## Exercices faciles

**E1 — `govulncheck` dans un script.**
Lancer `govulncheck -format json ./...` sur le programme vulnérable du cours. Écrire un petit programme Go qui
lit ce flux JSON et affiche, pour chaque vulnérabilité **atteignable**, son identifiant, le module, la version
corrigée et la première trace d'appel. Comment distinguer, dans le JSON, une vulnérabilité atteignable d'une
vulnérabilité seulement importée ?

**E2 — La vulnérabilité transitive.**
Créer un module `a` qui dépend de `golang.org/x/text@v0.3.5` et appelle `language.Parse`, puis un module
`svc` qui dépend de `a` (avec `replace` vers un répertoire local). `govulncheck` sur `svc` la trouve-t-il ?
Que disent `go mod why -m golang.org/x/text` et `go mod graph` ? Comment corriger depuis `svc` sans
attendre une nouvelle version de `a` ?

**E3 — La rotation.**
Étendre `Verifier` pour accepter **plusieurs** jetons valides (l'ancien et le nouveau pendant une rotation).
Le temps de `Check` doit rester indépendant du jeton qui correspond. Écrire le test d'une rotation complète :
seul A est valide, puis A et B, puis seul B.

**E4 — Pourquoi pas `math/rand` ?**
Écrire une fonction qui génère un « jeton » de 16 caractères avec `math/rand/v2`, à partir d'un générateur
initialisé par l'heure en secondes (`rand.NewPCG(uint64(time.Now().Unix()), 0)`). Connaissant l'heure
approximative de création (à une minute près) et un jeton produit, retrouver la graine, puis le jeton
**suivant**. Combien d'essais faut-il ?

**E5 — Les modules privés.**
Avec un chemin de module privé fictif (`git.exemple.interne/equipe/lib`), lancer `go get` d'abord sans
réglage, puis avec `GOPRIVATE=git.exemple.interne`. Quels messages d'erreur ? Qu'est-ce qui a été envoyé au
proxy public dans le premier cas (`GOFLAGS=-x`, ou les messages de `go`) ? Pourquoi est-ce une fuite
d'information ?

---

## Exercice intermédiaire — La revue de sécurité du service de prêts

Pour le service de prêts du niveau 10 (ou ton projet 4/6/7) :

1. Remplir la liste de durcissement du cours pour ce service, avec pour chaque point « fait », « à faire » ou
   « sans objet », et la preuve (le fichier et la ligne, ou la commande).
2. Écrire des **tests de sécurité** qui échouent si un point régresse :
   - l'adresse publique répond 404 à `/debug/pprof/` et à `/metrics` ;
   - le serveur a un `ReadHeaderTimeout` et un `MaxHeaderBytes` non nuls ;
   - les réponses ont les en-têtes de sécurité de N7 L9 ;
   - un corps trop grand est refusé ;
   - la configuration refuse un secret passé par option.
3. Lancer `govulncheck` et corriger ce qui est atteignable.
4. Lister les dépendances directes (`go list -m -f '{{if not .Indirect}}{{.Path}}{{end}}' all`). Pour
   chacune, dire à quoi elle sert et si la bibliothèque standard suffirait.

---

## Défi — Signer une publication

1. Signer le fichier `SHA256SUMS` de la leçon 2 avec une clé SSH : `ssh-keygen -Y sign -f clé -n fichier
   SHA256SUMS`. Écrire le fichier `allowed_signers` et vérifier avec `ssh-keygen -Y verify`.
2. Modifier un octet d'un binaire, puis de `SHA256SUMS` : quelle vérification échoue dans chaque cas ?
3. Écrire en Go un vérificateur qui, pour une publication téléchargée, vérifie la signature de `SHA256SUMS`
   (par `os/exec` vers `ssh-keygen`), puis la somme de chaque binaire (`crypto/sha256`). Il sort avec un code
   non nul au moindre écart.
4. Où doit vivre la clé privée ? Que faire si elle fuit ? Comparer avec la signature « sans clé » (Sigstore,
   `cosign`) dans la CI de la leçon 4 : qu'est-ce qui remplace la clé, et que vérifie l'utilisateur ?

---

## Quiz

1. Qu'apporte l'analyse d'atteignabilité de `govulncheck` ?
2. Que signifie le code de sortie 3 de `govulncheck` ?
3. À quoi sert le mode binaire ?
4. Que garantissent `go.sum` et `sum.golang.org` ? Que ne garantissent-ils pas ?
5. À quoi sert `GOPRIVATE` ?
6. Quelles sont les étapes du cycle de vie d'un secret ?
7. Comment changer un secret sans interrompre le service ?
8. Un mot de passe a été validé dans Git, puis supprimé au commit suivant. Que faire ?
9. Pourquoi comparer un jeton en temps constant, et pourquoi sur des empreintes ?
10. Qu'apporte une signature par rapport à un fichier de sommes de contrôle ?
