# Alphanet (retired)

The first public eCash rehearsal. It forked Bitcoin at block **963,648** on 2026-08-23 and was **retired on 2026-09-24**. Its pools no longer answer, and the mobile wallet stopped offering it on 2026-10-26: existing alphanet wallets still load and can be removed or copied to a live network.

| | |
|---|---|
| Fork height | 963,648 (= 2016 × 478) |
| Difficulty at fork | reset to 1 (briefly CPU-mineable) |
| Replay `nLockTime` | `499999999` |
| Network magic | `0xeca5a104` |
| `OP_DRIVECHAIN` | `OP_NOP5` (0xb4), as written in BIP300 |
| Repurposed (Patoshi) txids | 220 |
| DNS seeds | `seed.alpha.*` |
| BIP300 thresholds | hours-scale, so a full cycle could be rehearsed in a day: 72 ACKs in 144 blocks for withdrawals and used slots, 30 for unused slots |
| UTXO snapshot | fork-point `data.drivechain.dev/alphanet/utxo-963648.dat` |
| Source / binaries | `alphanet` branch; `L1-ecash-bitcoin/alphanet/` on releases.ecash.com; Docker `ghcr.io/ecash-com/bitcoin:alphanet` |
| Enforcer | `--network-preset=alphanet` |
| Explorer (historical) | `explorer.alpha.ecash.ninja`, hub `alpha.ecash.ninja` (not reachable as of 2026-10-27) |

Alphanet burns are credited on real ECX (see the mobile wallet's `docs/alphanet-burn.md`).
