# Leçon 1 — Exercices

Code dans `mes-solutions/niveau-05/lecon-01/`. Pour **chaque** test écrit : le faire échouer une
fois en cassant le code, et recopier la sortie d'échec dans un commentaire.

## Exercices faciles

**E1 — Premier test.**
Reprendre `Slug` (N4 L1) dans un paquet `slug` et écrire `TestSlug` avec trois vérifications.
Lancer `go test`, `go test -v`, puis `go test -v -run 'Slug'`. Relancer `go test` sans rien
changer : que signifie `(cached)` ? Comment forcer l'exécution ?

**E2 — `Error` ou `Fatal`.**
Écrire un test qui charge une configuration depuis un fichier temporaire puis vérifie trois
champs. Le faire échouer de deux façons : (a) fichier absent, (b) deux champs faux. Pour chaque
cas, quelle méthode (`Error`/`Fatal`) as-tu utilisée à chaque endroit, et combien de lignes
d'échec obtiens-tu ?

**E3 — `t.Helper`.**
Écrire une fonction utilitaire `assertEqual(t *testing.T, got, want int)` **sans** `t.Helper()`.
Provoquer un échec : quelle ligne est rapportée ? Ajouter `t.Helper()` : laquelle maintenant ?
*(Pour aller plus loin : rendre `assertEqual` générique avec `comparable`.)*

**E4 — Test externe.**
Pour le paquet `stats` de N3 L2, écrire les tests en `package stats_test`. Un des tests a besoin
d'une fonction non exportée : comment faire sans passer tout le fichier en `package stats` ?
*(Indice : un fichier `export_test.go` en `package stats`.)*

**E5 — Cleanup.**
Écrire `newTempStore(t *testing.T) *Store` qui crée un `Store` (N4 L7, défi) dans un répertoire
temporaire et enregistre un `t.Cleanup` qui affiche `t.Log("nettoyage")`. Appeler la fonction
deux fois dans un test. Dans quel ordre les nettoyages s'exécutent-ils ? Où le `t.Log`
apparaît-il ?

---

## Exercice intermédiaire — Tester `wordstat`

Écrire la suite de tests du **projet 1** (`wordstat`).

**Contraintes :**
1. Si le code du projet 1 n'est pas testable (lecture de `os.Stdin` ou `os.Args` dans les
   fonctions de calcul, affichage dans les fonctions de comptage), le **refactoriser d'abord**,
   en notant chaque changement et sa raison.
2. Couvrir au minimum les six tests listés dans l'énoncé du projet 1, et les difficultés D1 à D6.
3. Tester `run(args []string, stdin io.Reader, stdout, stderr io.Writer) int` de bout en bout :
   code de sortie, contenu de `stdout` et de `stderr`, sans lancer de processus.
4. Aucun fichier créé ailleurs que dans `t.TempDir()`.
5. `go test -race -shuffle=on -count=3 ./...` doit passer.

---

## Défi — Le test qui ment

Voici un test :

```go
func TestParsePort(t *testing.T) {
	for _, s := range []string{"80", "8080", "65535"} {
		p, err := ParsePort(s)
		if err != nil {
			t.Errorf("ParsePort(%q) : %v", s, err)
		}
		_ = p
	}
}
```

1. Écrire **trois** implémentations **fausses** de `ParsePort(string) (int, error)` qui passent ce
   test. *(Une seule ligne suffit pour la première.)*
2. Réécrire le test pour qu'aucune des trois ne passe.
3. Quels cas limites manquaient ? Les lister avant de regarder la leçon 2.
4. Formuler en une phrase le critère d'un test qui « ne ment pas ».

---

## Quiz

1. Quelles sont les deux conventions de nommage qui font qu'une fonction est un test ?
2. Pourquoi Go n'a-t-il pas de fonctions d'assertion ?
3. Dans quel ordre écrire « obtenu » et « attendu » dans un message ?
4. Différence entre `t.Error` et `t.Fatal` ? Quand utiliser chacun ?
5. Que fait `t.Helper()` ?
6. Différence entre `t.Cleanup` et `defer` ?
7. Différence entre `package foo` et `package foo_test` dans un fichier de test ?
8. Que signifie `(cached)` et comment l'éviter ?
9. Comment lancer un seul sous-test ?
10. À quoi sert `-shuffle=on` ?
