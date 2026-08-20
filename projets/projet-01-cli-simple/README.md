# Projet 1 — `wordstat`, un analyseur de texte en ligne de commande

**À réaliser après le niveau 1.** Durée indicative : 3 à 6 heures.
Écrire le code dans `mes-solutions/projets/projet-01/`.

---

## Objectif pédagogique

Mobiliser d'un seul coup tout le niveau 1 : slices, maps, structs, méthodes, erreurs,
pointeurs, tri, entrées/sorties. Et surtout : **séparer le calcul des entrées/sorties**,
ce qui rendra le projet testable au niveau 3.

---

## Spécifications

`wordstat` lit du texte et produit des statistiques.

```
$ wordstat fichier.txt
lignes    : 128
mots      : 1 042
caractères: 6 733

top 10 des mots :
  1. le          62
  2. de          58
  3. et          41
  …
```

### Comportements exigés

| # | Exigence |
|---|---|
| S1 | Sans argument, lit l'**entrée standard** : `cat f.txt \| wordstat` doit fonctionner. |
| S2 | Avec un ou plusieurs fichiers en arguments, les traite **tous** et cumule les statistiques. |
| S3 | Un fichier illisible produit un message sur `os.Stderr` et **n'interrompt pas** le traitement des autres. |
| S4 | Le code de sortie vaut `1` si au moins un fichier a échoué, `0` sinon. |
| S5 | Le comptage des mots ignore la casse et la ponctuation : `Bonjour,` et `bonjour` sont le même mot. |
| S6 | Le comptage des caractères compte les **caractères** (runes), pas les octets : `é` compte pour 1. |
| S7 | `-top N` change le nombre de mots affichés (défaut : 10). `-top 0` masque le classement. |
| S8 | `-min N` ignore les mots de moins de N caractères (défaut : 1). |
| S9 | En cas d'égalité de fréquence, les mots sont triés par ordre alphabétique — la sortie doit être **déterministe**. |
| S10 | `-h` affiche l'aide sur la sortie standard et sort avec le code `0`. |

---

## Contraintes

1. **Bibliothèque standard uniquement.** Aucune dépendance externe.
2. Le programme doit tenir dans **au moins trois fichiers** du même package (à toi de choisir le découpage pertinent).
3. **Aucune fonction de calcul ne doit afficher quoi que ce soit.** Les fonctions qui calculent retournent des valeurs ; seul le code proche de `main` écrit sur la sortie.
4. **Aucun `panic`**, aucun `os.Exit` ailleurs que dans `main`.
5. Le programme doit traiter un fichier de **1 Go** sans dépasser quelques mégaoctets de mémoire. *(Cette contrainte à elle seule élimine la solution naïve — réfléchis-y avant de coder.)*
6. `go vet ./...` doit être silencieux et `gofmt -l .` ne doit rien afficher.
7. Chaque fonction exportée porte un commentaire de documentation commençant par son nom.

---

## Difficultés glissées volontairement

Elles ne sont pas signalées dans les spécifications ci-dessus. À toi de les repérer :

- **D1.** Que se passe-t-il si le même fichier est passé deux fois ?
- **D2.** Comment traites-tu un mot comme `aujourd'hui` ou `c'est` ? Et `Jean-Pierre` ?
- **D3.** Que compte `-min` : des octets ou des runes ? (relis S6)
- **D4.** Un fichier vide doit-il être une erreur ?
- **D5.** Le fichier fait 1 Go **et** ne contient aucun retour à la ligne. Ta lecture ligne par ligne survit-elle ? *(indice : `bufio.Scanner` a une limite par défaut — laquelle ?)*
- **D6.** L'ordre d'itération d'une map en Go est délibérément aléatoire. Comment garantis-tu S9 ?

---

## Architecture suggérée

**À ne lire qu'après avoir dessiné ta propre conception.** Ce n'est qu'une piste parmi d'autres.

```
projet-01/
├── go.mod
├── main.go        analyse des drapeaux, orchestration, codes de sortie, affichage
├── stats.go       le type Stats : comptage, fusion de deux Stats, classement
└── scan.go        lecture d'un io.Reader et alimentation d'un Stats
```

L'idée directrice : `scan.go` accepte un `io.Reader`, pas un nom de fichier. Un fichier
ouvert et l'entrée standard sont alors interchangeables — et un test pourra lui passer
une simple chaîne. Demande-toi pourquoi c'est un choix de conception, pas un détail.

---

## Tests à écrire

Le niveau 3 formalisera tout ça, mais tu peux déjà écrire ces tests :

- comptage sur un texte connu à la main ;
- texte vide ;
- texte sans retour à la ligne final ;
- accents et emoji (vérifie S6) ;
- égalité de fréquence entre deux mots (vérifie S9) ;
- fusion de deux `Stats` : le résultat est-il bien la somme ?

---

## Ce qui sera évalué en revue de code

| Axe | Question posée |
|---|---|
| **Ça fonctionne** | Les 10 spécifications sont-elles satisfaites ? |
| **C'est correct** | Les 6 difficultés cachées sont-elles traitées, ou au moins conscientes et documentées ? |
| **C'est idiomatique** | Nommage, gestion d'erreurs, `io.Reader` plutôt que chemins, absence de sur-abstraction |
| **Prêt pour la production** | Mémoire bornée, erreurs claires, codes de sortie, aucun état global |

Rends ton code même incomplet : une conception discutée vaut mieux qu'un projet abandonné.
