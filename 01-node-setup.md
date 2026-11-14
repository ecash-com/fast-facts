# Full Node Setup

The eCash L1 is Bitcoin Core v32.1 plus a patch stack of a dozen commits. Operationally it behaves exactly like Bitcoin Core.

> Mainnet forked from Bitcoin at block **973,728** on 2026-10-29 18:59:55 UTC. Alphanet and betanet were rehearsals with different parameters. If you ran them, read [testnets/](testnets/README.md) for what changed, and never reuse their datadirs.

> ⚠️ **Version check.** The current release is **`ecash-v1.2.0-core32.1`** (commit `c8474e0b4f2b`, tagged 2026-10-29): the same eCash patch stack as v1.1.0, rebased from Bitcoin Core 31.1 onto 32.1 (`src/pow.cpp` and `src/repo_txns.h` are unchanged). `ecash-v1.1.0-core31.1` (`5bd8690fab`, 2026-10-26) introduced the post-fork difficulty rule and is the minimum. Nodes still on `ecash-v1.0.0-core31.1` (`6e582335d1`) follow the old 2,016-block rule and **split off the chain at block 974,016**, the first post-fork retarget.

## Where to get the software

| Source | Location |
|---|---|
| Source code | https://github.com/ecash-com/bitcoin, branch **`master`**, tag **`ecash-v1.2.0-core32.1`** |
| Binaries | https://releases.ecash.com/L1-ecash-bitcoin/mainnet/latest/release.json names the current files. They are `L1-ecash-bitcoin-<version>-<target>.zip` under `.../mainnet/<version>/` (now `.../mainnet/1.2.0-core32.1/`), for `aarch64-apple-darwin`, `x86_64-apple-darwin`, `x86_64-unknown-linux-gnu`, `x86_64-w64-msvc`. `index.json` lists every network. Each zip unpacks to `L1-ecash-bitcoin-1.2.0-core32.1-c8474e0b4f2b-<target>/` containing `bitcoind`, `bitcoin-cli`, `bitcoin-util` and the GUI `qt/bitcoin-qt`. **The unversioned `latest/L1-ecash-bitcoin-<target>.zip` is still served but is the old v1.1.0 build** (checked 2026-11-13); do not use it |
| Integrity | SHA-256 per file in `release.json` / `index.json`, and GitHub build attestations: `gh attestation verify L1-ecash-bitcoin-<version>-<target>.zip --repo ecash-com/bitcoin --signer-workflow ecash-com/releases/.github/workflows/publish.yml` |
| Docker | `ghcr.io/ecash-com/bitcoin:latest` (built from `master`; currently the same digest as `:sha-c8474e0`). Pin `:sha-c8474e0` in production. There is no `:mainnet` tag |
| GUI / activation client | BitWindow, https://ecash.com/download ([06](06-contacts-and-resources.md#binaries)) |

Building from source works exactly like upstream Bitcoin Core v32 (CMake). On Ubuntu 24.04 (from the branch README):

```sh
sudo apt install git cmake build-essential libsqlite3-dev libboost-all-dev libzmq3-dev pkgconf
cmake -B build -DBUILD_GUI=OFF -DBUILD_BENCH=OFF -DBUILD_FUZZ_BINARY=OFF -DBUILD_GUI_TESTS=OFF \
  -DBUILD_TESTS=OFF -DENABLE_IPC=OFF -DWITH_ZMQ=ON -DBUILD_UTIL=ON
cmake --build build -j $(nproc)
```

## Differences from stock Bitcoin Core

The complete mainnet patch stack on top of v32.1:

