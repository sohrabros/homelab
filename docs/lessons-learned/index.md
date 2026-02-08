# Lessons Learned

The most valuable part of this homelab isn't the architecture diagrams — it's the mistakes. Every section below is a real problem I hit, how long it took me to figure out, and what I'd do differently next time.

If you're building your own homelab, read these before you start. They'll save you hours.

---

## Hardware & Power

### [Pi NAS Power Supply Gotcha](pi-nas-power.md)
The official Raspberry Pi 5 USB-C power supply can't power a Radxa SATA HAT with four SSDs. The HAT needs 12V via barrel jack and back-feeds 5V to the Pi through GPIO. One power supply, but it's not the one you'd expect.

### [PoE Power Budget Mismatch](poe-power-budget.md)
The USW-Flex-2.5G-5 needs 802.3bt PoE++ (20-25W), but the USW-Lite-8-PoE only provides 802.3at PoE+ (~15W per port). This caused weeks of intermittent disconnections that looked like a software problem.

---

## Networking

### [The DHCP Collision That Broke Everything](dhcp-collision.md)
Three UniFi switches all got assigned the same IP address via DHCP. The resulting flapping was misdiagnosed as a PoE issue, a firmware bug, and a cable problem before we found the real cause.

### [VLAN Migration Without Downtime](vlan-migration.md)
Moving infrastructure from a flat network to tagged VLANs without losing management access. The trick: a temporary Virtual IP alias on the new VLAN interface that answers for the old gateway address.

### [Parallel Network Approach](parallel-network.md)
Running the old ISP router and OPNsense side by side during cutover. How to access devices on the new network from the old one, and why you should commit to the cutover sooner rather than later.

### [UniFi Adoption Across Subnets](unifi-adoption.md)
When the UniFi controller moves to a new subnet but the switches are still on the old one, adoption breaks. The fix involves `set-inform` via SSH and temporary routing between subnets.

---

## General Principles

After months of building this lab, here are the patterns that kept proving themselves:

**Always have a backup access path.** Before making any network change, verify you can reach OPNsense via WireGuard VPN on your phone. If you lock yourself out of the management network, you need another way in.

**Change one thing at a time.** When VLANs, firewall rules, and switch configs all change at once, you can't tell which one broke things. Make a change, verify, then move on.

**DHCP reservations over static IPs.** Let DHCP hand out addresses from OPNsense using MAC-based reservations. Centralises IP management in one place instead of configuring each device individually.

**Document while you build, not after.** The details you think you'll remember — the specific port number, the exact firewall rule order, the PCIe config flag — you won't.

**The VLAN-ID-equals-third-octet pattern.** Making VLAN 10 map to `10.0.10.0/24`, VLAN 20 to `10.0.20.0/24`, etc. was the single best design decision. Seeing an IP in any log instantly tells you which network segment it belongs to.
