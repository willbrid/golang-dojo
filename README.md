# 🥋 Golang Dojo

Formation Go de zéro à expert — cours, exercices et corrections.

**Version de Go ciblée : 1.27** (dernière stable, sortie le 19 août 2026).
Vérifiée auprès des sources officielles : [go.dev/VERSION](https://go.dev/VERSION?m=text) et
[l'historique des versions](https://go.dev/doc/devel/release).

---

## Comment ce dépôt est organisé

```
.
├── SYLLABUS.md                  Le plan détaillé des 10 niveaux (~96 leçons)
├── PROGRESSION.md               Mon journal : acquis, lacunes, erreurs récurrentes
├── ressources/                  Aide-mémoire, liens officiels, glossaire
├── niveau-01-fondamentaux/
│   ├── README.md                Sommaire et objectifs du niveau
│   └── lecon-01-environnement/
│       ├── cours.md             Le cours : explications, exemples, pièges
│       └── exercices.md         Les exercices et le quiz — sans les réponses
├── niveau-02-go-idiomatique/    …
├── …
├── projets/                     Les 9 projets, spécifications et contraintes
└── mes-solutions/               MON code d'apprentissage — ignoré par git, jamais poussé
```

### Où sont les corrections ?

**Sur une branche séparée, volontairement.** Elles ne sont pas sur `main` pour que je ne
tombe pas dessus par accident en lisant un exercice.

```bash
git fetch origin
git checkout corrections     # pour consulter
git checkout main            # pour revenir au cours
```

Sur cette branche, chaque leçon a un fichier `correction.md` et un répertoire `solution/`
avec le code Go commenté. **La règle reste : je ne les ouvre qu'après avoir vraiment
essayé**, ou quand je bloque depuis un moment.

### Où j'écris mon code

Tout dans `mes-solutions/`, en miroir de l'arborescence du cours :

```
mes-solutions/
└── niveau-01/
    └── lecon-01/
        ├── e02-runtime/
        ├── e03-args/
        └── tempconv/
```

Ce répertoire est dans `.gitignore` : mon code d'entraînement reste sur ma machine.

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

| Niveau | Thème | Leçons | Projet associé |
|---|---|---|---|
| 1 | [Fondamentaux](niveau-01-fondamentaux/) | 11 | Projet 1 — CLI simple |
| 2 | [Go idiomatique](niveau-02-go-idiomatique/) | 11 | Projet 2 — Gestionnaire de tâches |
| 3 | [Tests et qualité](niveau-03-tests-qualite/) | 8 | — |
| 4 | [Concurrence](niveau-04-concurrence/) | 11 | Projet 5 — Serveur concurrent |
| 5 | [Réseau et backend](niveau-05-reseau-backend/) | 12 | Projets 3 et 6 — API REST, auth |
| 6 | [Bases de données](niveau-06-bases-de-donnees/) | 8 | Projet 4 — API + PostgreSQL |
| 7 | [Architecture](niveau-07-architecture/) | 10 | Projet 7 — Workers et queues |
| 8 | [Performance](niveau-08-performance/) | 8 | — |
| 9 | [Outils et production](niveau-09-outils-production/) | 9 | Projet 8 — Microservice |
| 10 | [Expert](niveau-10-expert/) | 8 | Projet final |

Détail complet dans [SYLLABUS.md](SYLLABUS.md).

---

## Les quatre niveaux de qualité

Chaque exercice est évalué selon cette échelle, utilisée d'un bout à l'autre de la formation :

| Niveau | Question à laquelle il répond |
|---|---|
| **Ça fonctionne** | Le programme donne le bon résultat, aujourd'hui, sur mes données. |
| **C'est correct** | C'est vrai pour *toutes* les entrées, y compris les cas limites. |
| **C'est idiomatique** | Un développeur Go expérimenté l'aurait écrit ainsi et le comprend en 3 secondes. |
| **C'est prêt pour la production** | Ça survit aux erreurs, aux timeouts, à la concurrence ; ça se surveille et se maintient. |

## Règles de la formation

1. Aucune solution avant une tentative réelle.
2. Face à un code faux : *quoi*, *pourquoi*, un **indice** — puis je corrige moi-même.
3. Un quiz clôt chaque leçon ; une évaluation cumulative clôt chaque niveau.
4. La difficulté s'adapte à mes résultats, consignés dans [PROGRESSION.md](PROGRESSION.md).

## Sources officielles

- [Documentation Go](https://go.dev/doc/) · [Spécification](https://go.dev/ref/spec) · [Blog](https://go.dev/blog/)
- [pkg.go.dev](https://pkg.go.dev/) — documentation de tous les paquets
- [Effective Go](https://go.dev/doc/effective_go) · [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)
- [Notes de version Go 1.27](https://go.dev/doc/go1.27)