1. **Fork height** `EcashHeight = 973,728` (= 2016 × 483, a retarget boundary). The fork block must carry `nBits = 0x1901a6e1` (difficulty ~2.6e9).
2. **Post-fork difficulty algorithm**: retarget every 288 blocks with a dampened step ([below](#difficulty-after-the-fork)). Added in v1.1.0.
3. **Replay protection** via magic `nLockTime = 499999998` (`LOCKTIME_THRESHOLD - 2`). See [03](03-replay-protection-and-coin-splitting.md).
4. **`setRepurposeTx`**: a hard-coded list of 232 txids (`src/repo_txns.h`) whose input-script checks are skipped (the Patoshi coin reassignment). This is mainnet's own list; it shares no txids with betanet's.
5. **`OP_DRIVECHAIN`** = `OP_NOP8` (0xb7), made standard. BIP300 as written uses `OP_NOP5`; eCash moved it on betanet and kept it for mainnet.
6. OP_RETURN policy is **unchanged** from upstream v32 (no eCash patch): up to ~100 kB of data-carrier output, multiple outputs allowed.
7. **Own network magic and ports**: mainnet message-start `0xeca5dcb7`, P2P 8533, RPC 8532. The eCash binary's testnet3/testnet4/regtest use `0xeca5b114`/`0xeca5b124`/`0xeca5b134`. An eCash node cannot handshake with Bitcoin Core peers, and both can run side by side on default ports.
8. **Default datadir** `~/.ecash` (macOS `~/Library/Application Support/eCash`, Windows `%LOCALAPPDATA%\eCash`), config file `ecash.conf`.
9. **DNS seeds** replaced with `seed.ecash.ninja`, `seed.bip300.xyz`, `seed.ecash.drivecha.in`, `seed.ecash.zuexeuz.net`, `seed.ecash.eu.com`. No hard-coded fixed seeds ([below](#peers-and-seed-nodes)).
10. **`getblocktemplate`** requires the caller to acknowledge a `bip300301` rule and returns `!bip300301` in every template, so miners must fetch templates through `bip300301_enforcer` ([08](08-mining.md)). `-deprecatedrpc=getblocktemplate` restores the plain call, for testing only.
11. `CLIENT_NAME` is `eCash`; the P2P user agent is `/eCash Satoshi:1.2.0(Bitcoin Core 32.1.0)/`. `getnetworkinfo` reports `version` 320100 (the Core version).

Everything else (RPC, wallet, ZMQ, REST, P2P) is stock v32.1. Upstream v32 changes that integrators may notice: the PSBT-creating RPCs default to version 2 PSBTs (`psbt_version` selects another), `getdeploymentinfo` no longer lists `taproot`, and `bip125-replaceable` is gone from the mempool RPCs unless `-deprecatedrpc=bip125` is set (see `doc/release-notes.md` in the tag).

## Difficulty after the fork

From block 973,728 the node uses its own difficulty rule (`src/pow.cpp`, since v1.1.0):

| | |
|---|---|
| Retarget interval | every **288 blocks** (~2 days at 10-minute blocks), at heights divisible by 288 |
| First retarget | block **974,016**. It measured from the fork block 973,728 to 974,015 |
| Rule | `new_target = old_target × (6 × T + measured) / (7 × T)`, with `T` = 288 × 600 s = 48 h. This moves only 1/7 of the way toward the fully corrected value each window |
| Clamp | `measured` is clamped to [T/4, 4T]. Difficulty can therefore rise at most **~12%** per window and fall at most **~30%** (÷1.43) |
| Timewarp fix | each window starts at the previous window's last block, so no gap between windows can be exploited |

**What happened:** the fork difficulty of ~2.6e9 gives 10-minute blocks at only ~18.6 PH/s, and far more hashrate turned up. Difficulty rose by the maximum 12% in almost every window for the first three days, the first 20 windows (5,760 blocks) took under 8 hours, and the chain reached block 984,268 by 2026-11-13: about 10,500 blocks in 15 days, where a 10-minute chain would have produced about 2,200. Difficulty peaked near 8.9e10 on 2026-11-04, fell to 6.4e10, and was ~8.0e10 on 2026-11-13 (explorer: `/api/v1/mining/difficulty-adjustments/1m`).

**Implications for integrators:** do not tie anything to "N blocks ≈ N × 10 minutes". Block intervals still swing as hashrate moves and difficulty follows at most ~12% up or ~30% down per window. Use block heights for confirmations and BIP300 windows, and expect elevated reorg risk.

## Network identity

eCash has **its own network magic and its own default ports (P2P 8533, RPC 8532)**. It cannot peer with Bitcoin Core nodes or with alphanet/betanet nodes, and it won't collide with a Bitcoin node's default ports on the same machine.

Cautions:

- `getblockchaininfo` still reports `chain: main`, and RPC amounts are labeled `BTC`. Monitoring must verify the chain by block hash, not by chain name:

```sh
# Last shared block: equals Bitcoin's block 973,727
bitcoin-cli -datadir=./ecash-mainnet getblockhash 973727

# First eCash block
bitcoin-cli -datadir=./ecash-mainnet getblockhash 973728
# must be 0000000000000001441a6a4ae49d679d4a30581db3e3eaeeb2fba5da6dc146ed
# (Bitcoin's 973,728 is 0000000000000000000104150b08ff89bbb2d11bab42c245c82b8165ab2e2c11)
# and match https://esplora.ecash.ninja/block-height/973728
```

- **Use a dedicated `-datadir`**. Every eCash stage defaults to `~/.ecash`, and on macOS and Windows betanet's `ecash` folder and mainnet's `eCash` folder are the same directory on case-insensitive filesystems. A mainnet binary started on a betanet datadir sees a chain that left Bitcoin at 967,680.

## Peers and seed nodes

| Seed | Status 2026-11-13 |
|---|---|
| `seed.ecash.ninja` | resolves, port 8533 open (this is also the P2P address in [drivechain.dev/config](https://drivechain.dev/config)) |
| `seed.ecash.zuexeuz.net` | resolves |
| `seed.bip300.xyz` | resolves |
| `seed.ecash.drivecha.in` | resolves |
| `seed.ecash.eu.com` | resolves |

There are no fixed seeds compiled in. If DNS seeding fails (firewalled DNS, seeds down), add a peer explicitly:

```ini
addnode=seed.ecash.ninja:8533
```

An eCash node syncs Bitcoin's pre-fork history (blocks 0 to 973,727) from other eCash nodes only, because the magic differs. Before the fork, new Bitcoin blocks reached the eCash network through bridge nodes (the `mainnet-bridge` branch), which peered with Bitcoin nodes and relayed their data with the magic translated. Since block 973,728 the network follows its own chain.

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

A full sync from genesis is ~890 GB (Bitcoin's history up to the fork, then the post-fork chain). The fast path is the published snapshot of Bitcoin's UTXO set at height 965,000, which is pinned in the mainnet node:

```sh
curl -O https://releases.ecash.com/assumeutxo/mainnet/utxo-965000.dat     # 9,468,312,119 bytes
curl -O https://releases.ecash.com/assumeutxo/mainnet/SHA256SUMS
sha256sum -c SHA256SUMS            # 89351f7106051c0685880f9c64026e993335308de63ae6980e94ba3fafdc7dda

bitcoind -datadir=./ecash-mainnet
# wait until getblockchaininfo shows headers past 965,000, then:
bitcoin-cli -datadir=./ecash-mainnet -rpcclienttimeout=0 loadtxoutset utxo-965000.dat
```

- Pinned parameters (`src/kernel/chainparams.cpp`): height 965,000, block `00000000000000000001595977e6000ce56129f5c9b4073e31ccc30b90b97da9`, `hash_serialized` `4a8d794337118c0c615b574f817c7306c687584a537184b8d233df42bf477ec2`. A tampered snapshot is rejected. The upstream heights 840,000 / 880,000 / 910,000 / 935,000 are also accepted.
- The pre-fork UTXO set is identical to Bitcoin's, so you can also produce the file yourself on a synced Bitcoin Core node: `dumptxoutset utxo-965000.dat rollback=965000`.
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
- Expect erratic block intervals and elevated reorg risk while difficulty re-equilibrates ([above](#difficulty-after-the-fork)).
