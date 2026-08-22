---
title: Why I Switched from Lighthouse to Teku — Client Diversity, SSV, and the Road to IDVTC
date: 2026-08-21
category: ethereum
tags: teku, lighthouse, ssv, dvt, lido, idvtc, client diversity, ethereum staking
excerpt: The Lighthouse to Teku migration wasn't a preference — it was a requirement. Here's why client diversity matters for DVT clusters, what the migration actually looked like, and the gotchas that almost killed the switch.
---

## It Started With a Cluster

In mid-2026 I was invited to join an IDVTC cluster — a 4-seat Lido Distributed Validator Technology cluster forming for ICS Round 6. The other seats: Colinka in Prague (ICS-approved operator, runs Obol Divine Dragon), chuygarcia from SSV Foundation in Mexico, and an ICS operator in Argentina. I'm the US East seat.

An IDVTC cluster isn't just a technical configuration — it's a credibility signal toward Lido's Institutional DVT Cluster program. The September 21st submission deadline was real, the ICS decision landing September 7th was real, and there was a hard technical requirement I wasn't meeting.

Three of the four operators in the cluster were running Lighthouse. Against a 3-of-4 signing threshold, that's a single-client majority failure scenario. If a Lighthouse bug took down three of four nodes simultaneously, the cluster couldn't sign. The fix was client diversity, and that meant I had to move.

## Why Teku

The field for Ethereum consensus clients is real but narrower than it looks in practice. Prysm, Lighthouse, Teku, Nimbus, Lodestar. Most DVT operators default to Lighthouse because it's fast, well-documented, and the tooling ecosystem around it is mature.

Teku is the ConsenSys implementation — Java-based, enterprise-focused, built for environments where operational predictability matters more than raw performance. For a DVT cluster seat where uptime and attestation reliability are the primary metrics, Teku's operational profile made sense.

There was also a practical consideration: I was already running Lighthouse on Hoodi testnet for CSM validators and wanted to keep that distinct from the mainnet consensus client. Running Teku mainnet and Lighthouse Hoodi gives clear separation with no port conflicts and independent failure domains.

## The Gotchas — Read These Before You Start

If you're migrating from Lighthouse to Teku on mainnet, the documentation will not prepare you for these.

### Java version

Teku requires **Java 25**. Not 21. Not 17. 25.

The failure mode if you install the wrong version is genuinely misleading:

```
Unrecognized option: --sun-misc-unsafe-memory-access=allow
Error: Could not create the Java Virtual Machine.
```

Nothing in that error message says "wrong Java version." You'll spend time wondering if the binary is corrupted or the config is malformed before you think to check `java -version`. Pin `JAVA_HOME` explicitly in the systemd unit file so an `apt upgrade` can't silently downgrade it:

```ini
[Service]
Environment=JAVA_HOME=/usr/lib/jvm/java-25-openjdk-amd64
```

### The binary is not on GitHub

Teku's GitHub releases page has source archives and checksums. The actual compiled binary is on Cloudsmith — ConsenSys's artifact repository:

```bash
TAG="26.8.0"
curl -L "https://artifacts.consensys.net/public/teku/raw/names/teku.tar.gz/versions/${TAG}/teku-${TAG}.tar.gz" -o teku.tar.gz
```

If you go to the GitHub releases page looking for a `.tar.gz` you'll find it empty. This cost me an hour.

### QUIC port is independent

Teku has two P2P ports: `--p2p-port` and `--p2p-quic-port`. The QUIC port defaults to 9001 regardless of what you set `--p2p-port` to. It does not derive from the TCP port.

When Lighthouse was running mainnet at 9000/9001 (P2P/QUIC) and I set Teku's `--p2p-port=9003`, I assumed QUIC would follow at 9004. It didn't — Teku tried to bind 9001 and collided with the (now disabled but not yet cleaned up) Lighthouse QUIC listener.

Set both explicitly:

```bash
--p2p-port=9003 \
--p2p-quic-port=9004
```

And add matching port forwards in your router. If mainnet CL P2P is on 9003/9004 and your router is still forwarding 9000/9001, peers will plateau at single digits while everything looks fine locally.

### Docker bridge IP auto-detection

This one is subtle and will haunt you. Teku tries to detect its own network interface automatically. On a machine running Docker, it will find the Docker bridge at `172.17.0.1` or similar and advertise that as its P2P address. The result: 0-1 peers, no inbound connections, the node looks synced but is isolated.

The fix is explicit:

```bash
--p2p-interface=10.13.1.10 \
--p2p-advertised-ip=<YOUR WAN IP>
```

Your WAN IP may change. Check it with `curl -s https://api.ipify.org` before setting it — don't rely on what you think it is.

### SSV integration port

SSV Node needs to know where to find the consensus client's REST API. After the Lighthouse-to-Teku swap, the port moved from 5052 (which Lighthouse used and which Teku also uses by default) — but SSV was connecting via the Docker bridge IP, not localhost.

In the SSV Node config, the beacon node address needs to be:

