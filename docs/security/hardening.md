# Security Hardening

## Defence in Depth

GoozLab uses multiple overlapping security layers. No single layer is relied upon alone.

```
┌──────────────────────────────────────────┐
│ Layer 1: Network Segmentation (VLANs)    │
│  ┌────────────────────────────────────┐  │
│  │ Layer 2: OPNsense Firewall Rules   │  │
│  │  ┌──────────────────────────────┐  │  │
│  │  │ Layer 3: Suricata IDS/IPS    │  │  │
│  │  │  ┌────────────────────────┐  │  │  │
│  │  │  │ Layer 4: CrowdSec      │  │  │  │
│  │  │  │  ┌──────────────────┐  │  │  │  │
│  │  │  │  │ Layer 5: DNS     │  │  │  │  │
│  │  │  │  │ Filtering        │  │  │  │  │
│  │  │  │  └──────────────────┘  │  │  │  │
│  │  │  └────────────────────────┘  │  │  │
│  │  └──────────────────────────────┘  │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

## Suricata IDS/IPS

OPNsense's built-in Suricata provides network intrusion detection and prevention.

**Configuration:**

- **Mode:** Started in IDS (detection only) to build baseline, then switched to IPS (blocking)
- **Rulesets:** ET Open, Abuse.ch, and OPNsense-curated rules
- **Interface:** Monitors the WAN interface for inbound threats
- **Pattern matcher:** Hyperscan (fastest option for x86)

**Tip:** Run in IDS mode for at least a week before switching to IPS. Review alerts in the Suricata log to identify false positives and create suppression rules before enabling blocking. Otherwise you'll block legitimate traffic and spend hours troubleshooting "why can't I reach X."

## CrowdSec

CrowdSec is a collaborative threat intelligence tool. It analyses OPNsense logs, detects attack patterns, and shares threat data with the CrowdSec community. In return, you get blocklists of known-bad IPs from other CrowdSec users.

**Installation:** Available as an OPNsense plugin. After installation:

1. Register at [app.crowdsec.net](https://app.crowdsec.net) for community blocklists
2. Configure the OPNsense bouncer to automatically block flagged IPs at the firewall
3. Review decisions in the CrowdSec dashboard

## DNS Security

- **DNS-over-TLS:** All upstream queries encrypted via Quad9
- **DNSSEC:** Validates DNS responses — prevents poisoning and spoofing
- **Blocklists:** OISD and Hagezi lists block known ad/tracking/malware domains
- **DNS redirect:** NAT rule forces all port 53 traffic to Unbound, preventing devices from bypassing local DNS

## Access Security

- **No public-facing services:** Zero ports open to the internet except WireGuard VPN
- **WireGuard VPN:** Only way to access the network remotely. Pre-shared keys for each client.
- **SSH key-based auth:** Password authentication disabled on all infrastructure
- **OPNsense 2FA:** TOTP two-factor authentication on the firewall admin interface

## VLAN Isolation

The most impactful security measure. See [VLAN Segmentation](../architecture/vlan-segmentation.md) for the full traffic matrix. Key isolation rules:

- **IoT devices** cannot reach any internal network (RFC1918 block rule)
- **Guest WiFi** gets internet only — no LAN access at all
- **Conduit proxy** is completely isolated on its own VLAN with no internal access
- **Management VLAN** is only reachable from Trusted and WireGuard VLANs
