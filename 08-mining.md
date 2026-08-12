# Mining

## Fast facts

| | |
|---|---|
| Algorithm | **SHA-256d**, identical to Bitcoin. Any Bitcoin miner works unmodified: ASICs, home miners (Bitaxe, NerdAxe), cpuminer |
| Network | **drynet4** (live). Difficulty reset to 1 at fork block 961,632, then normal 2,016-block retargeting |
| Current difficulty | ~16,000 as of 2026-08-11 (Bitcoin: ~10¹⁴). Network hashrate is about 1 TH/s, so one small home ASIC finds blocks. Live value: [explorer](https://explorer.drynet4.drivechain.dev) |
| Block reward | ~3.125 ECX + fees; spendable after 100 confirmations (standard coinbase maturity) |
| Public pool | `stratum+tcp://pool.drynet4.drivechain.dev:3333` ([dashboard](https://pool.drynet4.drivechain.dev)) |
| Pool login | username = a **Thunder sidechain address** (optionally `<address>.<rig_label>`), password ignored |
| Solo mining | point any `getblocktemplate` SHA-256d miner at your own node ([below](#solo-mining)) |

## Mining at the public pool

The one public pool today is LayerTwo Labs' [simplepool](https://github.com/LayerTwo-Labs/simplepool) instance. Point any stratum miner at it:

```
URL:       stratum+tcp://pool.drynet4.drivechain.dev:3333
Username:  <your-thunder-address>.<rig_label>     # rig label optional
Password:  x                                      # ignored, any value
```

- **The username must be a Thunder address, not an L1/Bitcoin address.** The pool runs in PPS mode and pays out on the Thunder sidechain (drivechain #9). Each accepted share credits 1,000 sats × share difficulty, block or not; the operator takes a 1% fee, periodically deposits L1 rewards into Thunder via BIP300, and a payout worker settles miners' balances there. Get an address with `thunder-cli get-new-address` or from the Thunder wallet in BitWindow. An L1 address is rejected (`invalid thunder address`) and no shares accrue.
- Vardiff is on, so low-hashrate miners still accrue share credits. The [dashboard](https://pool.drynet4.drivechain.dev) shows pool stats, recent blocks, and per-worker pages.

## Solo mining

The recommended solo setup mines against the block template served by the [`bip300301_enforcer`](https://github.com/LayerTwo-Labs/bip300301_enforcer), not the node's own. Templates straight from the node contain ordinary transactions only; the enforcer's templates additionally carry the BIP300/301 coinbase data (sidechain proposal and withdrawal-bundle ACKs, BMM commitments), so blocks built from them earn BIP301 blind-merged-mining fees and participate in sidechain governance. This is the stack the official software (BitWindow, simplepool) is built around.

### 1. Run the node

A running, synced eCash node ([01](01-node-setup.md)) with the RPC server, credentials, and ZMQ enabled in `ecash.conf`:

```ini
server=1
rpcuser=user
rpcpassword=pass
zmqpubsequence=tcp://127.0.0.1:29000
```

Note: unlike drynet3, drynet4 dropped the patch that let `getblocktemplate` run with no peers or during IBD, so the node must be connected (e.g. `addnode=drynet4.drivechain.dev:8533`) and fully synced before templates are served. The fork block itself being minimum-difficulty remains a consensus rule.

### 2. Create a payout address

```sh
bitcoin-cli -datadir=./drynet4 createwallet mine
bitcoin-cli -datadir=./drynet4 getnewaddress
```

### 3. Run the enforcer

```sh
bip300301_enforcer \
  --node-rpc-addr=localhost:8532 \
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
minerd -a sha256d -o http://127.0.0.1:8122 --coinbase-addr=<your-address>
```

Or run a stratum pool against it so ordinary ASIC miners can connect without touching GBT; see [running your own pool](#running-your-own-pool-simplepool) below.

### Fallbacks

- **Node templates directly:** point the GBT miner at the node instead (`-o http://127.0.0.1:8532 -O user:pass`). Blocks are valid but contain no drivechain coinbase data, so no BMM fees and no ACK participation.
- **`generatetoaddress`:** still present as a hidden RPC (absent from `help` but works), no longer practical: at difficulty ~16,000 a block takes ~7×10¹³ hashes and `maxtries` caps at ~2.1 billion per call. It was only viable in the first hours after the fork at difficulty 1.

### Good to know

- **Reorg risk:** while difficulty is re-equilibrating, blocks arrive fast and erratically (currently ~1/minute) and reorgs are more likely than on Bitcoin. Do not treat freshly mined rewards as final.
- **Replay:** coinbase outputs are new post-fork coins and cannot be replayed. Later spends of them are eCash-only too; setting `nLockTime = 499999999` anyway costs nothing ([03](03-replay-protection-and-coin-splitting.md)).
- **Why mine now:** the team expects difficulty to find an equilibrium proportional to the USD value of the block reward. The low-difficulty window is when small miners matter, including for ACKing sidechain proposals ([05](05-sidechains-and-l2s.md)).
- The official per-network walkthrough lives in [09-drynets/DRYNET-4.md](09-drynets/DRYNET-4.md#mining).

## Running your own pool: simplepool

[simplepool](https://github.com/LayerTwo-Labs/simplepool) is a single-binary stratum server in C11. Miners connect over stratum with their payout address as the username; in the default `solo` mode each block's coinbase pays the finding miner directly, minus an operator fee, so the pool never custodies funds. Shares are recorded in SQLite; optional Redis pub/sub feeds a dashboard.

```
miners (stratum)
        |
   simplepool  ── getblocktemplate / submitblock ──>  bip300301_enforcer GBT server (8122)
                                                              |
                                                       eCash bitcoind (RPC 8532 + ZMQ)

   (fallback: simplepool can also talk to eCash bitcoind directly on 8532)
```

simplepool has no network-specific configuration; it only speaks JSON-RPC to whatever template backend you point it at. Connecting to the eCash network is entirely the node's job ([01](01-node-setup.md)).

### Build and run

```sh
git clone https://github.com/LayerTwo-Labs/simplepool
cd simplepool
# macOS: brew install sqlite curl        Debian/Ubuntu: apt install build-essential libsqlite3-dev libcurl4-openssl-dev
make
mkdir -p data && sqlite3 data/shares.db < schema.sql
cp proxy.conf.example proxy.conf
./build/simplepool proxy.conf
```

Minimal `proxy.conf`, pointing at the enforcer's template server (recommended; run it as in [solo mining](#solo-mining) above):

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

To use the node directly instead (plain templates, no drivechain coinbase data), set `bitcoind_url = http://127.0.0.1:8532` plus `bitcoind_user`/`bitcoind_pass` (cookie auth is not supported). Vardiff is on by default (`vardiff_target_spm = 12`).

### Pool modes

- **`solo`** (default): coinbase pays the finding miner directly, minus `fee_bps` to `operator_address`. No off-chain accounting. Simplest choice.
- **`pps-classic`**: what the public drynet4 pool runs. Miners authorize with a Thunder address and accrue `pps_credits` per share; the operator batches L1 funds into Thunder and a payout worker settles on the sidechain.
- **`pps`**: pays a BIP300 deposit output directly in the coinbase. Broken by design for now (the enforcer does not credit coinbase-source deposits); kept for shape validation only.

### Deployment

`scripts/deploy-to-server.sh` provisions a full host (systemd units for pool + dashboard, nginx, UFW). A Docker compose setup under `deploy/docker/` covers the stratum proxy, dashboard, and payout worker; it expects bitcoind, Thunder, the enforcer, and electrs on the host. Integration test: `tests/test_integration.sh` against a local regtest node.
