# OPNsense Setup

## Hardware

| Component | Specification |
|-----------|--------------|
| Board | ZimaBoard 2 (Model 1664) |
| CPU | Intel N150 (4 cores, up to 3.6GHz) |
| RAM | 16GB LPDDR5x |
| Storage | 64GB eMMC |
| Network | Dual Intel 2.5GbE NICs |
| Power | ~6W TDP |

The dual 2.5GbE NICs are critical — one for WAN, one for LAN. No need for VLAN-on-a-stick single-NIC setups.

## Installation

1. Download OPNsense AMD64 image from [opnsense.org](https://opnsense.org/download/)
2. Flash to USB drive using Rufus or balenaEtcher
3. Connect a monitor and keyboard to the ZimaBoard (or use JetKVM for remote BIOS)
4. Boot from USB and follow the installer — select ZFS on the eMMC for the root filesystem

## Initial Configuration

After install, connect to the web UI from a device on the LAN side:

1. **Set hostname and domain:** Something like `fw1.goozlab.net`
2. **WAN interface:** Configure PPPoE with ISP credentials (or DHCP if your ISP provides a modem with routing disabled)
3. **LAN interface:** Set your management subnet gateway IP
4. **DNS:** Change upstream servers to Quad9 with DNS-over-TLS (see below)

## DNS-over-TLS with Unbound

OPNsense's built-in Unbound resolver handles all DNS. Configuration:

- **Services → Unbound DNS → General:** Enable, listen on all VLAN interfaces
- **Services → Unbound DNS → DNS over TLS:** Enable, add Quad9 servers (`9.9.9.9` port 853, `149.112.112.112` port 853)
- **System → Settings → General:** Set DNS servers to Quad9 with "Use DNS over TLS" ticked
- **Services → Unbound DNS → Blocklists:** Enable OISD and Hagezi lists for ad/tracker blocking
- **Services → Unbound DNS → DNSSEC:** Enable validation

### Force all DNS through Unbound

Some devices (Google Home, Chromecast) hardcode DNS servers like `8.8.8.8`. Create a NAT port forward rule:

- **Firewall → NAT → Port Forward:** Redirect all outbound port 53 traffic to OPNsense's Unbound. This ensures every device on every VLAN uses your local resolver regardless of their DNS settings.

## VLAN Interfaces

Create one VLAN interface per segment under **Interfaces → Other Types → VLAN:**

| VLAN Tag | Parent | Interface Name | Purpose |
|----------|--------|----------------|---------|
| 10 | LAN | MGMT | Management |
| 20 | LAN | TRUSTED | Personal devices |
| 30 | LAN | IOT | Cameras, smart devices |
| 50 | LAN | GUEST | Visitor WiFi |
| 60 | LAN | WIREGUARD | VPN tunnel |
| 70 | LAN | CONDUIT | Psiphon proxy |

Assign each VLAN interface and configure its gateway IP as documented in [Network Design](../architecture/network-design.md).

## Firewall Rules

Implement the inter-VLAN traffic matrix from [VLAN Segmentation](../architecture/vlan-segmentation.md). Key principles:

- **Default deny** on every VLAN interface
- **Explicit allow** rules for permitted traffic
- **IoT and Guest VLANs** get a block rule for all RFC1918 ranges before the internet allow rule
- **Conduit VLAN** blocks everything except outbound internet

See [Firewall Rules](../security/firewall-rules.md) for the complete ruleset design.

## WireGuard VPN

1. **VPN → WireGuard → Instances:** Create a tunnel with a generated key pair
2. Assign the WireGuard interface to VLAN 60's subnet
3. Create peers for each client device (phone, laptop)
4. **Cloudflare DDNS:** Under **Services → Dynamic DNS**, configure Cloudflare API to update your public hostname automatically
5. Port forward the WireGuard UDP port from WAN to the tunnel

## Where FUTO Diverges

The [FUTO guide by Louis Rossmann](https://wiki.futo.org/index.php/Introduction_to_a_Self_Managed_Life:_a_13_hour_%26_28_minute_presentation_by_FUTO_software) uses pfSense, but Rossmann himself recommends OPNsense for new users. The guide also uses OpenVPN — WireGuard is the modern alternative with better performance and simpler configuration.

The FUTO guide uses pfBlockerNG for ad blocking. OPNsense's equivalent is Unbound's built-in blocklist feature, which achieves the same result without an additional plugin.
