# Leçon 8 — Exercices

Code dans `mes-solutions/niveau-11/lecon-08/`, en partant du paquet `opt` de l'exemple.

## Exercices faciles

**E1 — Amdahl avant d'écrire.**
Profiler `BenchmarkWrite/bufio` (leçon 4). Quelle part du temps revient à `fmt.Fprintf` et à ce qu'il
appelle ? En déduire, par la loi d'Amdahl, le gain maximal qu'on peut espérer en supprimant `fmt`. Le comparer
au gain mesuré avec `AppendRec`. Que représente le temps restant ?

**E2 — Lire aussi.**
Écrire trois lecteurs qui comptent les lignes d'un fichier de 10 000 lignes : `f.Read` d'un octet à la fois,
`bufio.Reader.ReadByte`, et `bufio.Scanner`. Compter les appels à `Read` sur le fichier (un `CountingReader`
sur le modèle de `CountingWriter`) et mesurer les temps.

**E3 — Le bug du pool.**
Écrire `RenderBad(recs) []byte` qui construit la réponse dans un tampon du pool, le **remet au pool**, puis
renvoie `buf.Bytes()`. Écrire un test où 8 goroutines appellent `RenderBad` en boucle avec des données
différentes et vérifient le contenu reçu. Le test échoue-t-il ? Le détecteur de *data races* le
signale-t-il ? Pourquoi ce bug est-il si dangereux ?

**E4 — Le tampon géant.**
Une réponse sur 1 000 fait 4 Mio, les autres 2 Kio. Appeler 100 000 fois `RenderPooled`, avec et sans la
condition `buf.Cap() <= 64<<10`, depuis 8 goroutines. Mesurer `HeapInuse` (`runtime.ReadMemStats`, après un
`runtime.GC()`) pendant la charge, et le temps total. Que garde le pool sans la condition ?

**E5 — Le seuil du parallélisme.**
(a) Trouver la taille à partir de laquelle `SumParallel` bat `Sum` sur ta machine. (b) Remplacer la somme par
un travail de **calcul** : la somme des `sha256.Sum256` de chaque élément converti en 8 octets. Mesurer
séquentiel contre 8 goroutines pour n = 1 000 et n = 100 000. Quel gain, et pourquoi est-il si différent de
celui de la somme ?

---

## Exercice intermédiaire — Optimiser un programme de bout en bout

Un programme `ventes` lit un fichier CSV de 1 000 000 de lignes `date,produit,quantité,prix_unitaire` et
écrit un rapport trié par chiffre d'affaires décroissant : `produit;quantité totale;chiffre d'affaires`.

1. Écrire la version **naïve** :
   - lecture avec `encoding/csv` et `ReadAll` ;
   - `strconv.ParseFloat` pour les prix ;
   - une `map[string]*Stats` ;
   - `fmt.Fprintf` directement sur `os.Stdout`.

   Écrire aussi un générateur de fichier de test (10 000 produits) et un test qui fixe la sortie attendue
   sur un petit fichier.
2. Mesurer le temps total (`time ./ventes < ventes.csv > /dev/null`, dix fois, leçon 1) et profiler.
3. Appliquer les optimisations **une par une**, dans l'ordre de la leçon. Pour chacune :
   - faire la prédiction d'Amdahl d'après le profil ;
   - appliquer la modification ;
   - mesurer le gain réel ;
   - vérifier que la sortie est identique.

   Pistes : lire ligne par ligne au lieu de `ReadAll` ; `bufio` en sortie ; des prix en centimes
   (`int64`) au lieu de `float64` ; une map préallouée ; découper les lignes soi-même si le profil désigne
   `encoding/csv`.
4. Arrêter quand le profil est plat ou que le gain de l'étape suivante serait inférieur à 10 %. Présenter
   dans un tableau les étapes, la prédiction, le gain mesuré et le nombre de lignes de code ajoutées.
5. Une des étapes a-t-elle changé le **résultat** (arrondis des prix en `float64`) ? Laquelle était
   correcte ?

---

## Défi — Un pool par classe de taille

Les tailles des réponses d'un service suivent une distribution très étalée : 90 % sous 4 Kio, 9 % entre
4 et 64 Kio, 1 % entre 64 Kio et 1 Mio.

1. Écrire un `TieredPool` avec trois `sync.Pool` de `[]byte` (capacités 4 Kio, 64 Kio, 1 Mio) :
   - `Get(n)` rend un tampon d'au moins n octets ;
   - `Put(b)` le range dans le pool de sa capacité, et jette ce qui n'est pas à une capacité prévue.

   Attention : mettre un `[]byte` dans un `sync.Pool` alloue (voir le cours). Comment l'éviter ?
2. Comparer, sous 8 goroutines et avec cette distribution :
   - pas de pool ;
   - un seul `sync.Pool` sans plafond ;
   - un seul pool avec plafond de 64 Kio ;
   - le `TieredPool`.

   Pour chacun, mesurer le temps, les allocations, `HeapInuse` maximal et la part du GC.
3. Lire le code source de `bufio` et de `net/http` dans `$(go env GOROOT)/src` : la bibliothèque standard
   utilise-t-elle des pools par taille, et où ? (Chercher `sync.Pool` avec `grep`.)
4. Conclure : quel gain par rapport au pool simple avec plafond, et ce gain justifie-t-il le code en plus ?

---

## Quiz

1. Énoncer la loi d'Amdahl. Quel gain global maximal pour une partie qui prend 20 % du temps ?
2. Dans quel ordre essayer les optimisations ?
3. Pourquoi `bufio.Writer` est-il souvent la première optimisation à faire ? Quel piège l'accompagne ?
4. Quelle convention suivent les fonctions `strconv.AppendInt`, `time.Time.AppendFormat` ? Pourquoi
   permettent-elles de ne pas allouer ?
5. Quelles sont les règles d'utilisation de `sync.Pool` ?
6. Pourquoi ne pas remettre un tampon géant dans un pool ?
7. Pourquoi `sync.Pool` n'est-il pas un cache ?
8. Pourquoi 8 goroutines ne donnent-elles que ×1,2 sur la somme d'un grand tableau ?
9. Pourquoi éviter `unsafe` pour convertir `[]byte` en `string` ?
10. Quand s'arrêter d'optimiser ?
