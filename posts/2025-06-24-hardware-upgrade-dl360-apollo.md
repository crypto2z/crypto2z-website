# Doubling Down: Xeon Gold 6230R Swap on the DL360 and a New Beast Joins the Rack

*June 24, 2025 · Steve2z · Infrastructure*

---

The homelab has been quiet on the outside but loud on the inside this week — and I mean that literally. Two major hardware moves happened back-to-back: a dual-CPU upgrade on the DL360 Ethereum node, and the official rack debut of the HPE Apollo 4200 Gen10 NAS chassis. Both are in, both are humming, and the numbers speak for themselves.

Let's get into it.

---

## The DL360 Gets New Brains: Dual Xeon Gold 6230R

The DL360 Gen10 has been the workhorse of the crypto2z stack since I migrated off the Ryzen 5500 desktop — running Nethermind, Lighthouse, MEV-Boost, and three SSV operator nodes (mainnet #2258 and Hoodi testnet #621/#624) simultaneously, 24/7. It's been solid. But when a matched pair of Xeon Gold 6230R showed up at the right price on eBay, I couldn't say no.

**Before:** Dual Intel Xeon Gold 6140 (18 cores / 36 threads per socket)  
**After:** Dual Intel Xeon Gold 6230R (26 cores / 52 threads per socket)

| Spec | Xeon Gold 6140 | Xeon Gold 6230R |
|---|---|---|
| Cores / Threads | 18C / 36T per socket | 26C / 52T per socket |
| Base / Boost | 2.3 GHz / 3.7 GHz | 2.1 GHz / 4.0 GHz |
| L3 Cache | 24.75 MB | 35.75 MB |
| TDP | 150W | 150W |
| **Total (dual socket)** | **36 cores / 72 threads** | **52 cores / 104 threads** |

That's 44% more cores at the same TDP envelope, with a higher boost clock for single-threaded BLS signature work in the validator pipeline. Same power draw, substantially more throughput.

---

## Performance: Before vs. After

Here's the real-world delta, captured with the same sysbench workload run before the swap and again after the new CPUs settled in.

### CPU Benchmark — Multi-Thread (sysbench, prime numbers limit 20000)

| | Before (dual 6140) | After (dual 6230R) | Delta |
|---|---|---|---|
| Threads | 72 | 104 | +44% |
| Events/sec | 21,168 | 28,281 | **+33.6%** |
| Avg latency (ms) | ~3.0 (est.) | 3.68 | — |

### CPU Benchmark — Single-Thread

| Metric | After (6230R) |
|---|---|
| Events/sec | 495.74 |
| Avg latency (ms) | 2.02 |
| 95th percentile (ms) | 2.14 |
| Max latency (ms) | 8.77 |

### Lighthouse Beacon API Response

| Metric | Result |
|---|---|
| `/eth/v1/node/syncing` response time | **17ms** |
| Sync status | ✅ Fully synced |
| Head slot | 14,626,000 |
| EL offline | No |

### System Load (post-swap, services running)

| Metric | Value |
|---|---|
| CPU idle | 98.2% |
| Load avg (1m / 5m / 15m) | 19.16 / 6.39 / 4.31 |
| Uptime | 21 hours post-swap |

The 1-minute load spike to 19 reflects the sysbench run itself — steady-state with Nethermind, Lighthouse, MEV-Boost, and SSV all running is well under 10% CPU utilization. The machine is barely breathing with this workload now.

---

## The RAM Shuffle

The upgrade wasn't just CPUs. The outgoing Xeon Gold 6140s went directly into the Apollo 4200, along with matching RAM — zero hardware wasted. The DL360 keeps its 128GB of matched Micron DDR4 ECC (2x 64GB DIMMs), now properly paired to the 6230R's memory controllers for optimal channel utilization.

Clean cascade. Every component found a home.

---

## The Apollo 4200 Joins the Rack

The second big move: the **HPE Apollo 4200 Gen10** is now physically racked and powered on. If you haven't seen one of these in person, it's a 2U chassis with 28 Large Form Factor (LFF) bays — an absolute unit of a NAS platform built for serious storage density.

![The crypto2z rack — patch panel and networking top, DL360 Ethereum node mid-rack, HPE Apollo 4200 NAS chassis filling the bottom. Those empty drive bays won't stay empty long.](/images/serverrack.jpg)

**Current state of the Apollo:**

| Component | Status |
|---|---|
| CPUs | Dual Xeon Gold 6140 (from DL360) ✅ |
| RAM | 128GB DDR4 ECC ✅ |
| iLO | Updated to v3.20, static IP 192.168.50.12 ✅ |
| OS | TrueNAS SCALE — install imminent |
| HBA | LSI 9400-16i (in box, full-height bracket pending) |
| Drives | Inbound — SAS SSDs + Seagate Exos 24TBs |

---

## The Storage Architecture

When the drives arrive, the Apollo is getting a purpose-built ZFS layout:

**Fast SSD Pool — Revenue Workloads**  
4x 3.84TB SAS SSDs in RAIDZ1. This is where Erigon archive, Graph/PostgreSQL, and the DCA bot will live once migrated off DigitalOcean. Low latency, high IOPS, 100% local.

**Bulk Storage Pool — Long-Term Archive**  
9x Seagate Exos 24TB in RAIDZ2 with a hot spare. Roughly 192TB raw, ~144TB usable. Chain snapshots, backups, media — everything that needs to exist but doesn't need to be fast.

**Write Acceleration — SLOG**  
A mirrored pair of Intel Optane P4800X 375GB drives. Optane's write latency is in a different league from NAND — this will keep ZFS sync writes from becoming a bottleneck on the bulk pool even under heavy load.

Total raw capacity when fully loaded: well north of 200TB.

---

## What's Next

The Apollo is waiting on two things before TrueNAS goes in:

1. **LSI 9400-16i full-height bracket** — the HBA is ready but needs the correct bracket for the Apollo's PCIe slot
2. **Drive order arriving** — SAS SSDs and Exos 24TBs are inbound

Once those land: TrueNAS SCALE install, pool configuration, and VM deployment. The DCA bot migration off DigitalOcean is one of the first planned workloads — cutting the cloud droplet and bringing everything home where it belongs.

Also on the near-term list: the **Protectli V1410** (OPNsense firewall) arrives July 3rd, and the **TP-Link SG3428X-M2** 24-port 2.5GbE managed switch is already in hand. Together those implement the 4-VLAN architecture — Management, Trusted, TrueNAS, and Crypto — that properly segments this whole stack.

The rack is filling up. Busy few weeks ahead.

---

*Enjoying the build? Support the stack at [crypto2z.com/support](https://crypto2z.com/support) — Lightning tips always welcome at steve2z@crypto2z.com.*

*Follow along on Nostr: npub1j23mjaze3w0ces5ttwjxpk9lddgdvl35u46n6ndh3y482uzg3wwqyr3m6q*
