# PoE Power Budget Mismatch

## The Problem

My USW-Flex-2.5G-5 switch kept going offline and coming back, cycling every few minutes. The UniFi controller showed it as disconnected, then connected, then disconnected again. This went on for **weeks**.

## The Red Herrings

I chased several wrong theories:

1. **Cable quality** — Swapped the cable between the Flex and the 8-port switch. No change.
2. **Firmware bug** — Checked for updates, rebooted the Flex. No change.
3. **VLAN misconfiguration** — Spent hours reviewing port profiles and trunk settings. Everything looked correct.
4. **Spanning tree** — Suspected an STP loop was causing the flapping. Checked topology — no loops.

The real clue was in the timing. The Flex would work for a few minutes, draw more power as traffic increased, hit the limit, and the upstream switch would cut power. After a brief cooldown, it would re-power and start the cycle again.

## The Root Cause

**PoE standards mismatch.**

| Standard | Max Power per Port | What Uses It |
|----------|-------------------|--------------|
| 802.3af (PoE) | 15.4W | IP phones, basic APs |
| 802.3at (PoE+) | 30W | NanoHD APs (~10.5W each) |
| **802.3bt (PoE++)** | **60-90W** | **USW-Flex-2.5G-5 (~20-25W)** |

The USW-Lite-8-PoE only supports 802.3at (PoE+). While it can deliver up to ~30W per port, the Flex 2.5G draws 20-25W at peak, which should theoretically fit — but in practice, the PoE negotiation and overhead meant the Lite-8 couldn't reliably supply what the Flex needed.

## The Fix

Two options:

1. **Power the Flex via USB-C** — The Flex has a USB-C power port. A standard USB-C PD adapter bypasses PoE entirely. This is what I did.
2. **Use a PoE++ source** — An injector or switch that supports 802.3bt.

## The Lesson

When a PoE device intermittently drops, **always check the power standard it requires versus what the source provides**. The UniFi controller doesn't always warn you about this mismatch — it just shows the device as offline.

Also: the total PoE budget matters. The USW-Lite-16-PoE has a 45W total budget across all ports. With two NanoHDs (~21W combined) and cameras, you can exceed that budget quickly. Plan your PoE power budget the same way you'd plan compute resources.
