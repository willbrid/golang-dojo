# Leçon 4 — Dates, heures et durées

> Le temps est le domaine où un code « qui marche » le plus longtemps finit par casser :
> deux fois par an (changement d'heure), une fois par mois (le 31), une fois tous les quatre
> ans (le 29 février), et chaque fois qu'un serveur n'est pas dans le même fuseau que son
> développeur. Le paquet `time` de Go est bien conçu ; encore faut-il savoir ce qu'il
> distingue.

## Objectifs

1. Distinguer un **instant** (`time.Time`) d'une **durée** (`time.Duration`).
2. Formater et analyser des dates avec la **date de référence**.
3. Manipuler les fuseaux horaires sans bug de changement d'heure.
4. Comparer des instants correctement, et mesurer une durée avec l'horloge monotone.

## Explication

### Deux types, deux notions

| Type | Ce qu'il représente | Exemple |
|---|---|---|
| `time.Time` | un **instant** précis, plus un fuseau pour l'afficher | « le 1er novembre 2026 à 10 h UTC » |
| `time.Duration` | un **écart**, en nanosecondes (`int64`) | « 90 minutes » |

```go
d := 90 * time.Minute            // une Duration
fmt.Println(d)                   // 1h30m0s
fmt.Println(d.Minutes())         // 90 (float64)
deadline := time.Now().Add(d)    // Time + Duration = Time
elapsed := time.Since(start)     // Time − Time = Duration
```

`time.Duration` est un `int64` : ±292 ans au maximum, à la nanoseconde. On l'écrit **toujours**
comme un multiple d'une constante (`5 * time.Second`), jamais comme un nombre nu :
`time.Sleep(5)` dort 5 **nanosecondes**.

Piège de conversion : `time.Duration(n) * time.Second` est correct si `n` est un entier ;
`n * time.Second` ne compile pas si `n` est un `int` (types différents — niveau 1, leçon 2).

`time.ParseDuration("1h30m")` accepte `ns`, `us`, `ms`, `s`, `m`, `h` — **pas** `d` (un jour
n'a pas toujours 24 heures, on va le voir).

### Formater : la date de référence

Go n'utilise pas `%Y-%m-%d` ni `yyyy-MM-dd`. Il utilise un **exemple** : la façon dont on
écrirait une date précise, la **date de référence** :

```
Mon Jan 2 15:04:05 MST 2006
  1   2  3  4  5       6     ← moyen mnémotechnique : 01/02 03:04:05 PM '06 -07
```

Chaque composant a une valeur unique : `2006` = année, `01` = mois, `02` = jour, `15` = heure
sur 24 h, `03` = heure sur 12 h, `04` = minutes, `05` = secondes, `Mon`/`Monday` = jour,
`Jan`/`January` = mois, `MST` = abréviation du fuseau, `-07:00` = décalage.

```go
t.Format("2006-01-02")              // 2026-11-01
t.Format("02/01/2006 à 15h04")      // 01/11/2026 à 10h00
t.Format(time.RFC3339)              // 2026-11-01T10:00:00Z  ← format d'échange par défaut
```

Deux pièges :

- `"2006-01-02 03:04"` affiche l'heure sur **12 h** sans indication AM/PM : 15 h devient `03`.
- Un format **inconnu** n'est pas une erreur : `t.Format("YYYY-MM-DD")` affiche littéralement
  `YYYY-MM-DD`. Aucun avertissement.

Les noms de jours et de mois sont toujours en **anglais** : la bibliothèque standard ne gère pas
la localisation. Pour du français, une table de traduction à la main ou un paquet externe.

### Analyser

```go
t, err := time.Parse("2006-01-02", "2026-02-30")
// parsing time "2026-02-30": day out of range
```

`time.Parse` **valide** : un 30 février est refusé. Ne jamais valider une date avec une
expression régulière (leçon 2).

**Le piège majeur** : sans indication de fuseau dans la chaîne, `time.Parse` suppose **UTC**.

```go
time.Parse("2006-01-02 15:04", "2026-11-01 10:00")
// 2026-11-01 10:00:00 +0000 UTC  ← même si l'utilisateur est à Paris
```

Pour une heure saisie par un humain dans **son** fuseau, il faut `time.ParseInLocation` :

```go
paris, err := time.LoadLocation("Europe/Paris")
t, err := time.ParseInLocation("2006-01-02 15:04", "2026-11-01 10:00", paris)
```

### Fuseaux horaires

`time.LoadLocation` lit la base de fuseaux de l'IANA (`Europe/Paris`, `America/New_York`). Elle
vient du système d'exploitation — **absente** des images Docker minimales (`scratch`,
`distroless/static`, niveau 12). Deux parades : installer le paquet `tzdata` dans l'image, ou
importer `_ "time/tzdata"` dans le programme, qui embarque la base dans le binaire (~450 Ko).

Règles de conception :

- **Stocker et échanger en UTC** (ou avec un décalage explicite, RFC 3339).
- **Convertir dans le fuseau de l'utilisateur au moment de l'affichage** : `t.In(paris)`.
- Ne **jamais** utiliser `time.Local` (le fuseau du serveur) pour une logique métier : le même
  binaire se comporterait différemment selon la machine.

### Le changement d'heure

En France, le 29 mars 2026, à 2 h du matin, on passe directement à 3 h. Et le 25 octobre 2026,
on vit **deux fois** la plage 2 h–3 h.

```go
time.ParseInLocation(layout, "2026-03-29 02:30", paris) // 03:30 CEST — 02:30 n'existe pas
time.ParseInLocation(layout, "2026-10-25 02:30", paris) // 02:30 CET  — existe deux fois, Go en choisit une
```

Conséquence directe : **un jour ne fait pas toujours 24 heures.**

```go
a := time.Date(2026, 3, 28, 12, 0, 0, 0, paris)
a.Add(24 * time.Hour)   // 2026-03-29 13:00 CEST  ← « 24 h plus tard »
a.AddDate(0, 0, 1)      // 2026-03-29 12:00 CEST  ← « demain, même heure »
```

`Add` ajoute une **durée physique** ; `AddDate` ajoute des **unités de calendrier**. « Rappelle-moi
demain à la même heure » est un `AddDate`. « Expire dans 24 heures » est un `Add`.

### `AddDate` et la normalisation

```go
time.Date(2026, 1, 31, 0, 0, 0, 0, time.UTC).AddDate(0, 1, 0)  // 2026-03-03
```

Un mois après le 31 janvier serait le « 31 février », que Go **normalise** en 3 mars. Pour
« le même jour le mois prochain, ou le dernier jour du mois », il faut le coder (exercices).
`time.Date` normalise de la même façon : `time.Date(2026, 2, 30, …)` donne le 2 mars, sans
erreur — contrairement à `time.Parse`.

### Comparer

```go
u := time.Date(2026, 11, 1, 10, 0, 0, 0, time.UTC)
p := u.In(paris)
u == p        // false
u.Equal(p)    // true
```

`==` compare la **structure** (instant + fuseau + lecture monotone), pas l'instant. Toujours
`Equal`, `Before`, `After`, `Compare` (Go 1.20, pour `slices.SortFunc`). Même raison pour ne
pas utiliser un `time.Time` comme **clé de map** : deux instants égaux peuvent être deux clés
différentes. Utiliser `t.UTC()` ou `t.Unix()` comme clé.

### L'horloge monotone

`time.Now()` contient **deux** lectures : l'horloge murale (qui peut sauter — synchronisation
NTP, réglage manuel) et une horloge **monotone** (qui ne fait qu'avancer). `time.Since(start)`
et `t2.Sub(t1)` utilisent la monotone quand les deux instants en ont une : une mesure de durée
reste juste même si l'horloge système est corrigée pendant ce temps.

La lecture monotone est perdue dès qu'on transforme l'instant (`Round(0)`, `In`, `UTC`,
sérialisation) — ce qui est voulu : elle n'a de sens que dans le processus.

### La zéro-valeur

`var t time.Time` vaut `0001-01-01 00:00:00 UTC`. On la teste avec `t.IsZero()`. C'est la
convention pour « pas de date » ; un `*time.Time` nil est l'autre option, plus lourde.

### Horodatage Unix

`t.Unix()` (secondes depuis le 1er janvier 1970 UTC), `t.UnixMilli()`, et `time.Unix(sec, nsec)`
pour revenir. Indépendant des fuseaux : pratique pour stocker, illisible pour un humain.

## Exemple

```go
package main

import (
	"fmt"
	"time"
)

var jours = [...]string{"dimanche", "lundi", "mardi", "mercredi", "jeudi", "vendredi", "samedi"}
var mois = [...]string{"", "janvier", "février", "mars", "avril", "mai", "juin",
	"juillet", "août", "septembre", "octobre", "novembre", "décembre"}

// Long formate une date en français ; la stdlib ne le fait pas.
func Long(t time.Time) string {
	return fmt.Sprintf("%s %d %s %d à %s", jours[t.Weekday()], t.Day(), mois[t.Month()], t.Year(), t.Format("15h04"))
}

// NextMonthSameDay rend le même jour le mois suivant, ou le dernier jour
// de ce mois s'il est plus court (31 janvier → 28 février).
func NextMonthSameDay(t time.Time) time.Time {
	y, m, d := t.Date()
	firstNext := time.Date(y, m+1, 1, t.Hour(), t.Minute(), t.Second(), t.Nanosecond(), t.Location())
	lastDay := firstNext.AddDate(0, 1, -1).Day()
	return firstNext.AddDate(0, 0, min(d, lastDay)-1)
}

func main() {
	paris, err := time.LoadLocation("Europe/Paris")
	if err != nil {
		fmt.Println(err)
		return
	}

	// Une heure saisie par un utilisateur parisien.
	const layout = "02/01/2006 15:04"
	rdv, err := time.ParseInLocation(layout, "28/03/2026 12:00", paris)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println("stocké   :", rdv.UTC().Format(time.RFC3339))
	fmt.Println("affiché  :", Long(rdv))
	fmt.Println("+24h     :", Long(rdv.Add(24*time.Hour)))
	fmt.Println("demain   :", Long(rdv.AddDate(0, 0, 1)))

	nuit := time.Date(2026, 3, 29, 0, 0, 0, 0, paris)
	fmt.Println("29 mars  :", nuit.AddDate(0, 0, 1).Sub(nuit), "de long")

	jan31 := time.Date(2026, 1, 31, 9, 0, 0, 0, time.UTC)
	fmt.Println("AddDate  :", jan31.AddDate(0, 1, 0).Format("2006-01-02"))
	fmt.Println("corrigé  :", NextMonthSameDay(jan31).Format("2006-01-02"))

	ny, _ := time.LoadLocation("America/New_York")
	a, b := rdv, rdv.In(ny)
	fmt.Println("== :", a == b, "Equal :", a.Equal(b), "à New York :", b.Format("15:04 MST"))

	if _, err := time.Parse("2006-01-02", "2026-02-29"); err != nil {
		fmt.Println("refusé   :", err)
	}
}
```

Sortie :

```
stocké   : 2026-03-28T11:00:00Z
affiché  : samedi 28 mars 2026 à 12h00
+24h     : dimanche 29 mars 2026 à 13h00
demain   : dimanche 29 mars 2026 à 12h00
29 mars  : 23h0m0s de long
AddDate  : 2026-03-03
corrigé  : 2026-02-28
== : false Equal : true à New York : 07:00 EDT
refusé   : parsing time "2026-02-29": day out of range
```

## Explication du code

| Élément | Ce qui compte |
|---|---|
| `[...]string` | Un **tableau** dont la taille est comptée par le compilateur : la table ne peut pas grandir par erreur. |
| `jours[t.Weekday()]` | `Weekday` est un entier (dimanche = 0) : il indexe directement la table. |
| `ParseInLocation(…, paris)` | Sans lui, 12 h serait interprété comme 12 h **UTC**, soit 13 h à Paris. |
| `rdv.UTC().Format(time.RFC3339)` | Ce qu'on stocke : UTC, format standard. 12 h à Paris le 28 mars = 11 h UTC (heure d'hiver, +1). |
| `+24h` contre `demain` | Le 29 mars n'a que 23 heures : `Add(24h)` tombe à 13 h. |
| `NextMonthSameDay` | `time.Date(y, m+1, 1, …)` : le mois 13 est normalisé en janvier suivant. `AddDate(0, 1, -1)` depuis le 1er = dernier jour du mois. |
| `a == b` faux | Même instant, fuseaux différents. |
| `07:00 EDT` | New York est déjà passé à l'heure d'été (8 mars aux États-Unis) : l'écart avec Paris n'est que de 5 h ce jour-là, pas 6. |

