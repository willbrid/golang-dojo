# Leçon 7 — `defer`, `panic` et `recover`

## Objectifs

1. Maîtriser `defer` : ordre d'exécution, moment d'évaluation des arguments, pièges en boucle.
2. Utiliser `defer` avec des **résultats nommés** pour enrichir une erreur ou fermer proprement.
3. Comprendre ce qu'est réellement une `panic` et pourquoi on ne s'en sert presque jamais.
4. Savoir où `recover` est légitime — et où il ne l'est pas.

## Explication

### `defer` : exécuter à la sortie de la fonction

```go
func read(path string) error {
	f, err := os.Open(path)
	if err != nil {
		return err
	}
	defer f.Close()      // exécuté quoi qu'il arrive à la sortie de read
	// … cent lignes, dix return possibles …
	return nil
}
```

`defer` planifie un appel pour la **sortie de la fonction englobante** — pas la fin du bloc,
pas la fin de la boucle. Il s'exécute :
- après un `return`, quel qu'il soit ;
- après une `panic`, pendant le déroulement de la pile ;
- **jamais** après `os.Exit`, qui termine le processus immédiatement.

L'intérêt est de placer la libération **juste après** l'acquisition : on ne peut plus oublier
un chemin de sortie.

### Trois règles à connaître par cœur

**1. Ordre LIFO — dernier planifié, premier exécuté.**
```go
for i := range 3 {
	defer fmt.Print(i)
}
// affiche 210
```
C'est ce qu'on veut : les ressources se libèrent dans l'ordre inverse de leur acquisition.

**2. Les arguments sont évalués **immédiatement**, l'appel est différé.**
```go
x := 1
defer fmt.Println("valeur :", x)   // capture 1 MAINTENANT
x = 2
// affiche « valeur : 1 »
```
Pour capturer la valeur finale, il faut une closure :
```go
defer func() { fmt.Println("valeur :", x) }()   // affiche 2
```
La différence est subtile et compte réellement — c'est le piège n°1 de `defer`.

**3. Le récepteur d'une méthode est évalué immédiatement lui aussi.**
```go
defer mu.Unlock()   // mu évalué maintenant : correct
```

### `defer` en boucle : le piège

```go
// FUITE : les fichiers ne se ferment qu'à la sortie de la FONCTION
for _, path := range paths {
	f, err := os.Open(path)
	if err != nil { continue }
	defer f.Close()          // 10 000 fichiers ouverts simultanément
	process(f)
}
```

Sur dix mille fichiers, on épuise les descripteurs du processus. Deux corrections :

```go
// A : extraire le corps dans une fonction
for _, path := range paths {
	if err := processFile(path); err != nil { … }   // defer à l'intérieur
}

// B : fonction anonyme immédiate
for _, path := range paths {
	func() {
		f, err := os.Open(path)
		if err != nil { return }
		defer f.Close()
		process(f)
	}()
}
```

La version A est préférable : la fonction est nommée, testable, et la boucle reste lisible.

### `defer` + résultats nommés : deux usages majeurs

C'est le seul cas où nommer les résultats est **fonctionnellement** nécessaire, car le
`defer` s'exécute **après** l'affectation du résultat mais **avant** le retour effectif.

**Usage 1 — ne pas perdre l'erreur de `Close` en écriture :**
```go
func write(path string, data []byte) (err error) {
	f, err := os.Create(path)
	if err != nil {
		return fmt.Errorf("création de %s : %w", path, err)
	}
	defer func() {
		if cerr := f.Close(); cerr != nil && err == nil {
			err = fmt.Errorf("fermeture de %s : %w", path, cerr)
		}
	}()
	_, err = f.Write(data)
	return err
}
```

Sur un fichier ouvert en **écriture**, `Close` peut échouer — disque plein, quota dépassé,
erreur de vidage du tampon. Un `defer f.Close()` nu perdrait cette erreur, et le programme
signalerait un succès alors que le fichier est tronqué. C'est un critère « prêt pour la
production ». En **lecture seule**, `defer f.Close()` sans vérification est acceptable.

**Usage 2 — enrichir l'erreur uniformément :**
```go
func doWork(id int) (err error) {
	defer func() {
		if err != nil {
			err = fmt.Errorf("travail %d : %w", id, err)
		}
	}()
	// … dix return err possibles, tous enrichis automatiquement …
}
```

À utiliser avec parcimonie : ce qu'on gagne en concision, on le perd en traçabilité, car
l'enrichissement n'est plus visible au point de retour.

