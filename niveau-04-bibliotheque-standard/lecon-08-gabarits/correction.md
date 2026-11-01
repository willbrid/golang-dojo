# Leçon 8 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — L'injection, mesurée

Sorties réelles (Go 1.27), `text/template` puis `html/template` pour chaque valeur :

```
<p title="Tom & Jerry">Tom & Jerry</p><script>var x = Tom & Jerry;</script><a href="Tom & Jerry">l</a>
<p title="Tom &amp; Jerry">Tom &amp; Jerry</p><script>var x = "Tom & Jerry";</script><a href="Tom%20&amp;%20Jerry">l</a>

<p title=""><img src=x onerror=alert(1)>">"><img src=x onerror=alert(1)></p><script>var x = "><img src=x onerror=alert(1)>;</script><a href=""><img src=x onerror=alert(1)>">l</a>
<p title="&#34;&gt;&lt;img src=x onerror=alert(1)&gt;">&#34;&gt;&lt;img …&gt;</p><script>var x = "\"><img src=x onerror=alert(1)>";</script><a href="%22%3e%3cimg%20src=x%20onerror=alert%281%29%3e">l</a>

<p title="javascript:alert(1)">javascript:alert(1)</p><script>var x = javascript:alert(1);</script><a href="javascript:alert(1)">l</a>
<p title="javascript:alert(1)">javascript:alert(1)</p><script>var x = "javascript:alert(1)";</script><a href="#ZgotmplZ">l</a>

<p title="</script><script>alert(1)</script>"></script><script>alert(1)</script></p><script>var x = </script><script>alert(1)</script>;</script>…
<p title="&lt;/script&gt;…">&lt;/script&gt;…</p><script>var x = "</script><script>alert(1)</script>";</script>…
```

`text/template` exécute du code dans **trois cas sur quatre** :

| Valeur | Code exécuté avec `text/template` ? |
|---|---|
| `Tom & Jerry` | non, mais le `<script>` est cassé (`var x = Tom & Jerry;` : erreur de syntaxe JS) |
| `"><img … onerror=…>` | **oui** : la balise `<img>` est injectée, `onerror` s'exécute |
| `javascript:alert(1)` | **oui** au clic sur le lien |
| `</script><script>alert(1)</script>` | **oui** : le premier `</script>` ferme le bloc, le second script s'exécute |

