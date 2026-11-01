# Leçon 4 — Exercices

Code dans `mes-solutions/niveau-04/lecon-04/`.

## Exercices faciles

**E1 — Formats.**
Pour l'instant `2026-07-14T21:05:09+02:00`, produire **exactement** ces sorties, chacune avec
un seul appel à `Format` (sauf la dernière) :

```
14/07/2026
21:05:09
09:05PM
2026-07-14T21:05:09+02:00
Tue, 14 Jul 2026 21:05:09 +0200
mardi 14 juillet
```

Puis essayer le format `"DD/MM/YYYY"` : que s'affiche-t-il, et pourquoi n'y a-t-il pas d'erreur ?

**E2 — UTC par défaut.**
Un utilisateur parisien saisit `2026-11-01 10:00`. Analyser la chaîne avec `time.Parse` puis
avec `time.ParseInLocation`, et afficher les deux résultats en UTC. Quel écart ? Lequel est
juste ? Refaire avec `2026-07-01 10:00` : l'écart a-t-il changé ? Pourquoi ?

**E3 — Les jours de 23 et 25 heures.**
Écrire `DayLength(date time.Time, loc *time.Location) time.Duration` qui rend la durée réelle
d'un jour civil. L'appliquer à tous les jours de 2026 à Paris, et afficher ceux qui ne font pas
24 h.

**E4 — Mesurer.**
Mesurer la durée d'un `time.Sleep(200 * time.Millisecond)` avec `time.Since`. Afficher le
résultat arrondi à la milliseconde (`Round`). Pourquoi n'est-il jamais exactement 200 ms ?
Pourquoi cette mesure resterait-elle juste si l'horloge système était reculée d'une heure
pendant le sommeil ?

**E5 — Égalité.**
Créer deux `time.Time` représentant le même instant dans deux fuseaux. Les utiliser comme clés
d'une `map[time.Time]string`. Combien d'entrées contient la map ? Corriger pour qu'il n'y en
ait qu'une.

---

## Exercice intermédiaire — `remind`, un planificateur de rappels

```go
type Rule struct {
	Every    string // "day", "week", "month"
	At       string // "08:30", heure locale
	Location *time.Location
}

// Next rend la prochaine occurrence STRICTEMENT après from.
func (r Rule) Next(from time.Time) (time.Time, error)
```

**Contraintes :**
1. `"month"` : le même jour du mois que la **première** occurrence, ou le dernier jour du mois
   s'il est plus court. Attention : un rappel créé le 31 janvier doit tomber le 28 février,
   **puis le 31 mars** — pas le 28 mars. Que faut-il mémoriser dans `Rule` pour y arriver ?
   Ajuster la structure.
2. Un rappel « tous les jours à 02:30 » à Paris : que se passe-t-il la nuit du 29 mars ? Et la
   nuit du 25 octobre ? Choisir un comportement, le documenter, le tester.
3. `At` invalide (`"25:00"`, `"8h30"`) : erreur à la construction, pas au premier appel. Ajouter
   un constructeur `NewRule`.
4. Afficher les 10 prochaines occurrences de trois règles à partir du 20 mars 2026 à midi.
5. Les durées ne passent **jamais** par `Add(24 * time.Hour)`.

---

## Défi — Heures ouvrées

Écrire `AddBusinessHours(start time.Time, d time.Duration, loc *time.Location) time.Time` :
ajouter `d` en ne comptant que les heures ouvrées (lundi–vendredi, 9 h–12 h et 14 h–18 h,
heure locale), en sautant une liste de jours fériés fournie.

1. Un ticket ouvert le vendredi 27 mars 2026 à 17 h avec un délai de 4 heures ouvrées : quand
   expire-t-il ? (Le week-end contient le changement d'heure.)
2. Écrire au moins 8 cas de test sous forme de tableau (entrée, délai, attendu) et une boucle
   qui les vérifie. *(Le niveau 5 formalisera ce patron.)*
3. Que se passe-t-il si `start` est un samedi ? Un jour férié ? À 12 h 30 ?
4. Les fêtes mobiles (lundi de Pâques) : faut-il les calculer ou les recevoir en paramètre ?
   Argumenter.

---

## Quiz

1. Différence entre `time.Time` et `time.Duration` ?
2. Que fait `time.Sleep(5)` ?
3. Quelle est la date de référence de Go ? Que représente `15` dans un format ?
4. Dans quel fuseau `time.Parse` interprète-t-il une chaîne sans indication de fuseau ?
5. Pourquoi `t.Add(24 * time.Hour)` n'est-il pas toujours « demain à la même heure » ?
6. Que donne `AddDate(0, 1, 0)` appliqué au 31 janvier ?
7. Pourquoi ne pas comparer deux `time.Time` avec `==` ?
8. Qu'est-ce que l'horloge monotone, et quand est-elle perdue ?
9. Pourquoi `LoadLocation` peut-il échouer dans un conteneur, et comment l'éviter ?
10. Dans quel fuseau stocker les dates ?
