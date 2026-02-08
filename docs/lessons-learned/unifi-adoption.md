# UniFi Adoption Across Subnets

## The Problem

After migrating the UniFi controller to a new Management VLAN, the switches couldn't find it. The controller was on one subnet, the switches were still on the legacy subnet, and adoption kept failing — devices would appear briefly as "Pending Adoption," then disappear or show "Adoption Failed."

The controller logs showed the switches attempting to connect, but the inform URL was pointing to an IP address on the old subnet that the switches could no longer reach.

## Why UniFi Adoption Breaks

UniFi devices use an "inform URL" to communicate with the controller. By default, this is `http://<controller-ip>:8080/inform`. When the controller moves to a different IP or subnet, the devices still try to reach the old address.

The devices won't automatically discover the controller on a new subnet — there's no broadcast/multicast discovery across VLANs. You have to manually tell them where the controller moved.

## The Fix

### Step 1: Ensure routing exists between subnets

On OPNsense, I added a **Virtual IP alias** on the LAN interface so that the old subnet could still reach the new management subnet. This temporary bridge let the switches talk to the controller during the transition.

### Step 2: SSH into the switch and update the inform URL

```bash
ssh admin@<switch-ip>
set-inform http://<new-controller-ip>:8080/inform
```

The switch responds with "Adoption request sent to [URL]. Use UniFi Network to complete the adopt process."

### Step 3: Adopt in the controller

In the UniFi controller, the device reappears as "Pending Adoption." Click Adopt.

### Step 4: If adoption keeps failing

Sometimes the device gets stuck in a loop. The fix:

1. In the controller, click **Forget** on the stuck device
2. Wait 30 seconds
3. SSH back in and run `set-inform` again
4. The device reappears fresh — adopt it

If even that fails, a factory reset on the switch (`set-default` via SSH) followed by `set-inform` always works.

## The Lesson

**Plan the controller migration as part of the network migration.** Don't assume devices will just find the controller after a subnet change. Have the `set-inform` command ready for each device.

Better yet: use a **DNS name** for the inform URL instead of an IP address. In the UniFi controller settings, set the inform host to something like `unifi.goozlab.net`. Then, when the controller's IP changes, you only need to update the DNS record — every device will find it automatically on the next inform cycle.
