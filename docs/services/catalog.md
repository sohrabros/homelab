# Service Catalog

## Active Services

| Service | Purpose | Status |
|---------|---------|--------|
| OPNsense | Firewall, routing, DNS, VPN, IDS/IPS | ✅ Production |
| Proxmox VE | Virtualisation and container management | ✅ Production |
| OpenMediaVault 7 | NAS — file sharing (NFS/SMB), RAID management | ✅ Production |
| UniFi Controller | Switch and AP management | ✅ Production |
| Prometheus | Metrics collection | ✅ Production |
| Grafana | Dashboards and visualisation | ✅ Production |
| Uptime Kuma | Service availability monitoring | ✅ Production |
| Psiphon Conduit | Internet freedom proxy | ✅ Production |
| Frigate NVR | Camera recording + AI detection | 🔶 Awaiting cameras |

## Planned Services

These follow the established [Docker Host LXC pattern](../setup/docker-services.md):

| Service | Purpose | FUTO Alignment |
|---------|---------|----------------|
| Jellyfin | Media streaming server | Replaces Netflix/Plex cloud |
| Immich | Photo management | Replaces Google Photos (FUTO-sponsored!) |
| Home Assistant | Smart home automation | Reduces IoT cloud dependencies |
| Vaultwarden | Password manager | Replaces cloud-based Bitwarden |

## Deployment Pattern

All containerised services use the Docker Host LXC architecture. See [Docker Services](../setup/docker-services.md) for the deployment pattern and conventions.
