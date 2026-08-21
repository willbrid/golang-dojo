# 🥋 Golang Dojo

Formation Go de zéro à expert — cours, exercices et corrections.

Le dépôt est **autonome et réutilisable** : il ne contient que du matériel de formation.
Aucune donnée personnelle, aucun suivi de progression, aucun résultat d'évaluation n'y est
publié — quiconque clone ce dépôt peut suivre le parcours du début à la fin.

**Version de Go ciblée : 1.27** (dernière stable, sortie le 19 août 2026).
Vérifiée auprès des sources officielles : [go.dev/VERSION](https://go.dev/VERSION?m=text) et
[l'historique des versions](https://go.dev/doc/devel/release).

---

## Comment ce dépôt est organisé

```
.
├── SYLLABUS.md                  Le plan détaillé des 13 niveaux (~124 leçons)
├── ressources/                  Aide-mémoire, liens officiels, glossaire
├── niveau-01-fondamentaux/
│   ├── README.md                Sommaire et objectifs du niveau
│   └── lecon-01-environnement/
│       ├── cours.md             Le cours : explications, exemples, pièges
│       └── exercices.md         Les exercices et le quiz — sans les réponses
├── niveau-02-types-abstraction/ …
├── …
├── projets/                     Les 9 projets, spécifications et contraintes
└── mes-solutions/               Le code d'entraînement du lecteur — ignoré par git
```

### Où sont les corrections ?

**Sur une branche séparée, volontairement.** Elles ne sont pas sur `main` pour qu'on ne
tombe pas dessus par accident en lisant un exercice.

```bash
git fetch origin
git checkout corrections     # pour consulter
git checkout main            # pour revenir au cours
```

Sur cette branche, chaque leçon a un fichier `correction.md` et un répertoire `solution/`
avec le code Go commenté. **La règle reste : ne les ouvrir qu'après avoir vraiment
essayé**, ou après être resté bloqué un moment.

### Où écrire son code

Tout dans `mes-solutions/`, en miroir de l'arborescence du cours :

```
mes-solutions/
└── niveau-01/
    └── lecon-01/
        ├── e02-runtime/
        ├── e03-args/
        └── tempconv/
```

Ce répertoire est dans `.gitignore` : le code d'entraînement reste local, il n'est jamais
poussé. Il en va de même pour tout journal de progression ou résultat d'évaluation, qui
n'ont pas leur place dans un dépôt destiné à être réutilisé.

---

## Démarrage

```bash
# 1. Vérifier la version de Go (attendu : go1.27.0)
go version

# 2. Lire le plan
less SYLLABUS.md

# 3. Commencer
less niveau-01-fondamentaux/lecon-01-environnement/cours.md
```

### Installer / mettre à jour Go 1.27 sur Linux

```bash
sudo rm -rf /usr/local/go
curl -LO https://go.dev/dl/go1.27.0.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.27.0.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin:$(go env GOPATH)/bin   # à ajouter dans ~/.bashrc
go version
```

---

## Le parcours en un coup d'œil

| Niveau | Thème | Leçons | Projet à la sortie |
|---|---|---|---|
| 1 | [Fondamentaux du langage](niveau-01-fondamentaux/) | 11 | — |
| 2 | [Types, méthodes et abstraction](niveau-02-types-abstraction/) | 10 | Projet 1 — CLI simple |
| 3 | [Go moderne et génériques](niveau-03-go-moderne-generiques/) | 6 | — |
| 4 | [Bibliothèque standard essentielle](niveau-04-bibliotheque-standard/) | 8 | Projet 2 — Gestionnaire de tâches |
| 5 | [Tests et qualité](niveau-05-tests-qualite/) | 8 | — |
| 6 | [Concurrence et coordination](niveau-06-concurrence/) | 12 | Projet 5 — Serveur concurrent |
| 7 | [Réseau, HTTP et services](niveau-07-reseau-http/) | 12 | Projet 3 — API REST |
| 8 | [Bases de données](niveau-08-bases-de-donnees/) | 8 | Projets 4 et 6 — PostgreSQL, auth |
| 9 | [Réflexion et métaprogrammation](niveau-09-reflexion/) | 4 | — |
| 10 | [Architecture](niveau-10-architecture/) | 10 | Projet 7 — Workers et queues |
| 11 | [Performance](niveau-11-performance/) | 8 | — |
| 12 | [Outils et production](niveau-12-outils-production/) | 9 | Projet 8 — Microservice |
| 13 | [Expert](niveau-13-expert/) | 8 | Projet final |

Détail complet et ordre des leçons dans
[SYLLABUS.md](SYLLABUS.md).

### L'ordre est une contrainte, pas une suggestion

Le parcours respecte une règle stricte : **aucune leçon n'utilise un concept qui n'a pas
encore été vu**. C'est ce qui explique quelques choix inhabituels — pointeurs avant structs,
méthodes avant interfaces, erreurs en deux temps (usage au niveau 1, mécanique au niveau 2,
parce qu'`error` *est* une interface). Le tableau des décisions d'ordonnancement et leurs
raisons est en tête du [SYLLABUS](SYLLABUS.md#principe-dordonnancement).

## Les quatre niveaux de qualité

Chaque exercice est évalué selon cette échelle, utilisée d'un bout à l'autre de la formation :

| Niveau | Question à laquelle il répond |
|---|---|
| **Ça fonctionne** | Le programme donne le bon résultat, aujourd'hui, sur les données d'essai. |
| **C'est correct** | C'est vrai pour *toutes* les entrées, y compris les cas limites. |
| **C'est idiomatique** | Un développeur Go expérimenté l'aurait écrit ainsi et le comprend en 3 secondes. |
| **C'est prêt pour la production** | Ça survit aux erreurs, aux timeouts, à la concurrence ; ça se surveille et se maintient. |

## Règles de la formation

1. Aucune solution avant une tentative réelle.
2. Face à un code faux : *quoi*, *pourquoi*, un **indice** — la correction se fait soi-même.
3. Un quiz clôt chaque leçon ; une évaluation cumulative clôt chaque niveau.
4. L'ordre des leçons se respecte : chacune suppose acquis tout ce qui précède.

## Sources officielles

- [Documentation Go](https://go.dev/doc/) · [Spécification](https://go.dev/ref/spec) · [Blog](https://go.dev/blog/)
- [pkg.go.dev](https://pkg.go.dev/) — documentation de tous les paquets
- [Effective Go](https://go.dev/doc/effective_go) · [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)
- [Notes de version Go 1.27](https://go.dev/doc/go1.27)
