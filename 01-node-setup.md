# Full Node Setup

The eCash L1 is Bitcoin Core v31.1 plus a patch stack of about a dozen commits. Operationally it behaves exactly like Bitcoin Core.

> Mainnet forks Bitcoin at block **973,728** (est. 2026-10-29 as of 2026-10-27; the height is what counts). Alphanet and betanet were rehearsals with different parameters. If you ran them, read [testnets/](testnets/README.md) for what changed, and never reuse their datadirs.

> ⚠️ **Required upgrade.** Release `ecash-v1.1.0-core31.1` (commit `5bd8690fab`, 2026-10-26) changes post-fork difficulty retargeting. Nodes still on `ecash-v1.0.0-core31.1` (`6e582335d1`) follow the old 2,016-block rule and **split off the chain at block 974,016**, the first post-fork retarget. Upgrade before then.

## Where to get the software

| Source | Location |
|---|---|
| Source code | https://github.com/ecash-com/bitcoin, branch **`master`**, tag **`ecash-v1.1.0-core31.1`** |
| Binaries | https://releases.ecash.com/L1-ecash-bitcoin/mainnet/latest/: `release.json` plus `L1-ecash-bitcoin-<target>.zip` for `aarch64-apple-darwin`, `x86_64-apple-darwin`, `x86_64-unknown-linux-gnu`, `x86_64-w64-msvc`. Pinned path: `.../mainnet/5bd8690fab60/`. `index.json` lists every network. Each zip unpacks to `L1-ecash-bitcoin-31.1.0-5bd8690fab60-<target>/` containing `bitcoind`, `bitcoin-cli`, `bitcoin-util` and the GUI `qt/bitcoin-qt` |
| Integrity | SHA-256 per file in `release.json` / `index.json`, and GitHub build attestations: `gh attestation verify L1-ecash-bitcoin-<target>.zip --repo ecash-com/bitcoin` |
| Docker | `ghcr.io/ecash-com/bitcoin:latest` (built from `master`; currently the same digest as `:sha-5bd8690`). Pin `:sha-5bd8690` in production. There is no `:mainnet` tag |
| GUI / activation client | BitWindow, https://ecash.com/download ([06](06-contacts-and-resources.md#binaries)) |

Building from source works exactly like upstream Bitcoin Core v31 (CMake). On Ubuntu 24.04 (from the branch README):

```sh
sudo apt install git cmake build-essential libsqlite3-dev libboost-all-dev libzmq3-dev pkgconf
cmake -B build -DBUILD_GUI=OFF -DBUILD_BENCH=OFF -DBUILD_FUZZ_BINARY=OFF -DBUILD_GUI_TESTS=OFF \
  -DBUILD_TESTS=OFF -DENABLE_IPC=OFF -DWITH_ZMQ=ON -DBUILD_UTIL=ON
cmake --build build -j $(nproc)
```

## Differences from stock Bitcoin Core

The complete mainnet patch stack on top of v31.1:

