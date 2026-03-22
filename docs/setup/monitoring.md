# Monitoring Setup

## Stack

| Tool | Purpose |
|---|---|
| Prometheus | Metrics collection and storage (30 scrape targets) |
| Grafana | Dashboards and visualisation (Command Center + Freedom Fleet) |
| InfluxDB | Time-series storage for fleet and long-term metrics |
| Uptime Kuma | Service availability monitoring |
| Homepage | Service dashboard at dash.goozlab.net |
| Frigate Exporter | Custom Python exporter for camera/NVR metrics |
| Blackbox Exporter | ICMP ping monitoring for cameras |
| SNMP Exporter | UniFi switch and AP metrics |

All services run in a single Docker Host LXC container on the Management VLAN (docker-mon, 10.0.10.40).

## Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │                    Scrape Targets                           │
  ├─────────────────────────────────────────────────────────────┤
  │                                                             │
  │  Proxmox Nodes          NAS             OPNsense            │
  │  pve1 :9100             Pi5 :9100       (FreeBSD metrics)   │
  │  pve2 :9100                                                 │
  │  pve3 :9100                                                 │
  │                                                             │
  │  Frigate NVR            Home Assistant   UniFi (SNMP)       │
  │  via exporter :9102     :8123/api/prom   4 devices via      │
  │                                          snmp-exporter      │
  │                                                             │
  │  Cameras (ICMP)         Conduit Fleet (via WireGuard)       │
  │  4 cameras via          5× app metrics :9101                │
  │  blackbox-exporter      5× node_exporter :9100 (pending)   │
  │                         1× homelab node :9090 + :9100       │
  └──────────────────────────┬──────────────────────────────────┘
                             │
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  Monitoring LXC (docker-mon, 10.0.10.40)                   │
  │                                                             │
  │  ┌────────────┐  ┌─────────┐  ┌────────────┐  ┌─────────┐ │
  │  │ Prometheus │  │ Grafana │  │ Uptime     │  │InfluxDB │ │
  │  │ :9090      │  │ :3000   │  │ Kuma :3001 │  │ :8086   │ │
  │  └────────────┘  └─────────┘  └────────────┘  └─────────┘ │
  │  ┌────────────┐  ┌──────────────┐  ┌────────────────────┐  │
  │  │ Homepage   │  │ Frigate      │  │ Blackbox Exporter  │  │
  │  │ :3002      │  │ Exporter     │  │ (ICMP ping)        │  │
  │  │            │  │ :9102        │  │                     │  │
  │  └────────────┘  └──────────────┘  └────────────────────┘  │
  │  ┌────────────────────┐                                     │
  │  │ SNMP Exporter      │                                     │
  │  │ (UniFi devices)    │                                     │
  │  └────────────────────┘                                     │
  └─────────────────────────────────────────────────────────────┘
```

## Deployment

The monitoring stack uses the standard [Docker Host LXC](docker-services.md) pattern. The full compose file includes all services:

```yaml
# docker-compose.yml (simplified — key services)
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"

  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    restart: unless-stopped
    volumes:
      - uptime_data:/app/data
    ports:
      - "3001:3001"

  influxdb:
    image: influxdb:2
    container_name: influxdb
    restart: unless-stopped
    volumes:
      - influxdb_data:/var/lib/influxdb2
    ports:
      - "8086:8086"

  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    restart: unless-stopped
    environment:
      - HOMEPAGE_ALLOWED_HOSTS=dash.goozlab.net,10.0.10.40,localhost
    volumes:
      - ./homepage/config:/app/config
    ports:
      - "3002:3000"

  frigate-exporter:
    image: python:3.11-slim
    container_name: frigate-exporter
    restart: unless-stopped
    volumes:
      - ./frigate-exporter:/app
    working_dir: /app
    command: python exporter.py
    ports:
      - "9102:9102"

  blackbox-exporter:
    image: prom/blackbox-exporter:latest
    container_name: blackbox-exporter
    restart: unless-stopped
    volumes:
      - ./blackbox/blackbox.yml:/config/blackbox.yml
    command: --config.file=/config/blackbox.yml
    ports:
      - "9115:9115"

  snmp-exporter:
    image: prom/snmp-exporter:latest
    container_name: snmp-exporter
    restart: unless-stopped
    volumes:
      - ./snmp/snmp.yml:/etc/snmp_exporter/snmp.yml
    ports:
      - "9116:9116"

volumes:
  prometheus_data:
  grafana_data:
  uptime_data:
  influxdb_data:
