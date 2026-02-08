# VLAN Migration Without Downtime

## The Problem

The infrastructure was running on a flat network with OPNsense's LAN interface serving as the gateway. I needed to move everything to a proper tagged Management VLAN (VLAN 10) — but the moment I changed the switch port config, the devices would lose their gateway. And the moment I changed the gateway on the devices, they'd lose connectivity until the switch ports matched.

Classic chicken-and-egg: **you can't change the switch port and the device gateway independently without breaking one or the other.**

## The Solution: A Temporary Bridge

The trick was adding a **Virtual IP alias** on the new VLAN interface that answered for the old gateway address.

### How it worked

Before the migration:

- Devices used the LAN interface IP as their gateway (let's call it Gateway A)
- The Management VLAN interface had its own IP (Gateway B)

The migration steps:

1. **Add a Virtual IP alias** on the Management VLAN interface that also answers for Gateway A
2. Now the Management VLAN responds to **both** addresses
3. **Change switch port** for a device from untagged to tagged VLAN 10
4. The device's traffic now arrives on the VLAN interface — and it still works because Gateway A is answered there too
5. **Change the device's gateway** from A to B
6. Verify connectivity
7. **Repeat** for each device
8. **Remove the Virtual IP** once all devices are migrated

### Why this works

The temporary VIP acts as a bridge between the old and new configurations. At no point does any device lose its gateway — it's always reachable on whichever interface the traffic arrives on.

## The Execution

I did this one device at a time, testing connectivity after each one:

1. Proxmox Node 1 — change switch port, verify, change gateway, verify
2. Proxmox Node 2 — same process
3. UniFi Controller container — stop, edit network config, start
4. Clean up — remove Virtual IP, disable old LAN DHCP

**Backup access throughout:** WireGuard VPN on my phone provided an independent path to OPNsense in case I lost management access from my workstation.

## Prerequisites

Before touching anything:

- Downloaded OPNsense config backup
- Downloaded UniFi controller backup
- Verified WireGuard VPN worked from my phone on mobile data
- Verified I could reach OPNsense, UniFi controller, and both Proxmox nodes

If any of those checks had failed, I would have fixed them first. Never make network changes without a verified backup access path.

## The Lesson

**Plan for the transition state, not just the end state.** It's easy to draw diagrams of "before" and "after" — the hard part is the path between them. A temporary Virtual IP that spans both configurations is a powerful technique for any gateway migration.

Also: **migrate one device at a time.** If something breaks, you know exactly which change caused it and you can roll back just that one step.
