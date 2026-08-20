# Leçon 10 — Exercices

Code dans `mes-solutions/niveau-02/lecon-10/`.

## Exercices faciles

**E1 — `main` + `run`.**
Prendre n'importe quel programme écrit jusqu'ici et le refactoriser selon le patron
`main` / `run() error`. Vérifier qu'il ne reste **qu'un seul** `os.Exit` (`grep -rn "os.Exit"`).
Puis ajouter un `defer fmt.Println("nettoyage")` dans `run` et vérifier qu'il s'exécute même en
cas d'erreur.

**E2 — Entrées/sorties en paramètres.**
Modifier `run` pour qu'elle prenne `args []string, stdout, stderr io.Writer`. Écrire dans
`main` un appel normal, puis un second appel dans le même programme qui capture la sortie dans
un `bytes.Buffer` et l'affiche transformée. **Que vient-on de rendre possible ?**

**E3 — Les quatre stades.**
Prendre un programme d'environ 300 lignes et l'organiser successivement selon les quatre
stades du cours. À chaque étape, noter ce qu'on gagne et ce qu'on perd. À quel stade
s'arrêterait-on raisonnablement pour ce programme ?

**E4 — `internal/` en pratique.**
Créer un projet avec `internal/core` et `pkg/api`. Depuis un second module local, tenter
d'importer les deux. Relever ce qui fonctionne et ce qui échoue. Puis répondre : `pkg/`
a-t-il apporté quoi que ce soit de vérifiable par le compilateur ?

**E5 — Couche contre domaine.**
Concevoir sur papier (ou en répertoires vides) la même application — utilisateurs, commandes,
produits — dans les deux découpages. Puis simuler un changement : « une commande doit
maintenant vérifier le crédit de l'utilisateur ». Combien de packages sont touchés dans chaque
version ? Un cycle apparaît-il ?

---

## Exercice intermédiaire — restructurer `taskman`

Reprendre le `taskman` de la leçon 8 et le faire évoluer.

**Nouvelles fonctionnalités demandées :**
- persistance dans un fichier JSON ;
- deux binaires : `taskman` (CLI interactif) et `taskman-export` (export en CSV) ;
- configuration par variables d'environnement (chemin du fichier, format de date, couleurs) ;
- un mode `--dry-run` qui n'écrit rien.

**Contraintes :**
1. Restructurer **avant** d'écrire les nouvelles fonctionnalités, et écrire dans le README pourquoi cette restructuration est nécessaire maintenant et ne l'était pas avant.
2. `cmd/` devient justifié : expliquer en une phrase pourquoi.
3. Le cœur métier ne doit dépendre **ni** de JSON, **ni** de l'environnement, **ni** de la CLI. Le vérifier : `go list -deps ./internal/task` ne doit contenir que la bibliothèque standard.
4. Les deux binaires partagent le cœur sans duplication.
5. `main` de chaque binaire : moins de 30 lignes, patron `run() error`.
6. `--dry-run` ne doit **pas** être implémenté par un `if` disséminé dans le code. *(Indice : quelle abstraction du niveau 2 permet de remplacer l'écriture réelle par une écriture factice ? La réponse tient en un mot.)*
7. Zéro variable globale. Zéro `init()`.
8. Dessiner le graphe de dépendances dans le README et vérifier qu'il est **acyclique** avec `go mod graph` ou `go list -deps`.

*La contrainte 6 est le cœur de l'exercice : c'est le premier vrai usage architectural d'une
interface. La contrainte 3 est le second — c'est la définition opérationnelle d'un « domaine
pur », et c'est le point de départ du niveau 10.*

---

## Défi

**a) Analyser un vrai projet.**
Cloner un projet Go réputé de taille moyenne — par exemple `github.com/charmbracelet/bubbletea`,
`github.com/spf13/cobra` ou `github.com/go-chi/chi`. Puis :
- dessiner son arborescence de premier niveau ;
- dire s'il utilise `cmd/`, `internal/`, `pkg/`, et pourquoi ;
- déterminer s'il découpe par domaine ou par couche ;
- trouver **une** décision de structure qu'on aurait prise autrement, et l'argumenter.

**b) Détecter la sur-structuration.**
Écrire un petit programme Go qui parcourt un projet et calcule, par package : le nombre de
fichiers, de lignes, d'identifiants exportés, et le nombre de packages qui l'importent.
Puis définir un **critère chiffré** de sur-structuration et le défendre.
*(Il n'y a pas de bonne réponse. L'exercice est de rendre explicite une intuition — c'est
exactement ce qu'on demande à un développeur senior.)*

**c) Le débat `pkg/`.**
Rédiger un texte d'une page défendant l'usage de `pkg/`, puis un autre le condamnant. Les deux
doivent être honnêtes et convaincants.
Conclure par sa propre position et le critère qui la détermine.
*(Chercher les prises de position de Russ Cox et Dave Cheney à ce sujet. Savoir défendre les
deux camps est le meilleur test de compréhension d'une décision d'architecture.)*

---

## Quiz

1. Go impose-t-il une structure de projet ? `golang-standards/project-layout` est-il officiel ?
2. À partir de quand `cmd/` est-il justifié ?
3. Quelle différence **vérifiable par le compilateur** entre `internal/` et `pkg/` ?
4. Citer trois problèmes du découpage par couche technique.
5. Pourquoi le patron `main` + `run() error` est-il préférable ?
6. Pourquoi `os.Exit` doit-il rester dans `main` ?
7. Pourquoi passer `stdout` et `stderr` en paramètres ?
8. Où placer les fichiers de test en Go ?
9. Quel signal indique qu'un découpage en packages est mauvais ?
10. Quelle est la bonne trajectoire de structuration d'un projet qui grossit ?
