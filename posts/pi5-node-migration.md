---
title: Migrating my Bitcoin node from Pi 4 to Pi 5 with NVMe storage
date: 2025-05-01
category: node-setup
tags: raspberry pi, mynode, nvme, bitcoin
---

After running my Bitcoin and Lightning node on a Raspberry Pi 4 for a couple of years, I finally pulled the trigger on upgrading to a Pi 5 with NVMe storage. What I thought would be a weekend project turned into three days of debugging — but the end result is rock solid. Here's everything that happened.

## The old setup

My Pi 4 was running MyNode 0.3.42 with a Crucial X6 2TB USB drive. On paper this should work fine — but in practice the X6 uses a JMicron controller that is notoriously incompatible with the Pi 4's USB 3.0 implementation. Transfer speeds were bottlenecked at USB 2.0 rates (~40 MB/s) and occasionally the drive would drop off entirely under sustained load.

The blockchain itself was fine — no corruption — but sync times after a restart were painful.

## The new hardware

- **Raspberry Pi 5** (16GB RAM)
- **Argon ONE V5 case** with built-in NVMe slot
- **2TB NVMe SSD**
- Battery-backed via UPS on my rack

The Pi 5 has a PCIe Gen 2 interface through its M.2 HAT slot, giving real NVMe speeds instead of USB-bottlenecked storage. The Argon ONE V5 case integrates this cleanly into a single unit with active cooling.

## Step 1: Installing MyNode on the Pi 5

Flashed MyNode 0.3.43 to a microSD card, booted the Pi 5, and confirmed it came up. MyNode detected the NVMe as `nvme0n1` but wouldn't use it automatically — more on that in a moment.

## Step 2: Transferring the blockchain via rsync

Rather than re-sync from scratch (weeks), I transferred the existing blockchain data from the Pi 4 over the local network using rsync:

```bash
rsync -avz --progress pi4-ip:/mnt/hdd/mynode/bitcoin/ /mnt/nvme/mynode/bitcoin/
```

This took about 6 hours over gigabit ethernet. The Pi 4 stayed online and synced during the transfer — rsync handles the delta on a second pass automatically.

## Step 3: NVMe instability — the cmdline.txt fix

After the transfer, I started hitting intermittent NVMe dropoffs under load. The drive would disappear from `/dev/` during IBD (Initial Block Download catchup) and during Lightning channel operations.

The fix was three kernel parameters added to `/boot/firmware/cmdline.txt`:

```
nvme_core.default_ps_max_latency_us=0 pcie_aspm=off pcie_port_pm=off
```

These disable PCIe Active State Power Management, which was causing the NVMe controller to enter low-power states that the Pi 5 couldn't reliably wake it from. After adding these flags and rebooting, zero dropoffs in weeks of operation.

## Step 4: The missing .mynode marker file

MyNode uses a marker file to identify its data drive. When I moved to the NVMe, it wasn't detecting the partition as the MyNode data volume despite the data being there.

The fix:

```bash
sudo touch /mnt/nvme/.mynode
```

After a reboot, MyNode picked up the NVMe correctly and the dashboard came online showing the full blockchain and Lightning node.

## Current status

The Pi 5 has been running continuously at `192.168.50.70`, battery-backed on my rack. My LND node — alias `steve2z` — came back online without any channel closures. All peers reconnected automatically within a few hours.

NVMe speeds are dramatically better than the old USB setup. Initial sync operations that used to peg the old drive are now barely noticeable.

## Lessons learned

- Always check the USB controller chip in any USB SSD before buying for Pi use. JMicron = avoid.
- The Pi 5's PCIe power management is aggressive by default — the three cmdline.txt flags should probably be standard practice for any NVMe on Pi 5.
- rsync is the right tool for blockchain transfers. Don't re-sync from scratch if you can avoid it.
- MyNode's `.mynode` marker file is undocumented but essential when moving drives.

The old Pi 4 is now running a [Presearch node](/posts/presearch-pi4.html) — nothing wasted.
