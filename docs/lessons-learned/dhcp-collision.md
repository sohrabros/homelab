# The DHCP Collision That Broke Everything

## The Problem

The USW-Flex-2.5G-5 switch kept dropping offline and reconnecting every few minutes. The UniFi controller showed it flapping between "Connected" and "Disconnected" for days.

I'd already spent time investigating the [PoE power budget](poe-power-budget.md) as a possible cause. While that was a real issue, the flapping continued even after addressing power delivery.

## The Investigation

When I finally looked at the UniFi controller's device list more carefully, I noticed something that should have been obvious sooner: **three devices all had the same IP address.**

The USW-Lite-8-PoE, USW-Flex-2.5G-5, and a NanoHD were all sitting on the same IP. The DHCP server on the legacy network had handed the same lease to all three.

## Why It Happened

During the migration from the old flat network to the new VLAN-segmented architecture, the UniFi infrastructure devices were still on the legacy subnet. There was no dedicated Management VLAN yet, and the DHCP scope was too small — or the lease times were too short — so devices kept getting reassigned the same address as leases expired and renewed.

Without a Management VLAN with static DHCP reservations, the infrastructure was essentially playing musical chairs with IP addresses.

## The Fix

1. **Created Management VLAN (VLAN 10)** on OPNsense with its own subnet and DHCP scope
2. **Created static DHCP mappings** in OPNsense for every infrastructure device, keyed by MAC address — each device gets a reserved IP that never changes
3. **Migrated all UniFi devices** to the Management VLAN by changing "Network Override" to the Management network in the controller
4. Each device picked up its reserved IP via DHCP — no manual static config needed on the devices themselves

After the migration, every device had a unique, predictable IP address. The flapping stopped immediately.

## The Lesson

**Never let infrastructure devices rely on a general DHCP pool.** Switches, access points, controllers, hypervisors — anything you need to manage — should have reserved addresses from day one.

DHCP reservations (not static IPs configured on each device) are the right approach: IP management stays centralised in OPNsense, and if you need to change an address, you do it in one place.

Also: **when a device is flapping, check the basics first.** I spent hours on PoE, cables, and firmware before looking at something as simple as "do these devices actually have unique IPs?"
