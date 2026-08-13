# Saltspite Media Architecture

## Overview

This repository contains the declarative infrastructure for the Saltspite media stack, optimized specifically for the Synology DS923+. It is a fully automated, Usenet-exclusive media acquisition and presentation pipeline.

## Design Philosophy

This stack is built on strict engineering principles to maximize performance, ensure data integrity, and minimize system overhead on NAS hardware.

* **Atomic Hardlinks:** The file system is structurally unified under `/volumeX/data`. This eliminates I/O-intensive cross-volume copy operations. Media imports occur instantaneously via inode pointers.
* **Protocol Singularity:** BitTorrent and its associated peripheral services have been entirely deprecated in favor of a Usenet-only pipeline (SABnzbd). This reduces disk thrashing, eliminates the need for unpacking daemons, and accelerates acquisition.
* **Separation of Concerns:** Episodic television and Anime are segregated into two distinct Sonarr instances. This prevents metadata collisions and allows for highly specialized release profiling.
* **Principle of Least Privilege:** Security is enforced at the container level. The presentation layer (Plex/Jellyfin) is restricted to read-only (`:ro`) access to the media library, preventing accidental or malicious file deletion.
* **Hardware Awareness:** As the DS923+ lacks an iGPU for hardware acceleration, the stack relies strictly on software transcoding.

## Architectural Diagram

```text
[ Clients ] ---> (HTTP/S) ---> [ Caddy (Reverse Proxy) ]
                                      |
                                      v
                             [ Seerr (Requests) ]
                                      |
       +------------------------------+------------------------------+
       |                              |                              |
       v                              v                              v
  [ Radarr ]                   [ Sonarr-TV ]                [ Sonarr-Anime ] <--- [ Clonarr ]
       |                              |                              |          (TRaSH Sync)
       +--------------+---------------+---------------+--------------+
                      |                               |
                 (API Search)                    (Send NZB)
                      v                               v
                [ Prowlarr ]                     [ SABnzbd ] ---> (Extracts to /data/usenet/)
                      |                               |
                 (Indexers)                           +---> [ Autoscan ] ---> (API Trigger)
                                                      |                             |
                                                (Atomic Hardlink)                   v
                                                      |                         [ Plex ]
                                                      v                            |
                                              [ /data/media/ ] <--- (Read Only) ---+
                                                      ^                            |
                                                      |                       [ Tautulli ]
                                                 [ Bazarr ]

```

## Directory Structure

The host file system must be configured exactly as follows to ensure atomic operations and proper access control list (ACL) inheritance.

```text
/volumeX/
├── docker/
│   ├── appdata/           # Application config directories mapped to /config
│   ├── scripts/           # Host-level bash scripts (e.g., pullio.sh)
│   └── docker-compose.yml 
└── data/
    ├── usenet/            # SABnzbd working and complete directories
    └── media/             # Final library destinations
        ├── movies/
        ├── tv/
        └── anime/

```
To create, replace `/volumeX/` with the volume you wish to use
```bash
mkdir -p /volumeX/data/{usenet/{incomplete,complete}/{tv,movies,anime},media/{tv,movies,anime}}
mkdir -p /volumeX/docker/{appdata/{caddy,cloudflareddns,prowlarr,sabnzbd,radarr,sonarr-tv,sonarr-anime,bazarr,clonarr,seerr,plex,jellyfin,tautull,autoscan},scripts}
```

## Network Configuration

The stack utilizes a hybrid networking model to resolve broadcast limitations inherent to Docker.

1. **`mediacore` (Custom Bridge):** The secure internal subnet. Caddy, the Arrs, Prowlarr, and SABnzbd reside here, communicating exclusively via internal container hostnames (e.g., `http://sabnzbd:8080`).
2. **`host` (Native Synology Network):** Plex and Jellyfin operate on the host network. This exposes their ports directly to the local LAN, ensuring DLNA and local client discovery (GDM) function correctly without traversing Docker's internal NAT.

## Container Stack

*All primary images are sourced from the `hotio` registry to ensure unified user permission handling (`PUID`/`PGID`) and simplified update cycles.*

### Infrastructure & Access

* **Caddy:** Reverse proxy handling internal domain routing (`Saltspite.server`) and eventual public Cloudflare TLS termination.
* **Cloudflare DDNS:** Dynamic IP updater securely configured with a scoped Zone DNS API token.

