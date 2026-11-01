# Leçon 2 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Ce que RE2 refuse

```
(\w)\1          → error parsing regexp: invalid escape sequence: `\1`
^(?!admin$).+$  → error parsing regexp: invalid or unsupported Perl syntax: `(?!`
\d+(?= €)       → error parsing regexp: invalid or unsupported Perl syntax: `(?=`
(?i)bonjour     → accepté
```

Solutions :

- **Lettre doublée** : sans regexp, une boucle qui compare chaque rune à la précédente.
  Trois lignes, plus rapides et plus claires que n'importe quelle expression.
- **Tout sauf « admin »** : `s != "admin" && s != ""`. Le lookahead négatif servait à faire
  tenir une condition dans une seule expression ; en Go, on écrit la condition.
- **Nombre suivi de « € »** : capturer le contexte et ne garder que le groupe :
  `(\d+) €` puis `m[1]`. C'est le remplacement standard d'un lookahead positif.
- `(?i)` est un **drapeau**, pas un lookaround : il est supporté.

## E2 — L'ancre oubliée

```go
var codeRe = regexp.MustCompile(`^[A-Z]{3}[0-9]{4}$`)
```

| Entrée | Résultat |
|---|---|
| `ABC1234` | `true` |
| `xABC1234` | `false` |
| `ABC12345` | `false` |
| `ABC1234\n` | `false` |

**Aucune** ne passe à tort en Go. C'est la réponse piège : en Perl, PCRE (PHP) ou Python, `$`
accepte aussi la position **avant un `\n` final**, et `ABC1234\n` serait validé — une faille
classique (un en-tête HTTP ou un nom de fichier avec un saut de ligne injecté). En RE2, hors
mode multiligne `(?m)`, `$` signifie **fin du texte**, comme `\z`. Tu aurais pu le trouver dans
`go doc regexp/syntax` : *« $ at end of text (like \z not \Z) or line (flag m=true) »*.

Ne pas le croire : le **mesurer**, ce que l'exercice demandait.

## E3 — Remplacer avec des groupes

```go
var dateRe = regexp.MustCompile(`(?P<jour>\d{2})/(?P<mois>\d{2})/(?P<an>\d{4})`)

dateRe.ReplaceAllString(s, "$an-$mois-$jour")     // 2026-10-03
dateRe.ReplaceAllString(s, "$an-$mois-$jourT")    // 2026-10-   ← le jour a disparu !
dateRe.ReplaceAllString(s, "$an-$mois-${jour}T")  // 2026-10-03T
```

`$jourT` désigne le groupe `jourT`, qui n'existe pas : remplacé par une chaîne **vide**, sans
erreur. Le piège est silencieux. Réflexe : **toujours des accolades** quand un caractère de mot
suit la référence.

## E4 — `strings` suffit-il ?

| Besoin | Solution |
|---|---|
| commence par `#` | `strings.HasPrefix(line, "#")` |
| extension | `filepath.Ext(name)` (leçon 7) — gère les cas `archive.tar.gz`, `.bashrc` |
| entier positif | `n, err := strconv.Atoi(s)` puis `n > 0` — valide **et** convertit |
| hashtags | **regexp** : `#[\p{L}\p{N}_]+` — motif variable, plusieurs occurrences |
| espaces multiples | `strings.Join(strings.Fields(s), " ")` — sans regexp, et gère aussi tabulations et retours |

Une seule sur cinq justifie une expression régulière.

## E5 — Unicode

```
\w+        → ["L" "t" "Paris" "c" "tait" "g" "nial"]
\p{L}+     → ["L" "été" "à" "Paris" "c" "était" "génial"]
[\p{L}']+  → ["L'été" "à" "Paris" "c'était" "génial"]
```

`\w` découpe les mots au milieu des accents : inutilisable pour du français. `[\p{L}']+` garde
les élisions en un seul mot. **Souhaitable ?** Ça dépend du besoin — c'est la difficulté D2
du projet 1. Pour compter des mots, `l'été` devrait sans doute compter `l'` et `été`
séparément ; pour une recherche plein texte, on veut `été` seul. Il n'y a pas de bonne
réponse universelle, il y a une **décision documentée**.

## Exercice intermédiaire — `mdlinks`

```go
var (
	linkRe  = regexp.MustCompile(`(!?)\[([^\]]*)\]\(([^)\s]+)\)`)
	fenceRe = regexp.MustCompile("^\\s*```")
)

type Link struct {
	Line, Col int
	Image     bool
	Text, URL string
}

