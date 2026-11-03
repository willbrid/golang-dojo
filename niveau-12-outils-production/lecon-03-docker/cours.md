# Leçon 3 — Docker

> Un binaire Go statique (leçon 2) n'a besoin de presque rien pour tourner. Une image Docker peut donc ne
> contenir **que lui**, quelques certificats et les fuseaux horaires. On obtient une image de 8 Mo au lieu de
> près d'un gigaoctet, sans shell, sans gestionnaire de paquets, et sans rien qu'un attaquant pourrait
> utiliser. Cette leçon construit cette image pas à pas, et mesure ce que chaque choix change : la taille, le
> temps de reconstruction, l'arrêt du conteneur, les certificats, l'utilisateur.

## Objectifs

- Écrire un Dockerfile **multi-étapes** : construire avec l'image Go, livrer une image minimale.
- Choisir l'image de base (`scratch`, *distroless*) et savoir ce qui y manque.
- Ordonner les instructions pour profiter du cache, et accélérer `go build` avec des caches montés.
- Faire tourner le service sous un utilisateur non privilégié, avec une sonde de santé.
- Comprendre la gestion des signaux dans un conteneur, et ce qui fait attendre `docker stop` 10 secondes.

## Explication

### Rappel : image, couche, conteneur

- Une **image** est un système de fichiers en lecture seule, accompagné d'une configuration : la commande à
  lancer, l'utilisateur, les ports.
- Elle est faite de **couches** empilées. Chaque instruction `RUN`, `COPY` ou `ADD` d'un Dockerfile en ajoute
  une.
- Un **conteneur** est un processus lancé à partir d'une image, isolé du reste de la machine, avec une couche
  inscriptible par-dessus.
- `docker build` construit l'image à partir du **contexte** : le répertoire envoyé au moteur, moins ce
  qu'exclut `.dockerignore`.

Docker a déjà servi au niveau 8 pour lancer PostgreSQL (`docker run`, `docker exec`).

### Le Dockerfile multi-étapes

L'image officielle `golang:1.27.3` contient la chaîne de compilation, Git, gcc et une distribution Debian :
983 Mo. On en a besoin pour **construire**, pas pour **exécuter**. Un Dockerfile multi-étapes a plusieurs
`FROM` :

```dockerfile
FROM golang:1.27.3 AS build        # étape 1 : construire
…
RUN CGO_ENABLED=0 go build … -o /out/svc ./cmd/svc

FROM gcr.io/distroless/static-debian12:nonroot   # étape 2 : l'image livrée
COPY --from=build /out/svc /svc
ENTRYPOINT ["/svc"]
```

Seule la dernière étape devient l'image. Les précédentes ne servent qu'à fabriquer des fichiers qu'on y
copie (`COPY --from=build`). Le binaire doit être **statique** (`CGO_ENABLED=0`, leçon 2) : l'image finale n'a
pas de libc.

### Quelle image de base ?

| Base | Contenu | Taille de l'image de l'exemple |
|---|---|---|
| `golang:1.27.3` (pas de multi-étapes) | Debian, Go, gcc, Git, shell | 983 Mo |
| `scratch` | **rien** : un système de fichiers vide | 6,4 Mo |
| `gcr.io/distroless/static-debian12:nonroot` | certificats racine, fuseaux horaires, `/etc/passwd` avec un utilisateur `nonroot` (uid 65532), pas de shell | 8,5 Mo |

`scratch` est tentante, mais l'exemple mesure ce qui y manque :

- **Les certificats racine.** Tout appel HTTPS échoue : `x509: certificate signed by unknown authority`.
- **Les fuseaux horaires.** `time.LoadLocation("Europe/Paris")` échoue : `unknown time zone Europe/Paris`.
  Une conversion d'heure passée inaperçue en développement devient une erreur en production.
- **Un utilisateur non privilégié.** Le service tourne en `root` (uid 0).

