# Switching & Wireless Setup

## UniFi Ecosystem

The entire switching and wireless infrastructure uses Ubiquiti UniFi equipment, managed through a single UniFi Network Controller running as an LXC container on Proxmox.

### Why UniFi?

The FUTO guide recommends TP-Link Omada for switching and wireless. I went with UniFi because:

- Single pane of glass for all switches and APs
- Better VLAN management across the full stack
- Strong second-hand market (the NanoHDs were eBay purchases)
- Mature controller software with good visibility into network health

The trade-off is vendor lock-in within the switching/wireless layer. But since OPNsense handles all routing, firewalling, DNS, and VPN, the UniFi gear is just doing L2 switching and WiFi — replaceable if needed.

## Hardware

| Device | Ports | PoE Budget | Role |
|--------|-------|-----------|------|
| USW-Lite-16-PoE | 16× GbE | 45W PoE+ | Core switch — all infrastructure |
| USW-Lite-8-PoE | 8× GbE | 52W PoE+ | Camera/endpoint switch |
| USW-Flex-2.5G-5 | 5× 2.5GbE | N/A (USB-C powered) | High-bandwidth connections |
| NanoHD ×2 | — | ~10.5W each | Wireless APs (downstairs + upstairs) |

## UniFi Controller Setup

The controller runs in a privileged LXC container on Proxmox with Docker installed:

```yaml
# docker-compose.yml
services:
  unifi-controller:
    image: lscr.io/linuxserver/unifi-network-application:latest
    container_name: unifi
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
    volumes:
      - ./config:/config
    ports:
      - 8443:8443  # Web UI
      - 8080:8080  # Device inform
      - 3478:3478/udp  # STUN
    network_mode: host
```

> **Tip:** Use `network_mode: host` so the controller can receive L2 discovery broadcasts. This simplifies device adoption.

## Switch Configuration

### Port Profiles

Each switch port needs the correct profile based on what's connected:

**Trunk ports** (uplinks between switches, connection to OPNsense, connections to Proxmox nodes):

- Native VLAN: Management (VLAN 10) or Default
- Tagged VLANs: Allow All

**AP ports** (NanoHD access points):

- Native VLAN: Management
- Tagged VLANs: Trusted (20), IoT (30), Guest (50)

**Access ports** (single-VLAN devices like the NAS, cameras):

- Assigned to the appropriate VLAN
- No tagged VLANs

### VLAN Networks in UniFi

Create each VLAN as a network in the UniFi controller:

1. **Settings → Networks → Create New**
2. Set the VLAN ID (must match OPNsense)
3. Set "DHCP Mode" to "None" (OPNsense handles DHCP)
4. Repeat for each VLAN

Then create port profiles that reference these networks and assign them to the appropriate switch ports.

## Wireless Configuration

### SSIDs

| SSID | Security | VLAN | Band |
|------|----------|------|------|
| GoozLab | WPA2 | 20 (Trusted) | 2.4GHz + 5GHz |
| GoozLab-IoT | WPA2 | 30 (IoT) | 2.4GHz + 5GHz |
| GoozLab-Guest | WPA2 | 50 (Guest) | 2.4GHz + 5GHz |

Each SSID is tagged to its VLAN in the wireless network settings. The AP handles the tagging — when a device connects to `GoozLab-IoT`, its traffic is tagged with VLAN 30 and routed through OPNsense accordingly.

### Roaming

Both NanoHDs broadcast identical SSIDs. UniFi handles roaming automatically — devices switch between APs as signal strength changes. No special roaming configuration needed for a two-AP setup.

## Lessons Learned

**PoE budget is a hard limit.** The 16-port's 45W budget goes fast: two NanoHDs consume ~21W, leaving 24W. If you're adding PoE cameras, either use the 8-port switch (52W) or add PoE injectors. See [PoE Power Budget Mismatch](../lessons-learned/poe-power-budget.md).

**The Flex 2.5G needs PoE++ or USB-C.** Don't try to power it from a PoE+ switch. See [PoE Power Budget Mismatch](../lessons-learned/poe-power-budget.md) for the full story.

**DHCP collisions break everything.** Before implementing a Management VLAN, all three switches got the same DHCP address. See [The DHCP Collision](../lessons-learned/dhcp-collision.md).

**Use DHCP reservations, not static IPs.** Configure MAC-based DHCP reservations in OPNsense for all infrastructure devices. This centralises IP management and means you never need to SSH into a switch to change its IP.

**Set a DNS-based inform URL.** In the controller settings, set the inform host to a DNS name (e.g., `unifi.goozlab.net`) instead of an IP. When the controller moves, you update DNS once instead of re-informing every device. See [UniFi Adoption Across Subnets](../lessons-learned/unifi-adoption.md).