### `panic` : l'arrêt brutal

```go
panic("état impossible")
```

Une `panic` interrompt l'exécution normale, **déroule la pile** en exécutant tous les `defer`
rencontrés, puis termine le programme avec une trace et le code de sortie 2.

Le runtime en déclenche aussi tout seul : déréférencement de `nil`, indice hors bornes,
division entière par zéro, écriture dans une map `nil`, assertion de type ratée.

**`panic` est réservé à ce qui ne peut pas arriver** : un invariant violé, un bug de
programmation, une configuration impossible détectée au démarrage. Une erreur d'entrée/sortie,
une saisie invalide, un réseau coupé sont des situations **normales** du monde réel : elles se
retournent en `error`.

Les rares usages légitimes :
- `MustXxx` : `regexp.MustCompile`, `template.Must` — destinées aux variables de package,
  où une erreur signifie que le binaire est irrémédiablement cassé. Le préfixe `Must` est un
  contrat : **il prévient que ça peut paniquer**.
- L'initialisation au démarrage : une configuration invalide qui rend le service inutilisable.
- Un invariant interne violé qui indique un bug : `panic("unreachable")`.

**Jamais dans du code de bibliothèque, sur une entrée externe.**

### `recover` : rattraper, uniquement dans un `defer`

```go
func safe() (err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("panique rattrapée : %v", r)
		}
	}()
	risky()
	return nil
}
```

`recover` n'a d'effet **que s'il est appelé directement dans une fonction différée**. Ailleurs,
il retourne `nil` et ne fait rien.

Deux usages légitimes, et deux seulement :

1. **La frontière d'un serveur.** Un `panic` dans le traitement d'une requête ne doit pas tuer
   le processus entier. `net/http` fait cela nativement pour chaque requête. Un serveur gRPC ou
   un pool de workers doit le faire explicitement.
2. **La frontière d'une bibliothèque qui utilise `panic` en interne.** Certains analyseurs
   syntaxiques utilisent `panic` pour remonter d'une récursion profonde, et le convertissent en
   `error` à l'entrée publique. Technique valable, mais elle doit rester **invisible** de
   l'extérieur.

**Ce que `recover` ne doit jamais être : un `try/catch`.** Rattraper les paniques pour
« continuer quand même » masque des bugs et laisse le programme dans un état incohérent.

**Point capital, souvent ignoré :** `recover` ne rattrape que les paniques de **sa propre
goroutine**. Une panique dans une goroutine lancée sans protection tue **tout le processus**,
quel que soit le nombre de `recover` ailleurs. C'est la première cause d'arrêt inexpliqué en
production Go. On y reviendra au niveau 6.

Et deux erreurs fatales ne sont **pas** rattrapables : `fatal error: concurrent map writes` et
`fatal error: all goroutines are asleep - deadlock!`. Le runtime refuse volontairement de
laisser continuer.

## Exemple