## Erreurs fréquentes

1. **`time.Parse` sur une heure locale** → interprétée en UTC.
2. **`==` entre deux `time.Time`**, ou un `time.Time` comme clé de map.
3. **`Add(24 * time.Hour)` pour « demain »**.
4. **`AddDate(0, 1, 0)` le 31** sans penser à la normalisation.
5. **`time.Sleep(5)`** : 5 nanosecondes.
6. **Format `03` au lieu de `15`** : heure sur 12 h sans AM/PM.
7. **Format écrit en `YYYY-MM-DD`** : affiché tel quel, sans erreur.
8. **`time.Local` dans la logique métier**.
9. **Image Docker sans base de fuseaux** : `LoadLocation` échoue en production seulement.
10. **Valider une date par regexp** au lieu de `time.Parse`.

## Bonnes pratiques Go

- Stocker et échanger en **UTC**, format **RFC 3339**.
- Convertir dans le fuseau de l'utilisateur à l'affichage, avec un `*time.Location` explicite.
- `Equal`/`Before`/`After`/`Compare` pour comparer ; `IsZero` pour l'absence.
- `Add` pour une durée physique, `AddDate` pour du calendrier.
- `time.Since` / `Sub` pour mesurer : l'horloge monotone fait le travail.
- `_ "time/tzdata"` pour un binaire autonome qui manipule des fuseaux.
- Les durées en paramètre d'API sont des `time.Duration`, pas des `int` de secondes.

## Ce que je dois retenir

- `Time` = instant, `Duration` = écart (nanosecondes, `int64`).
- Format par l'exemple : `2006-01-02 15:04:05 -07:00`.
- `Parse` suppose UTC ; `ParseInLocation` pour une heure locale.
- Un jour ne fait pas toujours 24 h : `Add` ≠ `AddDate`.
- `AddDate` normalise (31 janvier + 1 mois = 3 mars).
- **Jamais `==`**, toujours `Equal`.
- L'horloge monotone protège les mesures de durée.

---

➡️ [Exercices](exercices.md)
