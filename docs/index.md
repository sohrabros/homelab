# GoozLab — Privacy-First Home Infrastructure

<div align="center">
<em>Self-hosted homelab built on open-source software and <a href="https://wiki.futo.org/index.php/Introduction_to_a_Self_Managed_Life:_a_13_hour_%26_28_minute_presentation_by_FUTO_software">FUTO principles</a>.<br>
Replacing cloud dependencies with local-first infrastructure while contributing to global internet freedom.</em>
</div>

---

## What's Inside

<div class="grid cards" markdown>

-   :material-shield-lock:{ .lg .middle } **OPNsense Firewall**

    ---

    Stateful firewall with 6-VLAN segmentation, Suricata IPS (Hyperscan), WireGuard VPN, and Quad9 DNS-over-TLS — running on a ZimaBoard 2.

    [:octicons-arrow-right-24: Setup Guide](setup/opnsense.md)

-   :material-server-network:{ .lg .middle } **Proxmox Cluster**

    ---

    Three-node cluster on HP EliteDesk 800 G3 Minis with 32GB RAM each, running all services as LXC containers and VMs with full quorum.

    [:octicons-arrow-right-24: Setup Guide](setup/proxmox.md)

-   :material-harddisk:{ .lg .middle } **RAID 5 NAS**

    ---

    Raspberry Pi 5 with Radxa SATA HAT, 4× Kingston 894GB SSDs in RAID 5 under OpenMediaVault 7. NFS/SMB shared to the Proxmox cluster.

    [:octicons-arrow-right-24: Setup Guide](setup/nas.md)

-   :material-earth:{ .lg .middle } **Internet Freedom**

    ---

    Psiphon Conduit proxy fleet across 6 nodes serving ~1,000 concurrent users in censored regions.

    [:octicons-arrow-right-24: Learn More](humanitarian/psiphon-conduit.md)

-   :material-chart-box:{ .lg .middle } **Full Observability**

    ---

    Prometheus metrics, Grafana dashboards, InfluxDB, Uptime Kuma, and a Homepage dashboard for complete infrastructure visibility.

    [:octicons-arrow-right-24: Monitoring Stack](setup/monitoring.md)

-   :material-cctv:{ .lg .middle } **Frigate NVR**

    ---

    AI-powered camera system with 4 PoE cameras, Intel iGPU VAAPI decode, OpenVINO detection, tiered retention, and mobile push notifications via Home Assistant.

    [:octicons-arrow-right-24: Setup Guide](setup/frigate-nvr.md)

-   :material-home-automation:{ .lg .middle } **Home Assistant**

    ---

    Smart home hub integrating Frigate cameras, FoxESS solar monitoring, Tigo optimizers, and MQTT.

    [:octicons-arrow-right-24: Setup Guide](setup/home-assistant.md)

-   :material-web:{ .lg .middle } **Caddy Reverse Proxy**

    ---

    Auto-HTTPS via Let's Encrypt with Cloudflare DNS-01 challenge. 10 services behind TLS at goozlab.net.

    [:octicons-arrow-right-24: Setup Guide](setup/caddy.md)

</div>

---

## Infrastructure at a Glance

| Component | Hardware / Platform | Role |
|-----------|-------------------|------|
| OPNsense | ZimaBoard 2 (N150, 16GB) | Firewall, VPN, DNS, IPS |
| pve1 | HP EliteDesk G3 (i7-6700T, 32GB) | Proxmox compute node |
| pve2 | HP EliteDesk G3 (i7-6700T, 32GB) | Proxmox compute node |
| pve3 | HP EliteDesk G3 (i7-7700, 32GB) | Proxmox compute node |
| NAS | Raspberry Pi 5 (4GB) + Radxa SATA HAT | OMV7, RAID 5, NFS/SMB |
| Switches | USW-Lite-16-PoE, USW-Lite-8-PoE, Flex Mini 2.5G | VLAN-aware switching |
| Wireless | 2× UniFi NanoHD | Dual-band WiFi |
| Cameras | 3× Reolink RLC-510A + 1× PoE Doorbell | Frigate NVR with AI detection |

---

## Project Status

| Phase | Status |
|-------|--------|
| OPNsense firewall + PPPoE | ✅ Complete |
| VLAN segmentation (6 VLANs) | ✅ Complete |
| Proxmox cluster (3 nodes, 32GB each) | ✅ Complete |
| Pi 5 NAS (RAID 5, NFS/SMB) | ✅ Complete |
| UniFi switching + wireless | ✅ Complete |
| Monitoring (Prometheus/Grafana/InfluxDB/Uptime Kuma) | ✅ Complete |
| Homepage dashboard | ✅ Complete |
| WireGuard VPN (multi-peer) | ✅ Complete |
| Suricata IPS (Hyperscan, LAN/igc0) | ✅ Complete — tuning in Alert mode |
| DNS hardening (Quad9 DoT, DNSSEC, DNSBL) | ✅ Complete |
| Caddy reverse proxy (10 services, auto-HTTPS) | ✅ Complete |
| Camera system (Frigate NVR, 4 cameras) | ✅ Complete |
| Home Assistant (Frigate, Solar, MQTT) | ✅ Complete |
| Frigate mobile notifications (HA blueprint) | ✅ Complete |
| Psiphon Conduit fleet (6 nodes) | ✅ Operational |
| Ansible fleet management | 📋 Planned |
| Tor Snowflake + Watchtower (Conduit fleet) | 📋 Planned |
| Conduit Fleet Intelligence Dashboard | 📋 Planned — Phase 1 |
| Wazuh SOC | 📋 Planned — Phase 2 |
| Ollama + Open WebUI (local AI) | 📋 Planned — Phase 3 |
| n8n workflow automation | 📋 Planned — Phase 4 |
| Immich (photo management) | 📋 Planned |
| Syncthing (phone backup) | 📋 Planned |
| Suricata: Alert → Drop mode | 📋 Planned |

---

## Network Design

```
Internet ──► OPNsense (ZimaBoard 2)
              │
              ├── VLAN 10: Management    (10.0.10.x) — Proxmox, NAS, UniFi, Monitoring
              ├── VLAN 20: Trusted       (10.0.20.x) — Workstations, laptops
              ├── VLAN 30: IoT           (10.0.30.x) — Cameras, smart devices
              ├── VLAN 50: Guest         (10.0.50.x) — Isolated guest WiFi
              ├── VLAN 60: WireGuard     (10.0.60.x) — VPN peers
              └── VLAN 70: Conduit       (10.0.70.x) — Humanitarian proxy (fully isolated)
```

---

## Key Principles

- **FUTO-aligned:** Local control, no cloud dependency, open-source software
- **Defense in depth:** VLAN isolation, Suricata IPS, CrowdSec, DNSBL, WireGuard encryption
- **Humanitarian purpose:** Dedicated proxy fleet serving users in censored regions
- **Documentation-first:** Every build, every mistake, every lesson — documented publicly

---

<div align="center">
<a href="https://github.com/sohrabros/homelab">View on GitHub</a> · 
<a href="setup/opnsense/">Get Started</a> · 
<a href="humanitarian/psiphon-conduit/">Internet Freedom</a>
</div>
