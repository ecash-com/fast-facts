# Mining

## Fast facts

| | |
|---|---|
| Algorithm | **SHA-256d**, identical to Bitcoin. Any Bitcoin miner works unmodified: ASICs, home miners (Bitaxe, NerdAxe), cpuminer |
| Network | **alphanet** (live since 2026-08-23). Forked Bitcoin at block **963,648** (hash `0000000000b360c17636b7a6c366e3effbe91a847eb5d61b7a7b29476439e924`), difficulty reset to 1 there, then normal 2,016-block retargeting |
| Current state | Height, difficulty, hashrate, block times, next retarget, and blocks per pool: [explorer.alpha.ecash.ninja/mining](https://explorer.alpha.ecash.ninja/mining) |
| Block reward | 3.125 ECX + fees; spendable after 100 confirmations (standard coinbase maturity) |
| Public pool | `stratum+tcp://pool.alpha.bip300.xyz:3334` ([dashboard](https://pool.alpha.bip300.xyz)), run by LayerTwo Labs. Solo mode, 1% fee |
| Other pools | third-party pools listed at [pool.drivechain.info](https://pool.drivechain.info) |
| Solo mining | point a `getblocktemplate` miner at the **enforcer's** template server, never at the node directly ([below](#solo-mining)) |
| Next networks | **betanet** forks at block **967,680** (~2026-09-19) and starts at difficulty ~10⁹, not 1. **Mainnet** forks at ~973,728 (~2026-10-31). See [what changes next](#what-changes-with-betanet-and-mainnet) |

## Mining at the public pool

LayerTwo Labs runs a [simplepool](https://github.com/LayerTwo-Labs/simplepool) instance at [pool.alpha.bip300.xyz](https://pool.alpha.bip300.xyz). Point any stratum miner at it:

```
URL:       stratum+tcp://pool.alpha.bip300.xyz:3334
Username:  <your-bitcoin-address>.<rig_label>     # rig label optional
Password:  x                                      # ignored, any value
```

- **The username is an L1 Bitcoin address**. The pool runs in solo mode: the block's coinbase pays whoever found it, minus a 1% fee, spendable after 100 confirmations. Nothing is held by the pool.
- **Betanet:** [pool.beta.bip300.xyz](https://pool.beta.bip300.xyz) is already up ahead of the fork (`stratum+tcp://pool.beta.bip300.xyz:3334`).

## Other public pools

Third-party pools are listed at [pool.drivechain.info](https://pool.drivechain.info), with stratum URL, mode, fee, and what the username must be (a Bitcoin address for coinbase-paid pools, a Thunder address for Thunder-paid ones). The list is maintained in [`LayerTwo-Labs/mining-pools`](https://github.com/LayerTwo-Labs/mining-pools) and is what the explorer uses to attribute blocks.

## Solo mining

**Alphanet's node refuses plain `getblocktemplate` calls.** The `alphanet` branch adds a `bip300301` rule to the RPC: a `template` request that does not acknowledge the rule fails with `you should not be calling getblocktemplate from this daemon, instead call it from bip300301_enforcer`, and every template is returned with `!bip300301` in `rules`. Node templates contain no BIP300/301 coinbase data (sidechain and withdrawal ACKs, BMM commitments), so a block built from one would orphan sidechain activity. Only the enforcer adds those commitments. 

So the solo stack is: **eCash node → `bip300301_enforcer` (template server) → your GBT miner or stratum pool.** 

### 1. Run the node

A synced, **unpruned** alphanet node with RPC and ZMQ enabled (the enforcer refuses pruned nodes; assumeutxo-bootstrapped nodes are fine). Get it from the `alphanet` branch, the Docker image `ghcr.io/ecash-com/bitcoin:alphanet`, or the prebuilt binaries at [releases.ecash.com](https://releases.ecash.com/) (`L1-ecash-bitcoin/alphanet/`, verifiable with `gh attestation verify L1-ecash-bitcoin-<target>.zip --repo ecash-com/bitcoin`; mirrored as `L1-ecash-bitcoin-alphanet-<target>.zip` on [releases.drivechain.info](https://releases.drivechain.info/)).

Alphanet has its own network magic (`0xeca5a104`), ports P2P 8533 / RPC 8532, datadir `~/.ecash`, config `ecash.conf`, and built-in DNS seeds (`seed.alpha.ecash.ninja`, `seed.alpha.bip300.xyz`, `seed.alpha.ecash.drivecha.in`, `seed.alpha.ecash.zuexeuz.net`), so a plain `bitcoind` finds peers on its own. Full node setup in [01-node-setup.md](01-node-setup.md).

```ini
# alphanet/ecash.conf
server=1
rpcuser=user
rpcpassword=pass
zmqpubsequence=tcp://127.0.0.1:29000
# no prune=; the enforcer needs full blocks
```

```sh
bitcoind -datadir=./alphanet
# optional fast bootstrap (~9.5 GB snapshot at the fork block, pinned in the node):
#   curl -O https://data.drivechain.dev/alphanet/utxo-963648.dat
#   bitcoin-cli -datadir=./alphanet loadtxoutset utxo-963648.dat
bitcoin-cli -datadir=./alphanet getblockhash 963648
# 0000000000b360c17636b7a6c366e3effbe91a847eb5d61b7a7b29476439e924
```

`getblocktemplate` requires a connected, fully synced node. 

### 2. Choose a payout address

Any L1 address you control. The node's wallet is one way to get one:

```sh
bitcoin-cli -datadir=./alphanet createwallet mine
bitcoin-cli -datadir=./alphanet getnewaddress
```

### 3. Run the enforcer 

```sh
bip300301_enforcer \
  --network-preset=alphanet \
  --node-rpc-addr=127.0.0.1:8532 \
  --node-rpc-user=user --node-rpc-pass=pass \
  --node-zmq-addr-sequence=tcp://127.0.0.1:29000 \
  --enable-wallet \
  --enable-mempool        # with the wallet enabled, serves getblocktemplate
```

It serves `getblocktemplate` on `127.0.0.1:8122` by default (`--serve-rpc-addr`), with no authentication. `--gbt-cache-lifetime-s` controls template caching. Sanity check:

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

simplepool has no network-specific configuration. It only speaks JSON-RPC to the template backend. On alphanet that backend **must** be the enforcer.

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

- **`solo`** (default): coinbase pays the finding miner directly, minus `fee_bps` to `operator_address`. No off-chain accounting. Simplest choice.
- **`pps-classic`**: what the public drynet4 pool runs. Miners authorize with a Thunder address and accrue `pps_credits` per share; the operator batches L1 funds into Thunder and a payout worker settles on the sidechain.
- **`pps`**: pays a BIP300 deposit output directly in the coinbase. Broken by design for now (the enforcer does not credit coinbase-source deposits); kept for shape validation only.

### Deployment

`scripts/deploy-to-server.sh` provisions a full host (systemd units for pool + dashboard, nginx, UFW). A Docker compose setup under `deploy/docker/` covers the stratum proxy, dashboard, and payout worker; it expects bitcoind, the enforcer, and (for Thunder rails) a Thunder node on the host. Integration test: `scripts/regtest/` brings up the whole stack against a local regtest node.
