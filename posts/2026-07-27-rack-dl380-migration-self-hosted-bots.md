---
layout: post
title: "Rack, Rebuild, Repeat: Two Server Migrations and a NAS Build in Ten Weeks"
date: 2026-07-27
tags: [ethereum, dl380, homelab, rack, ssv, trading-bots, self-hosted, truenas]
description: "From a Ryzen tower on a desk to a proper 42U rack running a dual-Xeon Ethereum node, a NAS build in progress, and two trading bots migrated off the cloud — everything that changed since the last update."
---

Ten weeks, two server migrations, a $330 rack, and a DigitalOcean droplet finally put out to pasture. Since the [Celeron-to-Ryzen upgrade](/blog.html) in May, the homelab stopped being a collection of boxes on a desk and started being an actual rack. Here's what changed.

## the backstory

The Ryzen 5500 build from the last post was only ever meant to be a stepping stone. It proved the Ethereum stack could run well on real hardware, but a consumer AM4 board with 32GB of RAM was never going to be the endgame for running Nethermind, Lighthouse, MEV-Boost, and three SSV operators simultaneously — let alone leave headroom for trading bots and local LLM inference.

Two enterprise servers came through in quick succession: an HPE DL360 Gen10 first, then a DL380 Gen10 that became the permanent home. Along the way, a proper rack showed up, the NAS build started, and both trading bots got pulled off a DigitalOcean droplet and brought home.

## the DL380: dual xeon, 256GB RAM, and a 2.5 hour cutover

The final landing spot is an **HPE ProLiant DL380 Gen10** — dual Xeon Gold 6230R, 256GB of RAM across 4x 64GB DIMMs. Compared to the Ryzen 5500's 6 cores and 32GB, this is a different category of machine entirely.

| Component | Ryzen 5500 (May) | DL380 Gen10 (now) |
|---|---|---|
| CPU | 6c/12t, Zen 3 | Dual Xeon Gold 6230R (52c/104t total) |
| RAM | 32GB DDR4 | 256GB DDR4 |
| Boot drive | Samsung 500GB SATA SSD | 400GB Samsung SAS SSD via P408i-a |
| Chain storage | Samsung 990 Pro 4TB NVMe | Samsung 990 Pro 4TB NVMe (native slot) |
| NIC | Onboard gigabit | 622FLR-SFP28 CNA, 10G DAC to switch |
| Power draw | ~65W sustained | ~143W at moderate load |
| PSU | Standard ATX | Dual 800W Titanium, 240V 30A circuit |

The DL380 didn't come pre-built — it's a transplant job. The DL360's dual 6230R CPUs, 256GB RAM, HPE Smart Array P408i-a controller, 622FLR-SFP28 CNA, and the Samsung 990 Pro 4TB NVMe all moved over in July. The DL380 has no embedded SAS controller, so the P408i-a is mandatory just to see the boot drive. High-performance heatsinks (826706-B21 variant) were required to clear the dual-CPU thermal envelope.

**Cutover numbers:**
- Total downtime: ~2.5 hours
- Services restarted cleanly: Nethermind, Lighthouse (mainnet + Hoodi), MEV-Boost, SSV operators #2258, #621, #624, Grafana, Prometheus, both trading bots
- Post-migration thermals: CPU0 41°C, CPU1 38°C under load — Thermal Grizzly Kryonaut earning its keep
- Public IP changed from 104.229.34.32 to 104.229.32.146, which meant re-pointing SSV `HOST_ADDRESS` in both the mainnet and Hoodi `ssv.env` files

The one open item: an RTX 2070 that doesn't physically fit the DL380's riser (full-height card, no clearance). Ollama is running CPU-only for now on Mistral 7B and Phi-3 Mini — good enough for the AI trading agent's quick-screener duties, but a low-profile NVIDIA T600 8GB or A2 16GB is the likely fix once one turns up at a reasonable price.

The old DL360 chassis isn't gone — it's retained as a spare now that its guts live in the DL380.

## from desk to rack

The other big shift is physical. Everything used to sit loose on a desk. As of July, it's in a **42U Dell EMC rack** (acquired for $330), with real switching and a proper firewall:

| Gear | Role | Notes |
|---|---|---|
| TP-Link SG3428XPP-M2 | Core switch | 24x 2.5GbE PoE++, 4x 10G SFP+ |
| Protectli Vault V1410 | Firewall/router | OPNsense 26.1.11, replaced an ASUS router |
| APC UPS (240V) | DL380 power | Dual 800W Titanium PSUs on 30A circuit |
| CyberPower PR2200 (120V) | Networking gear power | Switch, firewall, AP |
| ASUS RT-BE90U | WiFi 7 AP | Access-point mode only now |

The DL380 connects to the switch over a 10G DAC on port 1/0/25. The network is still flat (10.13.1.x) for now, but VLANs are mapped out and waiting on implementation — management, data/NAS, DMZ, and WiFi segments, once there's time to do the cutover without breaking everything mid-sync.

## bringing the bots home

