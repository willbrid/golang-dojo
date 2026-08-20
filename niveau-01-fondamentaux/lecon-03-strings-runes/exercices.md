# Leçon 3 — Exercices

Code dans `mes-solutions/niveau-01/lecon-03/`.

## Exercices faciles

**E1 — Octets et runes.**
Pour chacune des chaînes `"hello"`, `"héllo"`, `"日本語"`, `"🙂👍"`, afficher : la longueur en
octets, le nombre de runes, et les octets bruts en hexadécimal (`% x`).
**Prédire chaque valeur avant d'exécuter.**

**E2 — Deux boucles.**
Parcourir `"café"` avec `for i := 0; i < len(s); i++` puis avec `for i, r := range s`,
en affichant l'indice à chaque tour. Expliquer en deux phrases pourquoi les indices diffèrent.

**E3 — Comptage.**
Écrire `CountVowels(s string) int` qui compte les voyelles, **accents compris**
(`a e i o u y à é è ê ï…`). *Indice : `strings.ContainsRune` sur une chaîne de voyelles,
après normalisation. Ne pas comparer des octets.*

**E4 — `string(42)`.**
Écrire un programme contenant `fmt.Println(string(42))` puis lancer `go vet ./...`.
Recopier l'avertissement. Écrire la version correcte.

**E5 — Découpage.**
Analyser la ligne `"host=localhost port=5432 user=app"` et afficher chaque paire
clé/valeur. Utiliser `strings.Fields` puis `strings.Cut`. Traiter le cas d'un champ sans `=`.

---

## Exercice intermédiaire — `slug`

Écrire `Slugify(s string) string` qui transforme un titre en identifiant d'URL :

```
"Bonjour le Monde !"        → "bonjour-le-monde"
"  Élégant   & Rapide  "    → "elegant-rapide"
"Go 1.27 : c'est sorti"     → "go-1-27-c-est-sorti"
"---"                       → ""
"日本語 test"                → "test"          (voir contrainte 5)
```

**Contraintes :**
1. Minuscules uniquement, chiffres autorisés.
2. Les accents latins sont réduits à leur lettre de base : `é → e`, `ç → c`, `ü → u`.
3. Tout autre caractère devient un tiret.
4. Jamais deux tirets consécutifs, jamais de tiret en début ou en fin.
5. Les caractères non latins et non ASCII sont supprimés, pas transformés en tirets… ou bien ? **Décider, documenter le choix en commentaire, et s'y tenir.**
6. Une seule passe sur la chaîne, un seul `strings.Builder`, **aucune** concaténation avec `+`.
7. La fonction n'affiche rien et ne retourne pas d'erreur.

*Indices : `unicode.IsLetter`, `unicode.IsDigit`, `unicode.Is(unicode.Latin, r)`. Pour la
contrainte 2, une table de correspondance explicite est acceptable au niveau 1 — la solution
générale passe par `golang.org/x/text/unicode/norm`, hors périmètre ici.*

Écrire au moins huit cas de vérification dans `main`, dont la chaîne vide et une chaîne
uniquement composée de ponctuation.

---

## Défi

**a)** Écrire `Reverse(s string) string` qui inverse une chaîne **sans corrompre l'UTF-8**.
Vérifier sur `"héllo"`, `"日本語"` et `"🙂👍"`.

**b)** Écrire `IsPalindrome(s string) bool` qui ignore la casse, les accents, les espaces
et la ponctuation. `"Élu par cette crapule"` doit retourner `true`.

**c)** Mesurer : comparer trois implémentations de la concaténation de 100 000 mots
(`+=`, `strings.Builder`, `strings.Join`). Chronométrer avec `time.Now()` / `time.Since`
et afficher les trois durées. **Prédire l'ordre de grandeur de l'écart avant de mesurer.**
Ce sera repris proprement au niveau 3 avec de vrais benchmarks.

---

## Quiz

1. Que vaut `len("héllo")` ? Pourquoi ?
2. De quel type est `s[0]` quand `s` est une `string` ?
3. Dans `for i, r := range s`, que représente exactement `i` ?
4. Pourquoi `s[0] = 'B'` ne compile-t-il pas ?
5. Que vaut `string(72)` ? Comment obtenir `"72"` ?
6. Pourquoi la concaténation avec `+` dans une boucle est-elle un problème de performance ?
7. Quelle différence entre `len([]rune(s))` et `utf8.RuneCountInString(s)` ?
8. Pourquoi `strings.EqualFold(a, b)` est-il préférable à `strings.ToLower(a) == strings.ToLower(b)` ?
9. Quel est le résultat de `range` sur une chaîne contenant un octet UTF-8 invalide ?
