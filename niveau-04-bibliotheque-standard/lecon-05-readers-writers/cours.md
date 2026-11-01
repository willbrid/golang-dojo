# Leçon 5 — Readers et Writers

> `io.Reader` et `io.Writer` sont deux interfaces d'**une** méthode chacune (niveau 2, leçon 4).
> Presque tout ce qui produit ou consomme des octets en Go les implémente : fichiers, sockets,
> requêtes HTTP, compresseurs, chiffreurs, hachages, buffers en mémoire. Les maîtriser, c'est
> pouvoir brancher n'importe quoi sur n'importe quoi — et c'est la condition pour comprendre
> JSON, fichiers et HTTP dans les leçons suivantes.

## Objectifs

1. Connaître le **contrat** exact de `Read` et de `Write`, avec ses cas surprenants.
2. Copier, limiter, dupliquer et combiner des flux avec le paquet `io`.
3. Bufferiser avec `bufio`, et savoir quand il faut appeler `Flush`.
4. Écrire ses propres `Reader`/`Writer` décorateurs.

## Explication

### Le contrat de `Read`

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

`Read` remplit **au plus** `len(p)` octets de `p` et rend combien il en a mis. Le contrat
complet, à connaître mot pour mot :

1. `Read` peut rendre **moins** que `len(p)` octets **sans** que ce soit la fin : une socket
   rend ce qui est arrivé. Ne jamais supposer qu'un `Read` remplit le buffer.
2. À la fin du flux, `Read` rend `io.EOF`. **EOF n'est pas une erreur** : c'est le signal
   normal de fin.
3. `Read` peut rendre `n > 0` **et** une erreur (y compris `io.EOF`) en même temps. Il faut
   **traiter les `n` octets d'abord**, puis regarder l'erreur.

D'où la boucle canonique — que l'on n'écrit presque jamais soi-même, parce que `io` le fait :

```go
buf := make([]byte, 4096)
for {
	n, err := r.Read(buf)
	process(buf[:n])          // D'ABORD les octets, même si err != nil
	if err == io.EOF {
		break
	}
	if err != nil {
		return err
	}
}
```

### Le contrat de `Write`

```go
type Writer interface {
	Write(p []byte) (n int, err error)
}
```

`Write` doit écrire **tous** les octets ou rendre une erreur : `n < len(p)` implique
`err != nil`. Et il ne doit **pas** modifier ni conserver `p` — l'appelant peut réutiliser son
buffer aussitôt.

Conséquence pratique : **toujours vérifier l'erreur de `Write`**, et surtout celle du dernier
`Write`/`Flush`/`Close` — c'est souvent là qu'un disque plein se révèle.

### Les outils du paquet `io`

| Fonction | Rôle |
|---|---|
| `io.Copy(dst, src)` | copie jusqu'à EOF, rend le nombre d'octets ; EOF n'est **pas** rendu comme erreur |
| `io.CopyN(dst, src, n)` | copie exactement `n` octets |
| `io.ReadAll(r)` | lit tout en mémoire — **à éviter** sur des données non bornées |
| `io.ReadFull(r, buf)` | remplit **tout** `buf`, ou erreur (`io.ErrUnexpectedEOF`) |
| `io.LimitReader(r, n)` | un `Reader` qui s'arrête après `n` octets |
| `io.TeeReader(r, w)` | ce qui est lu de `r` est **aussi** écrit dans `w` |
| `io.MultiWriter(w1, w2…)` | un `Writer` qui écrit dans tous |
| `io.MultiReader(r1, r2…)` | un `Reader` qui enchaîne les flux |
| `io.Pipe()` | relie un `Writer` à un `Reader` (entre deux goroutines, niveau 6) |
| `io.Discard` | un `Writer` qui jette tout |

`io.ReadAll` est la fonction la plus dangereuse de la liste : sur un corps de requête HTTP ou
un fichier fourni par un utilisateur, elle laisse un attaquant **remplir la mémoire**. Toujours
la combiner avec `io.LimitReader` quand la source n'est pas maîtrisée.

### Composer : tout est un décorateur

