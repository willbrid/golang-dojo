# Leçon 3 — Corrigé

> ⚠️ À ne lire qu'après avoir essayé.

## E1 — Réparer `scratch`

```go
// Fichier : cmd/svc/tzdata.go
//go:build tzdata

package main

import _ "time/tzdata" // embarque la base des fuseaux horaires dans le binaire (≈ 400 Kio)
```

```dockerfile
FROM golang:1.27.3 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -tags tzdata -trimpath -ldflags "-s -w" -o /svc ./cmd/svc

FROM scratch
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build /svc /svc
USER 65532:65532
ENTRYPOINT ["/svc"]
```

```
$ docker build -f Dockerfile.scratch -t svc:scratch2 . && docker run --rm -d --name s2 svc:scratch2
$ docker logs s2
2026/11/03 12:51:26 svc dev, commit (inconnu), uid 65532
2026/11/03 12:51:27 certificats : pool chargé, HTTPS : ok
2026/11/03 12:51:27 fuseau Europe/Paris : ok
$ docker images svc --format '{{.Tag}} {{.Size}}'
scratch2 7.03MB
distroless 8.51MB
scratch 6.39MB
```

- **Les certificats.** `crypto/x509` cherche notamment `/etc/ssl/certs/ca-certificates.crt`, qu'on copie
  depuis l'image de build (Debian).
- **Les fuseaux horaires.** `time/tzdata` embarque la base dans le binaire : 6 394 016 octets sans, 6 807 712
  avec, soit **404 Kio**. Le paquet n'est utilisé qu'en dernier recours, si le système n'a pas sa propre base.
  On l'a placé derrière une étiquette de build (leçon 2) pour ne pas alourdir les binaires qui s'exécutent
  ailleurs.
- **L'utilisateur.** `USER 65532:65532` suffit, même sans `/etc/passwd` : le noyau ne connaît que des
  numéros.

L'image fait 7,0 Mo contre 8,5 Mo pour *distroless*. Le gain est de 1,5 Mo, au prix de trois choses à
maintenir soi-même : la mise à jour des certificats, celle des fuseaux horaires, et l'absence de
`/etc/passwd`, que certains programmes attendent. Pour 1,5 Mo, *distroless* reste le meilleur choix par
défaut.

## E2 — L'ordre des couches

Avec `golang.org/x/sync` en dépendance (un `errgroup` dans `report`), et un fichier `.go` modifié entre deux
builds :

| Dockerfile | Reconstruction 1 | Reconstruction 2 |
|---|---|---|
| celui du cours | 1,52 s | 1,49 s |
| `COPY . .` d'abord, sans caches montés | 15,8 s | 20,5 s |

Les journaux de `docker build --progress=plain` montrent pourquoi :

```
# celui du cours
#11 [build 3/6] COPY go.* ./
#11 CACHED
#12 [build 4/6] RUN --mount=type=cache,target=/go/pkg/mod go mod download
#12 CACHED
#13 [build 5/6] COPY . .
#14 [build 6/6] RUN --mount=type=cache,… go build …
#14 DONE 0.6s

# COPY . . d'abord
#9 [build 3/5] COPY . .
#10 [build 4/5] RUN go mod download
#10 DONE 0.5s
#11 [build 5/5] RUN CGO_ENABLED=0 go build …
#11 DONE 19.4s
```

Dans le second Dockerfile, `COPY . .` change à chaque modification, et invalide tout ce qui suit :

- **le téléchargement des dépendances** est rejoué. Il ne prend ici que 0,5 s, car la dépendance est
  petite, mais il se compterait en dizaines de secondes avec un vrai `go.sum` ;
- **la compilation** repart de zéro : 19,4 s contre 0,6 s, car il faut recompiler la bibliothèque standard
  et les dépendances.

Le gain principal vient donc du **cache de compilation monté**, et l'ordre des couches s'occupe des
dépendances.

## E3 — Ce que gardent les couches

```
$ docker build --build-arg TOKEN=s3cr3t --target build -t t:build -f Dockerfile.secret .
$ docker history --no-trunc --format '{{.CreatedBy}}' t:build | grep -i "token\|s3cr3t"
RUN |2 VERSION=dev TOKEN=s3cr3t /bin/sh -c CGO_ENABLED=0 go build …
RUN |2 VERSION=dev TOKEN=s3cr3t /bin/sh -c echo "jeton : $TOKEN" > /tmp/t && rm /tmp/t
ARG TOKEN=s3cr3t
```

