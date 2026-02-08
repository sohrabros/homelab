# Psiphon Conduit — Internet Freedom

## What Is This?

A significant part of GoozLab's infrastructure is dedicated to running **[Psiphon Conduit](https://conduit.psiphon.ca/)** — a volunteer proxy service that helps people in censored countries access the open internet.

[Psiphon](https://psiphon.ca/) is an open-source censorship circumvention tool developed at the **University of Toronto's Citizen Lab**. When someone in a censored country (primarily Iran) tries to connect, the Psiphon network routes their traffic through volunteer proxy nodes like mine. The user gets access to the open internet; I donate bandwidth.

## Why It Matters

Since January 2026, Iran has experienced severe internet restrictions, including total blackouts, deep packet inspection, and the deployment of Russian Kalinka jammers that disabled Starlink terminals. Millions of people rely on tools like Psiphon to communicate, access news, and maintain some connection to the outside world.

Running a Conduit node isn't just a technical exercise — it directly helps real people access information that their government is trying to block.

## Architecture

The Conduit infrastructure runs on a dedicated, fully isolated VLAN:

```
┌────────────────────────────────────────────┐
│  VLAN 70 — Conduit (Isolated)              │
│                                            │
│  Firewall Rules:                           │
│  ❌ Conduit → Management VLAN              │
│  ❌ Conduit → Trusted VLAN                 │
│  ❌ Conduit → Any RFC1918 range            │
│  ✅ Conduit → Internet only                │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │  Docker Host LXC                     │  │
│  │                                      │  │
│  │  ┌────────────────────────────────┐  │  │
│  │  │  iptables (iran-only firewall) │  │  │
│  │  │  Blocks non-Iran UDP traffic   │  │  │
│  │  └────────────────────────────────┘  │  │
│  │            ↓                         │  │
│  │  ┌────────────────────────────────┐  │  │
│  │  │  Docker: Conduit               │  │  │
│  │  │  Official Psiphon image        │  │  │
│  │  │  Prometheus metrics on :9090   │  │  │
│  │  └────────────────────────────────┘  │  │
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

### Security: Two Layers of Isolation

1. **VLAN-level:** OPNsense firewall blocks all traffic from VLAN 70 to any internal network
2. **Container-level:** iptables inside the LXC further restrict UDP traffic to Iranian IP ranges only

Even if someone exploited the Conduit service, they could not reach any internal infrastructure.

### Fleet Beyond the Homelab

The homelab Conduit node is supplemented by VPS instances on Hetzner Cloud (German datacentres in Nuremberg/Falkenstein), which provide additional bandwidth and better routing to Iran. The VPS fleet uses cloud-init for automated deployment.

## Monitoring

Conduit exposes Prometheus metrics that are scraped by the monitoring stack:

- **Connected clients:** How many users are currently proxying through the node
- **Data transferred:** Bytes uploaded and downloaded (in GB)
- **Connection status:** Whether the node is live and registered with the Psiphon broker

These metrics feed into a Grafana dashboard for real-time visibility.

## How Conduit Works

1. Your node registers with the **Psiphon broker** (a central coordination service)
2. When someone in Iran opens Psiphon, the broker matches them with available nodes
3. Traffic flows: Iranian user → encrypted tunnel → your Conduit node → open internet
4. Your node sees encrypted proxy traffic — **no browsing history, no personal data**

Psiphon does not collect personal information, browsing activity, or message content. The Conduit node only facilitates encrypted network connections.

New nodes build **reputation** over time by staying online reliably. Don't worry if connections are sparse in the first few days — the broker preferentially routes to nodes with established uptime.

## Want to Run Your Own?

See [Running Your Own Node](run-a-node.md) for how to set up Conduit.

## Resources

- [Psiphon Official Site](https://psiphon.ca/)
- [Psiphon Conduit Volunteer Program](https://conduit.psiphon.ca/)
- [Citizen Lab — University of Toronto](https://citizenlab.ca/)
- [Open Technology Fund](https://www.opentech.fund/)