```
http://172.18.0.1:5052
```

Not `localhost:5052`. Docker containers see the host at the bridge gateway address, and that address varies between Docker networks. Verify with `docker network inspect` and match it to whatever bridge your SSV container is actually on.

### RSS memory is not a leak

Teku's RSS footprint runs at roughly 2x its configured heap (`-Xmx`). On a 16GB heap setting you'll see 16GB+ in `htop`. This is normal — RocksDB operates off-heap, the BLS and KZG cryptographic libraries have their own native memory allocation, and thread stacks add up. This is not a memory leak. It scared me for about 20 minutes until I confirmed the Teku forums have multiple threads with identical numbers.

### The REST API comes up before P2P

This one will bite you if you're checking health by curling the API. Teku starts the REST API endpoint immediately on launch, before P2P is initialized. If the service is crash-looping, the API may still respond for a few seconds before the process dies.

Always verify health with both:

```bash
systemctl is-active teku.service
curl http://localhost:5052/eth/v1/node/syncing
```

Not just the curl. The curl will lie to you.

## The Migration Itself

The actual cutover was staged. Lighthouse mainnet was kept running right up until Teku was fully synced and healthy.

**Step 1 — Install Java 25 and Teku binary**

```bash
sudo apt install openjdk-25-jdk
java -version  # Verify: 25.x

TAG="26.8.0"
curl -L "https://artifacts.consensys.net/public/teku/raw/names/teku.tar.gz/versions/${TAG}/teku-${TAG}.tar.gz" -o teku.tar.gz
tar xzf teku.tar.gz
sudo mv teku-${TAG} /opt/teku
```

**Step 2 — Configure the service**

The Teku systemd unit with all the production-required flags:

```ini
[Unit]
Description=Teku Ethereum Consensus Client (Mainnet)
Wants=nethermind.service
After=nethermind.service

[Service]
User=steve2z
Environment=JAVA_HOME=/usr/lib/jvm/java-25-openjdk-amd64
ExecStart=/opt/teku/bin/teku \
  --network=mainnet \
  --data-path=/mnt/ethereum/teku \
  --ee-endpoint=http://localhost:8551 \
  --ee-jwt-secret-file=/mnt/ethereum/secrets/jwt.hex \
  --rest-api-enabled=true \
  --rest-api-host-allowlist=* \
  --rest-api-interface=0.0.0.0 \
  --p2p-port=9003 \
  --p2p-quic-port=9004 \
  --p2p-interface=10.13.1.10 \
  --p2p-advertised-ip=<WAN_IP> \
  --metrics-enabled=true \
  --metrics-port=8008 \
  --validators-proposer-default-fee-recipient=<FEE_RECIPIENT> \
  --beacon-liveness-tracking-enabled=true \
  Xmx=8g
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

**Step 3 — Let it sync while Lighthouse still runs**

Teku does checkpoint sync quickly from a trusted beacon node. Let it run alongside Lighthouse for a day. Watch the peer count climb and confirm the beacon API is responding correctly.

**Step 4 — Cut SSV over to Teku**

Update SSV Node config to point `beacon_node_addr` at `http://172.18.0.1:5052` (or whichever bridge IP your SSV container uses). Restart SSV and confirm it reconnects.

**Step 5 — Disable Lighthouse mainnet**

```bash
sudo systemctl disable lighthousebeacon.service
sudo systemctl stop lighthousebeacon.service
```

Note: disable, don't delete. The data stays at `/mnt/ethereum/lighthouse/` as a rollback option. Storage is cheap.

## Current State

As of August 21, 2026:

- Teku v26.8.0 running mainnet CL — synced, healthy peer count
- Lighthouse v8.2.2 running Hoodi CL only — completely separate from mainnet
- SSV Node v2.4.3 connected to Teku at `172.18.0.1:5052`
- Lighthouse mainnet service disabled, data retained

The port forward situation at the router:

| Service | Port | Protocol |
|---|---|---|
| Teku P2P | 9003 | TCP + UDP |
| Teku QUIC | 9004 | UDP |
| Lighthouse Hoodi | 9100 | TCP + UDP |
| SSV TCP | 13001 | TCP |
| SSV P2P | 12001 | UDP |
| SSV DKG | 3030 | TCP |

## The Bigger Picture

The migration wasn't complicated once I knew the gotchas. The hard part was finding the gotchas — they're scattered across the Teku docs, GitHub issues, and Discord threads with no single source that covers all of them. Hopefully this post is that single source for the next person.

The IDVTC cluster submission is September 21st. The ICS decision comes September 7th. If ICS comes through, the cluster forms and the DVT journey that started with registering SSV operator #2258 in May gets its first real validators.

Client diversity in a DVT cluster isn't optional — it's the entire point of DVT. One client going down shouldn't take the cluster down. Moving to Teku was the prerequisite for making that promise credible.

More on the IDVTC cluster, the Lido CSM Hoodi dry run, and what it actually takes to qualify for ICS in a future post.
