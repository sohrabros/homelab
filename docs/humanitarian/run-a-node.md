# Running Your Own Conduit Node

## The Easy Way

If you just want to help and don't need to isolate the traffic:

1. Visit [conduit.psiphon.ca](https://conduit.psiphon.ca/)
2. Download the app for your platform (Android, iOS, Windows, Mac)
3. Run it — that's it

Your device becomes a proxy node. The app handles everything automatically.

## The Homelab Way (Docker)

For more control, bandwidth, and security isolation:

```yaml
# docker-compose.yml
services:
  conduit:
    image: ghcr.io/psiphon-inc/conduit/cli:latest
    container_name: conduit
    restart: unless-stopped
    volumes:
      - ./data:/home/conduit/data
    ports:
      - "9090:9090"  # Prometheus metrics
    command: start -v --max-clients 500 --bandwidth -1
```

- `--max-clients 500` — How many concurrent users (default 10, max 1000)
- `--bandwidth -1` — Unlimited bandwidth (or set a number in Mbps)

### Recommended: Isolate It

If you're running this on a homelab, put it on its own VLAN with firewall rules that block access to your internal networks. See the [GoozLab architecture](psiphon-conduit.md) for how I did this.

### Optional: Iran-Only Filtering

To prioritise Iranian users, you can restrict UDP data tunnels to Iranian IP ranges using iptables and an ipset. Community scripts exist for this — search for "iran-conduit-firewall" on GitHub.

## What Conduit Does NOT Do

- Does NOT expose your home IP as an exit node (traffic exits through Psiphon's infrastructure)
- Does NOT log browsing activity or personal data
- Does NOT require port forwarding or a public IP
- Does NOT give users access to your local network (if properly firewalled)

## Privacy

Review Psiphon's [privacy policy](https://psiphon.ca/en/privacy.html) for full details. The key points: Conduit only facilitates encrypted connections. No personal information, browsing history, or message content passes through in readable form.

## Resources

- [Psiphon on GitHub](https://github.com/Psiphon-Labs)
- [Conduit Volunteer Program](https://conduit.psiphon.ca/)
