# Leçon 1 — Exercices

Code dans `mes-solutions/niveau-04/lecon-01/`.

## Exercices faciles

**E1 — Prédire `Split`.**
Écrire **sur papier** le résultat (avec `%q`) de chaque appel, puis vérifier :

```go
strings.Split("a,b,c", ",")
strings.Split("a,b,c,", ",")
strings.Split(",", ",")
strings.Split("", ",")
strings.Split("abc", "")
strings.SplitN("a,b,c", ",", 0)
strings.SplitN("a,b,c", ",", -1)
strings.Fields("\t a \n b  ")
```

Lequel t'a surpris ? Trouver la phrase de `go doc strings.Split` qui l'explique.

**E2 — Le piège de `TrimLeft`.**
Un collègue a écrit `host := strings.TrimLeft(url, "https://")`. Trouver **trois** URL
réalistes pour lesquelles le résultat est faux. Corriger avec `CutPrefix` en traitant aussi
`http://`.

**E3 — Échapper.**
Écrire `EscapeCSV(field string) string` : si le champ contient `,`, `"` ou un retour à la
ligne, l'entourer de guillemets et **doubler** les guillemets internes. Exemples :
`a` → `a` ; `a,b` → `"a,b"` ; `dit "oui"` → `"dit ""oui"""`. Un `Replacer` est-il utile ici ?

**E4 — Le coût de `+=`.**
Construire une chaîne de `n` fois `"x"` avec `+=`, puis avec un `Builder`, pour
`n = 10 000, 20 000, 40 000, 80 000`. Mesurer avec `time.Since`. Quand `n` double, par
combien le temps est-il multiplié dans chaque cas ? Quel mot désigne ce comportement ?

**E5 — Positions.**
Dans `"Le café coûte 3€"`, trouver la position de `3` avec `strings.Index`. Afficher la
position en **octets** puis en **caractères**. Expliquer l'écart chiffre par chiffre.

---

## Exercice intermédiaire — `wrap`, justification de texte

Écrire `Wrap(text string, width int) string` qui reformate un texte en lignes d'au plus
`width` **caractères** (pas octets), en coupant entre les mots.

**Contraintes :**
1. Les espaces multiples et les retours à la ligne existants sont normalisés (un paragraphe =
   une suite de mots). Les **paragraphes** (séparés par une ligne vide) sont conservés.
2. Un mot plus long que `width` est placé seul sur sa ligne, sans être coupé.
3. Aucune ligne ne se termine par un espace. Vérifier avec `cat -A`.
4. Un seul `strings.Builder` pour tout le résultat.
5. `width <= 0` est une erreur : changer la signature en conséquence.
6. Tester sur un paragraphe français contenant des accents, avec `width = 20`. Vérifier qu'une
   ligne de 20 caractères accentués n'est **pas** coupée à tort (piège : `len`).

---

## Défi — Un `Replacer` qui ne fait pas ce qu'on croit

1. Que donne `strings.NewReplacer("a", "1", "aa", "2").Replace("aaa")` ? Et
   `strings.NewReplacer("aa", "2", "a", "1").Replace("aaa")` ? Formuler la règle de
   priorité **à partir de ces deux mesures**, puis la retrouver dans `go doc strings.NewReplacer`.
2. Écrire un convertisseur Markdown minimaliste → texte brut qui retire `**`, `*`, `` ` `` et
   transforme `[texte](url)` en `texte (url)`. Quelles parties un `Replacer` sait-il faire ?
   Laquelle ne sait-il pas faire, et pourquoi ? *(La leçon 2 apporte l'outil manquant.)*

---

## Quiz

1. Que rend `strings.Split("", ",")` ? Et `strings.Fields("")` ?
2. Différence entre `TrimLeft(s, "ab")` et `TrimPrefix(s, "ab")` ?
3. Pourquoi `EqualFold` plutôt que `ToLower(a) == ToLower(b)` ?
4. Pourquoi `strings.Title` est-il déprécié ?
5. `strings.Index` rend-il une position en caractères ?
6. Pourquoi trois `ReplaceAll` successifs peuvent-ils donner un résultat différent d'un `Replacer` ?
7. Que fait `strings.Map` quand la fonction rend `-1` ?
8. Pourquoi `s += x` en boucle est-il coûteux ? Quelle est sa complexité ?
9. Quand faut-il utiliser le paquet `bytes` plutôt que `strings` ?
10. Quelle commande affiche les exemples d'une fonction (Go 1.27) ?
