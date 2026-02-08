# Architecture Overview

GoozLab's architecture follows the layered approach recommended in [Louis Rossmann's FUTO guide](https://wiki.futo.org/index.php/Introduction_to_a_Self_Managed_Life:_a_13_hour_%26_28_minute_presentation_by_FUTO_software) — build the network foundation first, then compute, then storage, then services on top.

## Layers

### 1. Network Foundation

**OPNsense on ZimaBoard 2** handles all routing, firewalling, DNS, VPN, and intrusion detection. This is the single most critical piece of infrastructure — everything else depends on it.

The ZimaBoard 2's dual 2.5GbE Intel NICs provide clean WAN/LAN separation without VLAN-on-a-stick. WAN connects to the ISP via PPPoE; LAN trunks all VLANs to the core switch.

**Seven VLANs** segment the network by trust level. See [VLAN Segmentation](vlan-segmentation.md) for the full design.

**UniFi switching and wireless** provides managed switching with 802.1Q VLAN trunking and enterprise wireless with per-SSID VLAN tagging. See [Network Design](network-design.md) for the physical topology.

### 2. Compute

**Two-node Proxmox cluster** on HP EliteDesk 800 G3 Mini PCs (i7-6700, 32GB RAM each). Runs all services as LXC containers — lightweight, efficient, and fast to deploy. The nodes are clustered for unified management but services are manually distributed rather than using HA (two nodes can't achieve quorum for automatic failover).

### 3. Storage

**Raspberry Pi 5 NAS** running OpenMediaVault 7 with a Radxa Penta SATA HAT holding four SSDs in RAID 5. Provides ~2.6TB usable storage via NFS (for Proxmox) and SMB (for workstations). See the [NAS Setup Guide](../setup/nas.md) and the [power supply lesson](../lessons-learned/pi-nas-power.md) before you try this yourself.

### 4. Services

All services run as Docker containers inside **Docker Host LXC containers** on Proxmox. Each logical service group gets its own LXC with Docker and docker-compose installed. This pattern keeps things isolated, reproducible, and easy to back up. See [Docker Services](../setup/docker-services.md) for the deployment pattern.

### 5. Security

Defence in depth: OPNsense firewall rules as the first layer, Suricata IDS/IPS for traffic inspection, CrowdSec for collaborative threat intelligence, and VLAN segmentation to limit blast radius. See [Security Hardening](../security/hardening.md).

### 6. Monitoring

Prometheus scrapes metrics from all infrastructure, Grafana provides dashboards, and Uptime Kuma monitors service availability. See [Monitoring](../setup/monitoring.md).

## Design Decisions

**Why OPNsense over pfSense?** Louis Rossmann's FUTO guide actually uses pfSense, but he explicitly recommends OPNsense for new users. The feature sets are nearly identical (both FreeBSD-based), and OPNsense has a more modern UI and more frequent updates.

**Why Proxmox over bare-metal Docker?** The FUTO guide runs Docker on bare metal, which is simpler. I chose Proxmox because it supports both LXC containers and VMs, provides a web UI for management, and enables clustering across two nodes. The overhead is minimal and the flexibility is worth it.

**Why LXC containers over VMs for services?** LXCs share the host kernel, so they use far less RAM and start instantly compared to full VMs. For Docker workloads that don't need their own kernel, LXC is the right choice.

**Why a Pi 5 for the NAS?** It was hardware I already owned. The Radxa SATA HAT gives native PCIe SATA connectivity (not USB), and the whole thing draws about 15W running 24/7. For a homelab NAS serving a Proxmox cluster over Gigabit Ethernet, it saturates the link with plenty of headroom. See the [honest assessment](../setup/nas.md) for limitations.

**Why WireGuard over OpenVPN?** The FUTO guide recommends self-hosted VPN (which I agree with), but uses OpenVPN. WireGuard is faster, simpler to configure, and has a smaller attack surface. OPNsense has native WireGuard support.
