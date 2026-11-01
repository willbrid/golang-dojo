# Leçon 8 — Gabarits

> Générer un courriel, un fichier de configuration, un rapport ou une page HTML à partir de
> données : c'est le rôle des gabarits (*templates*). Go en fournit deux paquets à la syntaxe
> **identique** — `text/template` et `html/template` — dont la différence est une question de
> sécurité. Utiliser le mauvais pour du HTML, c'est ouvrir une faille XSS.

## Objectifs

1. Écrire et exécuter un gabarit : actions, pipelines, conditions, boucles, variables.
2. Organiser des gabarits réutilisables (`define`, `template`, `block`) et les embarquer.
3. Comprendre l'**échappement contextuel** de `html/template` et pourquoi il est indispensable.
4. Savoir où mettre la logique : dans le gabarit, ou dans le code Go.

## Explication

### Le principe

```go
t := template.Must(template.New("hello").Parse("Bonjour {{.Name}}, vous avez {{.Count}} messages.\n"))
err := t.Execute(os.Stdout, data)   // écrit dans n'importe quel io.Writer (leçon 5)
```

Le texte est recopié tel quel ; ce qui est entre `{{` et `}}` est une **action**, évaluée avec
les données. `.` (le point) désigne la donnée **courante** : au départ, la valeur passée à
`Execute`.

`template.Must` panique si le gabarit ne se compile pas — même raisonnement que
`regexp.MustCompile` (leçon 2) : légitime pour un gabarit écrit dans le code, analysé au
démarrage.

### Les actions

| Action | Rôle |
|---|---|
| `{{.Name}}` | champ (exporté) d'une struct, clé d'une map, ou méthode sans argument |
| `{{.User.Email}}` | chaîne d'accès |
| `{{.Name \| printf "%-10s"}}` | **pipeline** : le résultat devient le dernier argument de la fonction suivante |
| `{{if .Admin}}…{{else if .Mod}}…{{else}}…{{end}}` | condition : faux = zéro-valeur, nil, vide |
| `{{range .Items}}…{{.}}…{{else}}aucun{{end}}` | boucle ; `.` devient l'élément ; `else` si vide |
| `{{range $i, $e := .Items}}` | avec indice et variable |
| `{{with .Address}}…{{.City}}…{{end}}` | change `.` si la valeur est non vide |
| `{{$total := len .Items}}` | variable locale |
| `{{/* commentaire */}}` | |
| `{{- …}}`, `{{… -}}` | supprime les espaces et sauts de ligne avant/après |

Les fonctions prédéfinies : `len`, `index`, `printf`, `print`, `and`, `or`, `not`, `eq`, `ne`,
`lt`, `le`, `gt`, `ge`, `slice`, `html`, `js`, `urlquery`, `call`.

```go
{{if and .Active (gt .Count 0)}}…{{end}}   // les arguments se parenthèsent
```

### Ajouter ses fonctions

```go
t := template.New("r").Funcs(template.FuncMap{
	"upper": strings.ToUpper,
	"money": func(c Cents) string { return c.String() },
})
// Funcs AVANT Parse : l'analyseur doit connaître les noms.
```

Une fonction du `FuncMap` peut rendre une valeur, ou une valeur et une `error` — une erreur
interrompt l'exécution.

### Les erreurs : à l'analyse et à l'exécution

- Erreur de **syntaxe** : rendue par `Parse`.
- Erreur d'**exécution** : rendue par `Execute` — champ inexistant dans une struct
  (`can't evaluate field Nope in type …`), fonction qui échoue.

**Piège** : une clé **absente d'une map** n'est pas une erreur par défaut ; `text/template`
affiche `<no value>`, `html/template` une chaîne vide. L'option `Option("missingkey=error")` la transforme en erreur. Préférer des
**structs** comme données : une faute de frappe y devient une erreur.

**Autre piège** : `Execute` écrit **au fil de l'eau**. En cas d'erreur au milieu, la moitié de la
sortie a déjà été écrite dans le `Writer`. Pour une réponse HTTP ou un fichier, exécuter dans un
`bytes.Buffer`, et ne copier vers la destination qu'en cas de succès.

### Composer : `define`, `template`, `block`

```
{{define "base"}}<html><body>{{template "content" .}}</body></html>{{end}}
{{define "content"}}<p>{{.Message}}</p>{{end}}
```