```

## Prometheus Configuration

The Prometheus config scrapes 30 targets across all infrastructure. Fleet metrics are collected exclusively over WireGuard tunnels — no metrics endpoints are exposed to the public internet.

```yaml
# prometheus/prometheus.yml (structure — IPs redacted)
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'proxmox-nodes'
    static_configs:
      - targets:
        - '<pve1-ip>:9100'
        - '<pve2-ip>:9100'
        - '<pve3-ip>:9100'

  - job_name: 'nas'
    static_configs:
      - targets: ['<nas-ip>:9100']

  - job_name: 'frigate'
    static_configs:
      - targets: ['localhost:9102']

  - job_name: 'home-assistant'
    metrics_path: /api/prometheus
    bearer_token: '<long-lived-access-token>'
    static_configs:
      - targets: ['<ha-ip>:8123']

  - job_name: 'blackbox-cameras'
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets:
        - '<camera-front-ip>'
        - '<camera-side-ip>'
        - '<camera-rear-ip>'
        - '<camera-doorbell-ip>'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115

  - job_name: 'snmp-unifi'
    static_configs:
      - targets:
        - '<usw-lite-16-ip>'
        - '<usw-lite-8-ip>'
        - '<nanohd-1-ip>'
        - '<nanohd-2-ip>'
    metrics_path: /snmp
    params:
      auth: [public_v2]
      module: [if_mib]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9116

  # Conduit fleet — scraped via WireGuard tunnels
  - job_name: 'conduit-fleet-nodes'
    static_configs:
      - targets:
        - '10.0.60.11:9100'
        - '10.0.60.12:9100'
        - '10.0.60.13:9100'
        - '10.0.60.14:9100'
        - '10.0.60.15:9100'

  - job_name: 'conduit-fleet-apps'
    static_configs:
      - targets:
        - '10.0.60.11:9101'
        - '10.0.60.12:9101'
        - '10.0.60.13:9101'
        - '10.0.60.14:9101'
        - '10.0.60.15:9101'

  - job_name: 'conduit-homelab-node'
    static_configs:
      - targets: ['<conduit-homelab-ip>:9100']

  - job_name: 'conduit-homelab-app'
    static_configs:
      - targets: ['<conduit-homelab-ip>:9090']
```

> **Note:** Replace `<*-ip>` placeholders with your actual Management VLAN addresses. Internal IPs are kept out of public documentation.

## Custom Exporters

### Frigate Exporter

A custom Python exporter scrapes the Frigate API (`/api/stats`) and exposes metrics on port 9102:

- `frigate_camera_fps` — FPS per camera per stream role
- `frigate_detection_fps` — Detection frames per second
- `frigate_detector_inference_speed_ms` — AI inference latency
- `frigate_cpu_usage` / `frigate_mem_usage` — Resource consumption
- `frigate_up` — Service availability

### Blackbox Exporter

ICMP ping monitoring for all 4 cameras on the IoT VLAN. Tracks reachability and latency — useful for detecting camera drops or PoE switch issues.

### SNMP Exporter

Monitors UniFi switches and access points via SNMP v2c. SNMP is enabled globally via the UniFi controller's CyberSecure settings (not per-device). Four devices respond: USW-Lite-16-PoE, USW-Lite-8-PoE, and both NanoHD APs. The USW Flex Mini 2.5G is excluded by the controller (no SNMP support).

## Grafana Dashboards

Two primary dashboards:

- **GoozLab Command Center (v2):** Infrastructure overview with Proxmox compute, NAS storage, OPNsense memory, Frigate camera health, network throughput, and service uptime
- **GoozLab Freedom Fleet:** Conduit fleet system health (CPU, RAM, disk per node), app metrics (connected clients, data transferred, broker announcements), fleet status table

Additional community dashboards:

- **Node Exporter Full** (Dashboard ID: 1860) — CPU, RAM, disk, network for Linux hosts
- **Custom panels** for Home Assistant solar metrics (pending prometheus filter configuration)

## Uptime Kuma

Monitors service availability with HTTP, TCP, and ping checks for all critical services including OPNsense, all three Proxmox nodes, NAS, UniFi controller, Grafana, Frigate, Home Assistant, and all Caddy-fronted services.

## What To Monitor

At minimum, set up alerts for:

1. **Disk usage >80%** on any host — especially the NAS and Proxmox nodes
2. **RAM usage >90%** — containers will get OOM-killed
3. **Service down** — Uptime Kuma pings for every critical service
4. **RAID degradation** — monitor `mdstat` on the Pi NAS
5. **Conduit fleet health** — connected clients, broker announcements, node availability
6. **Camera reachability** — Blackbox exporter ICMP checks