Le `rm /tmp/t` n'y change rien : la **valeur** de l'argument est inscrite dans les métadonnées de **chaque**
couche construite après l'`ARG`, en clair. Quiconque obtient l'image de l'étape de build, depuis un cache de
CI ou un registre où on l'a poussée « pour accélérer les builds », lit le secret.

L'image **finale** ne le contient pas (`grep -c` → 0) : l'étape de build n'y est pas incluse. C'est rassurant,
mais fragile : il suffit de pousser l'étape de build, ou de mettre l'`ARG` dans l'étape finale.

La bonne méthode passe par un secret **monté** :

```dockerfile
RUN --mount=type=secret,id=token test -s /run/secrets/token && echo "secret lu, longueur $(wc -c < /run/secrets/token)"
```

```
$ docker build --secret id=token,src=token.txt --target build -t t:sm .
#12 0.183 secret lu, longueur 6
$ docker history --no-trunc --format '{{.CreatedBy}}' t:sm | grep -c s3cr3t
0
```

Le fichier n'existe que pendant ce `RUN`, dans `/run/secrets/token`, et n'apparaît ni dans une couche ni dans
l'historique. C'est la façon de passer à un build un jeton d'accès à un dépôt de modules privé
(`GOPRIVATE`, leçon 8).

## E4 — Arrêts lents

La route ajoutée, et un délai d'arrêt réglable :

```go
	mux.HandleFunc("GET /lent", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(15 * time.Second)
		fmt.Fprintln(w, "fini")
	})
	…
	grace := 5 * time.Second
	if d, err := time.ParseDuration(os.Getenv("SHUTDOWN_GRACE")); err == nil {
		grace = d // la leçon 5 fera de la vraie configuration
	}
	sctx, cancel := context.WithTimeout(context.Background(), grace)
```

Une requête sur `/lent`, puis `docker stop` une seconde plus tard :

| Délai du service | `docker stop` | Arrêt | Code de sortie | Le client reçoit |
|---|---|---|---|---|
| 5 s (cours) | `-t 10` | 5,3 s | 1 (`context deadline exceeded`) | rien : connexion coupée (`curl` code 52) au bout de 6 s |
| 5 s | `-t 30` | 5,3 s | 1 | idem |
| 20 s | `-t 10` | 10,3 s | **137** (`SIGKILL`) | rien, au bout de 11 s |
| 20 s | `-t 30` | 14,5 s | 0, « arrêt terminé » | **`fini`, code 200**, au bout de 15 s |

- **Le délai du service** décide combien de temps on attend les requêtes en cours. Avec 5 s, une requête de
  15 s est coupée, quel que soit `-t`.
- **Le délai de `docker stop`** est une limite **absolue** : au-delà, `SIGKILL`, sans aucun nettoyage (pas de
  journal « arrêt terminé », pas de fermeture propre de la base).
- **Il faut donc** : durée de la plus longue requête légitime < délai du service < délai de l'orchestrateur.
  Pour Docker, c'est `-t` ou `stop_grace_period` dans Compose ; pour Kubernetes, `terminationGracePeriodSeconds`,
  30 s par défaut (leçon 9).

  Une requête qui dépasse toujours ce budget doit devenir une tâche asynchrone (N10 L8), pas un
  délai d'arrêt d'une minute.

## E5 — Réduire les droits

Lancé avec `--read-only --cap-drop=ALL --security-opt=no-new-privileges`, et une adresse et un fichier
temporaire réglables par l'environnement :

| Lancement | Résultat |
|---|---|
| durci, utilisateur `nonroot`, `:8080` | fonctionne (uid 65532) |
| durci, `nonroot`, `ADDR=:80` | **fonctionne aussi** |
| durci, `-u 0`, `ADDR=:80` | fonctionne |
| durci, `TEMPFILE=1` | `open /tmp/svc-1314912431: read-only file system` |
| durci, `--tmpfs /tmp`, `TEMPFILE=1` | `fichier temporaire : ok` |

Le port 80 fonctionne sans privilège ni capacité : dans un conteneur Docker récent,
`/proc/sys/net/ipv4/ip_unprivileged_port_start` vaut **0** (contre 1024 sur la machine). On ne l'apprend
qu'en mesurant, et le cours a été corrigé en conséquence. Le port 8080 reste le choix portable.

Le service lui-même n'écrit rien : il fonctionne en lecture seule, sans aucune capacité. C'est le cas de la
plupart des services Go, et c'est la configuration à viser. Si un paquet a besoin d'un répertoire temporaire,
`--tmpfs /tmp` fournit un répertoire en mémoire, vidé à l'arrêt, sans rouvrir le reste du système de fichiers.