On peut les ajouter à la main : copier `/etc/ssl/certs/ca-certificates.crt` depuis l'étape de build,
importer `time/tzdata` (le paquet embarque la base des fuseaux dans le binaire), et déclarer `USER 65532`.
Mais l'image *distroless* `static` fournit exactement cela, maintenu et mis à jour. C'est le choix par défaut
pour un binaire Go statique. Le suffixe `:nonroot` fixe aussi l'utilisateur par défaut.

L'absence de shell est une **qualité**. Un attaquant qui obtient l'exécution de code dans le conteneur n'y
trouve ni `sh`, ni `curl`, ni gestionnaire de paquets. La contrepartie : on ne peut pas faire
`docker exec -it … sh` pour déboguer. La leçon 9 montre comment faire autrement.

### Le cache des couches

`docker build` réutilise une couche si l'instruction **et** ses entrées n'ont pas changé. Dès qu'une couche
change, toutes celles qui suivent sont reconstruites. D'où l'ordre :

```dockerfile
COPY go.* ./                      # change rarement
RUN go mod download               # rejoué seulement si go.mod ou go.sum changent
COPY . .                          # change à chaque modification du code
RUN go build …
```

Si l'on écrit `COPY . .` en premier, chaque modification d'un fichier `.go` télécharge à nouveau toutes les
dépendances.

Même ainsi, `go build` repart de zéro à chaque reconstruction : son **cache de compilation**
(`/root/.cache/go-build`) est dans une couche qui vient d'être invalidée. BuildKit, le moteur de construction
de Docker, permet de **monter** un cache qui persiste d'un build à l'autre, sans entrer dans l'image :

```dockerfile
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build …
```

Mesuré sur l'exemple, après la modification d'un fichier `.go`, la reconstruction prend 16,5 s sans les caches
montés et 1,7 s avec (×10). La première ligne du Dockerfile, `# syntax=docker/dockerfile:1`, demande la
syntaxe récente qui les permet.

### `.dockerignore`

Le contexte envoyé au moteur doit être **petit** et **sans secrets**. `.dockerignore` exclut :

- `.git` (volumineux) ;
- les binaires et les artefacts (`bin/`, `dist/`) ;
- les fichiers `.env`, qui finiraient dans une couche de l'étape de build, lisible par quiconque a l'image de
  cette étape.

Exclure `.git` a une conséquence vue à la leçon 1 : `go build` n'a plus d'informations Git, et la version
inscrite vaut `(devel)` avec un commit inconnu. On transmet donc la version par un argument de build :

```dockerfile
ARG VERSION=dev
RUN go build -ldflags "-X main.version=${VERSION}" …
```

```
docker build --build-arg VERSION=$(git describe --tags --always) -t svc:v1.3.0 .
```

### L'utilisateur

Un processus `root` dans un conteneur n'est pas `root` sur la machine, mais il en est plus proche : une faille
du noyau ou une configuration trop permissive (un volume monté, un *socket* Docker) suffit à franchir la
frontière. `USER nonroot:nonroot`, l'utilisateur 65532 de l'image *distroless*, réduit ce risque.

Le service écoute sur `:8080`. Sur une machine Linux, seuls les processus privilégiés peuvent écouter
sous le port 1024. Dans un conteneur Docker récent, ce seuil est abaissé à 0 : `ip_unprivileged_port_start`
vaut 0 dans le conteneur et 1024 sur la machine, mesuré. L'utilisateur `nonroot` pourrait donc écouter sur
`:80`, mais rien ne garantit que tous les environnements d'exécution font de même. Un port au-dessus de 1024
fonctionne partout. On publie ensuite le port de son choix (`-p 80:8080`).

On peut aller plus loin au lancement :
- `--read-only` : système de fichiers en lecture seule ;
- `--cap-drop=ALL` : aucune capacité spéciale ;
- `--security-opt=no-new-privileges` : pas d'élévation de privilèges.

### Les signaux et `docker stop`

`docker stop` envoie `SIGTERM` au **processus 1** du conteneur, attend 10 secondes, puis envoie `SIGKILL`.
Trois situations, mesurées :

