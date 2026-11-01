# Leçon 2 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Trop tard

Mesuré : le client reçoit **200**, sans l'en-tête `X-Trop-Tard`, avec `Content-Type: text/html;
charset=utf-8` (deviné à partir de `<p>`), et le serveur journalise :

```
http: superfluous response.WriteHeader call from main.main.func1 (main.go:9)
```

Le premier `Fprint` a envoyé un statut 200 implicite et les en-têtes connus à ce moment-là. Tout ce qui
suit ne concerne plus que le corps.

## E2 — Le `return` oublié

Le client reçoit **400** (premier statut écrit) et un corps qui contient **les deux** : le message de
`http.Error` puis le JSON — une réponse incohérente, et un journal `superfluous response.WriteHeader`
si le code du JSON appelle `WriteHeader`. Test :

```go
rec := httptest.NewRecorder()
h(rec, httptest.NewRequest("GET", "/bad", nil))
if rec.Code != 400 || strings.Contains(rec.Body.String(), "{") {
	t.Errorf("= %d %q : réponse d'erreur suivie d'autre chose", rec.Code, rec.Body.String())
}
```

`go vet` ne le détecte pas ; certains linters (`gocritic`) signalent du code après `http.Error` sans
`return`. La parade sûre : une fonction `writeError` suivie **systématiquement** de `return`, et des
tests sur les chemins d'erreur.

## E3 — Paramètres de requête

```go
func search(w http.ResponseWriter, r *http.Request) {
	q := r.URL.Query()
	text := q.Get("q")
	if n := utf8.RuneCountInString(text); n < 1 || n > 100 {
		http.Error(w, "paramètre q : 1 à 100 caractères", http.StatusBadRequest)
		return
	}
	page := 1
	if p := q.Get("page"); p != "" {
		v, err := strconv.Atoi(p)
		if err != nil || v < 1 {
			http.Error(w, fmt.Sprintf("paramètre page : entier ≥ 1 attendu, reçu %q", p), http.StatusBadRequest)
			return
		}
		page = v
	}
	tags := q["tag"]
	if len(tags) > 5 {
		http.Error(w, "paramètre tag : au plus 5 valeurs", http.StatusBadRequest)
		return
	}
	…
}
```

`q["tag"]` rend toutes les valeurs ; `q.Get("tag")` seulement la première. Longueur en **caractères**
(`RuneCountInString`, N1 L8). Messages qui nomment le paramètre et citent la valeur (`%q`).

## E4 — 413

Avec `MaxBytesReader` : 413 et `*http.MaxBytesError`. Avec `io.LimitReader` : la lecture s'arrête
silencieusement à 1 Mio — le JSON est **tronqué**, le décodeur rend `unexpected EOF` (→ 400 « JSON
invalide », trompeur), ou pire, pour un format ligne à ligne, les données sont **acceptées tronquées**
sans aucune erreur. `MaxBytesReader` signale le dépassement **comme tel**, et indique au serveur de
fermer la connexion (le reste du corps n'est pas lu).

## E5 — Le `Content-Type` deviné

- JSON : `text/plain; charset=utf-8` — **faux** : un client strict refusera de le décoder, ou un
  navigateur l'affichera comme texte.
- HTML : `text/html; charset=utf-8` — juste.
- PNG : `image/png` — juste (signature binaire reconnue).

`DetectContentType` reconnaît une quarantaine de signatures, pas le JSON. D'où la règle : toujours fixer
`Content-Type` explicitement, et ajouter `X-Content-Type-Options: nosniff` (leçon 9) pour que le
navigateur ne devine pas non plus.

## Exercice intermédiaire — Export CSV en streaming

```go
func export(w http.ResponseWriter, r *http.Request) {
	from, err1 := time.Parse("2006-01-02", r.URL.Query().Get("from"))
	to, err2 := time.Parse("2006-01-02", r.URL.Query().Get("to"))
	switch {
	case err1 != nil || err2 != nil:
		http.Error(w, "from et to au format AAAA-MM-JJ", http.StatusBadRequest)
		return
	case !from.Before(to):
		http.Error(w, "from doit précéder to", http.StatusBadRequest)
		return
	case to.Sub(from) > 31*24*time.Hour:
		http.Error(w, "période limitée à 31 jours", http.StatusBadRequest)
		return
	}

	w.Header().Set("Content-Type", "text/csv; charset=utf-8")
	w.Header().Set("Content-Disposition", `attachment; filename="ventes.csv"`)
	w.Header().Set("Trailer", "X-Export-Status")      // annoncé AVANT d'écrire le corps
	rc := http.NewResponseController(w)
	cw := csv.NewWriter(w)
	cw.Write([]string{"horodatage", "montant"})
	n := 0
	for t := from; t.Before(to); t = t.Add(time.Minute) {
		if err := r.Context().Err(); err != nil {
			return                                         // client parti
		}
		cw.Write([]string{t.Format(time.RFC3339), strconv.Itoa(saleAt(t))})
		n++
		if n%1000 == 0 {
			cw.Flush()
			if err := cw.Error(); err != nil {
				w.Header().Set("X-Export-Status", "incomplet")
				return
			}
			rc.Flush()
		}
	}
	cw.Flush()
	w.Header().Set("X-Export-Status", "complet")         // envoyé APRÈS le corps, comme trailer
}
```

- Mémoire constante : chaque ligne part dans le flux ; rien n'est accumulé.
- **Statut impossible à changer en cours de route** (le 200 est parti avec les premiers octets). Deux
  solutions : un **trailer** HTTP (en-tête envoyé **après** le corps, déclaré à l'avance dans `Trailer`),
  que les clients HTTP savent lire ; ou une ligne finale de contrôle (`#fin,43200 lignes`) que le client
  vérifie. Le trailer est plus propre mais moins connu des clients non techniques.