func scanLinks(r io.Reader) ([]Link, error) {
	var links []Link
	inFence := false
	sc := bufio.NewScanner(r)
	for n := 1; sc.Scan(); n++ {
		line := sc.Text()
		if fenceRe.MatchString(line) {
			inFence = !inFence
			continue
		}
		if inFence {
			continue
		}
		for _, m := range linkRe.FindAllStringSubmatchIndex(line, -1) {
			links = append(links, Link{
				Line:  n,
				Col:   utf8.RuneCountInString(line[:m[0]]) + 1,
				Image: m[3] > m[2],
				Text:  line[m[4]:m[5]],
				URL:   line[m[6]:m[7]],
			})
		}
	}
	return links, sc.Err()
}

func isRelative(u string) bool {
	return !strings.Contains(u, "://") && !strings.HasPrefix(u, "#") && !strings.HasPrefix(u, "mailto:")
}
```

Points de correction :

- **`FindAllStringSubmatchIndex`** : on a besoin des **positions** pour la colonne. Le slice
  rendu contient des paires `[début, fin]` : `m[0:2]` pour la correspondance entière, `m[2:4]`
  pour le groupe 1, etc.
- **Colonne en caractères** : `RuneCountInString(line[:m[0]]) + 1`. Les positions de `regexp`
  sont en octets, comme celles de `strings`.
- **Image** : le groupe 1 capture `!` ou rien ; `m[3] > m[2]` signifie « non vide ».
- **Fichiers relatifs** : résoudre le chemin **par rapport au répertoire du fichier Markdown**
  (`filepath.Join(filepath.Dir(mdPath), url)`), pas au répertoire courant — l'erreur la plus
  fréquente. Retirer aussi l'ancre `#section` avant `os.Stat`.
- La détection des blocs de code fonctionne pour ` ``` `, pas pour les blocs indentés de
  4 espaces — limite à documenter.

**Question des crochets imbriqués.** `[voir [1]](x.md)` n'est **pas** reconnu : `[^\]]*`
s'arrête au premier `]`. Les crochets **équilibrés** ne sont pas un langage régulier — aucune
expression régulière, RE2 ou non, ne sait compter des niveaux d'imbrication. Le supporter
exigerait un vrai analyseur. Pour un outil de vérification, ne pas le supporter et le
documenter est raisonnable ; pour un moteur de rendu Markdown, ce serait inacceptable.

## Défi — Mesurer RE2

Mesures sur la machine de rédaction :

| n | Go `(a+)+$` |
|---|---|
| 10 | 5 µs |
| 1 000 | 215 µs |
| 100 000 | 10 ms |
| 1 000 000 | 82 ms |

Grossièrement **linéaire** : ×100 sur l'entrée → ×50 à ×100 sur le temps.

Python (`re`, moteur à retour arrière) :

| n | Python |
|---|---|
| 20 | 0,10 s |
| 22 | 0,32 s |
| 24 | 1,30 s |

Chaque `a` supplémentaire **double** le temps : **exponentiel**. À n = 40, on parle de jours.

`strings.Contains` contre `regexp` sur 10 Mo : sur le motif **littéral** `needle`, les deux
sont du même ordre (quelques millisecondes) — le moteur détecte le littéral et utilise la même
recherche rapide. Avec `(?i)needle` ou `\bneedle\b`, la regexp passe à ~500 ms : facteur 50.
Une regexp n'est pas « lente » ou « rapide » dans l'absolu ; **ça se mesure, motif par motif**.

Où RE2 est décisif : tout service qui applique des expressions à des **entrées non fiables**
— passerelle d'API, WAF, moteur de recherche avec motifs fournis par l'utilisateur, filtre de
journaux multi-tenant. Un seul motif pathologique ou une seule entrée malveillante bloque un
cœur avec un moteur à retour arrière ; avec RE2, le pire cas est borné par la taille de
l'entrée, qu'on limite déjà par ailleurs.

## Quiz

1. Elles exigent un retour arrière, incompatible avec la garantie de temps linéaire de RE2.
2. Un déni de service par une entrée qui fait exploser le temps d'un moteur à retour arrière.
3. Légitime pour une expression constante écrite dans le code (erreur de programmation visible
   au démarrage) ; illégitime pour une expression venue d'un utilisateur ou d'une configuration.
4. `[][]string` : pour chaque correspondance, le texte complet puis chaque groupe.
5. Pas d'ancres : l'expression trouve `b` (ou `x`) **quelque part** dans la chaîne.
6. La valeur du groupe **nommé** `1x` — inexistant, donc chaîne vide.
7. `\p{L}`.
8. `.*` prend le plus possible (gourmand), `.*?` le moins possible (paresseux).
9. Préfixe/suffixe, découpe sur séparateur fixe, conversion de nombre, dates, URL, JSON/HTML…
10. Échappe tous les métacaractères pour qu'un texte soit cherché littéralement.
