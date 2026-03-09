# Frigate NVR Setup

## Overview

| Component | Detail |
|-----------|--------|
| Software | [Frigate](https://frigate.video/) v0.16.4 — open-source NVR with AI object detection |
| NVR Host | LXC container on Proxmox Node 1 |
| Cameras | 3× Reolink RLC-510A (5MP, PoE, RTSP) + PoE doorbell |
| AI Detection | Intel OpenVINO on HD 530 iGPU |
| Hardware Decode | Intel VAAPI (preset-vaapi) on HD 530 iGPU |
| Switch | USW-Lite-8-PoE (cameras on IoT VLAN) |
| Storage | Recordings stored on Pi 5 NAS via NFS bind mount |

## Status

✅ **Fully operational.** Four cameras installed, AI detection running, tiered retention configured, mobile push notifications via Home Assistant.

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

## Hardware Acceleration

Frigate uses the Intel HD 530 iGPU (built into the i7-6700T) for two things:

- **VAAPI hardware decode** — Offloads video decoding from CPU. Enabled globally with `preset-vaapi` in the Frigate config. This is critical for handling four 5MP streams without maxing out the CPU.
- **OpenVINO AI detection** — Runs object detection models on the iGPU at ~12ms inference time. Much faster than CPU-only detection and leaves the CPU free for recording and other tasks.

## Retention Strategy

Frigate v0.16.4 uses a tiered retention model using the `alerts` and `detections` schema (not the old `events` key):

| Retention Type | Duration | What It Keeps |
|---------------|----------|---------------|
| Motion | 3 days | Any segment with motion detected |
| Alerts | 14 days | Segments where tracked objects triggered alerts |
| Detections | 14 days | Segments where AI detected objects of interest |

This keeps storage under control on the 2.6TB NAS. At steady state, expect roughly 500–700 GiB of rolling footage across all four cameras — well within capacity with over a terabyte of headroom.

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

The `/mnt/nas/frigate` path is an NFS bind mount — the NFS share is mounted on the Proxmox host first, then bind-mounted into the LXC container. You cannot mount NFS directly inside an LXC.

## Home Assistant Integration

Frigate integrates with Home Assistant via MQTT (Mosquitto broker), enabling:

- **Live camera feeds** on the HA dashboard using picture-glance cards with native Frigate camera entities (no iframes needed)
- **Mobile push notifications** via the Frigate Notifications blueprint from HACS — sends snapshot with bounding box when a person is detected at the doorbell
- **Event-driven automations** based on object detection events

For full Home Assistant setup, see the [Home Assistant guide](home-assistant.md).

## Lessons Learned

- **Stream roles matter:** Main stream (5MP) goes to recording, sub-stream (640×480) goes to detection. This keeps CPU usage low while recording at full quality.
- **FFmpeg CPU warnings are cosmetic:** Frigate warns about high FFmpeg CPU usage per camera (~30% of one core), but total system CPU stays around 29% — the i7-6700T handles it easily.
- **NFS bind mounts:** Always mount NFS on the Proxmox host and bind-mount into the LXC. Direct NFS inside LXC doesn't work reliably.
- **Tiered retention saves storage:** The jump from flat 14-day retention to 3-day motion / 14-day alerts dropped NAS usage from 99% to ~30%.
