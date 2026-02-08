# Pi 5 NAS Setup

## Hardware

| Component | Specification |
|-----------|--------------|
| Board | Raspberry Pi 5 (4GB RAM) |
| Storage HAT | Radxa Penta SATA HAT (JMB585 PCIe-to-SATA) |
| Drives | 4× Kingston 894GB 2.5" SATA SSD |
| RAID | RAID 5 (~2.6TB usable, single-drive redundancy) |
| Filesystem | ext4 on mdraid |
| OS | OpenMediaVault 7 (Sandworm) on SD card |
| Network | Gigabit Ethernet (Management VLAN) |
| Power | 12V 5A (60W) DC adapter via barrel jack |

> ⚠️ **Read [Pi NAS Power Supply Gotcha](../lessons-learned/pi-nas-power.md) first.** The official Pi 5 USB-C PSU cannot power this setup. The HAT needs its own 12V supply and back-feeds power to the Pi through GPIO.

## Why This Setup

The Pi 5 + Radxa SATA HAT is a genuine PCIe SATA solution (not USB). The JMB585 controller connects via the Pi 5's native PCIe bus, giving real SATA performance. Over Gigabit Ethernet, the NAS saturates the link at ~115 MB/s — more than enough for a homelab Proxmox cluster.

**Limitations to be honest about:**

- 4GB RAM means ZFS is off the table (ZFS wants ~1GB per TB). Use mdraid + ext4 instead.
- Cannot boot from the SATA drives — the bootloader doesn't power the HAT during boot. SD card boot is required.
- Single Gigabit Ethernet — fine for a homelab, but won't scale if you need multi-client high-throughput.

## Installation

### Enable PCIe

Add to `/boot/firmware/config.txt`:

```ini
dtparam=pciex1
dtparam=pciex1_gen=3
usb_max_current_enable=1
```

Reboot. Verify with `lspci` — you should see the JMB585 SATA controller.

### Install OMV 7

Start with a fresh Raspberry Pi OS Lite (64-bit) flashed to an SD card. **Critical:** create a username other than "admin" (reserved for OMV) and do not configure WiFi — use wired Ethernet only.

```bash
# Update system
sudo apt-get update && sudo apt-get upgrade -y

# Pre-install script
sudo wget -O - https://github.com/OpenMediaVault-Plugin-Developers/installScript/raw/master/preinstall | sudo bash
sudo reboot

# Install OMV (takes 15-30 minutes)
wget -O - https://github.com/OpenMediaVault-Plugin-Developers/installScript/raw/master/install | sudo bash
```

Access the web interface at `http://<nas-ip>` with `admin` / `openmediavault`. Change the password immediately.

### RAID 5 Setup

1. **Storage → Disks:** Verify all 4 SSDs are detected
2. **Install the RAID plugin:** System → Plugins → search `openmediavault-md` → install
3. **Storage → Software RAID:** Create RAID 5 with all 4 drives
4. **Storage → File Systems:** Create ext4 on the RAID array, mount it
5. Wait — initial RAID sync takes 2-6 hours in the background. Performance will be degraded during sync.

### Network Shares

**NFS (for Proxmox):**

- Storage → Shared Folders → create a folder on the RAID array
- Services → NFS → Shares → add the folder, restrict to the Management VLAN subnet

**SMB (for workstations):**

- Services → SMB/CIFS → Shares → add folders for media, backups, ISOs

## Integration with Proxmox

On both Proxmox nodes:

1. **Datacenter → Storage → Add → NFS**
2. Enter NAS IP, export path, select content types (ISOs, templates, backups)

Both nodes now share the same storage pool. Container templates, ISOs, and backup files are accessible from either node.

## Troubleshooting: Drives Not Detected

If `lspci` shows the Broadcom bridge and Ethernet but no SATA controller:

1. **Check power** — Is the 12V barrel jack connected and powered?
2. **Check PCIe config** — Are the `dtparam=pciex1` lines in config.txt?
3. **Check the ribbon cable** — Reseat the FFC cable (contacts face down on both ends, latch fully closed)
4. **Check `dmesg | grep -i pcie`** — Look for errors during boot

If `dmesg` shows PCIe devices but no SATA, the ribbon cable is likely the issue. These are fiddly — it took me multiple attempts to get it seated correctly.
