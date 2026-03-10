# Docker Services

## The Pattern: Docker Host LXC

Rather than creating a separate VM for each service, GoozLab uses a consistent pattern: **privileged LXC containers with Docker installed inside.**

```
┌───────────────────────────────────┐
│  Proxmox Node                     │
│                                   │
│  ┌─────────────────────────────┐  │
│  │  LXC Container              │  │
│  │  • Debian 12 minimal        │  │
│  │  • Docker + docker-compose  │  │
│  │  • nesting=1, keyctl=1      │  │
│  │                             │  │
│  │  ┌─────────┐ ┌─────────┐   │  │
│  │  │ Service │ │ Service │   │  │
│  │  │  (Docker│ │  (Docker│   │  │
│  │  │  cont.) │ │  cont.) │   │  │
│  │  └─────────┘ └─────────┘   │  │
│  └─────────────────────────────┘  │
└───────────────────────────────────┘
```

### Why this pattern?

- **Lighter than VMs:** LXCs share the host kernel — ~50MB overhead vs ~512MB+ for a VM
- **Docker-compose for everything:** Services are defined in YAML, version-controlled, and reproducible
- **Easy backups:** Snapshot the entire LXC to capture both the Docker host and its data volumes
- **VLAN isolation:** Each LXC can be placed on a different VLAN via its network tag

### How to create a Docker Host LXC

```bash
# Create the LXC
pct create <ID> <template> \
  --hostname <service-name> \
  --cores 2 --memory 2048 \
  --rootfs local-zfs:16 \
  --net0 name=eth0,bridge=vmbr0,tag=<VLAN> \
  --features nesting=1,keyctl=1 \
  --unprivileged 0

# Start and enter
pct start <ID>
pct enter <ID>

# Install Docker
apt update && apt install -y curl
curl -fsSL https://get.docker.com | sh

# Install docker-compose
apt install -y docker-compose-plugin
```

### Standard docker-compose structure

Each service directory follows the same layout:

```
/opt/<service-name>/
├── docker-compose.yml
├── .env               # Environment variables (not committed to git)
├── .env.example        # Template showing required variables
└── data/               # Persistent data (bind-mounted volumes)
```

## Deploying a Service

```bash
cd /opt/<service-name>
cp .env.example .env
# Edit .env with your values
nano .env

# Start
docker compose up -d

# Check logs
docker compose logs -f
```

## Updating Services

```bash
cd /opt/<service-name>
docker compose pull
docker compose up -d
```

## Current Services

| Service | Host | Type | Purpose | Docs |
|---------|------|------|---------|------|
| UniFi Controller | Docker Host LXC on pve1 | LXC | Network management | [Switching & Wireless](switching-wireless.md) |
| Monitoring Stack | Docker Host LXC on pve1 | LXC | Prometheus + Grafana + Uptime Kuma + Homepage | [Monitoring](monitoring.md) |
| Frigate NVR | Docker Host LXC on pve1 | LXC | Camera recording + AI detection (4 cameras) | [Frigate](frigate.md) |
| Psiphon Conduit | Docker Host LXC on pve2 | LXC | Internet freedom proxy | [Psiphon Conduit](../humanitarian/psiphon-conduit.md) |
| Home Assistant | VM on pve2 | VM | Smart home — Frigate, solar, MQTT | [Home Assistant](home-assistant.md) |

> **Note:** Home Assistant runs as a HAOS VM (not a Docker Host LXC) because HAOS requires its own kernel and provides a managed add-on ecosystem.

## Planned Services

| Service | Purpose | Notes |
|---------|---------|-------|
| Jellyfin | Media streaming | Google Photos / Netflix replacement |
| Immich | Photo management | FUTO-sponsored project |
| Vaultwarden | Password management | Bitwarden-compatible, self-hosted |
| Syncthing | File sync and phone backup | Google/iCloud replacement |
