# Mining

## Fast facts

| | |
|---|---|
| Algorithm | **SHA-256d**, identical to Bitcoin. Any Bitcoin miner works unmodified: ASICs and home miners (Bitaxe, NerdAxe). The fork difficulty (~2.6e9) rules out CPU mining |
| Network | **mainnet**, live since 2026-10-29. The first eCash block was **973,728**, built on Bitcoin's block 973,727 with `nBits = 0x1901a6e1` (difficulty ~2.6e9). On 2026-11-13 the tip was 984,268 and difficulty ~8.0e10 (10-minute blocks at ~0.57 EH/s) |
| Difficulty after the fork | retarget **every 288 blocks**, `new = old × (6T + measured) / 7T`. At most ~12% harder or ~30% easier per retarget. First retarget was at **974,016**. Needs node **v1.1.0** or later; current is **v1.2.0** ([01](01-node-setup.md#difficulty-after-the-fork)) |
| Block reward | 3.125 ECX + fees; spendable after 100 confirmations (standard coinbase maturity) |
| Public pools | six in the mainnet registry as of 2026-11-13, four with a public stratum ([below](#pools)) |
| Solo mining | point a `getblocktemplate` miner at the **enforcer's** template server, never at the node directly. Needs enforcer **v1.0.0 or later** (current v1.0.1) ([below](#solo-mining)) |
| Live stats | https://explorer.ecash.ninja/mining |
| Test networks | betanet pool and enforcer preset: [testnets/betanet.md](testnets/betanet.md#mining-on-betanet) |

**What the 288-block rule means for miners:** difficulty climbs by at most ~12% every 288 blocks, so a surplus of hashrate takes many windows to absorb. That is what happened after the fork: the first 5,760 blocks took under 8 hours, and difficulty went from 2.6e9 to ~8.9e10 by 2026-11-04. In the other direction, if hashrate leaves, difficulty falls by at most ~30% per window, and each window still needs 288 blocks: the window ending at 983,232 took about five days. Expect stale blocks and reorgs when blocks are fast, and long gaps when hashrate drops.

## Pools

The pool list lives in [`LayerTwo-Labs/mining-pools`](https://github.com/LayerTwo-Labs/mining-pools) (`networks/mainnet/pools.json`), rendered at [pool.drivechain.info](https://pool.drivechain.info). The explorer uses the same list to attribute blocks by `coinbase_tag`. As of 2026-11-13 (registry file updated 2026-11-03):

| Pool | Mode, fee | Stratum | Notes |
|---|---|---|---|
| eCPool.tech | proportional, 1% | `stratum+tcp://mine.ecpool.tech:3334` | username is your address; the coinbase pays the PPLNS window directly. Port 3336 is solo |
| bip300.xyz | solo, 0.1% | `stratum+tcp://pool.bip300.xyz:3334` | username is your address; 3335 (Braiins) and 3336 (NiceHash) for rented hashrate |
| avonpool | pplns-thunder, 0.01% | `stratum+tcp://pool.avonpool.xyz:3334` | username is a **Thunder** address; balances settle on the Thunder sidechain |
| supapool | solo, 1% | `stratum+tcp://ecx.stratum.supapool.xyz:3383` (TLS 3384) | ckpool; username is your ECX address |
| Pow.re | solo, 0% | none | Pow.re's own fleet only (DATUM) |
| FreeBank | solo, 0% | none | private solo mining by the FreeBank sidechain operator |

Open listing PRs on 2026-11-13: SegPool ECX ([#32](https://github.com/LayerTwo-Labs/mining-pools/pull/32)), MergedHash ([#33](https://github.com/LayerTwo-Labs/mining-pools/pull/33)), eCashPool.io ([#35](https://github.com/LayerTwo-Labs/mining-pools/pull/35)). Over the week to 2026-11-13 the explorer attributed 615 of 1,090 blocks to eCPool.tech, 20 to Pow.re and 16 to bip300.xyz; 439 were untagged. A listing is not an endorsement; check a pool's terms and its share of blocks yourself.

The `ecash` entry in [drivechain.dev/config](https://drivechain.dev/config) still has no `mining_pool`. To list a pool, open a PR that appends to `networks/mainnet/pools.json` (with `"chain": "mainnet"` and a unique `coinbase_tag`) and run `tools/build.py`.

How the pools pay: the username is normally an **L1 Bitcoin-format address** (`<address>.<rig>`, any password). In `solo` and `pplns-coinbase` modes the block's own coinbase pays miners directly, so the pool never custodies funds. Check each pool's `mode` in the registry.

## Solo mining

**The eCash node refuses plain `getblocktemplate` calls.** A `template` request that does not acknowledge the `bip300301` rule fails with `you should not be calling getblocktemplate from this daemon, instead call it from bip300301_enforcer`, and every template is returned with `!bip300301` in `rules`. Node templates contain no BIP300/301 coinbase data (sidechain and withdrawal ACKs, BMM commitments). Only the enforcer adds those. (`-deprecatedrpc=getblocktemplate` re-enables the plain call for testing.)

So the solo stack is: **eCash node → `bip300301_enforcer` (template server) → your GBT miner or stratum pool.**

> **Mainnet enforcer status (2026-11-13):** mainnet support shipped. [PR #666](https://github.com/LayerTwo-Labs/bip300301_enforcer/pull/666) was merged on 2026-10-27 and released in **v1.0.0**; the current release is **v1.0.1** (2026-10-29), which supports Bitcoin Core majors 30 to 32 and so works with node v1.2.0. `--network-preset` is gone. The enforcer detects eCash from the node's user agent (`/eCash…`) or takes `--fork ecash`, and then applies `OP_NOP8`, activation height 973,728 and replay `nLockTime` `499999998` on its wallet sends. Binaries: `bip300301-enforcer/stable` in [releases.ecash.com/index.json](https://releases.ecash.com/index.json). **Do not run a pre-1.0 enforcer against a mainnet node**: without eCash parameters it assumes `OP_NOP5` and activation height 0.

### 1. Run the node

A synced, **unpruned** mainnet node running `ecash-v1.2.0-core32.1` with RPC and ZMQ enabled. The enforcer refuses pruned nodes; assumeutxo-bootstrapped nodes are fine. Setup: [01-node-setup.md](01-node-setup.md).

```ini
# ecash-mainnet/ecash.conf
server=1
rpcuser=user
rpcpassword=pass
zmqpubsequence=tcp://127.0.0.1:29000
# no prune=; the enforcer needs full blocks
```

```sh
bitcoind -datadir=./ecash-mainnet
bitcoin-cli -datadir=./ecash-mainnet getblockchaininfo   # headers/blocks at the tip, not in IBD
```

`getblocktemplate` requires a connected, fully synced node.

### 2. Choose a payout address

Any L1 address you control. Use a fresh, eCash-only key so you never mix coinbase ECX with pre-fork coins:

```sh
bitcoin-cli -datadir=./ecash-mainnet createwallet mine
bitcoin-cli -datadir=./ecash-mainnet getnewaddress
```

### 3. Run the enforcer (v1.0.1)

```sh
bip300301_enforcer \
  --fork=ecash \
  --node-rpc-addr=127.0.0.1:8532 \
  --node-rpc-user=user --node-rpc-pass=pass \
  --node-zmq-addr-sequence=tcp://127.0.0.1:29000 \
  --enable-wallet \
  --enable-mempool        # with the wallet enabled, serves getblocktemplate
```

`--fork=ecash` is optional when the node is an eCash build, because the fork is detected from its user agent. The enforcer serves `getblocktemplate` on `127.0.0.1:8122` by default (`--serve-rpc-addr`), with no authentication. `--gbt-cache-lifetime-s` controls template caching. Without a wallet, use `--enable-block-template-server --enable-mempool --coinbase-recipient=<address>` instead of `--enable-wallet`. Sanity check:

```sh
curl -s --data '{"method":"getblocktemplate","params":[{"rules":["segwit"]}]}' http://127.0.0.1:8122
# "height" should be the node's tip + 1 (above 984,000 in November 2026)
```

### 4. Point a miner at the enforcer

Any GBT-compatible SHA-256d miner, aimed at **8122** instead of the node's 8532:

```sh
minerd -a sha256d -o http://127.0.0.1:8122 --coinbase-addr=<any-address>
```

- **The reward goes to `--coinbase-recipient`** (or the enforcer wallet), not to the miner's own `--coinbase-addr`, because the miner uses the served coinbase as-is. A pool that wants to pay its miners replaces the reward output while keeping the commitment `OP_RETURN`s byte-for-byte.

Alternatively, run a stratum pool against the enforcer so ordinary ASICs can connect without touching GBT; see [running your own pool](#running-your-own-pool-simplepool) below.

## Running your own pool: simplepool

[simplepool](https://github.com/LayerTwo-Labs/simplepool) is a single-binary stratum server written in C11. Miners connect over stratum with their payout address as the username. In the default `solo` mode each block's coinbase pays the finding miner directly, minus an operator fee, so the pool never custodies funds. Shares are recorded in SQLite; optional Redis pub/sub feeds a dashboard.

```
miners (stratum)
        |
   simplepool  ── getblocktemplate / submitblock ──>  bip300301_enforcer GBT server (8122)
                                                              |
                                                       eCash bitcoind (RPC 8532 + ZMQ)
```

simplepool has no network-specific configuration. It only speaks JSON-RPC to the template backend, and on eCash that backend **must** be the enforcer.

### Build and run

```sh
git clone https://github.com/LayerTwo-Labs/simplepool
cd simplepool
# macOS: brew install sqlite curl hiredis node   Debian/Ubuntu: apt install build-essential libsqlite3-dev libcurl4-openssl-dev libhiredis-dev sqlite3
make
mkdir -p data && sqlite3 data/shares.db < schema.sql
cp proxy.conf.example proxy.conf
./build/simplepool proxy.conf
```

Minimal `proxy.conf`, pointing at the enforcer started as in [solo mining](#solo-mining):

```ini
listen_addr = 0.0.0.0
listen_port = 3334
bitcoind_url = http://127.0.0.1:8122
# omit bitcoind_user and bitcoind_pass entirely; the enforcer's GBT
# endpoint takes no basic-auth, and simplepool then sends none
bitcoind_poll_interval_ms = 30000

operator_address = <your bc1... address>
fee_bps = 100                      # 1% operator fee (0..1000)
coinbase_tag = /yourpool/          # register this tag in the pool registry

pool_mode = solo
db_path = ./data/shares.db
```

With fast early blocks, consider a shorter `bitcoind_poll_interval_ms` and make sure the template backend notifies on new blocks, or miners will work on stale templates.

### Pool modes

simplepool ships five, selected by `pool_mode`:

| `pool_mode` | Coinbase pays | Username | Who carries the variance |
|---|---|---|---|
| `solo` (default) | the miner who found the block | Bitcoin address | nobody; you are paid what you find |
| `pps-classic` | the pool | Thunder address | the operator, from a reserve |
| `pplns-thunder` | the pool | Thunder address | the miners |
| `pplns-btc` | the pool | Bitcoin address | the miners |
| `pplns-coinbase` | the whole PPLNS window, directly | Bitcoin address | the miners |

- **`solo`**: the coinbase pays the finding miner directly, minus `fee_bps` to `operator_address`. No off-chain accounting. The simplest choice.
- **`pps-classic`**: miners authorize with a bare base58 Thunder address and accrue `pps_credits` per share. The operator batches L1 funds into Thunder, and a payout worker settles on the sidechain. Requires the Thunder sidechain, which was activated on mainnet in slot 9 at block 978,324 ([05](05-sidechains-and-l2s.md)).
- **`pplns-thunder`** / **`pplns-btc`**: the coinbase pays the pool, and a matured block is split across the shares in the window (`pplns_window_diff_multiple` × network difficulty), so the pool never owes more than it has just been paid. Fees are included in the split. Paid over Thunder or in BTC-format L1 coins respectively.
- **`pplns-coinbase`**: what the LayerTwo Labs betanet pool ran. Same PPLNS accounting, but the block's own coinbase pays every miner in the window, so the pool holds nothing. Miners below 546 sats, or beyond the coinbase's output slots, roll to the front of the next block's queue.
- An earlier `pps` mode that paid a BIP300 deposit output in the coinbase has been dropped; the enforcer does not credit coinbase-source deposits.

### Deployment

`scripts/deploy-to-server.sh` provisions a full host (systemd units for pool and dashboard, nginx, UFW). A Docker Compose setup under `deploy/docker/` covers the stratum proxy, dashboard and payout worker; it expects bitcoind, the enforcer and (for Thunder rails) a Thunder node on the host. Integration test: `scripts/regtest/` brings up the whole stack against a local regtest node.
