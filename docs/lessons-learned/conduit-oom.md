# Conduit OOM Kill Loops

## The Problem

Three out of five Hetzner Conduit nodes entered an unrecoverable out-of-memory death spiral. The kernel's OOM killer was continuously killing Docker processes, only for them to restart and consume all memory again — an endless loop that made the nodes completely unresponsive to SSH.

## The Setup

Each Hetzner CX23 instance has 4GB RAM. The original deployment ran **two containers** (Conduit proxy + a second service) with **no memory limits**. Both containers would independently try to consume as much RAM as they needed, and with no swap configured, the kernel had nowhere to go but kill processes.

## What Happened

1. Both containers gradually consumed available RAM
2. At ~3.8GB total usage, the kernel OOM killer activated
3. OOM killer terminated one of the Docker containers
4. Docker's `restart: unless-stopped` policy immediately restarted it
5. The freshly restarted container began consuming memory again
6. Within seconds, OOM killer activated again
7. **Repeat forever** — the node was stuck in a kill-restart loop

SSH connections would hang or timeout because the system was spending all its time killing and restarting processes. The only way to access the nodes was through Hetzner's VNC console.

## The Fix

**Memory caps.** Every Conduit container now runs with `--memory=3g`, leaving ~1GB for the OS, Docker daemon, and other services. The homelab node (16GB LXC) uses a 12GB Docker memory cap for even more headroom.

```yaml
# docker-compose.yml
services:
  conduit:
    image: ghcr.io/ssmirr/conduit/conduit:d399071
    deploy:
      resources:
        limits:
          memory: 3g
```

## The Lesson

**Never run containers without memory limits on a constrained host.** It doesn't matter if a single container "should" only use 1GB — if two containers both have no ceiling, they'll fight for every byte until the OOM killer steps in. And with Docker's restart policy, the OOM killer can never win.

On VPS instances with limited RAM (2–4GB), always: set explicit memory caps on every container, leave at least 500MB–1GB for the host OS, and consider adding swap as a pressure relief valve (even a small swap file buys the kernel time to handle memory pressure gracefully instead of immediately killing processes).

## Timeline

- **Impact:** 3 of 5 fleet nodes down simultaneously
- **Detection:** Noticed via Prometheus scraping failures
- **Diagnosis:** ~30 minutes via Hetzner VNC console (`dmesg | grep -i oom` confirmed the kill loop)
- **Resolution:** Clean rebuild of all 5 nodes with memory caps — also took the opportunity to switch from the default Conduit compartment to shirokhorshid and migrate from Nuremberg (unreachable from our ISP) to Helsinki
