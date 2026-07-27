{
  "slug": "2026-07-27-rack-dl380-migration-self-hosted-bots",
  "title": "Rack, Rebuild, Repeat: Two Server Migrations and a NAS Build in Ten Weeks",
  "date": "2026-07-27",
  "category": "node-setup",
  "tags": [
    "ethereum",
    "dl380",
    "homelab",
    "rack",
    "ssv",
    "trading-bots",
    "self-hosted",
    "truenas"
  ],
  "excerpt": "From a Ryzen tower on a desk to a proper 42U rack running a dual-Xeon DL380 Ethereum node, a NAS build in progress, and two trading bots migrated off DigitalOcean — everything that changed in ten weeks."
}

---
layout: post
title: "From Celeron to Ryzen: Upgrading My Ethereum Node Hardware"
date: 2026-05-12
tags: [ethereum, node-setup, ryzen, nethermind, ssv, hardware]
description: "Real-world performance numbers from upgrading an Ethereum node from an Intel Celeron G3920 H110 build to a Ryzen 5500 AM4 platform with 32GB RAM."
---

286ms block processing vs 8,000ms. 65 MGas/s vs 1-2 MGas/s. Zero swap vs 2.6GB in use. Real numbers from a real upgrade on a real node.

## the backstory

After getting my Bitcoin Lightning node running on a Pi 5 with NVMe storage, I decided to go deeper and spin up an Ethereum full node to support SSV Network operator duties. The goal: run Nethermind + Lighthouse + SSV node stack, sync to mainnet, and register as an operator.

The hardware I had available was an old Intel H110 board running a **Celeron G3920** — a dual-core, 2.9GHz chip from 2016. No NVMe slots, 16GB DDR4 max, and about as much single-thread performance as a tired laptop. I knew it was going to be rough. What I didn't fully appreciate was *how* rough.

## the original build

| Component | Spec |
|---|---|
| CPU | Intel Celeron G3920 — 2 cores, 2.9GHz |
| RAM | 16GB DDR4 |
| OS Drive | 28GB USB stick → Samsung 500GB SATA SSD |
| Chain Data | Crucial X6 2TB via USB 3.2 |
| NVMe slots | None — H110 has no M.2 slots |
| Power draw | ~65W sustained |

The plan was to get synced on this hardware, learn the stack, then upgrade. What I didn't plan for was how long that sync would take.

## syncing on a celeron — the grind

Nethermind snap sync has several phases: Old Headers, Snap State, Old Bodies, Old Receipts, and finally StateNodes. Each phase has different bottlenecks. On the Celeron, everything bottlenecked everywhere simultaneously.

The iostat numbers told the story clearly:

```
Device    %util    r/s      rkB/s    w/s      wkB/s
sdb       89.20    339.50   9436.00  217.00   25020.00
(Crucial X6 USB drive — maxed out constantly)
```

The CPU was sitting at 15-31% **iowait** — meaning it was idle, just waiting for the USB drive to catch up. And the drive was at 89% utilization doing it.

Old Receipts phase — the longest phase — crawled along at **0-35 blocks per second**. It took approximately 5 days to complete. The whole sync took just under a week.

**Celeron peak stats during sync:**
- Memory: 13GB used / 2.6GB swap in use / 171MB free
- engine_newPayloadV4 avg: 7,964ms (max: 19,967ms)
- Lighthouse warnings: "system resources may be overloaded"
- Background task queue full — dropping transactions

The node did sync. The Celeron limped across the finish line. But running a live Ethereum node — processing a new block every 12 seconds — was clearly pushing it to the absolute limit.

## the upgrade: ryzen 5500 on am4

I happened to be upgrading my main gaming PC to an AM5 platform, which freed up the AM4 components. The Ryzen 5500 isn't new hardware — it's a Zen 3 chip from 2021 — but compared to a Celeron G3920 it's a fundamentally different machine.

| Component | Old (H110) | New (AM4) |
|---|---|---|
| CPU | Celeron G3920 — 2c/2t | Ryzen 5500 — 6c/12t |
| RAM | 16GB DDR4 | 32GB DDR4 |
| NVMe slots | 0 | 2x M.2 |
| Chain storage | USB 3.2 external | USB 3.2 external (temp) |
| OS storage | USB stick → SATA SSD | Samsung 500GB SATA SSD |