---

## Exercice intermédiaire — Le service et sa base dans Docker Compose

Une API minimale (notes) à la place du service de prêts : seul ce qui touche au conteneur compte ici. Elle
a trois sous-commandes, `migrate`, `serve` et `healthcheck`.

```go.mod
// Fichier : go.mod
module example.com/svc

go 1.27

toolchain go1.27.3

require github.com/jackc/pgx/v5 v5.9.2
```

```go
// Fichier : cmd/api/main.go
// Commande api : une API minimale et sa base, pour Docker Compose.
//
//	api migrate   applique le schéma et se termine
//	api serve     sert l'API (défaut)
package main

import (
	"context"
	"database/sql"
	"encoding/json"
	"errors"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
)

// Le schéma tient en une instruction ; un vrai service embarque ses fichiers de migration (N8 L6).
const schema = `CREATE TABLE IF NOT EXISTS notes (
    id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    text text NOT NULL
)`

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()
	cmd := "serve"
	if len(os.Args) > 1 {
		cmd = os.Args[1]
	}
	if err := run(ctx, cmd); err != nil {
		log.Fatal(err)
	}
}

func run(ctx context.Context, cmd string) error {
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		return errors.New("DATABASE_URL manquante")
	}
	db, err := sql.Open("pgx", dsn)
	if err != nil {
		return err
	}
	defer db.Close()
	switch cmd {
	case "migrate":
		return migrate(ctx, db)
	case "serve":
		return serve(ctx, db)
	case "healthcheck":
		return healthcheck()
	}
	return fmt.Errorf("commande inconnue : %s", cmd)
}

func migrate(ctx context.Context, db *sql.DB) error {
	if _, err := db.ExecContext(ctx, schema); err != nil {
		return fmt.Errorf("migration : %w", err)
	}
	log.Print("migrations appliquées")
	return nil
}

func serve(ctx context.Context, db *sql.DB) error {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) {
		if err := db.PingContext(r.Context()); err != nil {
			http.Error(w, "base indisponible", http.StatusServiceUnavailable)
			return
		}
		fmt.Fprintln(w, "ok")
	})
	mux.HandleFunc("POST /notes", func(w http.ResponseWriter, r *http.Request) {
		var n struct{ Text string }
		if err := json.NewDecoder(r.Body).Decode(&n); err != nil || n.Text == "" {
			http.Error(w, "texte attendu", http.StatusBadRequest)
			return
		}
		if _, err := db.ExecContext(r.Context(), "INSERT INTO notes (text) VALUES ($1)", n.Text); err != nil {
			http.Error(w, "erreur", http.StatusInternalServerError)
			return
		}
		w.WriteHeader(http.StatusCreated)
	})
	mux.HandleFunc("GET /notes/count", func(w http.ResponseWriter, r *http.Request) {
		var c int
		if err := db.QueryRowContext(r.Context(), "SELECT count(*) FROM notes").Scan(&c); err != nil {
			http.Error(w, "erreur", http.StatusInternalServerError)
			return
		}
		fmt.Fprintln(w, c)
	})
	srv := &http.Server{Addr: ":8080", Handler: mux, ReadHeaderTimeout: 5 * time.Second}
	errc := make(chan error, 1)
	go func() { errc <- srv.ListenAndServe() }()
	log.Print("API prête sur :8080")
	select {
	case err := <-errc:
		return err
	case <-ctx.Done():
	}
	log.Print("arrêt demandé")
	sctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := srv.Shutdown(sctx); err != nil {
		return err
	}
	log.Print("arrêt terminé")
	return nil
}

func healthcheck() error {
	c := http.Client{Timeout: 2 * time.Second}
	resp, err := c.Get("http://127.0.0.1:8080/healthz")
	if err != nil {
		return err
	}
	resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("santé : %s", resp.Status)
	}
	return nil
}
```

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.27.3 AS build
WORKDIR /src
COPY go.* ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
ARG VERSION=dev
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -ldflags "-s -w -X main.version=${VERSION}" -o /out/api ./cmd/api

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/api /api
EXPOSE 8080
ENTRYPOINT ["/api"]
CMD ["serve"]
```

```yaml
# compose.yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: notes
      POSTGRES_DB: notes
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets: [db_password]
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "notes", "-d", "notes"]
      interval: 2s
      timeout: 3s
      retries: 15

  migrate:
    build: .
    command: ["migrate"]
    env_file: .env # DATABASE_URL, hors du dépôt
    depends_on:
      db:
        condition: service_healthy

  api:
    build: .
    command: ["serve"]
    env_file: .env
    ports:
      - "127.0.0.1:18080:8080" # jamais 0.0.0.0 par défaut
    stop_grace_period: 10s
    healthcheck:
      test: ["CMD", "/api", "healthcheck"]
      interval: 5s
      timeout: 3s
      retries: 3
    depends_on:
      migrate:
        condition: service_completed_successfully