Les fonctions de `io` (et de dizaines d'autres paquets) **prennent** un `Reader`/`Writer` et
en **rendent** un autre. On compose donc comme des tuyaux Unix :

```go
f, _ := os.Open("data.csv.gz")
gz, _ := gzip.NewReader(f)             // décompresse
h := sha256.New()                      // un hash.Hash est un io.Writer
r := io.TeeReader(io.LimitReader(gz, 100<<20), h)  // ≤ 100 Mio, hachés au passage
io.Copy(os.Stdout, r)
fmt.Printf("%x\n", h.Sum(nil))
```

Un seul passage sur les données, mémoire constante, quatre comportements combinés. Aucun des
paquets ne connaît les autres : ils ne partagent que les deux interfaces.

### Les buffers en mémoire

| Type | Rôle |
|---|---|
| `strings.NewReader(s)` | un `Reader` sur une chaîne (tests, constantes) |
| `bytes.NewReader(b)` | idem sur un `[]byte` |
| `bytes.Buffer` | à la fois `Reader` **et** `Writer` : on écrit dedans, on relit ensuite |
| `strings.Builder` | `Writer` seulement, pour produire une `string` |

Leur zéro-valeur est prête à l'emploi : `var buf bytes.Buffer`. Et `*bytes.Buffer` (pas
`bytes.Buffer`) implémente `io.Writer` : on passe `&buf`.

### `bufio` : bufferiser

Chaque `Read` ou `Write` sur un fichier ou une socket est un **appel système**, coûteux
(quelques centaines de nanosecondes à quelques microsecondes). Écrire un fichier octet par
octet, c'est un appel système par octet.

`bufio.Writer` accumule les écritures en mémoire (4 Kio par défaut) et ne fait un appel
système que quand le buffer est plein. Le prix : **il faut appeler `Flush`** à la fin, sinon
les derniers octets restent en mémoire et sont perdus.

```go
w := bufio.NewWriter(f)
defer w.Flush()   // ⚠️ ignore l'erreur : acceptable pour un essai, pas en production
```

En production, on vérifie l'erreur du `Flush` (voir l'exemple).

