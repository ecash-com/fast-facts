# Full Node Setup

The eCash L1 is Bitcoin Core v31.1 plus a ~12-commit patch stack; operationally it behaves exactly like Bitcoin Core.

> The production chain forks Bitcoin at block ~963,648 (~August 22, 2026). Until then, integrate against **drynet3**, the live dry-run network that forked Bitcoin mainnet at block 957,600 with the same mechanics (difficulty reset, replay protection, coin repurposing). The launch network will work identically; new branch and binaries will be announced at https://drivechain.info/dev.txt. The official per-network docs (drynet1/2/3, including what changed between runs) are mirrored in [09-drynets](09-drynets/README.md).

## Where to get the software

| Source | Location |
|---|---|
| Source code | https://github.com/ecash-com/bitcoin, branch **`drynet3`** (launch branch TBA) |
| Binaries | https://releases.drivechain.info/ under `L1-ecash-bitcoin-drynet3-<platform>.zip` (macOS arm64/x86_64, Linux x86_64, Windows x86_64) |
| Integrity | SHA-256 hashes at https://releases.drivechain.info/hashes.json (no PGP signatures published yet, see [06](06-contacts-and-resources.md)) |
| Docker | `ghcr.io/ecash-com/bitcoin:drynet3` |
| GUI / activation client | BitWindow, https://layertwolabs.com/download |

Building from source works exactly like upstream Bitcoin Core v31 (CMake); see the repo's `README.md` and `INSTALL.md`.

## Differences from stock Bitcoin Core

The complete patch stack on top of v31.1:

1. Fork/activation height (`DrivechainHeight`): 957,600 on drynet3. Difficulty resets to 1 at this height (consensus rule: the fork block must have minimum difficulty).
2. Replay protection via magic `nLockTime = 499999999`. See [03](03-replay-protection-and-coin-splitting.md).
3. `setRepurposeTx`: hard-coded list of 122 txids (`src/repo_txns.h`) whose input-script checks are skipped (the Patoshi coin reassignment).
4. `OP_DRIVECHAIN` (repurposes `OP_NOP5`, 0xb4) added and made standard.
5. OP_RETURN limits removed.
6. `getblocktemplate` works with no peers and during IBD (solo mining during the low-difficulty period).
7. DNS seeds replaced with `seed.bip300.xyz`.
8. Default datadir `~/.drivechain-ecash` (macOS: `~/Library/Application Support/drivechain-ecash`), config file `drivechain-ecash.conf`.

Everything else (RPC, wallet, ZMQ, REST, P2P) is stock v31.1.

## Critical: network-identity footgun

**eCash keeps Bitcoin mainnet's network magic (`0xf9beb4d9`), chain name (`main`), default P2P port 8333, and default RPC port 8332.** An eCash node and a Bitcoin node can handshake, and they share identical history below the fork height.

- Never run an eCash node with an open peer policy on default ports next to Bitcoin infrastructure. Pin peers (`connect=`/`addnode=`) or rely only on `seed.bip300.xyz`.
- Always use a dedicated `-datadir`.
- drynet2 and drynet3 share all history up to 957,600; the same caution applies between drynets.
- Monitoring must verify the chain by block hash at or after the fork height, not by chain name (`getblockchaininfo` reports `main`):

```sh
bitcoin-cli -datadir=./drynet3 getblockhash 957600
# must return: 000000004ecdabd8c45d5623aa40008f24f1a5d45851265fc6b905250d2ee840
```

## Quick start (drynet3)

```sh
mkdir drynet3
bitcoind -datadir=./drynet3 -addnode=drynet3.drivechain.dev:8337
```

Production-style config (`drynet3/drivechain-ecash.conf`):

```ini
server=1
txindex=1                              # if you index by txid; omit if pruning
# prune=2000                           # pruned mode works (assumeutxo-friendly)
connect=drynet3.drivechain.dev:8337    # exclusive peering to a known eCash node
listen=0
rpcuser=<user>
rpcpassword=<password>
zmqpubsequence=tcp://127.0.0.1:29000   # real-time deposit detection
rest=1
```

`connect=` (exclusive) guarantees the node cannot wander onto the wrong network; switch to multiple `addnode=` lines once you trust your peer set.

## Bootstrapping: assumeutxo vs full sync

Full sync from genesis is ~850 GB (all of Bitcoin's history). The fast path is the official assumeutxo snapshot at the fork point:

```sh
curl -O https://data.drivechain.dev/drynet3/utxo-957600.dat     # ~9 GB
bitcoind -datadir=./drynet3 -addnode=drynet3.drivechain.dev:8337 -pause
bitcoin-cli -datadir=./drynet3 loadtxoutset utxo-957600.dat
```

- Snapshot parameters are consensus-pinned in the node (`hash_serialized` `4417b3a580ba4fbdd80287df16d72e875939ac799bf71c087a476227687867d4`, 166,322,090 coins); a tampered snapshot is rejected.
- The node is usable at the tip within minutes; a background chainstate re-validates history afterward.
- Pruned mode (`prune=2000`) works with this flow if you don't need `txindex`.

## Ports

| Service | Port |
|---|---|
| P2P | 8333 (default; public drynet3 node listens on **8337**) |
| RPC | 8332 |
| ZMQ (convention) | 28332 / 29000 |
| BIP300/301 enforcer gRPC (optional) | 50051 |

## Mining

At the fork block, difficulty resets to 1 and the chain is CPU-mineable until difficulty re-equilibrates (roughly the first 2,016 blocks):

```sh
bitcoin-cli -datadir=./drynet3 createwallet mine
bitcoin-cli -datadir=./drynet3 getnewaddress
bitcoin-cli -datadir=./drynet3 generatetoaddress 1 <your-address> 2000000000
```

Or point any SHA-256d getblocktemplate miner at RPC port 8332 (e.g. `minerd -a sha256d -o http://127.0.0.1:8332 -O user:pass --coinbase-addr=<addr>`).

## Operational notes

- `getblockchaininfo` reports `chain: main`, and RPC amounts display as `BTC` (`CURRENCY_UNIT` unchanged). Handle the ECX ticker at your application layer.
- Halving schedule, 21M cap, and block interval are inherited from Bitcoin; the ledger and supply schedule continue from the fork point.
- Expect fast, erratic blocks and elevated reorg risk during the difficulty re-equilibration window.
