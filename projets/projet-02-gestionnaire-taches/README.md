# Projet 2 — `todo`, un gestionnaire de tâches en ligne de commande

**À réaliser après le niveau 4.** Durée indicative : 8 à 15 heures.
Écrire le code dans `mes-solutions/projets/projet-02/`.

---

## Objectif pédagogique

Mobiliser le niveau 4 sur un outil qu'on utilise vraiment : **dates** et fuseaux, **JSON**,
**fichiers** écrits de façon sûre, erreurs enveloppées, et une interface en ligne de commande à
sous-commandes. Le projet introduit aussi la question qui reviendra dans tous les suivants :
**séparer le domaine (les tâches) du stockage (le fichier) et de l'interface (la CLI)**.

---

## Spécifications

```
$ todo add "Rédiger le rapport" --due 2026-11-05 --priority high --tag travail
ajoutée : #7 Rédiger le rapport (échéance mer. 5 nov., priorité haute)

$ todo list
 #  P  ÉCHÉANCE        TÂCHE                          ÉTIQUETTES
 3  ●  aujourd'hui     Appeler le plombier            maison
 7  ●  dans 4 jours    Rédiger le rapport             travail
 5  ·  —               Lire « Effective Go »          go

$ todo done 3
terminée : #3 Appeler le plombier

$ todo list --all --tag travail --json
[{"id":7,"title":"Rédiger le rapport", …}]
```

### Comportements exigés