1. **Fork height** `EcashHeight = 973,728` (= 2016 × 483, a retarget boundary). The fork block must carry `nBits = 0x1901a6e1` (difficulty ~2.6e9).
2. **Post-fork difficulty algorithm**: retarget every 288 blocks with a dampened step ([below](#difficulty-after-the-fork)). New in v1.1.0.
3. **Replay protection** via magic `nLockTime = 499999998` (`LOCKTIME_THRESHOLD - 2`). See [03](03-replay-protection-and-coin-splitting.md).
4. **`setRepurposeTx`**: a hard-coded list of 232 txids (`src/repo_txns.h`) whose input-script checks are skipped (the Patoshi coin reassignment). This is mainnet's own list; it shares no txids with betanet's.
5. **`OP_DRIVECHAIN`** = `OP_NOP8` (0xb7), made standard. BIP300 as written uses `OP_NOP5`; eCash moved it on betanet and kept it for mainnet.
6. OP_RETURN policy is **unchanged** from upstream v31 (no eCash patch): up to ~100 kB of data-carrier output, multiple outputs allowed.
7. **Own network magic and ports**: mainnet message-start `0xeca5dcb7`, P2P 8533, RPC 8532. The eCash binary's testnet3/testnet4/regtest use `0xeca5b114`/`0xeca5b124`/`0xeca5b134`. An eCash node cannot handshake with Bitcoin Core peers, and both can run side by side on default ports.
8. **Default datadir** `~/.ecash` (macOS `~/Library/Application Support/eCash`, Windows `%LOCALAPPDATA%\eCash`), config file `ecash.conf`.
9. **DNS seeds** replaced with `seed.ecash.ninja`, `seed.bip300.xyz`, `seed.ecash.drivecha.in`, `seed.ecash.zuexeuz.net`, `seed.ecash.eu.com`. No hard-coded fixed seeds ([below](#peers-and-seed-nodes)).
10. **`getblocktemplate`** requires the caller to acknowledge a `bip300301` rule and returns `!bip300301` in every template, so miners must fetch templates through `bip300301_enforcer` ([08](08-mining.md)). `-deprecatedrpc=getblocktemplate` restores the plain call, for testing only.
11. `CLIENT_NAME` is `eCash`; the P2P user agent is `/eCash Satoshi:31.1.0/`.

Everything else (RPC, wallet, ZMQ, REST, P2P) is stock v31.1.

## Difficulty after the fork

From block 973,728 the node switches to its own difficulty rule (`src/pow.cpp`, v1.1.0):

| | |
|---|---|
| Retarget interval | every **288 blocks** (~2 days at 10-minute blocks), at heights divisible by 288 |
| First retarget | block **974,016**. It measures from the fork block 973,728 to 974,015 |
| Rule | `new_target = old_target × (6 × T + measured) / (7 × T)`, with `T` = 288 × 600 s = 48 h. This moves only 1/7 of the way toward the fully corrected value each window |
| Clamp | `measured` is clamped to [T/4, 4T]. Difficulty can therefore rise at most **~12%** per window and fall at most **~30%** (÷1.43) |
| Timewarp fix | each window starts at the previous window's last block, so no gap between windows can be exploited |

**Implications for integrators:** the fork difficulty of ~2.6e9 gives 10-minute blocks at only ~18.6 PH/s. A capped ~12% rise per 288 blocks means that if much more hashrate turns up, blocks will stay fast for many windows. Do not tie anything to "N blocks ≈ N × 10 minutes" in the first weeks. Use block heights for confirmations and BIP300 windows, and expect elevated reorg risk.

## Network identity

eCash has **its own network magic and its own default ports (P2P 8533, RPC 8532)**. It cannot peer with Bitcoin Core nodes or with alphanet/betanet nodes, and it won't collide with a Bitcoin node's default ports on the same machine.

Cautions:

- `getblockchaininfo` still reports `chain: main`, and RPC amounts are labeled `BTC`. Monitoring must verify the chain by block hash, not by chain name:

```sh
# Before the fork: your node tracks the shared chain, so its tip equals Bitcoin's
bitcoin-cli -datadir=./ecash-mainnet getblockhash 973727   # equals Bitcoin's block 973,727 once mined

# After the fork: block 973,728 is eCash-only
bitcoin-cli -datadir=./ecash-mainnet getblockhash 973728
# must DIFFER from Bitcoin's 973,728 and MATCH https://esplora.ecash.ninja/block-height/973728
```

- **Use a dedicated `-datadir`**. Every eCash stage defaults to `~/.ecash`, and on macOS and Windows betanet's `ecash` folder and mainnet's `eCash` folder are the same directory on case-insensitive filesystems. A mainnet binary started on a betanet datadir sees a chain that left Bitcoin at 967,680.

## Peers and seed nodes

| Seed | Status 2026-10-27 |
|---|---|
| `seed.ecash.ninja` | resolves, port 8533 open (this is also the P2P address in [drivechain.dev/config](https://drivechain.dev/config)) |
| `seed.ecash.zuexeuz.net` | resolves, port 8533 open |
| `seed.bip300.xyz` | not resolving yet |
| `seed.ecash.drivecha.in` | not resolving yet |
| `seed.ecash.eu.com` | not resolving yet |

There are no fixed seeds compiled in. If DNS seeding fails (firewalled DNS, seeds down), add a peer explicitly:

```ini
addnode=seed.ecash.ninja:8533
```

**Before the fork**, an eCash node still syncs Bitcoin's history, but only from other eCash nodes (the magic differs). New Bitcoin blocks reach the eCash network through bridge nodes: the `mainnet-bridge` branch implements one, which peers with Bitcoin nodes and relays their data to eCash peers with the magic translated. As a result, an eCash node's tip tracks Bitcoin's until block 973,727 (the public mainnet explorer did so on 2026-10-27).

## Quick start

```sh
mkdir ecash-mainnet
bitcoind -datadir=./ecash-mainnet       # mainnet is the default network; no -chain flag
```

Production-style config (`ecash-mainnet/ecash.conf`):

```ini
server=1
txindex=1                              # if you index by txid; omit if pruning
# prune=2000                           # pruned mode works (but not with the enforcer)
listen=0                               # set listen=1 and open 8533 to help the network
rpcuser=<user>
rpcpassword=<password>
zmqpubsequence=tcp://127.0.0.1:29000   # real-time deposit detection
zmqpubhashblock=tcp://127.0.0.1:28332
rest=1
```

## Bootstrapping: assumeutxo

A full sync from genesis is ~860 GB (Bitcoin's history up to the fork, then the post-fork chain). The fast path is the published snapshot of Bitcoin's UTXO set at height 965,000, which is pinned in the mainnet node:

```sh
curl -O https://releases.ecash.com/assumeutxo/mainnet/utxo-965000.dat     # 9,468,312,119 bytes
curl -O https://releases.ecash.com/assumeutxo/mainnet/SHA256SUMS
sha256sum -c SHA256SUMS            # 89351f7106051c0685880f9c64026e993335308de63ae6980e94ba3fafdc7dda

bitcoind -datadir=./ecash-mainnet
# wait until getblockchaininfo shows headers past 965,000, then:
bitcoin-cli -datadir=./ecash-mainnet -rpcclienttimeout=0 loadtxoutset utxo-965000.dat
```

- Pinned parameters (`src/kernel/chainparams.cpp`): height 965,000, block `00000000000000000001595977e6000ce56129f5c9b4073e31ccc30b90b97da9`, `hash_serialized` `4a8d794337118c0c615b574f817c7306c687584a537184b8d233df42bf477ec2`. A tampered snapshot is rejected. The upstream heights 840,000 / 880,000 / 910,000 / 935,000 are also accepted.
- The pre-fork UTXO set is identical to Bitcoin's, so you can also produce the file yourself on a synced Bitcoin Core v31 node: `dumptxoutset utxo-965000.dat rollback=965000`.
- The node is usable at the snapshot height within minutes. A background chainstate re-validates history afterward.
- Pruned mode (`prune=2000`) works with this flow if you don't need `txindex`.

## Ports

| Service | Port |
|---|---|
| P2P | **8533** (Tor: 8534) |
| RPC | **8532** |
| ZMQ (convention) | 28332 / 29000 |
| BIP300/301 enforcer gRPC (optional) | 50051 |
| BIP300/301 enforcer block template server (miners) | 8122 |

The eCash binary's testnet3/testnet4/signet/regtest use 18533/48533/38533/18644 (P2P) and 18532/48532/38532/18643 (RPC).

## Mining

See [08-mining.md](08-mining.md).

## Operational notes

- `getblockchaininfo` reports `chain: main`, and RPC amounts display as `BTC` (`CURRENCY_UNIT` unchanged). Handle the ECX ticker at your application layer.
- The halving schedule, 21M cap, 100-block coinbase maturity and 10-minute target spacing are inherited from Bitcoin. The block subsidy at the fork is 3.125 ECX.
- Expect fast, erratic blocks and elevated reorg risk while difficulty re-equilibrates ([above](#difficulty-after-the-fork)).
