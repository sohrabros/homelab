# Frigate NVR Setup

## Overview

| Component | Detail |
|-----------|--------|
| Software | [Frigate](https://frigate.video/) — open-source NVR with AI object detection |
| NVR Host | LXC container on Proxmox Node 2 |
| Cameras | 3× Reolink RLC-510A (5MP, PoE, RTSP) + PoE doorbell |
| Switch | USW-Lite-8-PoE (cameras on IoT VLAN) |
| Storage | Recordings stored on NAS via NFS mount |

## Status

🔶 **In progress.** LXC container is provisioned and Frigate is deployed. Physical camera cabling (~77 metres of Cat6 outdoor cable) is being run. Three external cameras and one PoE doorbell will connect to the USW-Lite-8-PoE on the IoT VLAN.

## Why Frigate?

The FUTO philosophy is clear: own your surveillance footage locally. Cloud-based camera systems (Ring, Nest) send your video to someone else's servers. Frigate runs entirely locally, stores recordings on your own NAS, and does AI-based object detection without phoning home.

The Reolink RLC-510A cameras support RTSP streaming, which Frigate ingests directly. No cloud account required.

## Camera Network Design

Cameras are placed on the **IoT VLAN (VLAN 30)** with strict firewall rules:

- ✅ Cameras → NVR (Frigate) on Management VLAN: Allowed (RTSP streams)
- ✅ Cameras → Internet: Allowed (firmware updates only, can be blocked after initial setup)
- ❌ Cameras → Trusted/Guest/Other VLANs: Blocked
- ❌ Cameras → Any RFC1918 range (except NVR): Blocked

This means a compromised camera cannot access your trusted network, NAS, or other infrastructure.

## Deployment

Frigate runs in a Docker Host LXC using the standard [deployment pattern](docker-services.md):

```yaml
# docker-compose.yml
services:
  frigate:
    image: ghcr.io/blakeblackshear/frigate:stable
    container_name: frigate
    restart: unless-stopped
    privileged: true
    volumes:
      - ./config:/config
      - /mnt/nas/frigate:/media/frigate
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1000000000
    ports:
      - "5000:5000"  # Web UI
      - "8554:8554"  # RTSP restream
      - "8555:8555/tcp"  # WebRTC
      - "8555:8555/udp"
```

## Cabling Notes

Running ~77 metres of outdoor-rated Cat6 for three camera positions plus the doorbell. Using the USW-Lite-8-PoE (52W budget) as the dedicated camera switch — the 16-port's 45W budget is already committed to the APs and infrastructure.
