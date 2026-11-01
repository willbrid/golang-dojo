# Leçon 3 — Exercices

Code dans `mes-solutions/niveau-06/lecon-03/`. Tout avec `-race`.

## Exercices faciles

**E1 — Les trois paniques.**
Provoquer `send on closed channel`, `close of closed channel` et `close of nil channel`. Pour
chacune, écrire en une phrase **quelle règle** a été violée.

**E2 — Buffer et rendez-vous.**
Avec un channel de capacité 2, une goroutine envoie 5 valeurs en affichant `envoi i` avant chaque
envoi ; `main` attend 100 ms puis reçoit les 5 valeurs. Prédire combien de lignes « envoi » sont
affichées avant la première réception. Vérifier. Recommencer avec une capacité 0.

**E3 — `ok` et `range`.**
Écrire `func drain(ch <-chan int) (sum, n int)` de deux façons : avec `range`, puis avec une boucle
`for` et `v, ok`. Laquelle est plus lisible ? Dans quel cas la seconde est-elle nécessaire ?
*(Indice : leçon 4.)*

**E4 — Réécrire le pipeline.**
Reprendre le pipeline à trois étages de la leçon 2 (sentinelle `eof`) et le réécrire avec `close` et
`range`. Combien de lignes disparaissent ? Où va maintenant l'erreur éventuelle de `sc.Err()` ?

**E5 — La fuite du résultat unique.**
Écrire `func slow() <-chan int` qui lance une goroutine calculant pendant 200 ms puis envoyant le
résultat sur un channel **non bufferisé**. Dans `main`, appeler `slow()` 1 000 fois sans lire les
résultats, puis afficher `runtime.NumGoroutine()` après 500 ms. Corriger avec une capacité de 1 et
recompter.

---

## Exercice intermédiaire — `merge` et `split`

1. Écrire `func Merge[T any](ins ...<-chan T) <-chan T` générique (N3), qui ferme sa sortie quand
   **toutes** les entrées sont fermées et vidées.
2. Écrire `func Split[T any](in <-chan T, n int) []<-chan T` qui distribue les valeurs en
   **tourniquet** sur `n` sorties, et les ferme toutes quand `in` est fermé.
3. Tester (table, `-race`, N5) : `Merge(Split(source, 4)...)` rend exactement les mêmes valeurs que
   `source`, dans un ordre quelconque — comparer après tri. Cas limites : aucune entrée, entrées
   vides, `n = 1`.
4. Le test doit échouer en **moins d'une seconde** si une fermeture est oubliée, plutôt que de
   bloquer jusqu'au délai global de `go test` (10 minutes). Comment ? *(Indice : `t.Context()` ou un
   `time.After` — leçon 4 ; ou l'option `-timeout`.)*

---

## Défi — La bonne capacité

Un producteur génère une valeur toutes les 1 ms **en moyenne**, mais par rafales : 50 valeurs d'un
coup, puis 50 ms de silence. Le consommateur traite une valeur en 0,9 ms, régulièrement.

1. Simuler (avec `time.Sleep` dans le producteur et le consommateur, c'est acceptable pour une
   simulation) avec des capacités 0, 1, 10, 50, 1 000. Mesurer le temps total et le **temps
   d'attente maximal** de l'émetteur.
2. Quelle capacité choisir ? Pourquoi 1 000 n'est-il pas mieux que 50 ?
3. Passer le consommateur à 1,1 ms par valeur. Que se passe-t-il, quelle que soit la capacité ?
   Formuler la règle (elle sera le sujet de la leçon 11).

---

## Quiz

1. Quand un envoi sur un channel bufferisé bloque-t-il ?
2. Que perd-on par rapport à un channel non bufferisé ?
3. Que rend une réception sur un channel fermé dont le buffer contient encore des valeurs ?
4. Quand `range` sur un channel s'arrête-t-il ?
5. Quelles sont les trois opérations qui paniquent ?
6. Qui doit fermer un channel ? Et s'il y a plusieurs émetteurs ?
7. Est-il obligatoire de fermer un channel ?
8. Pourquoi `if len(ch) < cap(ch)` ne garantit-il rien ?
9. Pourquoi un résultat unique se rend-il souvent sur un channel de capacité 1 ?
10. Un buffer plus grand augmente-t-il le débit ?
