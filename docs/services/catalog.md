# Service Catalog

## Active Services

| Service | Purpose | Status |
|---------|---------|--------|
| OPNsense | Firewall, routing, DNS, VPN, IDS/IPS | ✅ Production |
| Proxmox VE | Virtualisation and container management (3-node cluster) | ✅ Production |
| OpenMediaVault 7 | NAS — file sharing (NFS/SMB), RAID management | ✅ Production |
| UniFi Controller | Switch and AP management | ✅ Production |
| Prometheus | Metrics collection | ✅ Production |
| Grafana | Dashboards and visualisation | ✅ Production |
| Uptime Kuma | Service availability monitoring | ✅ Production |
| Homepage | Service dashboard at dash.goozlab.net | ✅ Production |
| Caddy | Reverse proxy — auto-HTTPS for 10 services via Cloudflare DNS-01 | ✅ Production |
| Suricata | Network IDS (Hyperscan, igc0/LAN) — tuning in Alert mode | ✅ Production |
| CrowdSec | Collaborative threat intelligence and IP blocking | ✅ Production |
| Psiphon Conduit | Internet freedom proxy (6-node fleet) | ✅ Production |
| Frigate NVR | Camera recording + AI detection (4 cameras, OpenVINO) | ✅ Production |
| Home Assistant | Smart home — Frigate integration, solar monitoring, MQTT | ✅ Production |

## Planned Services

These follow the established [Docker Host LXC pattern](../setup/docker-services.md):

| Service | Purpose | FUTO Alignment |
|---------|---------|----------------|
| Jellyfin | Media streaming server | Replaces Netflix/Plex cloud |
| Immich | Photo management | Replaces Google Photos (FUTO-sponsored!) |
| Vaultwarden | Password manager | Replaces cloud-based Bitwarden |
| Syncthing | Phone backup and file sync | Replaces Google/iCloud backup |
| Wazuh | Security operations centre (SOC) | Centralised security monitoring |
| Ollama + Open WebUI | Local AI inference | Replaces cloud AI dependencies |
| n8n | Workflow automation | Connects HA, Frigate, Wazuh, Ollama |

## Deployment Pattern

All containerised services use the Docker Host LXC architecture, with the exception of Home Assistant which runs as a dedicated HAOS VM on Proxmox. See [Docker Services](../setup/docker-services.md) for the deployment pattern and conventions.
