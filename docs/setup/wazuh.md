# Wazuh SOC

Centralized security monitoring with Wazuh 4.14.4 — vulnerability scanning, intrusion detection, file integrity monitoring, CIS compliance benchmarks, and MITRE ATT&CK mapping across the entire GoozLab infrastructure.

---

## Architecture

Wazuh runs as an all-in-one deployment (server + indexer + dashboard) on a dedicated VM on pve2. Agents report from every core node in the homelab.

| Component | Detail |
|-----------|--------|
| **Server** | VM 200 on pve2, Ubuntu 24.04 |
| **IP** | 10.0.10.70 |
| **Resources** | 4 cores, 8GB RAM, 50GB disk |
| **Version** | Wazuh 4.14.4 |
| **Access** | wazuh.goozlab.net (VPN only) |

## Agents

| ID | Name | IP | Platform | Key Features |
|----|------|----|----------|--------------|
| 000 | wazuh (server) | 127.0.0.1 | Ubuntu 24.04 | Local monitoring |
| 001 | opnsense | 10.0.10.1 | FreeBSD 14.3 | Suricata eve.json, firewall logs, audit, DNS, IDS |
| 002 | pve1 | 10.0.10.100 | Debian 13 (Trixie) | System monitoring, FIM, vuln scan |
| 003 | pve2 | 10.0.10.101 | Debian 13 (Trixie) | System monitoring, FIM, vuln scan |
| 004 | pve3 | 10.0.10.102 | Debian 13 (Trixie) | System monitoring, FIM, vuln scan |
| 005 | docker-mon | 10.0.10.40 | Debian 12 (Bookworm) | Monitoring stack host |
| 006 | nas | 10.0.10.50 | Debian 13 (Trixie/ARM64) | Storage monitoring |
| 007 | frigate | 10.0.10.104 | Debian (LXC) | NVR container monitoring |

## Installation

### Server (VM on Proxmox)

Created as a full VM (not LXC) because the Wazuh indexer (OpenSearch) requires kernel-level `vm.max_map_count` tuning.

1. **Create VM** — Ubuntu 24.04 Server, 4 cores, 8GB RAM, 50GB disk on pve2
2. **Extend LVM** — Ubuntu Server only allocates half the disk by default:
   ```bash
   sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
   sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
   ```
3. **Set kernel parameter:**
   ```bash
   sudo sysctl -w vm.max_map_count=262144
   echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
   ```
4. **Run the installer:**
   ```bash
   curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
   ```
5. **Post-install:** Disable the Wazuh repo to prevent accidental upgrades:
   ```bash
   sudo sed -i 's/^deb /#deb /' /etc/apt/sources.list.d/wazuh.list
   ```

### Agent Enrollment

Auto-enrollment with password is unreliable between agent/server version mismatches. Manual key import works every time:

**On the Wazuh server:**
```bash
# Register an agent
sudo /var/ossec/bin/manage_agents -a <AGENT_IP> -n <AGENT_NAME>

# Extract the key
sudo /var/ossec/bin/manage_agents -e <AGENT_ID>
```

**On the agent (Linux):**
```bash
# Install
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | tee /etc/apt/sources.list.d/wazuh.list
apt-get update && apt-get install -y wazuh-agent

# Import key and configure manager
/var/ossec/bin/manage_agents -i <BASE64_KEY>
sed -i 's/MANAGER_IP/10.0.10.70/' /var/ossec/etc/ossec.conf

# Start and disable repo
systemctl daemon-reload && systemctl enable --now wazuh-agent
sed -i 's/^deb /#deb /' /etc/apt/sources.list.d/wazuh.list
```

**On OPNsense:**

1. Install `os-wazuh-agent` from System → Firmware → Plugins
2. Register manually on the Wazuh server, extract key
3. Import key on OPNsense via SSH: `/var/ossec/bin/manage_agents -i <KEY>`
4. Configure via GUI: Services → Wazuh Agent → Settings
5. Enable: Suricata (IDS events), filter (firewall), audit, resolver (DNS)
6. Enable: File integrity monitoring, System inventory

## Active Response

Wazuh is configured to automatically push firewall blocks to OPNsense when repeated firewall block events are detected:

```xml
<command>
  <name>opnsense-fw</name>
  <executable>opnsense-fw</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>

<active-response>
  <disabled>no</disabled>
  <command>opnsense-fw</command>
  <location>defined-agent</location>
  <agent_id>001</agent_id>
  <rules_id>87702</rules_id>
  <timeout>3600</timeout>
</active-response>
```

This triggers on rule 87702 (multiple firewall blocks from the same source) and blocks the offending IP for 1 hour via the OPNsense firewall.

## Capabilities

- **Vulnerability Detection** — Automated CVE scanning across all agents
- **File Integrity Monitoring (FIM)** — Tracks changes to critical system files
- **Security Configuration Assessment (SCA)** — CIS benchmarks for Debian 12/13 and Ubuntu 24.04
- **Suricata Integration** — IDS/IPS alerts from OPNsense eve.json correlated with MITRE ATT&CK
- **Active Response** — Automated firewall blocks pushed back to OPNsense
- **MITRE ATT&CK Mapping** — All alerts mapped to adversary tactics and techniques
- **Rootcheck** — Host-based anomaly detection
- **Log Analysis** — Centralized log collection from all infrastructure nodes

## Reverse Proxy

Accessed via `wazuh.goozlab.net` through Caddy on OPNsense with TLS insecure skip verify (Wazuh uses a self-signed certificate):

- **Domain:** wazuh.goozlab.net
- **Upstream:** https://10.0.10.70:443
- **TLS:** Enabled with insecure skip verify
- **Access:** VPN only (not exposed to the internet)

## Lessons Learned

- **Ubuntu Server LVM** only allocates approximately half the disk to the root logical volume. Always run `lvextend` after install.
- **Wazuh auto-enrollment** with password authentication fails silently between version mismatches (e.g., server 4.14.4 vs OPNsense agent 4.12.0). Manual key import via `manage_agents` is reliable.
- **OPNsense agent** is installed via the `os-wazuh-agent` plugin, which provides a GUI for configuration but uses the older FreeBSD Wazuh agent package (4.12.0).
- **Wazuh requires a full VM**, not an LXC container, because the OpenSearch indexer needs `vm.max_map_count=262144` which requires kernel-level access.
- **dpkg removal failures** after a botched install can be fixed by removing the pre/post-removal scripts from `/var/lib/dpkg/info/` and forcing purge.
