# Leçon 2 — Exercices

Code dans `mes-solutions/niveau-04/lecon-02/`.

## Exercices faciles

**E1 — Ce que RE2 refuse.**
Essayer de compiler chacune de ces expressions, recopier les messages, et pour chaque refus
proposer une solution Go **sans** regexp ou avec deux regexps :

```
(\w)\1                  mots avec une lettre doublée
^(?!admin$).+$          tout sauf « admin »
\d+(?= €)               un nombre suivi de « € »
(?i)bonjour             bonjour, toute casse
```

**E2 — L'ancre oubliée.**
Écrire `ValidCode(s string) bool` qui accepte exactement 3 lettres majuscules suivies de 4
chiffres (`ABC1234`). Le tester sur `ABC1234`, `xABC1234`, `ABC12345`, `ABC1234\n`. Une des
quatre passe-t-elle à tort même avec `^…$` ? *(Indice : `go doc regexp/syntax`, chercher `\z`.)*

**E3 — Remplacer avec des groupes.**
Convertir les dates `JJ/MM/AAAA` d'un texte en `AAAA-MM-JJ`. Écrire le gabarit avec des
**noms**. Puis écrire une version qui produit `AAAA-MM-JJT` (avec un `T` collé) : que se
passe-t-il avec `$jourT` ?

**E4 — `strings` suffit-il ?**
Pour chaque besoin, écrire la solution la plus simple — regexp **ou non** :

1. la ligne commence par `#` ;
2. extraire l'extension d'un nom de fichier ;
3. la chaîne est un entier positif ;
4. trouver tous les mots-clés `#hashtag` d'un tweet ;
5. remplacer les suites de plusieurs espaces par un seul.

**E5 — Unicode.**
Extraire tous les mots de `"L'été à Paris, c'était génial !"` avec `\w+`, puis avec `\p{L}+`,
puis avec `[\p{L}']+`. Lequel donne `l'été` et `c'était` en un mot ? Est-ce souhaitable ?

---

## Exercice intermédiaire — `mdlinks`, vérifier les liens d'un fichier Markdown

Écrire un programme qui lit un fichier Markdown (chemin en argument) et liste ses liens.

**Spécifications :**
1. Reconnaître les liens `[texte](url)` et les images `![alt](url)`, en les distinguant.
2. Afficher `ligne:colonne  type  url  texte` pour chacun. La **colonne** est en caractères,
   pas en octets.
3. Ignorer les liens situés dans un bloc de code délimité par ` ``` `.
4. Signaler en fin de sortie les URL **relatives** (`docs/x.md`) qui pointent vers un fichier
   inexistant (`os.Stat`, leçon 7 — en anticipation : `_, err := os.Stat(p); err != nil`).
5. Code de sortie 1 si au moins un lien est cassé.
6. Les expressions sont compilées **une fois**.

Tester sur le `README.md` de ce dépôt.

**Question :** un lien dont le texte contient des crochets, `[voir [1]](x.md)`, est-il reconnu ?
Faut-il le supporter ? Justifier sans écrire le code.

---

## Défi — Mesurer RE2

1. Avec `regexp`, mesurer (`time.Since`) le temps de `(a+)+$` sur `"a"×n + "!"` pour
   `n = 10, 1 000, 100 000, 1 000 000`. Tracer mentalement la courbe : linéaire ?
2. Si tu as Python ou Node installé, faire la même mesure avec leur moteur pour
   `n = 20, 22, 24, 26`. Que constates-tu ? *(Ne pas aller au-delà de 28 : le processus peut
   ne pas répondre pendant de longues minutes.)*
3. Comparer `strings.Contains(s, "needle")` et `regexp.MustCompile("needle").MatchString(s)`
   sur un texte de 10 Mo : facteur de temps ?
4. En 5 lignes : dans quel type de service le choix de RE2 est-il **décisif** ?

---

## Quiz

1. Pourquoi Go refuse-t-il les rétro-références ?
2. Qu'est-ce qu'un ReDoS ?
3. Quand `MustCompile` est-il légitime ? Quand ne l'est-il pas ?
4. Que rend `FindAllStringSubmatch` ?
5. Pourquoi `regexp.MustCompile(`[a-z]+`).MatchString("<b>x")` vaut-il `true` ?
6. Que produit `$1x` dans un gabarit de remplacement ?
7. Quelle classe utiliser pour « une lettre, accents compris » ?
8. Différence entre `.*` et `.*?` ?
9. Citer trois besoins pour lesquels une regexp est le mauvais outil.
10. Que fait `regexp.QuoteMeta` ?