volumes:
  pgdata:

secrets:
  db_password:
    file: ./db_password.txt # hors du dépôt (.gitignore)
```

Les deux fichiers locaux, **hors du dépôt** (dans `.gitignore`) :

```
# .env
DATABASE_URL=postgres://notes:exemple-local-seulement@db:5432/notes?sslmode=disable
# db_password.txt
exemple-local-seulement
```

```
$ docker compose up -d --build --wait
 Container n123-migrate-1  Exited
 Container n123-db-1  Healthy
 Container n123-api-1  Healthy
$ docker compose ps -a --format '{{.Service}} {{.State}} {{.Health}}'
api running healthy
db running healthy
migrate exited
$ docker compose logs migrate api --no-log-prefix
2026/11/03 12:55:55 API prête sur :8080
2026/11/03 12:55:54 migrations appliquées
$ curl -X POST -d '{"Text":"première"}' http://127.0.0.1:18080/notes -w '%{http_code}\n'
201
$ curl http://127.0.0.1:18080/notes/count
1
$ docker compose stop api && docker compose logs api --no-log-prefix | tail -2
2026/11/03 12:56:07 arrêt demandé
2026/11/03 12:56:07 arrêt terminé
$ docker compose down && docker compose up -d --wait && curl http://127.0.0.1:18080/notes/count
1
```

- **L'ordre.** `db` passe `healthy` (`pg_isready`). `migrate` démarre alors, applique le schéma et se
  termine avec le code 0 (`service_completed_successfully`). `api` démarre ensuite. Sans les conditions, l'API
  démarrerait avant la base et échouerait, ou pire, réussirait par chance la plupart du temps.
- **3. Les migrations à part.** Le service `migrate` s'exécute **une** fois, avant toute instance de l'API.
  Avec plusieurs instances qui migrent chacune au démarrage, il faut le verrou de N8 L6.

  Un échec de migration bloque le déploiement au lieu de faire tourner en boucle une API qui
  redémarre. Le même binaire et la même image servent aux deux : il n'y a pas de version de schéma
  désaccordée.
- **Les données survivent** à `down` puis `up` : elles sont dans le volume nommé `pgdata`, que `down` ne
  supprime pas (`down -v` le fait).
- **Le port** est publié sur `127.0.0.1:18080` seulement. Docker contourne en partie le pare-feu de la
  machine pour les ports publiés : un port 5432 ou 8080 publié sur toutes les interfaces serait exposé au
  réseau. La base n'est **pas** publiée du tout, et l'API la joint par le réseau interne de Compose (`db:5432`).
- **5. Ce qui ne figure pas dans `compose.yaml`** : les mots de passe et la chaîne de connexion complète. On
  les met dans `.env` et dans le fichier secret (`POSTGRES_PASSWORD_FILE` est lu par l'image `postgres`),
  tous deux hors du dépôt. Le dépôt contient au plus un `.env.example` avec des valeurs factices. Les
  ports publiés sur `0.0.0.0` n'y figurent pas non plus.

---

## Défi — Une image pour deux architectures

```dockerfile
# Dockerfile.multi : les lignes qui changent par rapport au Dockerfile du cours
FROM --platform=$BUILDPLATFORM golang:1.27.3 AS build
ARG TARGETOS TARGETARCH
…
    CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -trimpath -ldflags "-s -w -X main.version=${VERSION}" -o /out/svc ./cmd/svc