### Indexing & Acquisition

* **Prowlarr:** Centralized Usenet indexer management.
* **SABnzbd:** Primary download, repair, and extraction engine.

### Automation & Logic

* **Radarr:** Movie management.
* **Sonarr (TV):** Standard episodic television management.
* **Sonarr (Anime):** Dedicated Anime management.
* **Bazarr:** Subtitle acquisition and synchronization.
* **Clonarr:** Automated synchronization of TRaSH Guides custom formats and quality profiles to the Arr stack.

### Frontend & Presentation

* **Seerr:** Unified user interface for media discovery and requests.
* **Plex:** Primary media server.
* **Jellyfin:** Secondary fallback media server (Configured to `disabled` by default to prevent I/O contention).
* **Tautulli:** Telemetry, logging, and analytical tracking for Plex.
* **Autoscan:** Replaces standard filesystem `inotify` watches. Triggers targeted Plex library scans upon SABnzbd extraction completion.

## Deployment Instructions

### 1. Pre-Requisites

1. Establish a dedicated `media` user and `docker-media` group via the Synology Control Panel.
2. Grant the group exact Read/Write permissions to `/volumeX/docker` and `/volumeX/data`.
3. SSH into the NAS and retrieve the numerical IDs: `id media`.

### 2. Environment Configuration

Create a `.env` file in the same directory as the `docker-compose.yml`. Populate all required values.

```env
# Identifiers
PUID=1024
PGID=100
TZ=Europe/Istanbul
UMASK=002

# Paths
DOCKERCONFDIR=/volumeX/docker/appdata
DOCKERSTORAGEDIR=/volumeX/data

# Logging
DOCKERLOGGING_MAXFILE=10
DOCKERLOGGING_MAXSIZE=200k

# Pullio
PULLIO_UPDATE=true
PULLIO_NOTIFY=true

# Security & Tokens
CF_USER=your_cloudflare_email
CF_APITOKEN=your_scoped_zone_dns_token
CF_HOSTS=misbehaviour.training,*.misbehaviour.training

# Plex Setup
PLEX_CLAIM_TOKEN=claim-xxxxxxxxxxxxxxxxx
PLEX_ADVERTISE_URL=http://<NAS_IP>:32400
PLEX_NO_AUTH_NETWORKS=192.168.1.0/24,172.18.0.0/16

```

### 3. Initialization

Download the `docker-compose.yml` to your `/volumeX/docker/appdata` location so you can get your important stuff together.

```bash
wget https://raw.githubusercontent.com/McHearty/saltspite.local/refs/heads/main/docker-compose.yml -P /volumeX/docker/appdata/
```
Download this `.env.example` to your `/volumeX/docker/appdata` location next to the docker-compose.yml.

```bash
wget https://raw.githubusercontent.com/McHearty/saltspite.local/refs/heads/main/.env.example -O /volumeX/docker/appdata/.env
```
Ensure user has permissions, changing `USER` to the user managing the stack, if you are importing an existing library you will need to run these commands again after import.
```bash
sudo chown -R USER:users /volumeX/data /volumeX/docker
```
```bash
sudo chmod -R a=,a+rX,u+w,g+w /volumeX/data /volumeX/docker
```
Run the Docker Compose
```bash
cd /volumeX/docker/appdata
```
```bash
sudo docker-compose up -d
```

### 4. Local DNS Resolution

To access the services via the internal Caddy routing, you must configure your local DNS server (e.g., Pi-hole, router) to point `*.Saltspite.server` to the Synology NAS's static LAN IP address.

Additionally, Synology automatically binds to ports 443 and 80 even if you’re not using their built-in reverse proxy. To get around this run the following, and set it up as a boot time task.
```bash
sed -i -e 's/80/82/' -e 's/443/444/' /usr/syno/share/nginx/server.mustache /usr/syno/share/nginx/DSM.mustache /usr/syno/share/nginx/WWWService.mustache

synosystemctl restart nginx
```

SABnzbd requires hostname whitelisting, you can find this in Special -> host_whitelist

### 5. Automated Updates (Pullio)

Container updates are completely automated via `pullio`.

1. Download the `pullio` bash script to `/volumeX/docker/scripts/`.
2. Create a Scheduled Task in the Synology Control Panel.
3. Set the task to run daily as `root` executing: `bash /volumeX/docker/scripts/pullio.sh`.