`{{template "nom" .}}` inclut un gabarit nommé en lui passant une donnée (ne pas oublier le `.`,
sinon le sous-gabarit reçoit `nil`). `{{block "nom" .}}défaut{{end}}` définit **et** inclut, avec
un contenu par défaut qu'un autre fichier peut redéfinir — le mécanisme des mises en page.

`template.ParseFS(fsys, "templates/*.html")` analyse tous les fichiers d'un `fs.FS` — typiquement
un `embed.FS` (leçon 7) : les gabarits voyagent dans le binaire.

### `text/template` contre `html/template`

Même gabarit, même donnée hostile `"><script>alert('x')</script>&` :

```html
<a href="/u?q={{.Q}}" onclick="f({{.Q}})" title="{{.Q}}">{{.Q}}</a>
```

Avec **`text/template`** (aucune protection) :

```html
<a href="/u?q="><script>alert('x')</script>&" onclick="f("><script>alert('x')</script>&)" …
```

Le script est injecté dans la page : c'est une faille **XSS** (*cross-site scripting*). Le
navigateur de chaque visiteur exécute le code de l'attaquant, avec sa session.

Avec **`html/template`** :

```html
<a href="/u?q=%22%3e%3cscript%3e…" onclick="f(&#34;\&#34;><script…&#34;)"
   title="&#34;&gt;&lt;script&gt;…">&#34;&gt;&lt;script&gt;…</a>
```

La même valeur est échappée **quatre fois différemment** : encodage d'URL dans le `href`, chaîne
JavaScript dans le `onclick`, entités HTML dans l'attribut et dans le texte. C'est
l'**échappement contextuel** : `html/template` analyse le HTML et sait, à chaque `{{…}}`, dans
quel langage il se trouve. Aucune fonction d'échappement unique ne pourrait faire ça.

Il neutralise aussi les URL dangereuses : `href="{{.U}}"` avec `javascript:alert(1)` devient
`href="#ZgotmplZ"` — une valeur sentinelle qui signale « contenu refusé ».

Règle sans exception : **tout HTML se génère avec `html/template`**. `text/template` est pour le
texte brut, les configurations, le code généré, les courriels en texte.

### Contourner l'échappement : `template.HTML`

Pour insérer du HTML **de confiance** (produit par son propre code, par un convertisseur
Markdown qui assainit), on le convertit en `template.HTML`, `template.URL`, `template.JS`… :
`html/template` l'insère alors tel quel. Chaque conversion de ce type est une **affirmation de
sécurité** : jamais sur une donnée venue d'un utilisateur. En revue de code, chaque
`template.HTML(…)` mérite une question.

### Où mettre la logique

Le langage de gabarits est volontairement pauvre : pas d'arithmétique, pas d'affectation de
champ. C'est voulu. Un gabarit **présente** des données ; il ne les **calcule** pas. La bonne
pratique est de préparer dans le code Go une **struct de vue** qui contient exactement ce que le
gabarit affiche (totaux calculés, dates déjà formatées, listes déjà triées), et de garder le
gabarit trivial.

## Exemple

```go
package main

import (
	"bytes"
	"fmt"
	"html/template"
	"os"
	"strings"
)

const page = `{{define "page" -}}
<h1>{{.Title}}</h1>
{{template "list" .Items}}
<p>Total : {{.Total}}</p>
{{- end}}

{{define "list" -}}
<ul>
{{- range .}}
  <li><a href="/produit?id={{.ID}}">{{.Name | upper}}</a> — {{.Price}}{{if .Promo}} <b>promo</b>{{end}}</li>
{{- else}}
  <li>aucun produit</li>
{{- end}}
</ul>
{{- end}}`

type Item struct {
	ID    string
	Name  string
	Price string // déjà formaté par le code Go : le gabarit ne calcule pas
	Promo bool
}

// View contient EXACTEMENT ce que le gabarit affiche.
type View struct {
	Title string
	Items []Item
	Total string
}

var tmpl = template.Must(template.New("").Funcs(template.FuncMap{
	"upper": strings.ToUpper,
}).Parse(page))

// render exécute dans un buffer : rien n'est écrit en cas d'erreur à mi-chemin.
func render(name string, data any) ([]byte, error) {
	var buf bytes.Buffer
	if err := tmpl.ExecuteTemplate(&buf, name, data); err != nil {
		return nil, err
	}
	return buf.Bytes(), nil
}

func main() {
	v := View{
		Title: "Catalogue <été>",
		Items: []Item{
			{ID: "42", Name: "clavier", Price: "49,90 €", Promo: true},
			{ID: `1"><script>x()</script>`, Name: "<b>souris</b>", Price: "19,50 €"},
		},
		Total: "69,40 €",
	}
	out, err := render("page", v)
	if err != nil {
		fmt.Println(err)
		return
	}
	os.Stdout.Write(out)
	fmt.Println()

	empty, _ := render("list", []Item(nil))
	fmt.Println(string(empty))

	if _, err := render("page", struct{ Title string }{"x"}); err != nil {
		fmt.Println("erreur :", err)
	}
}
```

Sortie :

```
<h1>Catalogue &lt;été&gt;</h1>
<ul>
  <li><a href="/produit?id=42">CLAVIER</a> — 49,90 € <b>promo</b></li>
  <li><a href="/produit?id=1%22%3e%3cscript%3ex%28%29%3c%2fscript%3e">&lt;B&gt;SOURIS&lt;/B&gt;</a> — 19,50 €</li>
