# Caddy Reverse Proxy

## Overview

| Component | Detail |
|-----------|--------|
| Software | [Caddy](https://caddyserver.com/) via OPNsense plugin |
| TLS | Automatic Let's Encrypt via Cloudflare DNS-01 challenge |
| Domain | `goozlab.net` (Cloudflare DNS) |
| Services | 10+ internal services behind HTTPS |

## Why Caddy?

Caddy replaced Nginx to simplify certificate management. Caddy handles automatic HTTPS with zero manual renewal — it obtains and renews Let's Encrypt certificates automatically using the Cloudflare DNS-01 challenge. The OPNsense Caddy plugin makes it manageable from the firewall GUI.

## Configuration

**OPNsense WebGUI port change:** The default HTTPS port (443) must be free for Caddy. OPNsense's WebGUI was moved to port **8443** under System → Settings → Administration.

**DNS-01 challenge:** Caddy uses a Cloudflare API token to create TXT records for domain validation. This means no ports need to be open to the internet for certificate issuance — all validation happens via DNS.

**Individual domains:** Each service gets its own full subdomain (e.g., `dash.goozlab.net`, `ha.goozlab.net`). A wildcard certificate was not used due to a known Caddyfile generation bug in the OPNsense plugin.

**HTTPS upstream:** Services with self-signed certificates (Proxmox, OPNsense, UniFi) use HTTPS upstream with TLS insecure skip verify, so the browser sees a valid Let's Encrypt cert while the backend connection uses the service's own self-signed cert.

## Services Behind Caddy

| Subdomain | Backend | Notes |
|-----------|---------|-------|
| `dash.goozlab.net` | Homepage dashboard | Port 3002 |
| `ha.goozlab.net` | Home Assistant | Port 8123 |
| `frigate.goozlab.net` | Frigate NVR | Port 5000 |
| `grafana.goozlab.net` | Grafana | Port 3000 |
| `uptime.goozlab.net` | Uptime Kuma | Port 3001 |
| `pve1.goozlab.net` | Proxmox Node 1 | HTTPS upstream (self-signed) |
| `pve2.goozlab.net` | Proxmox Node 2 | HTTPS upstream (self-signed) |
| `pve3.goozlab.net` | Proxmox Node 3 | HTTPS upstream (self-signed) |
| `fw1.goozlab.net` | OPNsense WebGUI | HTTPS upstream, port 8443 |
| `unifi.goozlab.net` | UniFi Controller | HTTPS upstream (self-signed) |

## How It Works

```
Browser ──► Caddy (OPNsense, port 443)
              │
              ├── Valid Let's Encrypt cert (auto-renewed via Cloudflare DNS-01)
              │
              └── Reverse proxy to internal service
                    │
                    ├── HTTP backends: direct proxy
                    └── HTTPS backends: TLS with skip verify (self-signed)
```

All traffic stays internal — Caddy runs on the OPNsense firewall and proxies to services on the Management VLAN. No ports are exposed to the internet except WireGuard VPN, so these subdomains are only accessible when connected to the home network or via VPN.

## Setup Steps

1. **Install the Caddy plugin** in OPNsense (System → Firmware → Plugins → `os-caddy`)
2. **Move OPNsense WebGUI** to port 8443 (System → Settings → Administration → TCP Port)
3. **Create a Cloudflare API token** with Zone:DNS:Edit permissions for your domain
4. **Configure DNS provider** in Services → Caddy → General → DNS Provider (Cloudflare + token)
5. **Add domains** in Services → Caddy → General → Domains — one entry per subdomain
6. **Add handlers** in Services → Caddy → General → Handlers — map each domain to its backend address and port
7. **Enable Caddy** and verify certificates are issued (check the Caddy log for Let's Encrypt activity)

## Lessons Learned

- **Individual domains, not wildcard.** The OPNsense Caddy plugin has a known bug with wildcard certificate generation in the Caddyfile. Use individual full domains per service instead.
- **HTTPS upstream for self-signed backends.** Proxmox, OPNsense, and UniFi all use self-signed certificates. Configure Caddy to use HTTPS upstream with TLS insecure skip verify for these backends.
- **WebGUI port conflict.** Caddy needs port 443. Moving OPNsense to 8443 is the standard solution — update your bookmarks and any monitoring that checks the WebGUI.
