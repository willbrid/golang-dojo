# Leçon 5 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Le hash de l'exemple

`printf 'bonjour\nle monde\nde go\n' | sha256sum` commence par `0ae119f9` : identique au
programme, qui hache le texte **original**.

Si l'on place `upperReader` **sous** le `TeeReader` (`TeeReader(upperReader{…}, h)`), le hash
porte sur le texte en majuscules, et c'est `printf 'BONJOUR\nLE MONDE\nDE GO\n' | sha256sum`
(`96e7004a…`) qui correspond. L'ordre des décorateurs **est** le programme.

## E2 — Un `Read` partiel

```go
type oneByteReader struct{ r io.Reader }

func (o oneByteReader) Read(p []byte) (int, error) {
	if len(p) == 0 {
		return 0, nil
	}
	return o.r.Read(p[:1])
}
```

| Lecture | Résultat |
|---|---|
| un seul `Read` sur 16 octets | `"b"` |
| `io.ReadFull(r, buf[:7])` | `"bonjour"` |
| `io.ReadAll(r)` | `"bonjour"` |

Un `Read` unique rend ce qu'il a **sous la main** — un octet ici, ce qui est arrivé du réseau
dans la vraie vie. Code qui lit un protocole binaire sur une socket : **toujours** `io.ReadFull`
pour lire un en-tête de taille fixe. Un `conn.Read(header)` isolé marche en local (tout arrive
d'un coup) et casse en production, sur un réseau lent.

## E3 — `wc` en une copie

```go
func main() {
	var lines lineCounter
	var size byteCounter
	h := sha256.New()
	if _, err := io.Copy(io.MultiWriter(&lines, &size, h), os.Stdin); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Printf("%d %d %x\n", size, lines, h.Sum(nil))
}

type byteCounter int

func (c *byteCounter) Write(p []byte) (int, error) { *c += byteCounter(len(p)); return len(p), nil }

type lineCounter int

func (c *lineCounter) Write(p []byte) (int, error) {
	*c += lineCounter(bytes.Count(p, []byte{'\n'}))
	return len(p), nil
}
```

Un type défini sur `int` peut avoir des méthodes (N2 L3) : pas besoin de struct. Le compte de
lignes correspond à `wc -l`, qui compte les `\n` (un fichier sans saut final « perd » une
ligne — c'est le comportement POSIX).

## E4 — Le `Flush` oublié

Mesuré :

| Écrit | Taille du fichier |
|---|---|
| 10 lignes de 80 octets (800 o) | **0** |
| 10 000 lignes (800 000 o) | **798 720** |

800 octets tiennent dans le buffer de 4 096 : rien n'a été écrit. Pour 800 000 octets, le
buffer s'est vidé chaque fois qu'il était plein : 195 × 4 096 = 798 720 octets écrits, et les
**1 280** derniers octets (800 000 mod 4 096) sont restés en mémoire, perdus. Le fichier semble
complet à l'œil… et il lui manque 16 lignes.

## E5 — `ReadAll` borné

```go
var ErrTooLarge = errors.New("données trop volumineuses")

func ReadAtMost(r io.Reader, max int64) ([]byte, error) {
	b, err := io.ReadAll(io.LimitReader(r, max+1))
	if err != nil {
		return nil, err
	}
	if int64(len(b)) > max {
		return nil, fmt.Errorf("plus de %d octets : %w", max, ErrTooLarge)
	}
	return b, nil
}

```

Lire `max + 1` octets est la seule façon de distinguer « exactement `max` » de « plus que
`max` » : avec `LimitReader(r, max)` seul, les deux cas rendent `max` octets. `net/http` offre
l'équivalent pour les corps de requête : `http.MaxBytesReader` (niveau 7).

## Exercice intermédiaire — `linecat`

```go
type LineNumberWriter struct {
	w           io.Writer
	line        int
	atLineStart bool
}

func NewLineNumberWriter(w io.Writer) *LineNumberWriter {
	return &LineNumberWriter{w: w, atLineStart: true}
}

func (l *LineNumberWriter) Write(p []byte) (int, error) {
	written := 0
	for len(p) > 0 {
		if l.atLineStart {
			l.line++
			if _, err := fmt.Fprintf(l.w, "%6d\t", l.line); err != nil {
				return written, err
			}
			l.atLineStart = false
		}
		chunk := p
		if i := bytes.IndexByte(p, '\n'); i >= 0 {
			chunk = p[:i+1]
			l.atLineStart = true
		}
		n, err := l.w.Write(chunk)
		written += n
		if err != nil {
			return written, err
		}
		p = p[len(chunk):]
	}
	return written, nil
}

```

Points de correction :

- **L'état `atLineStart`** traverse les appels : c'est ce qui permet de recevoir les lignes
  découpées n'importe comment. Un `Writer` qui suppose recevoir des lignes entières casse dès
  qu'il est placé derrière un `bufio.Writer` ou `io.Copy`.
- **Le `n` rendu** compte les octets **de `p`** consommés. L'appelant (par exemple `io.Copy`)
  compare `n` à `len(p)` : rendre « octets écrits préfixes compris » ferait croire à une
  écriture **plus longue** que demandée, et `io.Copy` rendrait `io.ErrShortWrite` ou
  compterait faux. Le contrat porte sur `p`, pas sur ce qui sort en dessous.
- **Erreur en cours de route** : on rend `written` (ce qui a été effectivement transmis de
  `p`) et l'erreur.
- Vérification : `diff <(cat -n README.md) <(go run . < README.md)` ne doit rien afficher, y
  compris avec `oneByteReader` en entrée — c'est le test de la contrainte 1.

## Défi — `CountingReader` et progression

```go
type CountingReader struct {
	r    io.Reader
	n    int64
	last time.Time
	on   func(n int64)
}

func (c *CountingReader) Read(p []byte) (int, error) {
	n, err := c.r.Read(p)
	c.n += int64(n)
	if now := time.Now(); now.Sub(c.last) >= 100*time.Millisecond || err == io.EOF {
		c.last = now
		c.on(c.n)
	}
	return n, err
}
```

1. **10 fois par seconde sans goroutine** : on ne met à jour la barre que si 100 ms se sont
   écoulées depuis la dernière mise à jour, testé **à chaque `Read`**. `time.Now` coûte quelques
   dizaines de nanosecondes : négligeable devant un `Read` de 32 Kio. Afficher avec `\r` en début
   de ligne pour réécrire la même ligne du terminal.
2. **L'optimisation est perdue.** `io.Copy` cherche `WriterTo` sur la source : un `*os.File`
   en a un, ton `CountingReader` non. La copie retombe dans la boucle générique avec un buffer
   de 32 Kio en mémoire utilisateur, au lieu d'un éventuel `sendfile`/`copy_file_range` noyau.
   Un décorateur n'est pas gratuit : il **masque** les interfaces optionnelles de ce qu'il
   enveloppe. (Les décorateurs de la stdlib exposent parfois une méthode `Unwrap` ou
   réimplémentent `WriterTo` pour cette raison.)
3. Dans `io.copyBuffer` : deux cas particuliers avant la boucle — la source implémente
   `WriterTo`, ou la destination implémente `ReaderFrom`. Puis la boucle générique, qui gère
   aussi `ErrShortWrite` et le cas `n < 0` d'un `Writer` défectueux.

## Quiz

1. Oui. Rien n'oblige `Read` à remplir le buffer.
2. Non : c'est le signal normal de fin de flux.
3. Traiter les `n` octets d'abord, puis l'erreur.
4. Qu'une erreur non nil accompagne le retour.
5. Un client peut envoyer un corps arbitrairement gros et épuiser la mémoire du serveur.
6. `TeeReader` : ce qui est lu est aussi écrit ailleurs ; `MultiWriter` : une écriture est
   dupliquée vers plusieurs destinations.
7. Il regroupe les petites écritures en gros appels système ; prix : il faut `Flush`, et une
   erreur d'écriture peut n'apparaître qu'à ce moment.
8. Parce que l'erreur de `Flush` serait ignorée ; or c'est souvent là qu'un disque plein se révèle.
9. Non : celui qui ouvre ferme.
10. Parce que les méthodes `Write` et `Read` de `bytes.Buffer` ont un récepteur pointeur : seul
    `*bytes.Buffer` satisfait `io.Writer`.