</ul>
<p>Total : 69,40 €</p>
<ul>
  <li>aucun produit</li>
</ul>
erreur : template: :3:18: executing "page" at <.Items>: can't evaluate field Items in type struct { Title string }
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `{{define …}}` × 2 | Deux gabarits nommés dans une seule source ; `ExecuteTemplate` choisit lequel exécuter. |
| `{{template "list" .Items}}` | Le sous-gabarit reçoit **seulement** la liste : `.` y désigne `[]Item`. |
| `{{-` et `-}}` | Contrôlent les sauts de ligne : sans eux, la sortie serait pleine de lignes vides. |
| `{{range .}}…{{else}}` | Le cas vide est traité **dans** le gabarit, pas par un `if len`. |
| `{{.Name \| upper}}` | Pipeline vers une fonction du `FuncMap`, déclaré **avant** `Parse`. |
| `Price string` | La vue reçoit des valeurs déjà formatées : aucune logique monétaire dans le gabarit. |
| L'`ID` hostile | Encodé pour une URL (`%22%3e…`) ; le nom contenant `<b>` est échappé en entités. Aucun code injecté. |
| `render` dans un `bytes.Buffer` | Le dernier appel échoue **après** avoir produit le `<h1>` : grâce au buffer, rien n'est envoyé. |
| `can't evaluate field Items` | Avec une struct, un champ absent est une **erreur**. Avec une `map[string]any` sans clé `Items`, `html/template` aurait rendu une page à moitié vide **sans erreur**. |

## Erreurs fréquentes

1. **`text/template` pour du HTML** : XSS.
2. **`template.HTML(saisieUtilisateur)`** : on a désactivé la protection à la main.
3. **`Funcs` après `Parse`** : `function "upper" not defined`.
4. **`{{template "x"}}` sans le `.`** : le sous-gabarit reçoit `nil`.
5. **Exécuter directement vers la réponse** : sortie à moitié écrite en cas d'erreur.
6. **Des maps comme données** : les fautes de frappe donnent `<no value>`, sans erreur.
7. **De la logique métier dans le gabarit** : calculs, tris, formats.
8. **Analyser le gabarit à chaque requête** : on l'analyse une fois, au démarrage.
9. **Champ non exporté** dans la struct de vue : inaccessible au gabarit.

## Bonnes pratiques Go

- `html/template` pour tout ce qui finit dans un navigateur.
- Gabarits analysés **une fois** (variable de paquet, `Must`), embarqués avec `//go:embed` et `ParseFS`.
- Une **struct de vue** par page, préparée par le code Go.
- Exécuter dans un buffer, puis écrire.
- Chaque `template.HTML`/`URL`/`JS` justifié en commentaire.
- Pour du code Go généré : `text/template` puis `go/format.Source` pour un résultat `gofmt`.

## Ce que je dois retenir

- `{{.}}`, champs, pipelines `|`, `if`/`range`/`with`/`else`, `{{-  -}}`.
- `FuncMap` avant `Parse` ; `define`/`template`/`block` pour composer.
- Clé de map absente → `<no value>` par défaut ; préférer des structs.
- `Execute` écrit au fil de l'eau : passer par un buffer.
- **`html/template` = échappement contextuel** (HTML, attribut, URL, JS) ; `text/template` = aucun.
- `template.HTML` désactive la protection : seulement pour du contenu de confiance.

---

➡️ [Exercices](exercices.md)
