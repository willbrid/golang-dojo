# Leçon 2 — Exercices

Code dans `mes-solutions/niveau-06/lecon-02/`. Tout se lance aussi avec `-race`.

## Exercices faciles

**E1 — Provoquer l'interblocage.**
Écrire trois programmes qui se terminent par `all goroutines are asleep - deadlock!` : (a) un envoi
sans récepteur ; (b) deux goroutines qui s'attendent mutuellement ; (c) un channel déclaré sans
`make`. Pour chacun, recopier la ligne `goroutine N [...]` de la trace : que dit-elle de l'opération
bloquée ?

**E2 — Ping-pong.**
Deux goroutines s'échangent un compteur : la première envoie `n` sur `ping`, la seconde renvoie
`n+1` sur `pong`, cinq fois. `main` affiche la liste des valeurs reçues : `[1 2 3 4 5]`. Aucun
`time.Sleep`, aucun `WaitGroup`.

**E3 — Direction.**
Écrire `func generate(n int, out chan<- int)` et `func square(in <-chan int, out chan<- int, n int)`.
Essayer, dans `generate`, de lire sur `out` : recopier le message du compilateur. Pourquoi cette
erreur est-elle préférable à un bug à l'exécution ?

**E4 — Propriété transférée.**
Une goroutine remplit un `*bytes.Buffer`, l'envoie sur un channel, puis **continue** d'écrire
dedans. Le récepteur lit le buffer. Que dit `-race` ? Corriger de deux façons : (a) l'émetteur
n'y touche plus après l'envoi ; (b) l'émetteur envoie une copie (`buf.String()`).

**E5 — Requête-réponse.**
Écrire un « serveur de compteur » : une goroutine possède un `map[string]int` **à elle seule** et
traite des requêtes `incr(clé)` et `get(clé)` reçues sur un channel, chacune avec son channel de
réponse. Dix goroutines clientes font 100 `incr` chacune sur la même clé ; vérifier que `get` rend
1 000, avec `-race` silencieux. Pourquoi n'y a-t-il pas besoin de verrou ?

---

## Exercice intermédiaire — Un pipeline à trois étages

Construire un pipeline : `lire` (envoie les lignes d'un `io.Reader`) → `nettoyer` (minuscules,
ponctuation retirée) → `compter` (cumule dans une map, que seul cet étage possède).

**Contraintes :**
1. Chaque étage est une fonction avec des paramètres de channel **typés en direction**.
2. On ne connaît pas à l'avance le nombre de lignes. Sans fermeture de channel (leçon 3), comment
   signaler la fin ? Proposer une solution avec une valeur **sentinelle** (par exemple une ligne
   spéciale, ou une struct `{line string; eof bool}`), et noter ses défauts.
3. Le résultat de `compter` revient à `main` par un channel.
4. Mesurer l'intérêt : sur un texte de 100 Mo, le pipeline est-il plus rapide qu'une boucle
   séquentielle ? Expliquer le résultat (coût d'un échange sur un channel contre coût du travail
   par ligne).
5. Garder ce code : la leçon 3 le réécrira avec `close` et `range`.

---

## Défi — Le coût d'un rendez-vous

1. Écrire un benchmark (`b.Loop`, N5 L6) qui mesure un aller-retour ping-pong entre deux goroutines.
   Combien de nanosecondes par échange ?
2. Comparer avec un simple appel de fonction et avec un aller-retour sur un channel bufferisé de
   taille 1 (anticipation de la leçon 3).
3. Lancer avec `GOMAXPROCS=1` puis par défaut. L'échange est-il plus rapide ou plus lent quand les
   deux goroutines sont sur des cœurs différents ? Formuler une hypothèse.
4. Conclusion : à partir de quelle quantité de travail **par message** un pipeline à channels
   devient-il intéressant ?

---

## Quiz

1. Que se passe-t-il quand on envoie sur un channel non bufferisé sans récepteur ?
2. Que garantit un rendez-vous à l'émetteur au moment où l'envoi se termine ?
3. Que garantit le modèle mémoire pour ce qui a été écrit avant un envoi ?
4. Pourquoi utiliser `chan struct{}` pour un signal ?
5. Différence entre `chan<- T` et `<-chan T` ?
6. Quand le runtime détecte-t-il un interblocage ? Quand ne le détecte-t-il pas ?
7. Que fait une opération sur un channel nil ?
8. Pourquoi une requête peut-elle contenir son propre channel de réponse ?
9. Que signifie « céder la propriété » d'une valeur envoyée ?
10. Que signifie le proverbe « partager la mémoire en communiquant » ?
