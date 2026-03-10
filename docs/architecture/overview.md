# Architecture Overview

GoozLab's architecture follows the layered approach recommended in [Louis Rossmann's FUTO guide](https://wiki.futo.org/index.php/Introduction_to_a_Self_Managed_Life:_a_13_hour_%26_28_minute_presentation_by_FUTO_software) — build the network foundation first, then compute, then storage, then services on top.

## Layers

### 1. Network Foundation

**OPNsense on ZimaBoard 2** handles all routing, firewalling, DNS, VPN, and intrusion detection. This is the single most critical piece of infrastructure — everything else depends on it.

The ZimaBoard 2's dual 2.5GbE Intel NICs provide clean WAN/LAN separation without VLAN-on-a-stick. WAN connects to the ISP via PPPoE; LAN trunks all VLANs to the core switch.

**Six VLANs** segment the network by trust level. See [VLAN Segmentation](vlan-segmentation.md) for the full design.

**UniFi switching and wireless** provides managed switching with 802.1Q VLAN trunking and enterprise wireless with per-SSID VLAN tagging. See [Network Design](network-design.md) for the physical topology.

### 2. Compute

**Three-node Proxmox cluster** on HP EliteDesk 800 G3 Mini PCs. Nodes 1 and 2 run i7-6700T (35W) with 32GB RAM each; Node 3 runs i7-7700 (65W) with 32GB RAM. All services run as LXC containers and VMs — lightweight, efficient, and fast to deploy. With three nodes, the cluster has proper quorum for unified management and can tolerate one node going down without losing cluster functionality.

### 3. Storage

**Raspberry Pi 5 NAS** running OpenMediaVault 7 with a Radxa Penta SATA HAT holding four SSDs in RAID 5. Provides ~2.6TB usable storage via NFS (for Proxmox) and SMB (for workstations). See the [NAS Setup Guide](../setup/nas.md) and the [power supply lesson](../lessons-learned/pi-nas-power.md) before you try this yourself.

### 4. Services

All services run as Docker containers inside **Docker Host LXC containers** on Proxmox, with the exception of Home Assistant which runs as a dedicated VM. Each logical service group gets its own LXC with Docker and docker-compose installed. This pattern keeps things isolated, reproducible, and easy to back up. See [Docker Services](../setup/docker-services.md) for the deployment pattern.

Key services include Frigate NVR (AI-powered camera system with 4 PoE cameras), Home Assistant (smart home hub with solar monitoring), Caddy reverse proxy (auto-HTTPS for 10 services), Homepage dashboard, and the Psiphon Conduit proxy fleet.

### 5. Security

Defence in depth: OPNsense firewall rules as the first layer, Suricata IDS for traffic inspection (running on igc0 — the LAN parent of all VLANs — in Alert mode for tuning), CrowdSec for collaborative threat intelligence, and VLAN segmentation to limit blast radius. See [Security Hardening](../security/hardening.md).

### 6. Monitoring

Prometheus scrapes metrics from all infrastructure, Grafana provides dashboards, and Uptime Kuma monitors service availability. A Homepage dashboard at dash.goozlab.net provides a quick overview of all services. See [Monitoring](../setup/monitoring.md).

## Design Decisions

**Why OPNsense over pfSense?** Louis Rossmann's FUTO guide actually uses pfSense, but he explicitly recommends OPNsense for new users. The feature sets are nearly identical (both FreeBSD-based), and OPNsense has a more modern UI and more frequent updates.

**Why Proxmox over bare-metal Docker?** The FUTO guide runs Docker on bare metal, which is simpler. I chose Proxmox because it supports both LXC containers and VMs, provides a web UI for management, and enables clustering across three nodes. The overhead is minimal and the flexibility is worth it.

**Why LXC containers over VMs for services?** LXCs share the host kernel, so they use far less RAM and start instantly compared to full VMs. For Docker workloads that don't need their own kernel, LXC is the right choice. Home Assistant is the exception — it runs as a VM because HAOS requires its own kernel.

**Why a Pi 5 for the NAS?** It was hardware I already owned. The Radxa SATA HAT gives native PCIe SATA connectivity (not USB), and the whole thing draws about 15W running 24/7. For a homelab NAS serving a Proxmox cluster over Gigabit Ethernet, it saturates the link with plenty of headroom. See the [honest assessment](../setup/nas.md) for limitations.

**Why WireGuard over OpenVPN?** The FUTO guide recommends self-hosted VPN (which I agree with), but uses OpenVPN. WireGuard is faster, simpler to configure, and has a smaller attack surface. OPNsense has native WireGuard support.

**Why Caddy over Nginx?** Caddy provides automatic HTTPS with zero configuration via Let's Encrypt and Cloudflare DNS-01 challenge. The OPNsense Caddy plugin makes it simple to manage. Nginx was replaced during the migration to simplify certificate management.
