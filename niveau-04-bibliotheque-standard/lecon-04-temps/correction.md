# Leçon 4 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Formats

```go
t.Format("02/01/2006")           // 14/07/2026
t.Format("15:04:05")             // 21:05:09
t.Format("03:04PM")              // 09:05PM
t.Format(time.RFC3339)           // 2026-07-14T21:05:09+02:00
t.Format(time.RFC1123Z)          // Tue, 14 Jul 2026 21:05:09 +0200
fmt.Sprintf("%s %d %s", jours[t.Weekday()], t.Day(), mois[t.Month()]) // mardi 14 juillet
```

`"DD/MM/YYYY"` s'affiche **tel quel** : `DD/MM/YYYY`. `Format` remplace les composants de la
date de référence qu'il reconnaît et recopie le reste comme du texte littéral. `D` et `Y` ne
sont pas des composants ; il n'y a donc rien à signaler. C'est pourquoi un format se **teste**
toujours sur une date connue.

## E2 — UTC par défaut

| Saisie | `Parse` → UTC | `ParseInLocation(paris)` → UTC | Écart |
|---|---|---|---|
| `2026-11-01 10:00` | 10:00 UTC | 09:00 UTC | 1 h |
| `2026-07-01 10:00` | 10:00 UTC | 08:00 UTC | 2 h |

`ParseInLocation` est juste : l'utilisateur parlait de 10 h **à Paris**. L'écart change parce
que Paris est en UTC+1 l'hiver et UTC+2 l'été. Un bug de fuseau n'est donc pas un décalage
fixe qu'on « corrige » en ajoutant une heure : il change deux fois par an.

## E3 — Les jours de 23 et 25 heures

```go
func DayLength(date time.Time, loc *time.Location) time.Duration {
	y, m, d := date.In(loc).Date()
	start := time.Date(y, m, d, 0, 0, 0, 0, loc)
	return start.AddDate(0, 0, 1).Sub(start)
}
```

Sortie sur 2026 à Paris :

```
2026-03-29 23h0m0s
2026-10-25 25h0m0s
```

`AddDate` pour aller au lendemain minuit (calendrier), `Sub` pour mesurer l'écart physique.

## E4 — Mesurer

On mesure par exemple `201ms`. `Sleep` garantit de dormir **au moins** la durée demandée :
l'ordonnanceur du système et celui de Go (niveau 6) ajoutent un délai de réveil. Si l'horloge
système reculait d'une heure pendant le sommeil, `time.Since` resterait juste parce que
`time.Now()` contient une lecture de l'horloge **monotone**, et que `Since` l'utilise quand les
deux instants en ont une. Une soustraction d'horloges murales donnerait −59 min 59,8 s.

## E5 — Égalité

La map contient **2** entrées : les clés sont comparées avec `==`, qui compare la structure
(fuseau compris), pas l'instant. Correction : normaliser la clé, `m[t.UTC()]`, ou utiliser
`t.Unix()` (ou `t.UnixNano()`) comme clé. La documentation de `time.Time` le dit explicitement.

## Exercice intermédiaire — `remind`

Deux corrections de conception par rapport à l'énoncé :

1. **Le jour d'ancrage** : pour qu'un rappel du 31 janvier tombe le 28 février **puis** le 31
   mars, `Next` ne peut pas se baser sur l'occurrence précédente (le 28). Il faut mémoriser le
   jour d'origine dans la règle.
2. **Les erreurs au constructeur** : une fois `NewRule` validé, `Next` ne peut plus échouer, et
   sa signature perd son `error`. Valider tôt simplifie tous les appelants.

```go
type Rule struct {
	every      string
	hour, min  int
	loc        *time.Location
	dayOfMonth int // pour "month" : le jour d'ancrage (1..31), fixé à la création
}

func NewRule(every, at string, loc *time.Location, anchor time.Time) (Rule, error) {
	switch every {
	case "day", "week", "month":
	default:
		return Rule{}, fmt.Errorf("périodicité %q inconnue", every)
	}
	hm, err := time.Parse("15:04", at)
	if err != nil {
		return Rule{}, fmt.Errorf("heure %q : %w", at, err)
	}
	return Rule{every: every, hour: hm.Hour(), min: hm.Minute(), loc: loc, dayOfMonth: anchor.In(loc).Day()}, nil
}

// at construit la date y-m-d à l'heure de la règle. Une heure inexistante (passage à
// l'heure d'été) est décalée par time.Date ; une heure ambiguë prend l'une des deux.
func (r Rule) at(y int, m time.Month, d int) time.Time {
	return time.Date(y, m, d, r.hour, r.min, 0, 0, r.loc)
}

func lastDay(y int, m time.Month) int { return time.Date(y, m+1, 0, 0, 0, 0, 0, time.UTC).Day() }

func (r Rule) Next(from time.Time) time.Time {
	f := from.In(r.loc)
	y, m, d := f.Date()
	switch r.every {
	case "day", "week":
		step := 1
		if r.every == "week" {
			step = 7
		}
		c := r.at(y, m, d)
		for !c.After(from) {
			d += step
			c = r.at(y, m, d)
		}
		return c
	default: // month
		for i := 0; ; i++ {
			mm := m + time.Month(i)
			yy, mmN := y+int(mm-1)/12, (mm-1)%12+1
			c := r.at(yy, mmN, min(r.dayOfMonth, lastDay(yy, mmN)))
			if c.After(from) {
				return c
			}
		}
	}
}

```

