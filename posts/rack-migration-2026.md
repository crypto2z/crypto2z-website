---
title: "From Desktop to Rack: Building a Sovereign Crypto Infrastructure"
date: "2026-06-10"
author: "steve2z"
tags: ["ethereum", "bitcoin", "lightning", "ssv", "homelab", "rack", "hardware"]
---

# From Desktop to Rack: Building a Sovereign Crypto Infrastructure

When I started running crypto nodes, everything ran on a Raspberry Pi 4 with a USB drive. Fast forward to today and the basement now houses a proper rack with a CyberPower UPS, a 24-port patch panel, and an HPE ProLiant DL360 Gen10 server on the way. Here's how it happened and why.

## Why Self-Sovereign Infrastructure?

The core philosophy at crypto2z is simple: if you don't run it yourself, someone else controls it. That applies to Bitcoin, Ethereum, Lightning routing, and validator infrastructure. Managed services are convenient until they're not — and when they fail, you lose control at exactly the wrong moment.

Running your own nodes isn't just about ideology. It's increasingly about income. SSV Network pays operators to run distributed validators. Lightning routing fees compound with every new channel. The Graph will pay indexers for serving subgraph data. None of that works without hardware you own and control.

## The Hardware Journey

### Phase 1: Raspberry Pi Era

The journey started with a Pi 4 running MyNode — Bitcoin full node, LND Lightning, everything on a USB SSD. It worked, but the JMicron USB controller was a bottleneck and the Pi 4's 4GB RAM left little room to grow.

The upgrade to a **Raspberry Pi 5 (16GB, NVMe)** in the Argon ONE V5 case was transformative. Proper NVMe speeds, 16GB RAM, and MyNode 0.3.44 running cleanly. The Pi 5 remains dedicated to Bitcoin and Lightning — it does that job perfectly and there's no reason to change it.

### Phase 2: The Ethernet Node

As SSV Network grew, the need for a dedicated Ethereum node became clear. Validators need both an execution client (Nethermind) and a consensus client (Lighthouse) running reliably 24/7. A Ryzen 5500 with 32GB RAM and a Samsung 990 Pro 4TB NVMe became the Ethereum node.

The stack running on that machine:
- **Nethermind** — Ethereum execution client (mainnet + Hoodi testnet)
- **Lighthouse** — Ethereum consensus client (mainnet + Hoodi testnet)
- **SSV Node v2.4.2** — SSV Network Operator #2258 (mainnet)
- **Anchor v1.2.3** — SSV Network Operator #624 (Hoodi testnet, Rust client by Sigma Prime)
- **MEV-Boost** — connected to Flashbots, Ultra Sound, and Aestus relays
- **Presearch** — passive search node, 4,374 PRE staked

Running both mainnet and testnet on 32GB proved challenging — swap usage regularly climbed to 80% after a few days. That's what drove the next upgrade.

### Phase 3: The Rack Build

A 20U Quest half rack in the basement became the foundation. The build happened incrementally:

**Network layer (top):**
- 24-port CAT7 keystone patch panel
- TP-Link switch mounted on a custom 3D printed bracket
- SB8200 modem in a 3D printed 1U mount
- ASUS RT-BE90U WiFi 7 router sitting on top (metal enclosure kills WiFi 7 signals)

**Power layer (bottom):**
- CyberPower PR2200 2200VA rackmount UPS
- 10-outlet 3600J surge-protected PDU with lightning protection

The short Cat8 patch cables between the patch panel and switch made the biggest aesthetic difference — going from 3-foot cables to 6-inch runs transformed the look from "pile of gear" to professional installation.

### Phase 4: The Server Migration (In Progress)

The Ryzen 5500 served well but 32GB RAM running dual mainnet+testnet Ethereum clients was its ceiling. The solution: an **HPE ProLiant DL360 Gen10** with 192GB DDR4 ECC RAM and dual Xeon Gold 6140 processors.

The upgrade path:
- Fresh Ubuntu 24.04 install on a 200GB SAS SSD boot drive
- Samsung 990 Pro 4TB NVMe via PCIe adapter (existing chain data, no re-sync needed)
- CPU swap to dual Xeon Gold 6230R (26 cores each, 4.0GHz turbo) when they arrive
- Static IP at 192.168.50.10 for the new node
- All services migrate: Nethermind, Lighthouse, SSV, Anchor, MEV-Boost, Presearch

192GB RAM means the swap pressure problem disappears permanently.

## SSV Network: Becoming a Verified Operator

The most significant development in 2026 has been building toward SSV Verified Operator status. SSV Network lets node operators run distributed validators on behalf of ETH stakers, earning fees in SSV tokens for every validator managed.

The setup required:
1. **Mainnet Operator #2258** — Go SSV node, DKG enabled at port 3030
2. **Hoodi Testnet Operator #621** — Go SSV node for testnet validation
3. **Hoodi Testnet Operator #624** — Anchor v1.2.3 (Rust implementation by Sigma Prime)

Running Anchor alongside the Go implementation demonstrates commitment to client diversity — exactly what SSV Network needs as it matures into a multi-client protocol like Ethereum itself.

The Verified Operator application is submitted and under review. Once approved, the operator profile at app.ssv.network shows a verified badge that signals reliability to validators choosing their operator cluster.

## The DCA Bot: Automating Accumulation

Running on a DigitalOcean droplet, a Python bot executes Dollar Cost Averaging purchases on Coinbase every Monday, Wednesday, and Friday at 9am UTC:

- **BTC:** $15/run → auto-withdrawn to Lightning node after each purchase
- **ETH, ADA, AVAX, ATOM, POL, XLM, XRP:** $2/coin per run
- **Total:** $29/run, ~$87/week

Additional logic handles dip buying (per-coin thresholds of 7.5-13.5%), stop limits (5% 2-hour drop triggers sell + re-entry), and trailing stops (30% from 30-day high triggers 50% sell for non-BTC coins).

The BTC auto-withdrawal to Lightning means every scheduled purchase adds on-chain balance toward the next Lightning channel swap. Currently at 110,000+ sats in channels across 4 active channels.

## Lightning Network Growth

The Lightning node (alias: steve2z, pubkey: 027f5cbf...) has grown through LN+ triangle swaps:

- **3 completed swaps** — all rated 100% happy
- **4 active channels** — 240,223 sats total capacity
- **Inbound liquidity** — 122,807 sats receivable
- **Rank:** Mercury 4

The strategy is simple: DCA into BTC, auto-withdraw to Lightning, accumulate on-chain, open new channels via LN+ triangles. Each swap adds inbound liquidity and routing capability.

## What's Next

**Near term (this month):**
- Complete DL360 Gen10 migration
- Retire the Ryzen 5500 ethnode
- Attract first SSV validators after VO badge approval

**Medium term (2026):**
- Build toward 1M+ sats in Lightning channels
- Expand SSV validator count
- Stake earned SSV tokens at 24%+ APR in cSSV

**Long term (2027):**
- TrueNAS NAS build with 36-60 bay Supermicro chassis
- Ethereum archive node for RPC income
- The Graph indexer for query fee income
- Supermicro X13SEW-TF (WIO, LGA4677, DDR5, PCIe 5.0)

The infrastructure pays for itself as validators, routing fees, and archive node income compound. That's the sovereign crypto infrastructure thesis in practice.

---

*Running nodes at crypto2z.com. Lightning address: steve2z@crypto2z.com*  
*SSV Operator #2258 | Nostr: steve2z@crypto2z.com*