```

**1. Pourquoi c'est rapide.** `--platform=$BUILDPLATFORM` fait exécuter l'étape de build sur la plateforme
**de la machine** (amd64), et le compilateur Go produit le binaire ARM par cross-compilation (leçon 2). Seule
l'étape finale est « arm64 », et elle ne fait que copier un fichier : rien à exécuter. Sans cette ligne,
l'étape de build tourne dans une image Go arm64, donc dans un processeur ARM **émulé** : le compilateur
tourne lui-même sous émulation, des dizaines de fois plus lentement.

**2. Les mesures.** Sur cette machine, le moteur de build par défaut (pilote `docker`) n'exporte pas une image
multi-architecture d'un coup :

```
ERROR: OCI exporter is not supported for the docker driver.
```

On construit donc une variante par plateforme (`--platform linux/arm64 --load`) :

| Build | Durée | Résultat |
|---|---|---|
| `Dockerfile.multi`, `linux/amd64` | 2,1 s | image amd64 |
| `Dockerfile.multi`, `linux/arm64` | 17,9 s (premier build : téléchargement de l'image *distroless* arm64 et compilation sans cache) | image arm64 |
| `Dockerfile` du cours, `linux/arm64` | échec au bout de 25 s | `exec /bin/sh: exec format error` |

Sans émulation installée (aucune entrée QEMU dans `/proc/sys/fs/binfmt_misc`), la machine ne peut pas
exécuter le `/bin/sh` ARM de l'image `golang` : la variante sans `$BUILDPLATFORM` échoue dès le premier
`RUN`. La cross-compilation de Go rend l'émulation inutile.

**3. Vérification.**

```
$ id=$(docker create svc:multi-arm64) && docker cp $id:/svc svc-arm64 && docker rm $id
$ file svc-arm64
svc-arm64: ELF 64-bit LSB executable, ARM aarch64
$ docker image inspect svc:multi-arm64 --format '{{.Architecture}}'
arm64
```

Pour publier **une** image qui contient les deux variantes (une « liste de manifestes »), il faut un moteur
BuildKit dans un conteneur (`docker buildx create --driver docker-container --use`), ou le stockage d'images
containerd de Docker. On ne l'a pas fait ici. La CI (leçon 4) le fait avec l'action officielle de Docker.

**4. Reproductibilité.**

| Deux builds arm64 identiques (`--no-cache`) | Premier | Second |
|---|---|---|
| empreinte du **binaire** | `86355a5be886a961` | `86355a5be886a961` |
| identifiant de l'**image** | `dcdb156de2c8…` | `07e20361eb9d…` |
| avec `--build-arg SOURCE_DATE_EPOCH=1772359200` | `180cc43907b7…` | `bb66597859c2…` |

Le binaire est reproductible (leçon 2). L'image ne l'est pas : sa configuration contient la **date de
création**, et les fichiers des couches leurs dates de modification. `SOURCE_DATE_EPOCH` fixe la date de
création (les deux images affichent `2026-03-01T10:00:00Z`), mais les identifiants diffèrent encore : les
dates des fichiers dans les couches ne sont pas réécrites par l'exportateur utilisé ici.

Rendre l'image elle-même reproductible demande des options d'export de BuildKit qu'on n'a pas vérifiées
ici. Ce qu'on vérifie en pratique, c'est le **binaire** : il porte le code, et l'image n'est qu'un emballage.

---

## Quiz

1. Plusieurs `FROM` : une étape construit avec l'image Go complète, la dernière ne reçoit que le binaire
   (`COPY --from`). Pour Go, l'image livrée passe de près de 1 Go à quelques Mo, sans compilateur ni shell.
2. Les certificats racine (HTTPS : `certificate signed by unknown authority`), la base des fuseaux horaires
   (`unknown time zone`) et un utilisateur non privilégié.
3. Certificats, fuseaux horaires, `/etc/passwd` avec l'utilisateur `nonroot` (65532) par défaut, pas de shell
   ni de gestionnaire de paquets, et des mises à jour maintenues. Le tout en 2 Mo.
4. `go.mod` et `go.sum`, puis `go mod download`, **puis** le reste du code. La couche des dépendances reste en
   cache tant que `go.mod`/`go.sum` ne changent pas.
5. À garder, d'un build à l'autre, le cache des modules et le cache de compilation de Go, sans qu'ils entrent
   dans l'image : 16,5 s → 1,7 s dans le cours.
6. Pour un contexte petit et sans historique. Sans `.git`, `go build` n'inscrit ni version ni commit (leçon 1).
   On les passe par `--build-arg` et `-X`.
7. Il envoie `SIGTERM` au processus 1, attend 10 s, puis envoie `SIGKILL`. Un shell en processus 1 ne transmet
   pas `SIGTERM` à son enfant : rien ne se passe avant le `SIGKILL`.
8. Elle lance le programme directement, sans shell : il est le processus 1 et reçoit les signaux.
9. Le binaire lui-même : une sous-commande `healthcheck` qui interroge `/healthz` et sort avec 0 ou 1,
   déclarée par `HEALTHCHECK CMD ["/svc", "healthcheck"]`.
10. Sa valeur est inscrite en clair dans l'historique de chaque couche construite après lui
    (`docker history`). On utilise `RUN --mount=type=secret`, et à l'exécution, des variables d'environnement
    ou des fichiers fournis par l'orchestrateur (leçon 8).