| # | Exigence |
|---|---|
| S1 | Sous-commandes : `add`, `list`, `done`, `undo`, `edit`, `rm`, `export`, `import`. |
| S2 | Chaque tâche a un identifiant **stable** : supprimer #3 ne renumérote pas #4. |
| S3 | Une tâche a un titre (obligatoire, non vide après `TrimSpace`), une échéance facultative, une priorité (`low`, `normal`, `high` ; défaut `normal`), des étiquettes, une date de création et une date d'achèvement. |
| S4 | `--due` accepte `AAAA-MM-JJ`, `AAAA-MM-JJ HH:MM`, `today`, `tomorrow`, et `+3d`. Toute autre forme est une erreur qui cite la valeur. |
| S5 | `list` trie par : en retard d'abord, puis échéance la plus proche, puis priorité, puis identifiant. Les tâches sans échéance viennent après celles qui en ont. L'ordre est **déterministe**. |
| S6 | `list` affiche les échéances relatives (« aujourd'hui », « demain », « dans 4 jours », « en retard de 2 jours ») **dans le fuseau local de l'utilisateur**. |
| S7 | `list` masque les tâches terminées, sauf avec `--all`. Filtres `--tag` et `--priority` combinables. |
| S8 | `--json` sur `list` produit un tableau JSON (jamais `null`), dates en RFC 3339 UTC. |
| S9 | Le fichier de données est `$XDG_DATA_HOME/todo/tasks.json` (ou `~/.local/share/todo/tasks.json`), surchargé par la variable `TODO_FILE`. Le répertoire est créé au besoin. |
| S10 | `export` écrit le JSON sur la sortie standard ; `import FICHIER` fusionne des tâches en leur attribuant de **nouveaux** identifiants. |
| S11 | Messages d'erreur sur `os.Stderr`, code de sortie 1 ; 2 pour une erreur d'utilisation (sous-commande inconnue, drapeau invalide). |
| S12 | `todo help` et `todo CMD -h` affichent l'aide sur la sortie standard. |

---

## Contraintes

1. **Bibliothèque standard uniquement** (y compris pour l'analyse des arguments : paquet `flag`, un `flag.FlagSet` par sous-commande).
2. Trois paquets au moins sous `internal/` : le **domaine** (`task` : type, validation, tri), le **stockage** (`store` : lecture/écriture du fichier), la **CLI** (`cli` : analyse des arguments, affichage). `main.go` ne contient que l'appel à `cli.Run(os.Args[1:], os.Stdout, os.Stderr) int`.
3. Le paquet `task` n'importe **ni** `os`, **ni** `encoding/json`, **ni** `flag`.
4. **Aucune** fonction de `task` n'appelle `time.Now()` : l'heure courante est **passée en paramètre**. (Pourquoi ? Tu le comprendras en écrivant les tests du niveau 5.)
5. Écriture **atomique** du fichier ; une interruption au milieu d'une sauvegarde ne doit jamais corrompre les données.
6. Le format du fichier contient un champ `"version": 1` en tête, pour permettre une évolution future.
7. `go vet ./...` silencieux, `gofmt -l .` vide, chaque identifiant exporté documenté.

---

## Difficultés glissées volontairement

- **D1.** Une tâche due « le 5 novembre » sans heure : est-elle en retard le 5 à 10 h ? Le 6 à 0 h 01 ? Dans quel fuseau ? Ta réponse doit être **écrite** dans la documentation du paquet `task`.
- **D2.** `+1d` le samedi 28 mars 2026 à 12 h à Paris : `Add(24h)` ou `AddDate` ? (Leçon 4.)
- **D3.** Le fichier JSON est modifié à la main et contient une erreur de syntaxe. Le programme doit le dire clairement **et ne jamais l'écraser**.
- **D4.** Deux `todo add` lancés en même temps dans deux terminaux : une tâche est-elle perdue ? (Défi de la leçon 7.) Au minimum, documenter le comportement ; idéalement, l'empêcher.
- **D5.** Après `rm` de la tâche ayant le plus grand identifiant, le prochain `add` réutilise-t-il son numéro ? Pourquoi serait-ce un problème si un script externe garde des identifiants ?
- **D6.** Un titre contenant des retours à la ligne, des tabulations ou 500 caractères : que devient l'affichage en colonnes ?
- **D7.** Le fichier contient une tâche avec une priorité inconnue (`"urgent"`) écrite par une version future du programme. Refuser, ignorer, ou conserver tel quel ?

---

## Architecture suggérée

**À ne lire qu'après avoir dessiné ta propre conception.**

```
projet-02/
├── go.mod
├── main.go                     os.Exit(cli.Run(os.Args[1:], os.Stdout, os.Stderr))
└── internal/
    ├── task/
    │   ├── task.go             type Task, Priority, validation
    │   ├── due.go              ParseDue(s string, now time.Time, loc *time.Location) (time.Time, error)
    │   └── sort.go             Sort(tasks []Task, now time.Time)
    ├── store/
    │   └── file.go             Load(path) / Save(path, …) — format JSON versionné, écriture atomique
    └── cli/
        ├── cli.go              Run, routage des sous-commandes
        ├── add.go, list.go…    une sous-commande par fichier
        └── render.go           affichage en colonnes, dates relatives
```

L'idée directrice : `task` ne sait rien du disque ni du terminal ; `store` ne sait rien des
règles métier ; `cli` orchestre. `Run` reçoit ses arguments et ses flux de sortie au lieu de
lire `os.Args` et d'écrire sur `os.Stdout` — demande-toi ce que ça permettra au niveau 5.

---

## Tests à écrire

Le niveau 5 formalisera tout ça ; tu peux déjà écrire ces vérifications dans un `main` de test
ou en anticipant le paquet `testing` :

- `ParseDue` : chacune des cinq formes, une forme invalide, `+1d` la veille du changement d'heure ;
- le tri : deux tâches à égalité sur tous les critères sauf l'identifiant ;
- l'affichage relatif : « aujourd'hui » à 23 h 59 et à 0 h 00 le lendemain ;
- `Save` puis `Load` redonne exactement les mêmes tâches (aller-retour JSON) ;
- `Load` d'un fichier absent → liste vide ; d'un fichier corrompu → erreur, fichier intact ;
- `list --json` sur une liste vide → `[]`.

---

## Ce qui sera évalué en revue de code

| Axe | Question posée |
|---|---|
| **Ça fonctionne** | Les 12 spécifications sont-elles satisfaites ? |
| **C'est correct** | Les 7 difficultés sont-elles traitées, ou au moins documentées ? Les fuseaux et le changement d'heure sont-ils justes ? |
| **C'est idiomatique** | Découpage en paquets, dépendances dirigées vers `task`, erreurs enveloppées avec contexte, zéro-valeurs utiles, pas d'état global |
| **Prêt pour la production** | Écriture atomique, fichier corrompu jamais écrasé, codes de sortie, messages d'erreur exploitables, format versionné |

Rends ton code même incomplet : une conception discutée vaut mieux qu'un projet abandonné.
