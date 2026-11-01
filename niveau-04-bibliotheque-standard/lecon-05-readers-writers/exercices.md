# Leçon 5 — Exercices

Code dans `mes-solutions/niveau-04/lecon-05/`.

## Exercices faciles

**E1 — Le hash de l'exemple.**
Vérifier l'empreinte affichée par l'exemple du cours avec la commande
`printf 'bonjour\nle monde\nde go\n' | sha256sum`. Puis inverser l'ordre de `TeeReader` et
`upperReader` dans `process` : quelle commande `printf … | sha256sum` donne alors la même
empreinte que le programme ?

**E2 — Un `Read` partiel.**
Écrire un `Reader` `oneByteReader` qui ne rend jamais plus d'**un** octet par appel. Lire
`"bonjour"` à travers lui avec : (a) un seul `r.Read(buf)` sur un buffer de 16 octets ;
(b) `io.ReadFull` ; (c) `io.ReadAll`. Afficher ce que chacun obtient. Quelle leçon pour le code
qui lit des sockets ?

**E3 — `wc` en une copie.**
Écrire `wc` : lire l'entrée standard, et afficher octets, lignes et empreinte SHA-256 en **un
seul passage**, sans `ReadAll`, avec `io.Copy` et `io.MultiWriter`. Comparer avec
`wc -c -l` et `sha256sum` sur un gros fichier.

**E4 — Le `Flush` oublié.**
Écrire 10 lignes dans un fichier via `bufio.NewWriter` **sans** `Flush`. Combien d'octets
contient le fichier ? Et avec 10 000 lignes de 80 caractères ? Expliquer les deux chiffres.

**E5 — `ReadAll` borné.**
Écrire `ReadAtMost(r io.Reader, max int64) ([]byte, error)` qui rend une **erreur** (pas une
troncature silencieuse) si `r` contient plus de `max` octets. *(Indice : lire `max + 1`.)*

---

## Exercice intermédiaire — `linecat`, un `cat -n` composable

Écrire un `io.Writer` décorateur `LineNumberWriter` qui préfixe chaque ligne écrite à travers
lui par son numéro, au format `%6d\t` (comme `cat -n`).

```go
func NewLineNumberWriter(w io.Writer) *LineNumberWriter
```

**Contraintes :**
1. Les lignes peuvent arriver **découpées n'importe comment** : un `Write("bon")` puis
   `Write("jour\nle")` puis `Write(" monde\n")` doivent donner exactement
   `     1\tbonjour\n     2\tle monde\n`. Il faut donc mémoriser si l'on est en début de ligne.
2. Le `n` rendu par `Write` doit respecter le contrat : le nombre d'octets **de `p`** consommés,
   pas le nombre d'octets écrits en dessous (préfixes compris). Pourquoi ?
3. Une erreur du `Writer` sous-jacent est remontée immédiatement.
4. Un `main` qui fait `io.Copy(NewLineNumberWriter(os.Stdout), os.Stdin)`. Vérifier :
   `diff <(cat -n fichier) <(go run . < fichier)` ne doit rien afficher.
5. Tester avec un `oneByteReader` (E2) en entrée : la sortie doit être identique.

---

## Défi — `CountingReader` et progression

1. Écrire un `CountingReader` qui compte les octets lus, et l'utiliser pour afficher une barre
   de progression pendant la copie d'un gros fichier (`os.Stat` donne la taille totale —
   anticipation de la leçon 7). La barre se met à jour au plus 10 fois par seconde : comment
   le garantir sans goroutine ?
2. `io.Copy` détecte `io.WriterTo` sur la source. Si la source est un `*os.File` enveloppé dans
   ton `CountingReader`, l'optimisation est-elle encore possible ? Qu'est-ce que ça dit du coût
   caché d'un décorateur ?
3. Lire `go doc io.Copy` et `go doc -src io.copyBuffer`. Combien de cas particuliers y a-t-il
   avant la boucle générale ?

---

## Quiz

1. Un `Read` sur un buffer de 4 096 octets peut-il rendre 10 octets sans être à la fin ?
2. `io.EOF` est-il une erreur ?
3. Que faut-il faire quand `Read` rend `n > 0` et `err != nil` ?
4. Que garantit `Write` quand il rend `n < len(p)` ?
5. Pourquoi `io.ReadAll` est-il dangereux sur un corps de requête HTTP ?
6. À quoi servent `TeeReader` et `MultiWriter` ?
7. Pourquoi `bufio.Writer` améliore-t-il les performances ? Quel est son prix ?
8. Pourquoi ne pas mettre `defer w.Flush()` en production ?
9. Une fonction qui reçoit un `io.Reader` doit-elle le fermer ?
10. Pourquoi passer `&buf` et non `buf` quand `buf` est un `bytes.Buffer` ?