- Test d'annulation : `req = req.WithContext(ctx)` avec un contexte annulé après quelques écritures
  (un `Recorder` qui annule dans son `Write`), puis vérifier qu'on n'a pas tout exporté.

## Défi — Requêtes conditionnelles

```go
func article(w http.ResponseWriter, r *http.Request) {
	body := loadArticle(r.PathValue("id"))
	sum := sha256.Sum256(body)
	etag := `"` + hex.EncodeToString(sum[:8]) + `"`
	w.Header().Set("ETag", etag)
	w.Header().Set("Cache-Control", "max-age=60")
	if r.Header.Get("If-None-Match") == etag {
		w.WriteHeader(http.StatusNotModified)            // pas de corps
		return
	}
	w.Header().Set("Content-Type", "text/html; charset=utf-8")
	w.Write(body)
}
```

`http.ServeContent(w, r, name, modtime, bytes.NewReader(body))` fait **tout** cela et plus : il gère
`If-None-Match` (avec un `ETag` fixé par l'appelant), `If-Modified-Since` (avec `modtime`), les requêtes
**partielles** `Range` (reprise de téléchargement, 206), `HEAD`, et déduit `Content-Type` de
l'extension de `name`. La version à la main gère mal les listes d'ETags (`If-None-Match: "a", "b"`) et
les ETags faibles (`W/"…"`) — raison de plus de réutiliser la bibliothèque standard.

## Quiz

1. `r.PathValue("id")`, `r.URL.Query().Get("k")`, `r.Header.Get("Nom")`.
2. Il rend une erreur typée au dépassement (→ 413) au lieu de tronquer silencieusement, et fait fermer
   la connexion.
3. Non : le serveur le ferme à la fin du handler.
4. En-têtes, puis `WriteHeader`, puis corps.
5. Un statut 200 est envoyé implicitement.
6. Il est ignoré, avec un avertissement dans le journal du serveur.
7. 400 : requête mal formée ; 422 : bien formée mais invalide selon les règles. 401 : non authentifié ;
   403 : authentifié mais non autorisé.
8. Sinon le serveur le devine à partir du contenu, et se trompe (JSON en `text/plain`).
9. `http.NewResponseController(w).Flush()` après chaque morceau.
10. Sinon le handler continue et écrit une seconde réponse incohérente.
