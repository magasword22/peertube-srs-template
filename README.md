# PeerTube + SRS — Stack de streaming temps réel

Stack Docker autonome pour diffuser des **directs vidéo en basse latence** : ingestion RTMP (drones, caméras, OBS), transcodage temps réel, diffusion **WebRTC** (< 1 s) et hébergement **VOD / replay** via PeerTube.

## Architecture

```
                         ┌──────────────────────────────┐
   Drone DJI / OBS       │            SRS               │
   ──── RTMP ──────────► │  rtmp://<serveur>:1935       │
                         │                              │
                         │  transcode "dji_fix" :       │      ┌──────────────┐
                         │   • calage flux à 30 fps     ├─────►│   PeerTube   │
                         │   • H.264 baseline 1.5 Mbps  │ RTMP │  :9000 (API) │
                         │   • AAC 64 kbps 44.1 kHz     │      └──────┬───────┘
                         │                              │             │
                         │  rtc : WebRTC à chaud        │             ▼
                         │   (bframe discard)           │      HLS / VOD / replay
                         └──────────┬───────────────────┘             ▲
                                    │                                 │
                                    ▼                    ┌────────────┴───────────┐
                         ┌──────────────────────┐        │  Navigateurs / mobiles │
                         │  Supervision WebRTC  │        │  HLS (latence ~10-30 s)│
                         │  udp://:8000         │        └────────────────────────┘
                         └──────────────────────┘
```

**Deux modes de consommation :**
- **WebRTC temps réel** (latence < 1 s) — supervision, murs d'affichage
- **HLS via PeerTube** (latence 10-30 s) — diffusion large, replay et VOD

## Composants

| Service | Image | Rôle | Ports |
|---|---|---|---|
| `peertube` | `chocobozzz/peertube:production` | Plateforme vidéo (VOD, lives, HLS, replay) | 9000 (HTTP, à placer derrière reverse proxy) |
| `srs` | `srs-custom:latest` (build local) | Ingestion RTMP, transcodage, WebRTC | 1935/tcp (RTMP), 1985/tcp (API), 8000/udp (WebRTC) |
| `postgres` | `postgres:17-alpine` | Base de données PeerTube | interne |
| `redis` | `redis:8-alpine` | Cache / files PeerTube | interne |

## Déploiement

```bash
# 1. Configuration
cp .env.example .env
# → renseigner POSTGRES_PASSWORD, PEERTUBE_SECRET (openssl rand -hex 32),
#   PEERTUBE_WEBSERVER_HOSTNAME, SMTP...

# 2. Lancement
docker compose up -d

# 3. PeerTube (premier démarrage)
#    → récupérer le mot de passe root dans les logs :
docker compose logs peertube | grep -A1 "User password"
```

### Points de configuration obligatoires

| Variable / fichier | Usage |
|---|---|
| `.env` → `PEERTUBE_WEBSERVER_HOSTNAME` | Nom de domaine public de l'instance (HTTPS assuré par un reverse proxy en amont) |
| `.env` → `PEERTUBE_SECRET` | Clé secrète de l'instance (`openssl rand -hex 32`) |
| `docker-volume/srs/srs.conf` → `candidate` | **IP publique ou FQDN** joignable par les clients WebRTC (UDP 8000) — à adapter avant le premier lancement |

## Utilisation

### Mettre un direct en ligne

1. Créer un live dans PeerTube → récupérer la clé de stream
2. Publier le flux : `rtmp://<serveur>:1935/live/<clé>`
3. SRS transcode à la volée (moteur `dji_fix`) puis réinjecte dans PeerTube : `rtmp://peertube:1935/live/<stream>`

### Pourquoi le transcodage « dji_fix » ?

Les drones DJI publient des flux à cadence non standard (29.97 fps, GOP irrégulier) qui cassent le HLS de PeerTube. Le moteur transcode en direct : **H.264 baseline, 30 fps fixes, 1.5 Mbps, zerolatency** — cadence stable garantie en aval.

### Regarder en temps réel

Le flux est exposé en WebRTC par SRS (port UDP 8000) : lecture directe via l'URL WHEP
`http://<serveur>:1985/rtc/v1/whep/?app=live&stream=<stream>`, adaptée aux murs d'affichage et à la supervision temps réel.

## Arborescence

```
├── docker-compose.yml                  # Stack complète (peertube + srs + postgres + redis)
├── .env.example                        # Modèle de configuration (à copier en .env)
└── docker-volume/
    └── srs/
        ├── Dockerfile                  # SRS 5 + ffmpeg (pour le transcodage)
        └── srs.conf                    # Config SRS (RTMP, transcode dji_fix, WebRTC)
```

Les volumes runtime (`data/`, `db/`, `redis/`) sont créés au premier lancement et ignorés par git.
La configuration fine de PeerTube se fait via l'interface d'administration une fois la stack démarrée (ou via un volume `/config` si tu veux versionner un `production.json`).

## Notes d'exploitation

- Le service `peertube` est limité à 8 CPU (`deploy.resources`) — ajuster selon l'hôte
- Les lives PeerTube acceptent le **replay** et une fenêtre DVR de 30 min côté SRS
- Le port 1935 de PeerTube est volontairement non publié — seul SRS est point d'entrée RTMP
- Le HTTPS de PeerTube est attendu en amont (reverse proxy) : ne pas exposer le 9000 directement
