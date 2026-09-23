# Mining

## Fast facts

| | |
|---|---|
| Algorithm | **SHA-256d**, identical to Bitcoin. Any Bitcoin miner works unmodified: ASICs, home miners (Bitaxe, NerdAxe). Betanet's difficulty (~1e9 at the fork) rules out CPU mining |
| Network | **betanet** (live since 2026-09-19). Forked Bitcoin at block **967,680** (hash `00000000000000030101ba5cfea54b22becc79f95dc6040beb76e01dd9d04042`), difficulty reset to ~1e9 there, then normal 2,016-block retargeting (first retarget at 969,696 hit the +300% cap) |
| Current state | Height, difficulty, hashrate, block times, next retarget, and blocks per pool: [explorer.beta.ecash.ninja/mining](https://explorer.beta.ecash.ninja/mining) |
| Block reward | 3.125 ECX + fees; spendable after 100 confirmations (standard coinbase maturity) |
| Public pool | `stratum+tcp://stratum.beta.bip300.xyz:3334` ([dashboard](https://pool.beta.bip300.xyz)), run by LayerTwo Labs. PPLNS paid directly in the coinbase, 1% fee |
| Other pools | third-party pools listed at [pool.drivechain.info](https://pool.drivechain.info) |
| Solo mining | point a `getblocktemplate` miner at the **enforcer's** template server, never at the node directly ([below](#solo-mining)) |
| Other networks | **Alphanet** (fork 963,648, difficulty reset to 1) is retired on 2026-09-24; its pools stop answering then. **Mainnet** forks at ~973,728 (~2026-10-31) |

## Mining at the public pool

LayerTwo Labs runs a [simplepool](https://github.com/LayerTwo-Labs/simplepool) instance at [pool.beta.bip300.xyz](https://pool.beta.bip300.xyz). Point any stratum miner at it:

```
URL:       stratum+tcp://stratum.beta.bip300.xyz:3334   # pool.beta.bip300.xyz:3334 also answers
Username:  <your-bitcoin-address>.<rig_label>           # rig label optional
Password:  x                                            # ignored, any value
```

| Port | Starting difficulty | For |
|---|---|---|
| 3334 | 1 (vardiff) | individual miners |
| 3335 | 1,048,576 | Braiins-style rented or aggregated hashrate |
| 3336 | 2,000,000 | NiceHash-style rented hashrate |

- **The username is an L1 Bitcoin address**. The pool runs in `pplns-coinbase` mode: each block is divided among the shares in the PPLNS window and paid **in that block's own coinbase**, one output per miner, after a 1% operator fee. The pool never receives the reward and keeps no balance; a reorged block simply never paid.
- A coinbase has limited outputs. If your share of a block is under 546 sats, or the block runs out of output slots, you get no output in that one; you are then first in line for the next block. Small miners are paid less often, not less.
- Coinbase outputs mature after 100 confirmations (~16 hours at 10-minute blocks).
- Live status: https://pool.beta.bip300.xyz/api/status (mode, fee, hashrate, blocks).

## Other public pools

Third-party pools are listed at [pool.drivechain.info](https://pool.drivechain.info), one tab per network, with stratum URL, mode, fee, and what the username must be (a Bitcoin address for coinbase-paid pools, a Thunder address for Thunder-paid ones). The list is maintained in [`LayerTwo-Labs/mining-pools`](https://github.com/LayerTwo-Labs/mining-pools) (`networks/betanet/pools.json`; `networks.json` carries each stage's launch and sunset dates) and is what the explorer uses to attribute blocks. As of 2026-09-22 betanet lists bip300.xyz, eCPool.tech, ePool, avonpool, and Pow.re (private). To list a pool, open a PR against that file; mainnet listings are accepted ahead of launch.

## Solo mining

**The eCash node refuses plain `getblocktemplate` calls.** The `betanet` branch (like alphanet) adds a `bip300301` rule to the RPC: a `template` request that does not acknowledge the rule fails with `you should not be calling getblocktemplate from this daemon, instead call it from bip300301_enforcer`, and every template is returned with `!bip300301` in `rules`. Node templates contain no BIP300/301 coinbase data (sidechain and withdrawal ACKs, BMM commitments), so a block built from one would orphan sidechain activity. Only the enforcer adds those commitments. (`-deprecatedrpc=getblocktemplate` re-enables the plain call for testing.)

So the solo stack is: **eCash node → `bip300301_enforcer` (template server) → your GBT miner or stratum pool.** 

### 1. Run the node

A synced, **unpruned** betanet node with RPC and ZMQ enabled (the enforcer refuses pruned nodes; assumeutxo-bootstrapped nodes are fine). Get it from the `betanet` branch, the Docker image `ghcr.io/ecash-com/bitcoin:betanet`, or the prebuilt binaries at [releases.ecash.com](https://releases.ecash.com/) (`L1-ecash-bitcoin/betanet/`, verifiable with `gh attestation verify L1-ecash-bitcoin-<target>.zip --repo ecash-com/bitcoin`).

Betanet has its own network magic (`0xeca5b104`), ports P2P 8533 / RPC 8532, datadir `~/.ecash`, config `ecash.conf`, and built-in DNS seeds (`seed.beta.ecash.ninja`, `seed.beta.bip300.xyz`, `seed.beta.ecash.drivecha.in`, `seed.beta.ecash.zuexeuz.net`), so a plain `bitcoind` finds peers on its own. Full node setup in [01-node-setup.md](01-node-setup.md).

```ini
# betanet/ecash.conf
server=1
rpcuser=user
rpcpassword=pass
zmqpubsequence=tcp://127.0.0.1:29000
# no prune=; the enforcer needs full blocks
```

```sh
bitcoind -datadir=./betanet
# optional fast bootstrap: betanet pins no fork-point snapshot, only upstream
# height 935,000 (see 01-node-setup.md); loadtxoutset that and sync the rest
bitcoin-cli -datadir=./betanet getblockhash 967680
# 00000000000000030101ba5cfea54b22becc79f95dc6040beb76e01dd9d04042
```

`getblocktemplate` requires a connected, fully synced node. 

### 2. Choose a payout address

Any L1 address you control. The node's wallet is one way to get one:

```sh
bitcoin-cli -datadir=./betanet createwallet mine
bitcoin-cli -datadir=./betanet getnewaddress
```

### 3. Run the enforcer 

```sh
bip300301_enforcer \
  --network-preset=betanet \
  --node-rpc-addr=127.0.0.1:8532 \
  --node-rpc-user=user --node-rpc-pass=pass \
  --node-zmq-addr-sequence=tcp://127.0.0.1:29000 \
  --enable-wallet \
  --enable-mempool        # with the wallet enabled, serves getblocktemplate
```

The preset selects betanet's fork height, magic, `OP_NOP8` opcode, and BIP300 thresholds. It serves `getblocktemplate` on `127.0.0.1:8122` by default (`--serve-rpc-addr`), with no authentication. `--gbt-cache-lifetime-s` controls template caching. Without a wallet, use `--enable-block-template-server --enable-mempool --coinbase-recipient=<address>` instead of `--enable-wallet`. Sanity check:

```sh
curl -s --data '{"method":"getblocktemplate","params":[{"rules":["segwit"]}]}' http://127.0.0.1:8122
```

### 4. Point a miner at the enforcer

Any GBT-compatible SHA-256d miner, aimed at **8122** instead of the node's 8532, e.g. cpuminer:

```sh
minerd -a sha256d -o http://127.0.0.1:8122 --coinbase-addr=<any-address>
```

- **The reward goes to `--coinbase-recipient`**, not to the miner's own `--coinbase-addr`, because the miner uses the served coinbase as-is. A pool that wants to pay its miners replaces the reward output while keeping the commitment `OP_RETURN`s byte-for-byte.

Or run a stratum pool against the enforcer so ordinary ASICs can connect without touching GBT; see [running your own pool](#running-your-own-pool-simplepool) below.


## Running your own pool: simplepool

[simplepool](https://github.com/LayerTwo-Labs/simplepool) is a single-binary stratum server in C11. Miners connect over stratum with their payout address as the username; in the default `solo` mode each block's coinbase pays the finding miner directly, minus an operator fee, so the pool never custodies funds. Shares are recorded in SQLite; optional Redis pub/sub feeds a dashboard.

```
miners (stratum)
        |
   simplepool  ── getblocktemplate / submitblock ──>  bip300301_enforcer GBT server (8122)
                                                              |
                                                       eCash bitcoind (RPC 8532 + ZMQ)

```

simplepool has no network-specific configuration. It only speaks JSON-RPC to the template backend. On eCash that backend **must** be the enforcer.

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
coinbase_tag = /yourpool/

pool_mode = solo
db_path = ./data/shares.db
```


### Pool modes

simplepool ships five, selected by `pool_mode`:

| `pool_mode` | Coinbase pays | Username | Who carries the variance |
|---|---|---|---|
| `solo` (default) | the miner who found the block | Bitcoin address | nobody, you are paid what you find |
| `pps-classic` | the pool | Thunder address | the operator, from a reserve |
| `pplns-thunder` | the pool | Thunder address | the miners |
| `pplns-btc` | the pool | Bitcoin address | the miners |
| `pplns-coinbase` | the whole PPLNS window, directly | Bitcoin address | the miners |

- **`solo`**: coinbase pays the finding miner directly, minus `fee_bps` to `operator_address`. No off-chain accounting. Simplest choice.
- **`pps-classic`**: miners authorize with a bare base58 Thunder address and accrue `pps_credits` per share; the operator batches L1 funds into Thunder and a payout worker settles on the sidechain.
- **`pplns-thunder`** / **`pplns-btc`**: the coinbase pays the pool, and a matured block is split across the shares in the window (`pplns_window_diff_multiple` × network difficulty), so the pool never owes more than it has just been paid. Fees are included in the split. Paid over Thunder or in BTC respectively.
- **`pplns-coinbase`**: what the public betanet pool runs. Same PPLNS accounting, but the block's own coinbase pays every miner in the window, so the pool holds nothing. Miners below 546 sats or beyond the coinbase's output slots roll to the front of the next block's queue.
- An earlier `pps` mode that paid a BIP300 deposit output in the coinbase has been dropped; the enforcer does not credit coinbase-source deposits.

### Deployment

`scripts/deploy-to-server.sh` provisions a full host (systemd units for pool + dashboard, nginx, UFW). A Docker compose setup under `deploy/docker/` covers the stratum proxy, dashboard, and payout worker; it expects bitcoind, the enforcer, and (for Thunder rails) a Thunder node on the host. Integration test: `scripts/regtest/` brings up the whole stack against a local regtest node.
