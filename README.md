# PeerTube + SRS — Stack de streaming temps réel pour drones DJI

Stack Docker conçue **spécifiquement pour corriger les problèmes de transmission des drones DJI** (RM500, RM700B avec DJI Pilot 2) : stabilisation du flux RTMP, cadence forcée à 30 fps, acquittements RTMP, audio factuel AAC — diffusion **WebRTC** (< 1 s) et hébergement **VOD / replay** via PeerTube.

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

## 🎯 Le problème DJI : cinq anomalies de transmission, cinq corrections

Les flux RTMP émis par les radiocommandes DJI (RM500, RM700B avec DJI Pilot 2) s'écartent des standards habituels de diffusion web. Ce stack existe pour les fiabiliser — voici les cinq anomalies rencontrées et leur correction, toutes implémentées dans [`docker-volume/srs/srs.conf`](docker-volume/srs/srs.conf).

### 1. Absence de métadonnées de cadence — le bug « 720p1000 »

**Comportement DJI** : la radiocommande n'indique pas de cadence d'images exploitable dans l'en-tête du conteneur FLV.

**Impact** : PeerTube et son décodeur interne se calent sur la base de temps brute et interprètent le flux à 1 000 images par seconde. L'encodage HLS s'effondre, provoquant d'importantes saccades ou un gel total de l'image.

**Correction** : intercaler SRS qui transcode le flux à la volée avec FFmpeg en imposant une fréquence fixe (`vfps 30;`, ce qui injecte `-r 30.00`). PeerTube reçoit ainsi un flux certifié en 30 fps propre.

### 2. Télémétrie injectée dans la vidéo — « missing picture in access unit »

**Comportement DJI** : DJI Pilot 2 injecte en continu des paquets de télémétrie (coordonnées GPS, altitude, inclinaison nacelle, batterie) au milieu des trames vidéo sous forme de paquets H.264 SEI de 90 à 600 octets.

**Impact** : le décodeur de FFmpeg tente d'extraire des pixels de ces paquets et remplit les journaux d'erreurs : `missing picture in access unit` et `no frame!`.

**Correction** : c'est une fausse alerte non bloquante — FFmpeg ignore ces fragments et continue d'encoder la vidéo normale. En reconstruisant un conteneur H.264 propre, le transcodeur évite à PeerTube de recevoir ces paquets mal formés.

### 3. Absence d'accusés de réception RTMP — coupures après 60 à 90 secondes

**Comportement DJI** : DJI Pilot 2 attend des accusés de réception réguliers au niveau applicatif RTMP (paquets ACK). Par défaut, SRS est configuré sur `in_ack_size 0;` (aucun retour envoyé).

**Impact** : ne voyant aucun retour du serveur après un certain volume de données transmises, l'application du drone estime que la connexion est saturée ou figée. Elle ouvre une seconde connexion concurrente en secours, se heurte au rejet de SRS (`StreamBusy 1028`), puis coupe la liaison principale.

**Correction** : forcer l'envoi d'acquittements dans `srs.conf` :

```nginx
in_ack_size 2500000;
out_ack_size 2500000;
```

### 4. Retard cumulatif et buffer saturé — « speed < 1.0x »

**Comportement DJI / serveur** : en raison du double encodage (FFmpeg dans SRS + FFmpeg dans PeerTube), le processeur n'arrive pas à traiter le direct à la vitesse de capture réelle (`speed=0.63x` à `0.89x`).

**Impact** : le serveur accumule entre vingt et trente secondes de décalage par minute de vol (retard de plusieurs minutes affiché sur le lecteur). La mémoire tampon de SRS finit par saturer, provoquant la déconnexion du drone.

**Correction** — allègement drastique de la charge de calcul dans SRS :

```nginx
vprofile  baseline;      # supprime le décodage lourd CABAC et les images B
vpreset   ultrafast;     # préréglage ultra-rapide
vparams { tune zerolatency; }
vwidth   0;              # pas de redimensionnement logiciel
vheight  0;
```

**Résultat** : la vitesse d'encodage se verrouille à **1.0x** constant.

### 5. Absence ou instabilité de la piste audio — écrans noirs du lecteur

**Comportement DJI** : selon les configurations, la télécommande n'envoie aucune piste audio ou diffuse le son parasite de son microphone d'ambiance.

**Impact** : si l'audio est totalement supprimé avec l'option `-an`, les navigateurs web et le moteur HLS de PeerTube peinent à synchroniser l'horloge de lecture, ce qui déclenche des temps de chargement infinis ou un écran noir.

**Correction** : forcer la génération d'un **silence numérique AAC** :

```nginx
acodec       aac;
abitrate     64;
asample_rate 44100;
achannels    2;
```

Le lecteur web détecte une piste audio standard valide, synchronise instantanément la lecture, et le direct démarre sans bruit parasite.

## Utilisation

### Mettre un direct en ligne

1. Créer un live dans PeerTube → récupérer la clé de stream
2. Publier le flux depuis DJI Pilot 2 ou OBS : `rtmp://<serveur>:1935/live/<clé>`
3. SRS transcode à la volée (moteur `dji_fix`) puis réinjecte dans PeerTube : `rtmp://peertube:1935/live/<stream>`

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
