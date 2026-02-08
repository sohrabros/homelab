# Pi NAS Power Supply Gotcha

## The Problem

I had a Raspberry Pi 5 with a Radxa Penta SATA HAT and four Kingston 894GB SSDs. I assumed the official Raspberry Pi 27W USB-C power supply would be enough to power everything.

It wasn't. The SSDs didn't show up. `lsblk` showed nothing. `lsusb` showed nothing. `lspci` showed the Pi's built-in Broadcom bridge and Ethernet but no SATA controller.

## The Investigation

At first I thought it was a driver issue — maybe OMV7 didn't support the HAT. Then I thought the PCIe ribbon cable was bad. I reseated it multiple times, tried a replacement cable, and even double-checked the contact orientation (contacts face down on both ends).

The breakthrough came from checking `dmesg | grep -i pcie` — no errors, but also no SATA controller detected at all. The HAT simply wasn't getting enough power to initialise.

## The Root Cause

The Radxa Penta SATA HAT connects to the Pi 5 via **PCIe** (FFC ribbon cable), not USB. It uses a **JMB585 PCIe-to-SATA bridge chip** that needs 12V power to operate.

The official Pi 5 USB-C PSU outputs:

- 5.1V @ 5.0A
- 9.0V @ 3.0A
- 12.0V @ 2.25A (27W)

Even at 12V, it only provides 2.25A (27W). Four SSDs plus the JMB585 controller need approximately 36-48W for stable operation with headroom for write spikes.

More importantly, the **connector is wrong** — the HAT expects a 5.5×2.5mm barrel jack (center-positive), not USB-C.

## How the HAT Actually Powers Everything

This is the non-obvious part. When you plug 12V into the HAT's barrel jack:

1. The HAT uses 12V directly for the SSDs and SATA controller
2. The HAT **converts 12V down to 5V and back-feeds it to the Pi through the GPIO pins**
3. The Pi boots and operates entirely from power supplied by the HAT

So you end up with **one power supply powering everything** — but it's a 12V barrel jack adapter to the HAT, not the USB-C supply to the Pi. The Pi's USB-C port goes unused.

## The Fix

Bought a universal 12V 5A (60W) DC adapter with interchangeable tips — around £17. Use the 5.5mm outer / 2.5mm inner barrel tip, confirm center-positive polarity.

**Minimum spec:** 12V 4A (48W). I went with 5A for headroom.

## Additional PCIe Config Required

Even with correct power, the Pi 5 needs PCIe explicitly enabled. Add to `/boot/firmware/config.txt`:

```
dtparam=pciex1
dtparam=pciex1_gen=3
usb_max_current_enable=1
```

The `pciex1_gen=3` enables Gen 3 speeds. The Pi 5 is only certified for Gen 2 but Gen 3 works reliably for most users. If you get stability issues, drop to `pciex1_gen=2`.

## What I'd Do Differently

Read the HAT's power requirements **before** ordering a power supply. The Pi ecosystem makes you assume everything runs off USB-C, but any HAT with its own power connector is telling you something important.

Also: **don't use both power sources simultaneously.** Running the 12V barrel jack and the Pi's USB-C at the same time can cause ground loop issues. Pick one. The HAT's barrel jack is the right one.
