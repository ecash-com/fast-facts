# Test Networks (Drynets, Alphanet, Betanet)

eCash was rehearsed on a series of throwaway forks of Bitcoin before mainnet. Each one was a fresh fork of Bitcoin mainnet with its own branch, network magic, seeds and replay marker. **Test-network coins are not native mainnet ECX** (beta coins are called "practice ECX" or pECX). Mainnet starts again from Bitcoin's ledger at block 973,727; test-network balances do not automatically carry over. Any separate burn or redemption programme has its own terms ([below](#redemption-programmes)).

The main docs in this repo describe **mainnet only**. This folder keeps the test-network parameters for anyone still running a rehearsal or reading old logs.

| Stage | Fork height | Date | Status | Branch | Magic | Replay `nLockTime` | `OP_DRIVECHAIN` |
|---|---|---|---|---|---|---|---|
| drynet1-4 | various | 2026 | retired | `drynet1`-`drynet4` | | | |
| [Alphanet](alphanet.md) | 963,648 | 2026-08-23 | retired 2026-09-24 | `alphanet` | `0xeca5a104` | `499999999` | `OP_NOP5` (0xb4) |
| [Betanet](betanet.md) | 967,680 | 2026-09-19 | live, no sunset date announced | `betanet` | `0xeca5b104` | `499999999` | `OP_NOP8` (0xb7) |
| **Mainnet** ([main docs](../README.md)) | **973,728** | est. 2026-10-29 | upcoming, permanent | `master` (tag `ecash-v1.1.0-core31.1`) | `0xeca5dcb7` | **`499999998`** | `OP_NOP8` (0xb7) |

## What changes between betanet and mainnet

If you tested on betanet, these are the values to change for mainnet:

| | Betanet | Mainnet |
|---|---|---|
| Replay `nLockTime` | `499999999` (`LOCKTIME_THRESHOLD - 1`) | **`499999998`** (`LOCKTIME_THRESHOLD - 2`). With a non-final input sequence, a mainnet send stamped with the betanet value is non-final on both eCash and Bitcoin |
| Fork height / last shared block | 967,680 / 967,679 | 973,728 / 973,727 |
| Difficulty at fork | ~1e9 (`0x19044b7e`) | ~2.6e9 (`0x1901a6e1`) |
| Retargeting after the fork | Bitcoin's 2,016-block rule | **every 288 blocks**, dampened `(6 × old + measured) / 7` ([details](../01-node-setup.md#difficulty-after-the-fork)) |
| Network magic | `0xeca5b104` | `0xeca5dcb7` |
| DNS seeds | `seed.beta.*` | `seed.ecash.ninja`, `seed.bip300.xyz`, `seed.ecash.drivecha.in`, `seed.ecash.zuexeuz.net`, `seed.ecash.eu.com` |
| Repurposed (Patoshi) txids | 232 | 232, a **different** set (each stage pins its own list) |
| Unused-slot sidechain activation | more than 1,008 of 2,016 blocks | more than 1,815 of 2,016 blocks |
| macOS / Windows datadir | `.../ecash` | `.../eCash` (same folder on case-insensitive filesystems, so always pass an explicit `-datadir`) |
| Hostnames | `*.beta.ecash.ninja`, `*.beta.bip300.xyz` | `*.ecash.ninja` ([04](../04-explorers-and-apis.md)) |
| Enforcer | `--network-preset=betanet` | not merged yet ([08](../08-mining.md#solo-mining)) |
| UTXO snapshot | none at the fork point (upstream 935,000 only) | `utxo-965000.dat`, pinned in the node |

On both networks, replay protection also requires at least one input **`nSequence < 0xffffffff`**. With all-final sequences, Bitcoin ignores either locktime value. A successful betanet split does not split mainnet coins ([replay guide](../03-replay-protection-and-coin-splitting.md)).

Unchanged: ports 8533/8532, address and key formats, `OP_DRIVECHAIN` = `OP_NOP8`, the withdrawal thresholds (13,150 ACKs in 26,300 blocks), and the `getblocktemplate` `bip300301` rule.

## Redemption programmes

Some test-network balances have their own programmes. Alphanet burns are credited on real ECX (see the mobile wallet's `docs/alphanet-burn.md`), and the Solana `wbECX` beta token is not redeemable for real ECX after 2026-10-31. These programmes are separate from the fork and have their own terms. A beta coin split does not split mainnet coins.