```go
package main

import (
	"errors"
	"fmt"
	"os"
	"strings"
)

// writeReport écrit un rapport et NE PERD PAS l'erreur de Close.
// Le résultat nommé err est indispensable ici.
func writeReport(path string, lines []string) (err error) {
	f, err := os.Create(path)
	if err != nil {
		return fmt.Errorf("création de %s : %w", path, err)
	}
	defer func() {
		// Ne masque jamais une erreur déjà présente.
		if cerr := f.Close(); cerr != nil && err == nil {
			err = fmt.Errorf("fermeture de %s : %w", path, cerr)
		}
	}()

	for i, line := range lines {
		if _, werr := fmt.Fprintf(f, "%d. %s\n", i+1, line); werr != nil {
			return fmt.Errorf("écriture ligne %d : %w", i+1, werr)
		}
	}
	return nil
}

// processAll montre la bonne façon d'utiliser defer dans une boucle :
// une fonction par itération.
func processAll(paths []string) error {
	var errs []error
	for _, p := range paths {
		if err := processOne(p); err != nil {
			errs = append(errs, err)
		}
	}
	return errors.Join(errs...)
}

func processOne(path string) error {
	f, err := os.Open(path)
	if err != nil {
		return fmt.Errorf("ouverture de %s : %w", path, err)
	}
	defer f.Close() // en LECTURE seule, ignorer l'erreur de Close est acceptable
	// … traitement …
	return nil
}

// mustUpper illustre la convention Must : elle panique, et son nom le dit.
func mustUpper(s string) string {
	if s == "" {
		panic("mustUpper: chaîne vide")
	}
	return strings.ToUpper(s)
}

// safeWorker est une FRONTIÈRE : elle rattrape la panique pour que le
// programme survive à un traitement défaillant. Usage légitime de recover.
func safeWorker(job string, fn func(string) string) (result string, err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("tâche %q : panique rattrapée : %v", job, r)
		}
	}()
	return fn(job), nil
}

func main() {
	// Ordre LIFO
	for i := range 3 {
		defer fmt.Printf("defer %d\n", i) // affiche 2, 1, 0 à la toute fin
	}

	// Évaluation immédiate des arguments
	x := 1
	defer fmt.Println("capturé immédiatement :", x) // 1
	defer func() { fmt.Println("évalué à la sortie :", x) }() // 2
	x = 2

	if err := writeReport("/tmp/rapport.txt", []string{"alpha", "beta"}); err != nil {
		fmt.Fprintln(os.Stderr, err)
	} else {
		fmt.Println("rapport écrit")
	}

	// Frontière de sécurité
	for _, job := range []string{"ok", ""} {
		res, err := safeWorker(job, mustUpper)
		if err != nil {
			fmt.Fprintf(os.Stderr, "%v\n", err)
			continue
		}
		fmt.Println("résultat :", res)
	}
	fmt.Println("le programme continue")
}
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `(err error)` nommé dans `writeReport` | Sans le nom, le `defer` ne pourrait pas modifier la valeur retournée. |
| `cerr != nil && err == nil` | On ne remplace jamais une erreur existante par celle de `Close` : la première est plus informative. |
| `defer f.Close()` dans `processOne` | En lecture seule, l'erreur de `Close` n'apporte rien. Le contraste avec `writeReport` est volontaire. |
| Une fonction par itération | La parade correcte au `defer` en boucle. |
| `mustUpper` | Le préfixe `Must` **est** la documentation : cette fonction panique par contrat. |
| `safeWorker` | `recover` à une **frontière** de traitement, pour qu'un travail défaillant ne tue pas le processus. |
| `defer` de `main` | Affiche 2, 1, 0 — à la toute fin, après tout le reste. |

## Erreurs fréquentes

1. **`defer` dans une boucle** : les ressources s'accumulent jusqu'à la sortie de la fonction.
2. **Croire que les arguments sont évalués à la sortie** : ils le sont à la planification.
3. **`defer f.Close()` sur un fichier en écriture** : l'erreur est perdue, le fichier peut être tronqué.
4. **Oublier de nommer le résultat** quand le `defer` doit le modifier : le `defer` s'exécute mais la modification est ignorée.
5. **`recover` hors d'un `defer`** : retourne `nil`, ne fait rien.
6. **`recover` comme `try/catch`** : masque les bugs.
7. **Croire que `recover` protège les autres goroutines** : il ne protège que la sienne.
8. **`panic` sur une erreur d'entrée/sortie ou une saisie utilisateur.**
9. **`defer` après `os.Exit`** : jamais exécuté.
10. **`defer` avant de vérifier l'erreur d'ouverture** : `defer f.Close()` sur un `f` nil panique.

## Bonnes pratiques Go

- `defer` **immédiatement après** l'acquisition réussie de la ressource — jamais avant le test d'erreur.
- Une fonction par itération plutôt qu'un `defer` en boucle.
- Traiter l'erreur de `Close` en **écriture**, via un résultat nommé.
- `panic` uniquement pour un bug de programmation ; jamais en bibliothèque sur une entrée externe.
- Préfixe `Must` obligatoire pour toute fonction qui panique par contrat.
- `recover` seulement aux **frontières** : requête, tâche, goroutine de worker.
- Protéger explicitement chaque goroutine lancée si une panique y est concevable.
- Ne pas abuser du `defer` d'enrichissement d'erreur : la traçabilité en souffre.

## Ce que je dois retenir

- `defer` s'exécute à la sortie de la **fonction**, en ordre **LIFO**, y compris pendant une panique — mais jamais après `os.Exit`.
- Les **arguments** d'un `defer` sont évalués **immédiatement** ; une closure diffère l'évaluation.
- `defer` dans une boucle accumule les ressources : extraire une fonction.
- Avec un **résultat nommé**, un `defer` peut modifier l'erreur retournée — indispensable pour `Close` en écriture.
- `panic` = bug de programmation, pas condition d'exécution.
- `recover` ne fonctionne que dans un `defer`, et **uniquement pour sa propre goroutine**.

➡️ [Exercices](exercices.md)