| Processus 1 | `docker stop` | Code de sortie |
|---|---|---|
| le service, avec `signal.NotifyContext(…, SIGTERM)` et arrêt propre (N7 L7) | 0,18 s | 0, « arrêt terminé » dans les journaux |
| un programme Go **sans** gestion des signaux | 0,16 s | 143 (= 128 + 15, tué par `SIGTERM`) |
| un **shell** qui lance le programme (`CMD /svc; echo fin`) | **10,25 s** | 137 (= 128 + 9, tué par `SIGKILL`) |

- **Un programme Go s'arrête**, même sans rien prévoir : son *runtime* termine le processus à la réception de
  `SIGTERM`, y compris en processus 1. Mais il s'arrête **brutalement**, sans finir les requêtes en cours.
- **Le shell ne transmet pas `SIGTERM`** à son enfant. Rien ne se passe pendant 10 s, puis tout est tué.

D'où deux règles :

- la forme **tableau** de `ENTRYPOINT`/`CMD` (`["/svc"]`), qui lance le programme directement, sans shell ;
- un service qui gère `SIGTERM` (arrêt propre) et termine avant le délai de `docker stop` (ou celui de
  l'orchestrateur, leçon 9).

### La sonde de santé

`HEALTHCHECK` fait exécuter périodiquement une commande **dans** le conteneur. Docker marque le conteneur
`healthy` ou `unhealthy` selon son code de retour. L'image *distroless* n'a ni `curl` ni `wget` : c'est le
**binaire lui-même** qui sert de sonde, avec une sous-commande `healthcheck` qui interroge `/healthz` et sort avec
0 ou 1. Un client HTTP de quelques lignes dans le binaire vaut mieux que d'ajouter un outil à l'image.

Les orchestrateurs (Kubernetes, leçon 9) ont leurs propres sondes et ignorent `HEALTHCHECK`. Celui-ci reste
utile avec `docker run` et Docker Compose.

## Exemple

Le service (une version réduite d'un vrai service). Au démarrage, il affiche ce que son environnement lui
fournit : version, utilisateur, certificats, fuseaux horaires. Puis il sert `/healthz` et s'arrête proprement
sur `SIGTERM`.

```go.mod
// Fichier : go.mod
module example.com/svc

go 1.27

toolchain go1.27.3
```

```go
// Fichier : cmd/svc/main.go
// Commande svc : un service minimal, pour observer son comportement dans un conteneur.
package main

import (
	"context"
	"crypto/x509"
	"errors"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"runtime/debug"
	"syscall"
	"time"
)

var version = "dev" // remplacée par -ldflags -X au build de l'image

func main() {
	if len(os.Args) > 1 && os.Args[1] == "healthcheck" { // pour HEALTHCHECK : pas de shell ni de curl dans l'image
		os.Exit(healthcheck())
	}
	if err := run(); err != nil {
		log.Fatal(err)
	}
}

func run() error {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()
	report()
	mux := http.NewServeMux()
	mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) { fmt.Fprintln(w, "ok") })
	srv := &http.Server{Addr: ":8080", Handler: mux, ReadHeaderTimeout: 5 * time.Second}
	errc := make(chan error, 1)
	go func() { errc <- srv.ListenAndServe() }()
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
	if err := <-errc; !errors.Is(err, http.ErrServerClosed) {
		return err
	}
	log.Print("arrêt terminé")
	return nil
}

// report affiche ce que l'environnement fournit : identité, version, certificats, fuseaux horaires.
func report() {
	commit := "(inconnu)"
	if bi, ok := debug.ReadBuildInfo(); ok {
		for _, s := range bi.Settings {
			if s.Key == "vcs.revision" {
				commit = s.Value[:12]
			}
		}
	}
	log.Printf("svc %s, commit %s, uid %d", version, commit, os.Getuid())
	if pool, err := x509.SystemCertPool(); err != nil {
		log.Printf("certificats : %v", err)
	} else {
		_, err := http.Get("https://proxy.golang.org/")
		log.Printf("certificats : pool chargé, HTTPS : %v", errOrOK(err))
		_ = pool
	}
	if _, err := time.LoadLocation("Europe/Paris"); err != nil {
		log.Printf("fuseau Europe/Paris : %v", err)
	} else {
		log.Print("fuseau Europe/Paris : ok")
	}
}

func errOrOK(err error) string {
	if err != nil {
		return err.Error()
	}
	return "ok"
}

func healthcheck() int {
	c := http.Client{Timeout: 2 * time.Second}
	resp, err := c.Get("http://127.0.0.1:8080/healthz")
	if err != nil || resp.StatusCode != http.StatusOK {
		return 1
	}
	return 0
}
```

Le Dockerfile :

```dockerfile
# syntax=docker/dockerfile:1

# --- Étape 1 : la construction, dans l'image officielle de Go ---
FROM golang:1.27.3 AS build
WORKDIR /src

# Les dépendances d'abord : cette couche ne change que si go.mod ou go.sum changent.
COPY go.* ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download

# Puis le code. Le cache de compilation survit d'un build à l'autre.
COPY . .
ARG VERSION=dev
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -ldflags "-s -w -X main.version=${VERSION}" -o /out/svc ./cmd/svc

# --- Étape 2 : l'image livrée, qui ne contient que le binaire ---
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/svc /svc
USER nonroot:nonroot
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --retries=3 CMD ["/svc", "healthcheck"]
ENTRYPOINT ["/svc"]
```

Le `.dockerignore` :

```
.git
bin/
dist/
*.test
*.out
.env
Dockerfile*
```

Pour comparer, deux autres Dockerfiles : `Dockerfile.naif` construit et livre dans l'image Go, et
`Dockerfile.scratch` livre le binaire seul dans `scratch`.

```dockerfile
# Dockerfile.naif
FROM golang:1.27.3
WORKDIR /src
COPY . .
RUN go build -o /svc ./cmd/svc
CMD ["/svc"]
```

```dockerfile
# Dockerfile.scratch
FROM golang:1.27.3 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o /svc ./cmd/svc

FROM scratch
COPY --from=build /svc /svc
ENTRYPOINT ["/svc"]
```

```
$ docker build -f Dockerfile.naif -t svc:naif .
$ docker build -f Dockerfile.scratch -t svc:scratch .
$ docker build --build-arg VERSION=v1.3.0 -t svc:distroless .
$ docker images svc --format '{{.Tag}} {{.Size}}'
distroless 8.51MB
scratch 6.39MB
naif 983MB
```

Chaque image lancée par `docker run --rm -d --name svc-… svc:…`, puis `docker logs` :

```
== naif
2026/11/03 12:45:33 svc dev, commit (inconnu), uid 0
2026/11/03 12:45:34 certificats : pool chargé, HTTPS : ok
2026/11/03 12:45:34 fuseau Europe/Paris : ok
== scratch
2026/11/03 12:45:38 svc dev, commit (inconnu), uid 0
2026/11/03 12:45:38 certificats : pool chargé, HTTPS : Get "https://proxy.golang.org/": tls: failed to verify certificate: x509: certificate signed by unknown authority
2026/11/03 12:45:38 fuseau Europe/Paris : unknown time zone Europe/Paris
== distroless
2026/11/03 12:45:42 svc v1.3.0, commit (inconnu), uid 65532
2026/11/03 12:45:42 certificats : pool chargé, HTTPS : ok
2026/11/03 12:45:42 fuseau Europe/Paris : ok
```

La sonde de santé et l'arrêt :

```
$ docker run -d --name hc svc:distroless && sleep 13      # sans --rm : les journaux restent lisibles après l'arrêt
$ docker inspect --format 'santé : {{.State.Health.Status}}' hc
santé : healthy
$ docker stop hc && docker logs hc 2>&1 | tail -2
2026/11/03 12:50:02 arrêt demandé
2026/11/03 12:50:02 arrêt terminé
$ docker inspect --format 'code de sortie {{.State.ExitCode}}' hc
code de sortie 0
```

## Explication du code

| Élément | Rôle |
|---|---|
| `os.Args[1] == "healthcheck"` | Le même binaire sert de sonde : pas d'outil à ajouter dans une image sans shell. Il sort avec 0 si `/healthz` répond 200. |
| `signal.NotifyContext(…, os.Interrupt, syscall.SIGTERM)` | `SIGTERM` (envoyé par `docker stop`) annule le contexte, et le service s'arrête proprement : `Shutdown` attend la fin des requêtes, 5 s au plus. |
| `x509.SystemCertPool()` | Charge les certificats racine du système. Dans `scratch`, le pool est vide : la fonction réussit, mais toute vérification TLS échoue ensuite. |
| `time.LoadLocation("Europe/Paris")` | Lit la base des fuseaux du système (`/usr/share/zoneinfo`), absente de `scratch`. |
| `COPY go.* ./` puis `go mod download` | Couche des dépendances, réutilisée tant que `go.mod`/`go.sum` ne changent pas. `go.*` copie `go.sum` s'il existe, sans erreur s'il n'existe pas (ce module n'a pas de dépendances). |
| `--mount=type=cache,…` | Caches de modules et de compilation conservés par BuildKit entre deux builds, sans entrer dans l'image. |
| `ARG VERSION=dev` | Argument de build, avec une valeur par défaut. Il n'existe qu'à la construction, pas dans l'image finale. |
| `USER nonroot:nonroot` | Uid 65532. Déjà le défaut de l'image `:nonroot`, répété pour que l'intention soit lisible. |
| `ENTRYPOINT ["/svc"]` | Forme tableau : `/svc` est le processus 1 et reçoit les signaux. |
| `HEALTHCHECK … CMD ["/svc", "healthcheck"]` | Toutes les 10 s, 3 s au plus, `unhealthy` après 3 échecs. |

## Erreurs fréquentes

1. **Livrer l'image de build** (983 Mo, un compilateur et un shell en production).
2. **Un binaire lié dynamiquement** dans `scratch` ou *distroless* `static` : `no such file or directory` au
   démarrage.
3. **`scratch` sans certificats ni fuseaux horaires**, découvert à la première requête HTTPS.
4. **`COPY . .` avant `go mod download`** : toutes les dépendances à chaque modification du code.
5. **Pas de `.dockerignore`** : `.git`, binaires et `.env` dans le contexte, et parfois dans une couche.
6. **La forme shell de `CMD`** (`CMD /svc --port 8080`) : selon le shell, `SIGTERM` n'arrive jamais au
   service.
7. **Tourner en `root`** sans raison.
8. **Un secret passé en `ARG` ou copié dans l'image** : il reste lisible dans l'historique des couches
   (`docker history`). Les secrets arrivent à l'exécution (leçons 5 et 8).

## Bonnes pratiques Go

- Multi-étapes : `golang:<version exacte>` pour construire, `distroless/static:nonroot` pour livrer.
- `CGO_ENABLED=0 go build -trimpath -ldflags "-s -w -X main.version=${VERSION}"` (leçon 2).
- `COPY go.* ./` et `go mod download` avant `COPY . .` ; caches montés pour `go build`.
- `.dockerignore` qui exclut `.git`, les artefacts et les secrets.
- `ENTRYPOINT` en forme tableau ; arrêt propre sur `SIGTERM`, plus court que le délai de l'orchestrateur.
- Le binaire comme sonde de santé ; `USER` non privilégié ; port au-dessus de 1024, qui fonctionne partout.
- La même version de Go dans l'image de build et dans `go.mod` (`toolchain`) : sinon la version de Go,
  et donc le binaire, dépendent de l'image (leçon 2, défi).

## Ce que je dois retenir

- Un binaire Go statique tient dans une image de quelques mégaoctets : multi-étapes, et `distroless/static`.
- `scratch` n'a ni certificats, ni fuseaux horaires, ni utilisateur ; *distroless* les fournit.
- L'ordre des couches et les caches montés font passer une reconstruction de 16,5 s à 1,7 s.
- `docker stop` envoie `SIGTERM` au processus 1 ; un shell en processus 1 fait attendre 10 s, puis tue.
- Sans `.git` dans le contexte, la version vient d'un argument de build.

---

➡️ [Exercices](exercices.md)