`html/template` neutralise les quatre, avec un échappement différent selon le contexte :
entités dans le texte et les attributs, chaîne JSON dans le script (où `<` devient `<`
pour qu'un `</script>` ne ferme jamais le bloc), encodage d'URL dans `href`, et `#ZgotmplZ`
pour un schéma d'URL dangereux.

## E2 — Espaces

```
Courses :
{{- range .}}
- {{.}}
{{- end}}
Total : {{len .}} articles
```

`{{-` mange **tout** l'espace blanc qui précède, saut de ligne compris. Sans les tirets, on
obtient une ligne vide après `Courses :` et après chaque article : **4 lignes vides** (une avant
chaque `- …` et une avant `Total`), visibles avec `cat -A` sous forme de `$` seuls.

La règle pratique : placer le `-` du côté où se trouve le saut de ligne à manger.

## E3 — `<no value>`

| Exécution | Sortie |
|---|---|
| `text/template`, map, clé `nom` | `Bonjour <no value>` — aucune erreur |
| `html/template`, même map | `Bonjour ` — aucune erreur |
| `Option("missingkey=error")` | erreur : `map has no entry for key "Nom"` |
| struct avec champ `Name` (faute de frappe) | erreur : `can't evaluate field Nom in type struct { Name string }` |

La **struct** protège le mieux : l'erreur est systématique, sans option à penser. Avec une map,
la faute de frappe (`nom` contre `Nom`) passe en production.

## E4 — `FuncMap`

```go
var funcs = template.FuncMap{
	"money": func(c Cents) string { return c.String() },
	"plural": func(n int, one, many string) string {
		if n <= 1 {
			return one
		}
		return many
	},
	"date": func(t time.Time) string { return t.Format("02/01/2006") },
}

t := template.Must(template.New("r").Funcs(funcs).Parse(src))
// {{.N}} {{plural .N "article" "articles"}}
```

`Funcs` après `Parse` : `template: :1: function "upper" not defined` — l'erreur vient de
`Parse`, qui doit connaître chaque nom de fonction pour analyser le gabarit.

`plural` : en français, 0 et 1 sont au singulier (« 0 article »), contrairement à l'anglais.
D'où `n <= 1`. La localisation commence ici.

## E5 — Sortie partielle

Vers `os.Stdout`, tout ce qui précède l'erreur est **déjà affiché**, puis l'erreur est rendue
(dans la mesure : `Bonjour ` puis le message). Dans une réponse HTTP, l'en-tête `200 OK` serait
déjà parti avec une page tronquée, et il serait trop tard pour envoyer une erreur 500.

```go
var buf bytes.Buffer
if err := t.Execute(&buf, data); err != nil {
	return err          // rien n'a été écrit
}
_, err := buf.WriteTo(w)
```

## Exercice intermédiaire — Générateur de rapport

Points clés :

```go
//go:embed templates
var templatesFS embed.FS

var (
	txtTmpl  = tt.Must(tt.New("").Funcs(funcs).ParseFS(templatesFS, "templates/rapport.txt.tmpl"))
	htmlTmpl = ht.Must(ht.New("").Funcs(hfuncs).ParseFS(templatesFS,
		"templates/base.html.tmpl", "templates/rapport.html.tmpl"))
)

type ReportView struct {
	GeneratedAt string // déjà formaté
	Total       int
	ErrorRate   string // "3,2 %", déjà formaté
	Rows        []RowView
}

type RowView struct {
	Route  string
	Count  int
	Status int
	Alert  bool // calculé par le code : Status >= 500
}
```

```html
{{/* base.html.tmpl */}}
{{define "base"}}<!doctype html><html><body>{{block "content" .}}{{end}}</body></html>{{end}}

{{/* rapport.html.tmpl */}}
{{define "content"}}
<table>
{{- range .Rows}}
  <tr{{if .Alert}} style="color:red"{{end}}><td>{{.Route}}</td><td>{{.Count}}</td><td>{{.Status}}</td></tr>
{{- end}}
</table>
{{end}}
```

- **Les deux gabarits** reçoivent la même `ReportView` : la vue est indépendante du format.
- `Alert` est calculé en Go : le gabarit ne compare pas `ge .Status 500` (possible, mais c'est une
  règle métier qui n'a rien à faire dans la présentation).
- Contrainte 4 : exécuter les **deux** gabarits dans deux buffers d'abord ; écrire les fichiers
  (atomiquement) seulement si les deux ont réussi.
- `{{if .Alert}} style="color:red"{{end}}` à l'intérieur d'une balise : `html/template` comprend
  aussi ce contexte (nom d'attribut) et l'accepte parce que la valeur est une constante du gabarit.
- Le chemin `<script>` des données de test apparaît comme texte `&lt;script&gt;` dans la page.

## Défi — Générer du code Go

```go
const src = `// Code generated by gen; DO NOT EDIT.

package {{.Package}}

import "fmt"

type Level int

const (
{{- range $i, $v := .Values}}
	{{title $v}}{{if eq $i 0}} Level = iota{{end}}
{{- end}}
)

var levelNames = [...]string{ {{- range .Values}}"{{.}}", {{end -}} }

func (l Level) String() string {
	if l < 0 || int(l) >= len(levelNames) {
		return fmt.Sprintf("Level(%d)", int(l))
	}
	return levelNames[l]
}

func ParseLevel(s string) (Level, error) {
	for i, n := range levelNames {
		if n == s {
			return Level(i), nil
		}
	}
	return 0, fmt.Errorf("niveau inconnu %q", s)
}
`
```

1. `text/template` : on produit du Go, pas du HTML. `html/template` échapperait les `"` en
   `&#34;` et rendrait le code invalide.
2. ```go
   var buf bytes.Buffer
   if err := tmpl.Execute(&buf, data); err != nil { … }
   out, err := format.Source(buf.Bytes())   // erreur si le code n'est pas du Go valide
   if err != nil {
   	return fmt.Errorf("code généré invalide : %w", err)
   }
   ```
   Bonus : la sortie est formatée exactement comme `gofmt`, quelle que soit l'indentation du
   gabarit.
3. Le format exact, d'après `go help generate` : une ligne qui correspond à
   `^// Code generated .* DO NOT EDIT\.$`. Les outils (`gopls`, GitHub, les linters) la
   reconnaissent : ils signalent qu'il ne faut pas éditer le fichier à la main et l'excluent
   souvent des analyses.
4. `//go:generate go run ./gen low,medium,high` dans un fichier du paquet, puis
   `go generate ./...`. `go generate` n'est **jamais** lancé par `go build` : le fichier généré
   doit être commité.

*(Pour le nom des constantes, une petite fonction `title` qui met la première lettre en
majuscule suffit : `strings.ToUpper(s[:1]) + s[1:]` pour de l'ASCII.)*

## Quiz

1. La donnée courante : la valeur passée à `Execute`, puis l'élément courant dans `range`, la
   valeur dans `with`, l'argument dans `template`.
2. Une suite d'actions séparées par `|` : chaque résultat devient le **dernier** argument de la
   suivante.
3. Ils suppriment les espaces blancs (sauts de ligne compris) avant ou après l'action.
4. `<no value>` en `text/template`, chaîne vide en `html/template` ; `Option("missingkey=error")`
   en fait une erreur — ou utiliser une struct.
5. Parce que `Execute` écrit au fil de l'eau : une erreur à mi-chemin laisse une sortie tronquée
   déjà envoyée.
6. L'analyseur doit connaître les noms de fonctions pour analyser le gabarit.
7. L'échappement adapté au langage du point d'insertion (texte HTML, attribut, URL, JavaScript,
   CSS), déterminé en analysant le HTML.
8. Une valeur refusée par `html/template` (typiquement une URL `javascript:`).
9. Pour du contenu **de confiance** produit par le programme ; risque : désactiver la protection
   XSS si la valeur vient d'un utilisateur.
10. Dans le code Go, qui prépare une struct de vue ; le gabarit ne fait que présenter.