The migration was straightforward: clean shutdown of all services, swap the motherboard, move the Samsung SSD over, boot up. The only hiccup was Ubuntu's netplan config still had the old interface name (`enp1s0`) instead of the new one (`enp34s0`) — a quick two-line fix and we were back online.

## the numbers don't lie

The difference was immediate and dramatic. Here's Nethermind processing blocks on both platforms:

| Metric | Celeron G3920 | Ryzen 5500 | Improvement |
|---|---|---|---|
| Block processing time | ~8,000ms avg | 286ms | 28x faster |
| Block throughput | 1-2 MGas/s | 46-65 MGas/s | ~30x faster |
| Transactions per second | 15-25 tps | 447-800 tps | ~30x faster |
| RAM used | 13GB (swap: 2.6GB) | 14GB (swap: 0) | Zero swap |
| Available memory | 171MB free | 16GB available | Comfortable |
| engine_newPayloadV4 errors | 2 errors / 8s avg | 0 errors / <1s avg | Clean |

The Ethereum node spec says blocks should be processed in under 12 seconds. On the Celeron, we were averaging 8 seconds with occasional spikes to 20 seconds — cutting it dangerously close. On the Ryzen, we're at 286ms. **That's processing a block in 2.4% of the allowed time.**

## memory pressure: gone

The 32GB RAM upgrade eliminated swap pressure entirely. On the Celeron, the system was constantly thrashing swap — Nethermind, Lighthouse, and the SSV node stack competing for 16GB meant the OS was regularly paging to disk. On the Ryzen, all three clients run comfortably under 20GB with 11GB sitting in cache and 16GB available.

```
Before (Celeron):
Mem:   15Gi   13Gi used   171Mi free   2.6Gi swap IN USE

After (Ryzen 5500):
Mem:   31Gi   14Gi used   6.0Gi free   0B swap used
```

## what's still the bottleneck

The chain data is still on the Crucial X6 USB drive — that's a deliberate temporary situation. A Samsung 990 Pro 4TB NVMe is en route and will slide into the second M.2 slot. Once chain data moves off USB onto NVMe, the last remaining bottleneck is gone.

| Spec | Crucial X6 (USB) | Samsung 990 Pro 4TB |
|---|---|---|
| Interface | USB 3.2 Gen 2 | PCIe 4.0 x4 |
| Sequential read | ~540 MB/s | 7,450 MB/s |
| Random 4K IOPS | ~95K | 1,400K |
| Endurance (TBW) | N/A | 2,400 TBW |

## current stack

As of May 2026, the Ethereum node is running:

| Layer | Software | Version |
|---|---|---|
| OS | Ubuntu Server 24.04 LTS | Noble |
| Execution client | Nethermind | 1.37.1 |
| Consensus client | Lighthouse | v8.1.3 |
| DVT operator | SSV Node | v2.4.2 |
| MEV | MEV-Boost | 1.12 |
| Monitoring | Grafana + Prometheus + Telegram bot | — |

Fully synced, verified, 90+ peers, SSV Operator [#2258](https://app.ssv.network/operators/2258) registered on mainnet.

## lessons learned

**Don't run an Ethereum node on a Celeron.** It'll sync — eventually — but running a live node at the edge of its processing capacity is bad for the network and bad for your operator reputation. The 12-second block time is not a suggestion.

**The X6 USB drive is fine for getting started** — it's how I learned the stack without spending money on hardware I wasn't sure about. But NVMe storage is the right long-term choice for chain data.

**Moving the OS drive is easy.** Ubuntu's LVM setup made expanding from a 29GB USB stick to a 500GB SATA SSD a 5-minute operation. The `dd` clone + `growpart` + `lvextend` + `resize2fs` sequence works exactly as documented.

**Ryzen 5500 is genuinely good node hardware.** 6 cores of Zen 3 at 65W TDP, solid single-thread performance, two M.2 slots on any B550 board. If you have one sitting around from a gaming PC upgrade, it makes an excellent Ethereum node.

**Hardware recommendation for a home Ethereum node (2026):**
- CPU: Ryzen 5500 or better (any Zen 3+)
- RAM: 32GB minimum
- Storage: 2TB+ NVMe for chain data, any SSD for OS
- Power: 65W idle, 24/7 capable PSU
- OS: Ubuntu Server 24.04 LTS

Next up: NVMe install, chain data migration off the X6, and enabling DKG on the SSV node to start attracting validators. More to come.