Both trading bots — the DCA bot and the AI trading agent — were running on a DigitalOcean droplet. As of July 25, both are fully self-hosted on the DL380, and the droplet is pending cancellation. This closes the loop on "self-sovereign infrastructure" for the whole stack, not just the nodes.

**What moved:**
- **DCA bot** (`/home/steve2z/dca-bot/`) — Mon/Wed/Fri scheduled buys across an 11-coin basket via Coinbase Advanced Trade, per-coin dip thresholds (7.5% on BTC up to 13.5% on AVAX), plus a daily trailing-stop check for non-BTC positions.
- **AI trading agent** (`/home/steve2z/ai-agent/`) — trades roughly 400 Coinbase USDC pairs using Claude for decisioning on top of RSI/EMA/MACD/Bollinger signals, Kelly-criterion position sizing, and trailing stops that arm at +2.5% and exit on a 2% retrace.
- **AI watchdog and Telegram notifier** — stop-loss monitoring and trade alerts, now running as their own systemd units alongside the agent.

Two gotchas from the move worth writing down for future-me:

**Hardcoded paths.** Both bots had `/root/dca-bot/` and `/root/ai-agent/` baked into config and scripts from the DigitalOcean days. All of it had to be updated to `/home/steve2z/` post-migration — anything missed silently failed on the next restart.

**The Coinbase key newline trap.** The Coinbase API private key needs *real* newlines in the `.env` file, not literal `\n` characters, and the value needs to be quoted for `python-dotenv` to parse it correctly. Get this wrong and the bot fails auth with an error that gives no hint the problem is formatting.

One systemd fix made the whole migration far less painful: setting `KillMode=control-group` on both bot services. Without it, restarts were leaving orphaned duplicate processes running — which for a trading bot means two processes potentially placing the same trade twice.

## SSV and Lido ICS: getting close

SSV Operator #2258 is running clean on the DL380 on mainnet, and the validator-operator eligibility clock is close: the 90-day mark lands around **August 3, 2026**. On the Hoodi testnet side, CSM Node Operator #462 is live with 2 active validators and zero strikes.

The bigger deadline is Lido's ICS Round 6 evaluation on **September 7, 2026**. Current standing: High Signal rank 27 (score 46), Human Passport score 25.9 with all 8/8 Proof-of-Humanity points, and trust level 1 on the Lido research forum. The one thing still outstanding — and the critical path item before the Round 6 evaluation — is the **CSM self-bond on mainnet**, which is pending.

## and, as of today: presearch is done

Small closing note that landed right as this post was being written: Presearch shut the platform down today, July 27, 2026. The Pi 4 node that used to run it had already been retired earlier this year, and the 4,633 PRE sitting in the node wallet got withdrawn to MetaMask before the shutdown. One less service to monitor.

## what's next: the NAS

The next big build is already underway — an **HPE Apollo 4200 Gen10**, 28-bay LFF chassis, dual Xeon Gold 6140, 128GB RAM, with TrueNAS SCALE installation still pending. The storage design is planned but not yet built out:

| Pool | Hardware | Layout | Purpose |
|---|---|---|---|
| Fast SSD | 4x 3.84TB SAS SSD | RAIDZ1 | Revenue-generating workloads |
| Bulk | 9x Seagate Exos 24TB | RAIDZ2 + hot spare | Archive/bulk storage |
| SLOG | 2x Intel Optane P4800X 375GB | Mirror | Write acceleration |
| Boot | 2x 250GB SATA SSD | Mirror | TrueNAS OS |

An LSI 9400-16i Trimode HBA (flashed IT mode) handles the drive fanout. Once TrueNAS SCALE is installed and the pools are built, this becomes the home for chain data snapshots, bot logs/history, and offsite backup staging — instead of everything living wherever there happened to be free space on a node's boot drive.

## lessons learned

**Transplanting server internals is cheaper than buying two complete servers**, but budget real time for it — heatsink compatibility, controller requirements (the P408i-a wasn't optional), and cable routing between chassis generations aren't always drop-in even within the same HPE line.

**A rack changes how you think about the homelab.** Loose boxes on a desk invite "just SSH in and fix it live." A rack with a UPS, a real firewall, and 10G switching invites actually planning changes — which is exactly why the VLAN work is still "planned" and not "done."

**Self-hosting trading bots is a different risk profile than self-hosting a blockchain node.** A node that goes down for two hours during a migration just needs to resync. A trading bot with a stale lock file or a duplicate process from a bad restart can place a real trade twice. `KillMode=control-group` and a hard look at every hardcoded path earned their place on the pre-migration checklist.

**Public IP changes ripple further than expected.** Between SSV `HOST_ADDRESS` values, DNS, and Cloudflare Tunnel configs, one IP change touches more places than it looks like at first glance. Worth keeping a checklist of every place an IP is hardcoded before the next move.

Next up: TrueNAS SCALE install on the Apollo 4200, the CSM self-bond before the September 7 ICS deadline, and — eventually — a GPU that actually fits the DL380's riser.
