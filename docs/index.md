# GoozLab

A self-hosted homelab inspired by [Louis Rossmann's FUTO guide](https://wiki.futo.org/index.php/Introduction_to_a_Self_Managed_Life:_a_13_hour_%26_28_minute_presentation_by_FUTO_software). Privacy-first. Open source. Local control.

## Quick Links

<div class="grid cards" markdown>

-   **🔥 OPNsense Firewall**

    Router, firewall, DNS, VPN, and IDS/IPS on a ZimaBoard 2

    [→ Setup Guide](setup/opnsense.md)

-   **🖥️ Proxmox Cluster**

    Two-node virtualisation cluster running all services as LXC containers

    [→ Setup Guide](setup/proxmox.md)

-   **💾 Pi 5 NAS**

    RAID 5 storage on a Raspberry Pi with Radxa SATA HAT

    [→ Setup Guide](setup/nas.md)

-   **📡 Switching & Wireless**

    UniFi managed switches and NanoHD access points with VLAN trunking

    [→ Setup Guide](setup/switching-wireless.md)

-   **📊 Monitoring**

    Prometheus, Grafana, and Uptime Kuma for full infrastructure visibility

    [→ Monitoring Stack](setup/monitoring.md)

-   **Internet Freedom**

    Psiphon Conduit proxy helping people in censored regions

    [→ Learn More](humanitarian/psiphon-conduit.md)

-   **📸 Camera System**

    Frigate NVR with Reolink PoE cameras — local recording, no cloud

    [→ Setup Guide](setup/frigate.md)

-   **💡 Lessons Learned**

    Real mistakes, real troubleshooting, real fixes

    [→ Read Them](lessons-learned/index.md)

</div>

## Architecture at a Glance

Seven VLANs segment the network by trust level. The VLAN ID matches the third octet of each subnet — VLAN 10 is `10.0.10.0/24`, VLAN 30 is `10.0.30.0/24`, and so on.

[→ Full Architecture Documentation](architecture/overview.md)

## Guiding Principles

Every decision in GoozLab is guided by the philosophy from [Louis Rossmann's FUTO guide](https://wiki.futo.org/index.php/Introduction_to_a_Self_Managed_Life:_a_13_hour_%26_28_minute_presentation_by_FUTO_software):

1. **Own Your Data** — Self-host what matters. Your DNS, your files, your cameras.
2. **Open Source First** — Every component is open source.
3. **Privacy by Default** — Encrypted DNS, VPN-only remote access, no telemetry.
4. **Learn by Doing** — Build it, break it, fix it, [write down what you learned](lessons-learned/index.md).
5. **Give Back** — Surplus infrastructure helps others via [Psiphon Conduit](humanitarian/psiphon-conduit.md).

> *Credit where it's due — this project was kicked off by Louis Rossmann's 13.5-hour ["Introduction to a Self-Managed Life"](https://wiki.futo.org/index.php/Introduction_to_a_Self_Managed_Life:_a_13_hour_%26_28_minute_presentation_by_FUTO_software) presentation. If you haven't watched it, it's worth your time.*
