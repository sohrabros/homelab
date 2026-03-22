# Service Catalog

## Active Services

| Service | Purpose | Status |
|---|---|---|
| OPNsense | Firewall, routing, DNS, VPN, IDS/IPS | :material-check-circle:{ .green } Production |
| Proxmox VE | Virtualisation and container management (3-node cluster) | :material-check-circle:{ .green } Production |
| OpenMediaVault 7 | NAS — file sharing (NFS/SMB), RAID management | :material-check-circle:{ .green } Production |
| UniFi Controller | Switch and AP management | :material-check-circle:{ .green } Production |
| Prometheus | Metrics collection — 30 scrape targets across all infrastructure | :material-check-circle:{ .green } Production |
| Grafana | Dashboards — Command Center (v2) + Freedom Fleet | :material-check-circle:{ .green } Production |
| InfluxDB | Time-series storage for fleet and long-term metrics | :material-check-circle:{ .green } Production |
| Uptime Kuma | Service availability monitoring | :material-check-circle:{ .green } Production |
| Homepage | Service dashboard at dash.goozlab.net | :material-check-circle:{ .green } Production |
| Frigate Exporter | Custom Python exporter — camera FPS, detection, inference metrics | :material-check-circle:{ .green } Production |
| Blackbox Exporter | ICMP ping monitoring for IoT VLAN cameras | :material-check-circle:{ .green } Production |
| SNMP Exporter | UniFi switch and AP metrics via SNMP v2c | :material-check-circle:{ .green } Production |
| Caddy | Reverse proxy — auto-HTTPS for services via Cloudflare DNS-01 | :material-check-circle:{ .green } Production |
| Suricata | Network IDS (Hyperscan, igc0/LAN) — tuning in Alert mode | :material-check-circle:{ .green } Production |
| CrowdSec | Collaborative threat intelligence and IP blocking | :material-check-circle:{ .green } Production |
| Psiphon Conduit | Internet freedom proxy (6-node fleet, shirokhorshid compartment) | :material-check-circle:{ .green } Production |
| Tor Snowflake | Additional circumvention bridge — deployed on all Conduit nodes | :material-check-circle:{ .green } Production |
| Watchtower | Automated Docker container updates — all Conduit nodes | :material-check-circle:{ .green } Production |
| Frigate NVR | Camera recording + AI detection (4 cameras, OpenVINO) | :material-check-circle:{ .green } Production |
| Home Assistant | Smart home — Frigate, solar monitoring, MQTT, notifications | :material-check-circle:{ .green } Production |

## Planned Services

These follow the established [Docker Host LXC pattern](setup/docker-services.md):

| Service | Purpose | FUTO Alignment |
|---|---|---|
| Wazuh | Security operations centre (SOC) | Centralised security monitoring |
| Ollama + Open WebUI | Local AI inference | Replaces cloud AI dependencies |
| n8n | Workflow automation | Connects HA, Frigate, Wazuh, Ollama |
| Jellyfin | Media streaming server | Replaces Netflix/Plex cloud |
| Immich | Photo management | Replaces Google Photos (FUTO-sponsored!) |
| Vaultwarden | Password manager | Replaces cloud-based Bitwarden |
| Syncthing | Phone backup and file sync | Replaces Google/iCloud backup |

## Deployment Pattern

All containerised services use the Docker Host LXC architecture, with the exception of Home Assistant which runs as a dedicated HAOS VM on Proxmox. See [Docker Services](setup/docker-services.md) for the deployment pattern and conventions.
