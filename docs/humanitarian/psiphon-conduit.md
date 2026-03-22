# Psiphon Conduit — Internet Freedom

## What Is This?

A significant part of GoozLab's infrastructure is dedicated to running **[Psiphon Conduit](https://conduit.psiphon.ca/)** — a volunteer proxy service that helps people in censored countries access the open internet.

[Psiphon](https://psiphon.ca/) is an open-source censorship circumvention tool developed at the **University of Toronto's Citizen Lab**. When someone in a censored country (primarily Iran) tries to connect, the Psiphon network routes their traffic through volunteer proxy nodes like mine. The user gets access to the open internet; I donate bandwidth.

## Why It Matters

Since early 2026, Iran has experienced a near-total internet blackout — not just throttling or selective blocking, but a systematic shutdown affecting millions of people. Deep packet inspection, DNS poisoning, IP blocking, and even Russian-supplied Kalinka jammers targeting Starlink terminals have been deployed. Psiphon's Iranian user base dropped from up to six million unique daily users to fewer than 100,000 during the worst periods.

Running a Conduit node isn't just a technical exercise — it directly helps real people access information, communicate with the outside world, and maintain some connection that their government is trying to sever.

## Fleet Overview

The GoozLab freedom fleet consists of **6 nodes** — 1 homelab and 5 cloud — all running the **shirokhorshid** compartment, a community fork that resolved a Psiphon broker bug which had caused global connections to drop dramatically.

| Node | Location | Capacity | Stack |
|---|---|---|---|
| Homelab (CT 110) | pve3, VLAN 70 | 1,000 clients / 500 Mbps | Conduit + Snowflake + Watchtower |
| conduit-1 through conduit-5 | Hetzner CX23, Helsinki | 200 clients / 200 Mbps each | Conduit + Snowflake + Watchtower |

Every node runs three containers: the Conduit proxy itself, a **Tor Snowflake** proxy (additional circumvention bridge), and **Watchtower** for automated container updates.

## Architecture

### Homelab Node — Double Isolation

The homelab Conduit node runs on a dedicated, fully isolated VLAN with two layers of security:

```
┌────────────────────────────────────────────────┐
│  VLAN 70 — Conduit (Fully Isolated)            │
│                                                │
│  OPNsense Firewall Rules:                      │
│  ❌ Conduit → Management VLAN                  │
│  ❌ Conduit → Trusted VLAN                     │
│  ❌ Conduit → Any RFC1918 range                │
│  ✅ Conduit → Internet only                    │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │  Docker Host LXC (CT 110 on pve3)       │  │
│  │  16GB RAM / 12GB Docker memory cap       │  │
│  │                                          │  │
│  │  ┌────────────────┐ ┌────────────────┐  │  │
│  │  │ Conduit        │ │ Tor Snowflake  │  │  │
│  │  │ shirokhorshid  │ │ bridge proxy   │  │  │
│  │  │ 1000 clients   │ │                │  │  │
│  │  │ metrics :9090  │ └────────────────┘  │  │
│  │  └────────────────┘                     │  │
│  │  ┌────────────────┐                     │  │
│  │  │ Watchtower     │                     │  │
│  │  │ auto-updates   │                     │  │
│  │  └────────────────┘                     │  │
│  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘
```

Even if someone exploited the Conduit service, they could not reach any internal infrastructure — the VLAN firewall blocks all lateral movement.

### Hetzner Cloud Fleet

Five CX23 instances in **Helsinki, Finland** provide the bulk of the fleet's capacity and better routing to Iran. Each node is hardened:

- **SSH:** Key-only authentication, password auth disabled
- **Firewall:** UFW locked to SSH + WireGuard only — no metrics ports exposed publicly
- **Intrusion prevention:** fail2ban protecting SSH
- **Memory management:** Docker `--memory=3g` caps prevent OOM kill loops (a lesson learned the hard way — two uncapped containers on 4GB RAM caused kernel OOM spirals)

### WireGuard Tunnel Mesh

All five Hetzner nodes establish WireGuard tunnels back to OPNsense (wg0), forming a secure mesh for metrics collection:

```
┌─────────────┐         WireGuard Tunnels          ┌──────────────┐
│ Hetzner     │◄──────────────────────────────────► │ OPNsense     │
│ Helsinki    │  conduit-1  ◄─► 10.0.60.11         │ wg0          │
│             │  conduit-2  ◄─► 10.0.60.12         │ 10.0.60.0/24 │
│ 5× CX23    │  conduit-3  ◄─► 10.0.60.13         │              │
│             │  conduit-4  ◄─► 10.0.60.14         │     ▼        │
│             │  conduit-5  ◄─► 10.0.60.15         │ Prometheus   │
└─────────────┘                                     │ (docker-mon) │
                                                    └──────────────┘
```

Prometheus scrapes all fleet metrics exclusively over these tunnels. No metrics endpoints are exposed to the public internet.

## Monitoring

The fleet is monitored through a dedicated **"GoozLab Freedom Fleet"** Grafana dashboard, fed by Prometheus scraping:

- **Conduit app metrics** (port 9090 via tunnel): Connected clients, data transferred, broker announcements, connection status
- **System metrics** (node_exporter, port 9100): CPU, RAM, disk, network throughput per node
- **Fleet health:** All nodes visible in a single dashboard with per-node breakdown

The homelab node metrics are scraped directly on the management VLAN; cloud nodes exclusively via WireGuard tunnel.

## How Conduit Works

1. Your node registers with the **Psiphon broker** (a central coordination service)
2. When someone in Iran opens Psiphon, the broker matches them with available nodes
3. Traffic flows: Iranian user → encrypted tunnel → your Conduit node → open internet
4. Your node sees encrypted proxy traffic — **no browsing history, no personal data**

Psiphon does not collect personal information, browsing activity, or message content. The Conduit node only facilitates encrypted network connections.

New nodes build **reputation** over time by staying online reliably. Don't worry if connections are sparse in the first few days — the broker preferentially routes to nodes with established uptime.

## What Is Shirokhorshid?

The **shirokhorshid** compartment is a community fork of the Conduit image that addressed a critical Psiphon broker bug. The bug had caused global connection counts to drop from 29 million to 3 million. The name references the pre-revolutionary Iranian Lion and Sun emblem — a nod to the community this infrastructure serves.

The shirokhorshid image is pulled from `ghcr.io/ssmirr/conduit/conduit:d399071` and is deployed across all 6 nodes in the fleet.

## Want to Run Your Own?

See [Running Your Own Node](run-a-node.md) for how to set up Conduit.

## Resources

- [Psiphon Official Site](https://psiphon.ca/)
- [Psiphon Conduit Volunteer Program](https://conduit.psiphon.ca/)
- [Citizen Lab — University of Toronto](https://citizenlab.ca/)
- [Open Technology Fund](https://www.opentech.fund/)