Sortie pour une règle mensuelle créée le 31 janvier, puis une quotidienne à 02:30 :

```
2026-02-28 09:00 CET
2026-03-31 09:00 CEST
2026-04-30 09:00 CEST
2026-05-31 09:00 CEST
2026-03-28 02:30 CET
2026-03-29 03:30 CEST     ← 02:30 n'existe pas : time.Date avance d'une heure
2026-03-30 02:30 CEST
2026-10-25 02:30 CET      ← 02:30 existe deux fois : Go a pris la seconde (après le recul)
2026-10-26 02:30 CET
```

Points de correction :

- **`time.Parse("15:04", at)`** valide l'heure gratuitement : `25:00` → `hour out of range`,
  `8h30` → `cannot parse "h30" as ":"`. Aucune regexp.
- **`time.Date(y, m+1, 0, …)`** : le jour **0** d'un mois est normalisé en dernier jour du mois
  précédent. L'astuce la plus courte pour « dernier jour du mois ».
- **Nuit du 29 mars** : le rappel tombe à 03:30, c'est-à-dire **au moment** où 02:30 aurait dû
  arriver — comportement raisonnable, à documenter. **Nuit du 25 octobre** : une seule
  occurrence, et la documentation de `time.Date` précise que le choix entre les deux n'est pas
  garanti. Si le rappel doit partir **une seule fois et à la première**, il faut le coder
  explicitement.
- **`d += step` avec `time.Date`** plutôt que `Add(24h)` : le calendrier, pas la physique.

## Défi — Heures ouvrées

```go
type window struct{ from, to int } // heures locales
var windows = []window{{9, 12}, {14, 18}}

func AddBusinessHours(start time.Time, d time.Duration, loc *time.Location, holidays map[string]bool) time.Time {
	t := start.In(loc)
	for {
		y, m, day := t.Date()
		wd := t.Weekday()
		if wd != time.Saturday && wd != time.Sunday && !holidays[t.Format("2006-01-02")] {
			for _, w := range windows {
				ws := time.Date(y, m, day, w.from, 0, 0, 0, loc)
				we := time.Date(y, m, day, w.to, 0, 0, 0, loc)
				if !t.Before(we) {
					continue
				}
				if t.Before(ws) {
					t = ws
				}
				if avail := we.Sub(t); d <= avail {
					return t.Add(d)
				} else {
					d -= avail
					t = we
				}
			}
		}
		t = time.Date(y, m, day+1, 0, 0, 0, 0, loc)
	}
}

```

Sortie (jours fériés : lundi de Pâques 6 avril 2026) :

```
2026-03-27 17:00 4h0m0s → Mon 2026-03-30 12:00 CEST
2026-03-28 10:00 1h0m0s → Mon 2026-03-30 10:00 CEST
2026-04-03 17:30 1h0m0s → Tue 2026-04-07 09:30 CEST
2026-03-30 12:30 30m0s → Mon 2026-03-30 14:30 CEST
2026-03-30 11:00 2h0m0s → Mon 2026-03-30 15:00 CEST
2026-03-30 09:00 7h0m0s → Mon 2026-03-30 18:00 CEST
2026-03-30 18:00 0s → Tue 2026-03-31 09:00 CEST
```

1. **Vendredi 17 h + 4 h** : 1 h le vendredi (17–18 h), puis 3 h le lundi (9–12 h) → **lundi
   30 mars à 12 h**. Le changement d'heure du dimanche est sans effet parce que toutes les bornes
   sont construites avec `time.Date` dans le fuseau local.
2. Le tableau de cas ci-dessus est la forme attendue — c'est déjà un *table-driven test*
   (niveau 5, leçon 2).
3. Samedi : on saute au lundi 9 h. Jour férié : au jour ouvré suivant (`2026-04-03 17:30 + 1 h`
   finit le mardi 7 avril, le lundi 6 étant férié). 12 h 30 : on attend 14 h. **Cas limite** :
   à 18 h avec un délai nul, le code rend le lendemain 9 h — on pourrait défendre de rendre
   `start`. Le choix doit être **tranché et testé**.
4. **Jours fériés en paramètre.** Les calculer (Pâques via l'algorithme de Meeus, plus les
   fêtes fixes) couple la fonction à un **pays** et à une **réglementation** qui changent (lundi
   de Pentecôte travaillé ou non selon les entreprises). Une fonction qui reçoit la liste est
   plus simple, plus testable, et laisse la règle métier à qui la connaît.

## Quiz

1. `Time` est un instant (avec un fuseau d'affichage), `Duration` un écart en nanosecondes.
2. Il dort 5 **nanosecondes**.
3. `Mon Jan 2 15:04:05 MST 2006` ; `15` = l'heure sur 24 h.
4. En UTC.
5. Parce qu'un jour civil peut durer 23 ou 25 heures (changement d'heure) ; il faut `AddDate(0, 0, 1)`.
6. Le 3 mars (2 mars une année bissextile) : le « 31 février » est normalisé.
7. `==` compare aussi le fuseau et la lecture monotone ; deux représentations du même instant
   sont différentes. Utiliser `Equal`.
8. Une lecture d'horloge qui ne fait qu'avancer, utilisée pour mesurer des durées ; perdue dès
   qu'on transforme l'instant (`In`, `UTC`, `Round(0)`, sérialisation).
9. La base de fuseaux IANA est absente des images minimales ; installer `tzdata` ou importer
   `_ "time/tzdata"`.
10. En UTC.
