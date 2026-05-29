# Fixing an Omarchy Linux Boot Issue on a 2019 T2 MacBook Pro

This post was written purely with AI after debugging the issue on a real machine. It is provided mostly so it might show up in Google for the next person trying to boot Omarchy Linux on a T2 MacBook Pro and running into the same problem.

The machine was a 2019 Intel MacBook Pro with a T2 chip running Omarchy Linux.

After entering the disk password during boot, the screen would go dark. The Touch Bar still lit up, but input appeared not to work. The fans would spin up, the machine would get hot, and eventually it would power off or restart.

One strange detail: the system was more likely to boot successfully if a USB keyboard was plugged in.

## The Problem Was Not Missing `apple_bce`

A previous attempted fix added `apple_bce` directly to `/etc/mkinitcpio.conf` and regenerated the initramfs.

That was not the right fix here.

Omarchy already had the T2 input modules configured through:

```bash
/etc/mkinitcpio.conf.d/apple-t2.conf
```

That file contained:

```bash
MODULES+=(apple-bce usbhid hid_apple hid_generic xhci_pci xhci_hcd)
```

The generated EFI image also already included `apple-bce`, `hid-apple`, `usbhid`, and the other input-related modules. Kernel logs confirmed that the internal Apple keyboard and trackpad were being enumerated through `apple-bce`.

So the issue was not simply that `apple_bce` was missing.

## What Was Actually Broken

The concrete failure was `t2fanrd`.

It was crash-looping with:

```text
Error: Missing Fan2 in config file
```

The system exposed two fans:

```text
fan1_min=1836
fan1_max=5616

fan2_min=1700
fan2_max=5200
```

But `/etc/t2fand.conf` only defined `Fan1`:

```ini
[Fan1]
low_temp=55
high_temp=75
speed_curve=linear
always_full_speed=false
```

Because this MacBook has two fans, `t2fanrd` expected both `[Fan1]` and `[Fan2]`. With only one fan configured, the service repeatedly exited and systemd restarted it.

That explained the repeated fan-control failure and heat during boot.

## Snapshot Situation

The only working boot path was a Limine/Snapper snapshot:

```text
rootflags=subvol=/@/.snapshots/3/snapshot
```

The normal system root should boot from:

```text
rootflags=subvol=@
```

The known-good snapshot was:

```text
Snapshot 3
Date: 2026-05-28 00:08:26
Kernel: 7.0.9-arch1-Watanare-T2-3-t2
```

Because the machine was running from a snapshot, changing files in the live system was not enough. The fix had to be applied to the real `@` root subvolume.

## The Fix

First, the known-good snapshot was restored into the real root using Limine/Snapper's own restore mechanism.

This created a backup snapshot of the previous real root:

```text
Snapshot 6
Description: backup before restoring snapshot 3 with two-fan t2fand fix
```

Then the real root was mounted and `/etc/t2fand.conf` was changed to define both fans:

```ini
[Fan1]
low_temp=55
high_temp=75
speed_curve=linear
always_full_speed=false

[Fan2]
low_temp=55
high_temp=75
speed_curve=linear
always_full_speed=false
```

The main boot entry was restored to the known-good `7.0.9` T2 kernel image.

After rebooting from the normal Omarchy entry, the system booted from:

```text
rootflags=subvol=@
```

and `t2fanrd` was running:

```text
Active: active (running)
```

The logs showed both fans being detected and configured.

## What Changed From Base Omarchy

The important persistent change was:

```bash
/etc/t2fand.conf
```

It changed from only `[Fan1]` to both `[Fan1]` and `[Fan2]`.

The bad direct edit to `MODULES=()` in `/etc/mkinitcpio.conf` was not kept. The correct T2 module configuration remained in:

```bash
/etc/mkinitcpio.conf.d/apple-t2.conf
```

with:

```bash
MODULES+=(apple-bce usbhid hid_apple hid_generic xhci_pci xhci_hcd)
```

Limine/Snapper state also changed because the known-good snapshot was restored into the real root and the old real root was preserved as a backup snapshot.

## How To Verify This Fix

After rebooting, check that the system is not still booted from a snapshot:

```bash
cat /proc/cmdline
```

You want:

```text
rootflags=subvol=@
```

not:

```text
rootflags=subvol=/@/.snapshots/...
```

Check the fan daemon:

```bash
systemctl status t2fanrd --no-pager
journalctl -b -u t2fanrd --no-pager
```

A fixed boot should show:

```text
Active: active (running)
```

and logs showing both fan controllers.

## About Remaining Heat

After fixing `t2fanrd`, the machine may still run warm depending on power settings.

In this case, the system was using the `performance` power profile, the AMD discrete GPU was active, and the battery was charging. That is enough to keep this generation of MacBook Pro warm even when nothing is broken.

Useful checks:

```bash
powerprofilesctl get
sensors
ps -eo pid,comm,%cpu,%mem,args --sort=-%cpu | head
```

To reduce heat:

```bash
powerprofilesctl set balanced
```

or:

```bash
powerprofilesctl set power-saver
```

## Recovery Path

If the normal boot fails, use Limine's snapshot menu and boot the known-good snapshot:

```text
Omarchy -> linux-t2 -> Snapshots -> 2026-05-28 00:08:26 -> linux-t2
```

That snapshot boots with:

```text
rootflags=subvol=/@/.snapshots/3/snapshot
```

Once booted, inspect:

```bash
cat /proc/cmdline
uname -a
journalctl --list-boots --no-pager | tail -n 8
```

## Takeaway

On T2 MacBooks, a black screen or input problem after disk unlock does not automatically mean `apple_bce` is missing from the initramfs.

In this case, the T2 input stack was already present. The specific breakage was that `t2fanrd` expected config for two fans, but `/etc/t2fand.conf` only defined one.

Adding the missing `[Fan2]` section and restoring the known-good snapshot into the real root fixed the boot.
