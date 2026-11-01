# Leçon 4 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Lire la vue HTML

Avec la table de la leçon 2 (version corrigée de `Parse`), la seule ligne typiquement rouge est
le `return 0, err` qui suit `strconv.Atoi(s[:i])`. Peut-on l'atteindre ? `s[:i]` ne contient
que des chiffres et au moins un… mais un nombre **trop grand** (`"99999999999999999999h"`)
fait échouer `Atoi` avec `strconv.ErrRange`. Un cas à ajouter — et une découverte : la fonction
rend alors l'erreur **sans contexte** (pas de `%w` avec la chaîne fautive). La ligne rouge a
révélé un défaut de qualité, pas seulement un trou de test. C'est l'usage de l'outil.

Autre défaut que la couverture **ne** montre **pas** : `"1000000000h"` ne déborde pas
`Atoi` mais déborde `time.Duration(n) * d` (silencieusement, N1 L3). Aucune ligne n'est rouge ;
il faut **penser** au cas.

## E2 — 100 % et faux

```go
func Average(xs []float64) float64 {
	var s float64
	for _, x := range xs {
		s += x
	}
	return s / float64(len(xs))   // NaN pour une liste vide
}

func TestAverage(t *testing.T) {
	if got := Average([]float64{2, 4}); got != 3 {
		t.Errorf("…")
	}
}
```

100 % de couverture, et `Average(nil)` rend `NaN` (0/0 en flottant, N4 L3) sans que personne
ne le voie. Le cas `{"vide", nil, …}` révèle le bug ; le pourcentage, lui, **reste à 100 %** — il
n'a jamais rien su de ce cas.

## E3 — Couverture par condition

Deux cas suffisent pour 100 % de lignes (`ok(true, true)` et `ok(false, false)`) ; il en faut
quatre pour toutes les combinaisons. Et avec `ok(false, false)`, `b` n'est **jamais évalué**
(court-circuit). `-cover` mesure des **blocs** exécutés, pas des conditions : c'est une mesure
grossière, utile pour trouver des trous, pas pour certifier.

## E4 — Golden file instable

Le golden file contient la date de génération : il change chaque jour, le test échoue chaque
lendemain. Correction : passer l'instant de génération en paramètre (ou dans une struct
d'options), comme `now`. Le test passe un instant fixe ; `main` passe `time.Now()`. L'en-tête
reste, la sortie redevient déterministe. C'est encore une **dépendance cachée** (leçon 3).

## E5 — `-update` aveugle

Oui : `-update` réécrit le golden file **avec la sortie boguée**, et le test passe. Ce qui devait
arrêter le bug : relire `git diff testdata/` avant de commiter — on y aurait vu les colonnes
inversées — et, en équipe, la revue de la *pull request*, où le diff du golden file est visible.
Le golden file déplace la vérification du **code de test** vers le **diff** : si le diff n'est
pas lu, il n'y a plus de vérification du tout.

## Exercice intermédiaire — Golden files pour `todo list`

Structure attendue :

```
internal/cli/testdata/
├── list-vide.tasks.json
├── list-vide.golden
├── list-retards.tasks.json
├── list-retards.golden
├── list-all.golden          ← même entrée que list-retards, avec --all
…
```

```go
func TestList(t *testing.T) {
	paris, err := time.LoadLocation("Europe/Paris")
	if err != nil {
		t.Fatal(err)
	}
	now := time.Date(2026, 11, 10, 9, 0, 0, 0, paris)

	tests := []struct {
		name  string
		input string
		args  []string
	}{
		{"list-vide", "list-vide", nil},
		{"list-retards", "list-retards", nil},
		{"list-all", "list-retards", []string{"--all"}},
		{"list-json", "list-retards", []string{"--json"}},
		…
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			var out bytes.Buffer
			env := Env{Now: func() time.Time { return now }, Loc: paris,
				File: filepath.Join("testdata", tt.input+".tasks.json")}
			if code := Run(append([]string{"list"}, tt.args...), env, &out, io.Discard); code != 0 {
				t.Fatalf("code = %d", code)
			}
			compareGolden(t, tt.name, out.Bytes())
		})
	}
}
```

- **Le fuseau** fait partie de l'environnement injecté (`Loc`), pas lu dans `time.Local` : c'est
  ce qui rend la sortie identique avec `TZ=America/New_York`. Si le test passe en local et échoue
  avec un autre `TZ`, il reste un `time.Local` caché quelque part — souvent dans un `Format`
  appelé sur un `time.Time` non converti (`t.In(loc)` oublié).
- **Le fichier de tâches en entrée** est lu en **lecture seule** depuis `testdata` : un test ne
  modifie jamais `testdata` (sauf `-update`). Si la commande testée écrit (`add`, `done`), copier
  l'entrée dans `t.TempDir()` d'abord.
- `compareGolden` est un utilitaire avec `t.Helper()`, partagé par tous les tests du paquet.
- Typiquement sous 50 % après coup : la gestion des erreurs d'utilisation (drapeaux invalides) et
  `import`. Ce sont des cas pour une **table** d'erreurs, pas pour des golden files.

## Défi — Couverture d'intégration

```bash
go build -cover -o wordstat-cov .
mkdir -p /tmp/cov
GOCOVERDIR=/tmp/cov ./wordstat-cov testdata/a.txt
GOCOVERDIR=/tmp/cov ./wordstat-cov -top 3 testdata/a.txt testdata/b.txt
GOCOVERDIR=/tmp/cov ./wordstat-cov absent.txt
GOCOVERDIR=/tmp/cov ./wordstat-cov -top abc
echo "un deux deux" | GOCOVERDIR=/tmp/cov ./wordstat-cov
go tool covdata percent -i=/tmp/cov
go tool covdata textfmt -i=/tmp/cov -o integ.out
go tool cover -html=integ.out
```

Ce que l'on observe en général : la couverture d'intégration couvre bien `main`, l'analyse des
drapeaux et les chemins « heureux » — que les tests unitaires négligent souvent — mais mal les
cas limites du calcul. Les tests unitaires font l'inverse. Les lignes couvertes **uniquement**
par l'intégration signalent du code « de colle » non testé unitairement ; celles couvertes
uniquement par les tests unitaires, des cas qu'aucun scénario réel ne déclenche. Les deux mesures
sont complémentaires, et on peut les **fusionner** (`go tool covdata merge`).

## Quiz

1. `go test -coverprofile=cover.out` puis `go tool cover -html=cover.out`.
2. Non : elle a été exécutée, rien ne dit qu'un résultat a été vérifié.
3. Non : elle mesure des blocs de code exécutés.
4. Il pousse à écrire des tests sans valeur pour atteindre un chiffre, et ne garantit rien.
5. Il mesure la couverture de tous les paquets désignés par les tests de chaque paquet.
6. L'outil `go` l'ignore (jamais compilé) ; c'est l'emplacement conventionnel des données de test.
7. Le répertoire du paquet testé.
8. Un fichier de référence contenant la sortie attendue ; pour les sorties volumineuses.
9. Relire le diff des fichiers régénérés (`git diff testdata/`).
10. L'heure courante, l'ordre des maps, les chemins absolus, l'aléatoire, les fins de ligne.
