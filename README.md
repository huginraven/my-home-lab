# 🏠 my-home-lab

A modular homelab setup running on an Oracle Cloud VPS using Docker Compose and managed via [Dockhand](https://github.com/fnsys/dockhand).

Each service lives in its own folder inside `stacks/` with an independent `compose.yaml` and `.env.example` file. All public-facing containers communicate through a shared reverse proxy network (`proxy_net`).

## 📁 Stacks Overview

```text
stacks/
├── swag/          # Reverse proxy (Nginx, Cloudflare DNS-01 SSL, HTTP/3, CrowdSec bouncer)
├── crowdsec/      # Intrusion prevention system & log monitor
├── dockhand/      # Web UI for Docker container & stack management (GitOps)
├── homepage/      # Application dashboard & server status
├── vaultwarden/   # Self-hosted password vault (Bitwarden compatible)
├── decypharr/     # Debrid FUSE mount provider (/mnt:rshared)
├── radarr/        # Movie collection manager
├── sonarr/        # TV series collection manager
├── prowlarr/      # Indexer integration for *arr apps
├── bazarr/        # Automated subtitle downloader
└── recyclarr/     # TRaSH Guides quality profiles sync (cron)
```

## 🚀 Deployment Workflow

### 1. Prerequisites & Directory Setup
Prepare base data directories on your VPS and clone this repo:
```bash
# Create persistent data directories
mkdir -p /opt/appdata/config /opt/appdata/data

# Clone repository
git clone <repo-url> /opt/my-home-lab
```

Create the shared external Docker network:
```bash
docker network create proxy_net
```

### 2. Bootstrap (CLI)
Start the core security layer and Dockhand:

1. **SWAG** (Reverse proxy):
   ```bash
   cd /opt/my-home-lab/stacks/swag
   cp .env.example .env && nano .env
   docker compose up -d
   ```

2. **CrowdSec** (Intrusion Prevention System):
   ```bash
   cd /opt/my-home-lab/stacks/crowdsec
   cp .env.example .env && nano .env
   docker compose up -d
   ```

3. **Dockhand** (Management UI):
   ```bash
   cd /opt/my-home-lab/stacks/dockhand
   cp .env.example .env && nano .env
   docker compose up -d
   ```

### 3. Manage & Deploy via Dockhand (UI)
Once Dockhand is up:
1. Open the Dockhand web interface.
2. Link your Git repository / stacks directory (`/opt/my-home-lab/stacks`).
3. Deploy, configure `.env` variables, and manage all remaining stacks (`Homepage`, `Vaultwarden`, `*arr` media suite) directly from the UI.

---

## 🎬 Media Pipeline & Paths Layout (*arr + Decypharr)

To ensure seamless interaction between Decypharr (blackhole + FUSE debrid mount) and the `*arr` applications, the following folder structure is used on the host:

### Host Directory Structure
```bash
# Create all required *arr and Decypharr directories in one go:
mkdir -p /opt/appdata/config/{bazarr,prowlarr,radarr,recyclarr,sonarr}
mkdir -p /opt/appdata/config/decypharr/downloads/{radarr,sonarr}
mkdir -p /opt/appdata/data/{decypharr/mnt,radarr/movies,sonarr/tvseries}
```

```text
/opt/appdata/
├── config/
│   ├── decypharr/downloads/
│   │   ├── radarr/        # Blackhole watch folder for Radarr
│   │   └── sonarr/        # Blackhole watch folder for Sonarr
│   ├── radarr/            # Radarr configuration
│   ├── sonarr/            # Sonarr configuration
│   ├── bazarr/            # Bazarr configuration
│   ├── prowlarr/          # Prowlarr configuration
│   └── recyclarr/         # Recyclarr configuration
└── data/
    ├── decypharr/mnt/     # Debrid FUSE mount point (:rshared -> :rslave)
    ├── radarr/movies/     # Movies library root folder
    └── sonarr/tvseries/   # TV series library root folder
```

### Volume Mappings Quick Reference

| Service | Container Path | Host Path | Purpose |
| :--- | :--- | :--- | :--- |
| **Decypharr** | `/app`<br>`/mnt` | `${CONFIG_PATH}/decypharr`<br>`${DATA_PATH}/decypharr/mnt` | App config & blackhole downloads<br>FUSE virtual mount point (`rshared`) |
| **Radarr** | `/config`<br>`/movies`<br>`/app/downloads/radarr`<br>`/mnt` | `${CONFIG_PATH}/radarr`<br>`${DATA_PATH}/radarr/movies`<br>`${CONFIG_PATH}/decypharr/downloads/radarr`<br>`${DATA_PATH}/decypharr/mnt` | App config<br>Root movies library<br>Blackhole download folder<br>FUSE mount access (`rslave`) |
| **Sonarr** | `/config`<br>`/tv`<br>`/app/downloads/sonarr`<br>`/mnt` | `${CONFIG_PATH}/sonarr`<br>`${DATA_PATH}/sonarr/tvseries`<br>`${CONFIG_PATH}/decypharr/downloads/sonarr`<br>`${DATA_PATH}/decypharr/mnt` | App config<br>Root TV series library<br>Blackhole download folder<br>FUSE mount access (`rslave`) |
| **Bazarr** | `/config`<br>`/movies`<br>`/tv`<br>`/mnt` | `${CONFIG_PATH}/bazarr`<br>`${DATA_PATH}/radarr/movies`<br>`${DATA_PATH}/sonarr/tvseries`<br>`${DATA_PATH}/decypharr/mnt` | App config<br>Access to movies<br>Access to TV series<br>FUSE mount access (`rslave`) |

---

## ⚙️ Architecture & Mount Details

- **Reverse Proxy**: SWAG manages SSL wildcard certs via Cloudflare DNS validation and routes subdomains to containers on `proxy_net`.
- **FUSE Mount Propagation**: Decypharr runs with `SYS_ADMIN` capabilities and exports mounts with `:rshared`. Consuming media apps (`Radarr`, `Sonarr`, `Bazarr`) mount the shared directory using `:rslave`.
- **Secrets**: Environment files (`.env`) are excluded from git.

## 📄 License
Distributed under the [MIT License](LICENSE).
