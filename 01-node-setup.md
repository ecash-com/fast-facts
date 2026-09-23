# Full Node Setup

The eCash L1 is Bitcoin Core v31.1 plus a ~10-commit patch stack. Operationally it behaves exactly like Bitcoin Core.

> eCash rolls out in three stages, each a fresh fork of Bitcoin mainnet with its own branch, network magic, and seeds: **alphanet** (forked 2026-08-23 at block 963,648, retired 2026-09-24), **betanet** (live since 2026-09-19, fork block 967,680), and **mainnet** (fork block ~973,728, ~2026-10-31). Alpha and beta credit practice ECX. Only mainnet ECX is permanent.

## Where to get the software

| Source | Location |
|---|---|
| Source code | https://github.com/ecash-com/bitcoin, branch **`betanet`** (`alphanet` is retired, mainnet branch TBA) |
| Binaries | https://releases.ecash.com/ under `L1-ecash-bitcoin/<branch>/<commit>/L1-ecash-bitcoin-<target>.zip` (macOS arm64/x86_64, Linux x86_64, Windows x86_64); `index.json` lists the latest build per branch. |
| Integrity | GitHub build attestations: `gh attestation verify L1-ecash-bitcoin-<target>.zip --repo ecash-com/bitcoin`; SHA-256 per file in `index.json` |
| Docker | `ghcr.io/ecash-com/bitcoin:betanet` |
| GUI / activation client | BitWindow, https://layertwolabs.com/download |

Building from source works exactly like upstream Bitcoin Core v31 (CMake). The branch README has the exact `cmake` invocation and Ubuntu dependency list.

## Differences from stock Bitcoin Core

The complete patch stack on top of v31.1:

1. Fork/activation height (`EcashHeight`): 967,680 on betanet (= 480 × 2016, a retarget boundary). Difficulty resets to ~1e9 at this height (`EcashForkBits = 0x19044b7e`; consensus rule: the fork block must carry the reset target), then normal 2,016-block retargeting resumes. Alphanet used 963,648 with a reset to 1.
2. Replay protection via magic `nLockTime = 499999999`. See [03](03-replay-protection-and-coin-splitting.md).
3. `setRepurposeTx`: hard-coded list of 232 txids on betanet (220 on alphanet; `src/repo_txns.h`) whose input-script checks are skipped (the Patoshi coin reassignment).
4. `OP_DRIVECHAIN` added and made standard. Betanet repurposes `OP_NOP8` (0xb7); alphanet used `OP_NOP5` (0xb4). The enforcer's `--network-preset` selects the matching opcode.
5. OP_RETURN limits removed.
6. **Own network magic and ports**: message-start bytes `0xeca5b104` on betanet (`0xeca5a104` on alphanet; the third byte is the stage, the fourth byte selects testnet3/testnet4/regtest as `14`/`24`/`34`) and P2P/RPC ports 8533/8532, so an eCash node cannot handshake with Bitcoin Core peers and both can run side by side on default ports.
7. Default datadir `~/.ecash` (macOS: `~/Library/Application Support/ecash`, Windows: `AppData\Local\ecash`), config file `ecash.conf`.
8. DNS seeds replaced with `seed.beta.ecash.ninja`, `seed.beta.bip300.xyz`, `seed.beta.ecash.drivecha.in`, `seed.beta.ecash.zuexeuz.net` (alphanet used `seed.alpha.*`). No hard-coded fixed seeds.
9. `getblocktemplate` requires the caller to acknowledge a `bip300301` rule and returns `!bip300301` in every template, so miners must fetch templates through `bip300301_enforcer` ([08](08-mining.md)). `-deprecatedrpc=getblocktemplate` restores the plain call for testing only.
10. `CLIENT_NAME` identifies as `Bitcoin Core (eCash betanet)`.


Everything else (RPC, wallet, ZMQ, REST, P2P) is stock v31.1.

## Network identity

eCash has **its own network magic and its own default ports (P2P 8533, RPC 8532)**, so it cannot peer with Bitcoin Core nodes or with other eCash stages, and it won't collide with a Bitcoin node's default ports or datadir on the same machine.

Two cautions:

- `getblockchaininfo` still reports `chain: main` (the fork identifies as mainnet). Monitoring must verify the chain by block hash at or after the fork height, not by chain name:

```sh
bitcoin-cli -datadir=./betanet getblockhash 967680
# must return: 00000000000000030101ba5cfea54b22becc79f95dc6040beb76e01dd9d04042
```

- Use a dedicated `-datadir` per stage (alphanet, betanet, mainnet all default to `~/.ecash`, and their chains diverge from Bitcoin at different heights).

## Quick start (betanet)

```sh
mkdir betanet
bitcoind -datadir=./betanet
```

Production-style config (`betanet/ecash.conf`):

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

Full sync from genesis is ~850 GB (all of Bitcoin's history up to the fork, then the post-fork chain). The fast path is assumeutxo:

- **Betanet has no fork-point snapshot.** The `betanet` branch pins only the upstream Bitcoin Core snapshot heights (840,000, 880,000, 910,000, 935,000), and no `data.drivechain.dev/betanet/` download is published. Load a height-935,000 snapshot and sync the remaining ~32,700 pre-fork blocks plus the post-fork chain. The pre-fork UTXO set is identical to Bitcoin's, so any synced Bitcoin Core v31 node can produce the snapshot (it rolls back temporarily while dumping):

```sh
# on a synced Bitcoin Core (or eCash) node
bitcoin-cli -rpcclienttimeout=0 -named dumptxoutset utxo-935000.dat rollback=935000

# on the betanet node
bitcoind -datadir=./betanet
bitcoin-cli -datadir=./betanet loadtxoutset utxo-935000.dat
```

- Snapshot parameters are consensus-pinned in the node (`hash_serialized` `e4b90ef9eae834f56c4b64d2d50143cee10ad87994c614d7d04125e2a6025050` at height 935,000). A tampered snapshot is rejected.
- Alphanet did ship a pinned fork-point snapshot (`data.drivechain.dev/alphanet/utxo-963648.dat`); watch the DcInsiders Telegram for whether mainnet gets one.
- The node is usable at the snapshot height within minutes; a background chainstate re-validates history afterward.
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
