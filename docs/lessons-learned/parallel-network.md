# Parallel Network Approach

## The Approach

When I built OPNsense on the ZimaBoard 2, I didn't immediately replace my ISP's router. Instead, I ran both in parallel:

- **ISP router** stayed as the primary gateway on its original subnet
- **OPNsense** ran on a separate LAN subnet, with its WAN port connected to the ISP router as a client

This let me configure VLANs, test DNS, set up Suricata, and verify everything worked without affecting anyone else on the network. Family members stayed on the ISP router's WiFi, completely unaware anything was changing.

## The Access Problem

The downside: my PC was on the ISP router's network and couldn't reach devices on OPNsense's network. They were on different subnets with no routing between them.

**Solutions I considered:**

1. **Static route on my PC** pointing the OPNsense subnet via the ZimaBoard's WAN IP — plus a firewall rule on OPNsense allowing traffic from the old subnet. Clean but required firewall tweaking.
2. **USB Ethernet adapter** — plug a second NIC into my PC, connect it to the lab switch, give it a static IP on the OPNsense subnet. Instant dual-homed access.
3. **Just move my PC** to the new network and access old stuff via the ZimaBoard's routing.

I went with option 1 initially, then moved my PC over once I was confident in the OPNsense config.

## The Cutover

When it was time to go live:

1. Noted down ISP PPPoE credentials
2. Configured PPPoE on OPNsense's WAN interface
3. Physically moved the fibre ONT cable from the ISP router to the ZimaBoard
4. Verified internet connectivity through OPNsense
5. Powered down the ISP router

The actual cutover took about 5 minutes. The weeks of parallel testing meant I was confident everything would work.

## What I'd Do Differently

**Commit to the cutover sooner.** I ran parallel for weeks longer than necessary because I kept finding "one more thing" to configure. The problem is that some issues only surface when OPNsense is actually the primary gateway handling real traffic. The parallel environment was useful but it had diminishing returns.

Also: **plan for the cutover day.** Pick a time when a brief outage is acceptable (weekend morning, not during someone's work call). Have the ISP router ready to plug back in if things go wrong. Tell your household there might be a brief internet blip.

## The Lesson

Parallel testing is excellent for building confidence, but it's not a substitute for real-world traffic. The sooner you cut over, the sooner you discover the issues that only appear under production conditions — and the sooner you can fix them.
