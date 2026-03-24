# Ubuntu Server LVM Disk Allocation

## The Problem

Ubuntu Server's installer only allocates approximately **half** of the available disk space to the root logical volume by default. On a 50GB disk, you get a ~24GB root partition with 24GB sitting unused in the volume group.

This caused a Wazuh all-in-one installation to fail — the indexer (1.1GB), manager, and dashboard packages filled the root partition, and the dashboard installation failed with "No space left on device."

## The Symptoms

```
E: Write error - write (28: No space left on device)
E: IO Error saving source cache
```

The `df -h /` output showed the root filesystem at 100%, but `vgs` revealed plenty of free space in the volume group:

```
$ df -h /
/dev/mapper/ubuntu--vg-ubuntu--lv   24G  8.7G   14G  39% /

$ sudo vgs
  VG        #PV #LV #SN Attr   VSize   VFree
  ubuntu-vg   1   1   0 wz--n- <48.00g 24.00g
```

## The Fix

Extend the logical volume to use all available space and resize the filesystem:

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

Verify:

```bash
$ df -h /
/dev/mapper/ubuntu--vg-ubuntu--lv   48G  8.7G   37G  20% /
```

## The Lesson

**Always check and extend LVM after installing Ubuntu Server.** This is a known default behavior — the installer leaves room for snapshots that most homelab users never use. Run `lvextend` as part of your post-install checklist for any Ubuntu Server VM.
