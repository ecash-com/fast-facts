# Full Node Setup

The eCash L1 is Bitcoin Core v31.1 plus a ~10-commit patch stack. Operationally it behaves exactly like Bitcoin Core.

> eCash rolls out in three stages, each a fresh fork of Bitcoin mainnet with its own branch, network magic, and seeds: **alphanet** (live since 2026-08-23, fork block 963,648), **betanet** (fork block 967,680, expected ~2026-09-19), and **mainnet** (fork block ~973,728, ~2026-10-31). Alpha and beta credit practice ECX. Only mainnet ECX is permanent. 

## Where to get the software

| Source | Location |
|---|---|
| Source code | https://github.com/ecash-com/bitcoin, branch **`alphanet`** (`betanet` for the next stage, mainnet branch TBA) |
| Binaries | https://releases.ecash.com/ under `L1-ecash-bitcoin/<branch>/<commit>/L1-ecash-bitcoin-<target>.zip` (macOS arm64/x86_64, Linux x86_64, Windows x86_64); `index.json` lists the latest build per branch. |
| Integrity | GitHub build attestations: `gh attestation verify L1-ecash-bitcoin-<target>.zip --repo ecash-com/bitcoin`; SHA-256 per file in `index.json` |
| Docker | `ghcr.io/ecash-com/bitcoin:alphanet` |
| GUI / activation client | BitWindow, https://layertwolabs.com/download |

Building from source works exactly like upstream Bitcoin Core v31 (CMake). The branch README has the exact `cmake` invocation and Ubuntu dependency list.

## Differences from stock Bitcoin Core

The complete patch stack on top of v31.1:

1. Fork/activation height (`EcashHeight`): 963,648 on alphanet (= 478 × 2016, a retarget boundary). Difficulty resets to 1 at this height (consensus rule: the fork block must carry the reset target). Betanet moves the height to 967,680 and resets to 1e9.
2. Replay protection via magic `nLockTime = 499999999`. See [03](03-replay-protection-and-coin-splitting.md).
3. `setRepurposeTx`: hard-coded list of 220 txids (`src/repo_txns.h`) whose input-script checks are skipped (the Patoshi coin reassignment).
4. `OP_DRIVECHAIN` (repurposes `OP_NOP5`, 0xb4) added and made standard.
5. OP_RETURN limits removed.
6. **Own network magic and ports**: message-start bytes `0xeca5a104` on alphanet (`0xeca5b104` on betanet, testnet/regtest variants differ in the third byte) and P2P/RPC ports 8533/8532, so an eCash node cannot handshake with Bitcoin Core peers and both can run side by side on default ports.
7. Default datadir `~/.ecash` (macOS: `~/Library/Application Support/ecash`, Windows: `AppData\Local\ecash`), config file `ecash.conf`.
8. DNS seeds replaced with `seed.alpha.ecash.ninja`, `seed.alpha.bip300.xyz`, `seed.alpha.ecash.drivecha.in`, `seed.alpha.ecash.zuexeuz.net` (betanet: `seed.beta.*`).
9. `getblocktemplate` requires the caller to acknowledge a `bip300301` rule and returns `!bip300301` in every template, so miners must fetch templates through `bip300301_enforcer` ([08](08-mining.md)).
10. `CLIENT_NAME` identifies as `Bitcoin Core (eCash alphanet)`.


Everything else (RPC, wallet, ZMQ, REST, P2P) is stock v31.1.

## Network identity

eCash has **its own network magic and its own default ports (P2P 8533, RPC 8532)**, so it cannot peer with Bitcoin Core nodes or with other eCash stages, and it won't collide with a Bitcoin node's default ports or datadir on the same machine.

Two cautions:

- `getblockchaininfo` still reports `chain: main` (the fork identifies as mainnet). Monitoring must verify the chain by block hash at or after the fork height, not by chain name:

```sh
bitcoin-cli -datadir=./alphanet getblockhash 963648
# must return: 0000000000b360c17636b7a6c366e3effbe91a847eb5d61b7a7b29476439e924
```

- Use a dedicated `-datadir` per stage (alphanet, betanet, mainnet all default to `~/.ecash`, and their chains diverge from Bitcoin at different heights).

## Quick start (alphanet)

```sh
mkdir alphanet
bitcoind -datadir=./alphanet
```

Production-style config (`alphanet/ecash.conf`):

```ini
server=1
txindex=1                              # if you index by txid; omit if pruning
# prune=2000                           # pruned mode works (but not with the enforcer)
listen=0
rpcuser=<user>
rpcpassword=<password>
zmqpubsequence=tcp://127.0.0.1:29000   # real-time deposit detection
rest=1
```

## Bootstrapping: assumeutxo vs full sync

Full sync from genesis is ~850 GB (all of Bitcoin's history up to the fork, then the post-fork chain). The fast path is the official assumeutxo snapshot at the fork point:

```sh
curl -O https://data.drivechain.dev/alphanet/utxo-963648.dat     # ~9.5 GB
bitcoind -datadir=./alphanet
bitcoin-cli -datadir=./alphanet loadtxoutset utxo-963648.dat
```

- Snapshot parameters are consensus-pinned in the node (`hash_serialized` `9dcc897da984d93eda56e5c4d0c4d6b54e75df7d03f5c8ae1459f511fadf7210` at height 963,648). A tampered snapshot is rejected.
- The node is usable at the tip within minutes; a background chainstate re-validates history afterward.
- Pruned mode (`prune=2000`) works with this flow if you don't need `txindex`.

## Ports

| Service | Port |
|---|---|
| P2P | **8533** (Tor: 8534) |
| RPC | **8532** |
| ZMQ (convention) | 28332 / 29000 |
| BIP300/301 enforcer gRPC (optional) | 50051 |
| BIP300/301 enforcer block template server (miners) | 8122 |

Testnet3/testnet4/signet/regtest use 18533/48533/38533/18644 (P2P) and 18532/48532/38532/18643 (RPC).

## Mining

See [08-mining.md](08-mining.md).

## Operational notes

- `getblockchaininfo` reports `chain: main`, and RPC amounts display as `BTC` (`CURRENCY_UNIT` unchanged). Handle the ECX ticker at your application layer.
- Halving schedule, 21M cap, and block interval are inherited from Bitcoin; the ledger and supply schedule continue from the fork point.
- Expect fast, erratic blocks and elevated reorg risk while difficulty re-equilibrates.