`bufio.Reader` fait l'inverse en lecture et ajoute `ReadString('\n')`, `ReadRune`, `Peek`.
`bufio.Scanner` (déjà vu) découpe en lignes, mots ou jetons personnalisés, avec sa limite de
64 Kio par jeton (`sc.Buffer` pour l'augmenter).

### `Closer` et les interfaces composées

`io.Closer` (`Close() error`) se combine aux autres : `io.ReadCloser`, `io.WriteCloser`,
`io.ReadWriteCloser`. Règle : **celui qui ouvre ferme**. Une fonction qui reçoit un
`io.Reader` ne le ferme pas — elle ne sait même pas s'il peut l'être.

### Interfaces optionnelles

`io.Copy` vérifie si `src` implémente `io.WriterTo` ou si `dst` implémente `io.ReaderFrom`,
et délègue alors la copie. C'est le patron des **interfaces optionnelles** (défi de N2 L4) :
copier un fichier vers une socket peut ainsi utiliser `sendfile` côté noyau, sans passer les
octets par le programme. On en profite gratuitement en utilisant `io.Copy` plutôt qu'une
boucle à la main.

## Exemple

```go
package main

import (
	"bufio"
	"bytes"
	"crypto/sha256"
	"errors"
	"fmt"
	"io"
	"os"
	"strings"
)

// countingWriter est un décorateur : il compte les octets et les lignes
// qui le traversent, puis délègue au Writer enveloppé.
type countingWriter struct {
	w            io.Writer
	bytes, lines int
}

func (c *countingWriter) Write(p []byte) (int, error) {
	n, err := c.w.Write(p)
	c.bytes += n
	c.lines += bytes.Count(p[:n], []byte("\n"))
	return n, err
}

// upperReader transforme en majuscules ASCII ce qu'il lit.
type upperReader struct{ r io.Reader }

func (u upperReader) Read(p []byte) (int, error) {
	n, err := u.r.Read(p)
	for i := range p[:n] {
		if 'a' <= p[i] && p[i] <= 'z' {
			p[i] -= 'a' - 'A'
		}
	}
	return n, err // n octets valides ET éventuellement io.EOF : on transmet les deux
}

// process copie src vers dst en majuscules, au plus max octets,
// et rend l'empreinte SHA-256 de ce qui a été LU.
func process(dst io.Writer, src io.Reader, max int64) ([]byte, error) {
	h := sha256.New()
	r := upperReader{io.TeeReader(io.LimitReader(src, max), h)}

	bw := bufio.NewWriter(dst)
	if _, err := io.Copy(bw, r); err != nil {
		return nil, fmt.Errorf("copie : %w", err)
	}
	if err := bw.Flush(); err != nil { // l'erreur d'écriture finale se révèle souvent ICI
		return nil, fmt.Errorf("vidage : %w", err)
	}
	return h.Sum(nil), nil
}

// failAfter simule un disque qui se remplit après n octets.
type failAfter struct{ n int }

func (f *failAfter) Write(p []byte) (int, error) {
	if len(p) > f.n {
		w := f.n
		f.n = 0
		return w, errors.New("plus de place sur le périphérique")
	}
	f.n -= len(p)
	return len(p), nil
}

func main() {
	text := "bonjour\nle monde\nde go\n"

	cw := &countingWriter{w: os.Stdout}
	sum, err := process(cw, strings.NewReader(text), 1<<20)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Printf("%d octets, %d lignes, sha256=%x…\n", cw.bytes, cw.lines, sum[:4])

	// Limite : seuls 10 octets passent.
	var buf bytes.Buffer
	process(&buf, strings.NewReader(text), 10)
	fmt.Printf("%q\n", buf.String())

	// Disque plein : l'erreur ne sort qu'au Flush, parce que bufio a tout gardé en mémoire.
	_, err = process(&failAfter{n: 5}, strings.NewReader(text), 1<<20)
	fmt.Println("erreur :", err)
}
```

Sortie :

```
BONJOUR
LE MONDE
DE GO
23 octets, 3 lignes, sha256=0ae119f9…
"BONJOUR\nLE"
erreur : vidage : plus de place sur le périphérique
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `countingWriter` | Un `Writer` qui en enveloppe un autre : le patron décorateur de N2 L5, appliqué aux flux. Il compte `p[:n]`, pas `p` : seulement ce qui a vraiment été écrit. |
| `upperReader.Read` | Transforme `p[:n]` **avant** de regarder l'erreur, et rend `n` et `err` ensemble : le contrat de `Read` respecté. |
| Ordre des décorateurs | `LimitReader` → `TeeReader` → `upperReader` : le hash porte sur le texte **original** limité. Inverser `Tee` et `upper` hacherait le texte en majuscules. |
| `io.Copy(bw, r)` | Pas de boucle à la main ; EOF n'est pas une erreur pour `Copy`. |
| `bw.Flush()` vérifié | Avec `failAfter`, `Copy` **réussit** (tout tient dans le buffer de 4 Kio) ; c'est `Flush` qui échoue. Un `defer bw.Flush()` aurait avalé l'erreur. |
| `process(&buf, …)` | `&buf` : c'est `*bytes.Buffer` qui implémente `io.Writer`. |
| `"BONJOUR\nLE"` | 10 octets exactement : `LimitReader` coupe au milieu d'une ligne, sans état d'âme. |

## Erreurs fréquentes

1. **Supposer qu'un `Read` remplit le buffer.**
2. **Traiter EOF comme une erreur**, ou **ignorer les `n` octets** rendus avec une erreur.
3. **`io.ReadAll` sur une source non bornée.**
4. **Oublier `Flush`**, ou le mettre en `defer` sans vérifier son erreur.
5. **Fermer un `Reader` reçu en paramètre.**
6. **Passer `buf` au lieu de `&buf`** pour un `bytes.Buffer`.
7. **Écrire une boucle `Read`/`Write` à la main** là où `io.Copy` suffit (et se prive des
   optimisations `WriterTo`/`ReaderFrom`).
8. **Conserver le slice `p`** dans un `Write` personnalisé : l'appelant va le réutiliser.

## Bonnes pratiques Go

- Accepter `io.Reader`/`io.Writer` plutôt que des chemins de fichier ou des `[]byte`.
- `io.Copy` pour copier, `io.LimitReader` pour borner, `TeeReader`/`MultiWriter` pour dupliquer.
- `bufio` pour les écritures nombreuses et petites ; `Flush` explicite et vérifié.
- `strings.NewReader` et `bytes.Buffer` pour tester sans fichier.
- Celui qui ouvre ferme.
- Écrire un décorateur plutôt que de modifier le code qui lit ou écrit.

## Ce que je dois retenir

- `Read` : peut rendre moins que demandé ; `io.EOF` est la fin normale ; traiter `n` **avant** `err`.
- `Write` : tout ou erreur ; ne pas garder `p`.
- `io.Copy`, `LimitReader`, `TeeReader`, `MultiWriter`, `ReadFull`.
- `io.ReadAll` uniquement sur des sources bornées.
- `bufio.Writer` → `Flush`, et son erreur.
- Les flux se **composent** : chaque paquet ne connaît que les deux interfaces.

---

🧵 **Fil rouge :** un `bytes.Buffer` contient un slice ; un `Write` qui garde `p` crée un partage invisible. Une ligne de plus pour *[Copié ou partagé ?](../../ressources/copie-ou-partage.md)*.

➡️ [Exercices](exercices.md)
