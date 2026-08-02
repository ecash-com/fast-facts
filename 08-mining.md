# Mining

eCash is SHA-256d proof-of-work, same as Bitcoin. At the fork block, difficulty resets to 1 and normal retargeting resumes, so for roughly the first 2,016 blocks the chain is CPU-mineable by anyone; after that, difficulty climbs toward whatever the ECX block reward is worth and ordinary SHA-256 ASIC economics take over. Everything below works on drynet3 today (`-datadir=./drynet3`) and will work identically on the production network at launch.

## How to mine

The recommended setup is to mine against the block template served by the [`bip300301_enforcer`](https://github.com/LayerTwo-Labs/bip300301_enforcer), not the node's own. Templates straight from the node contain ordinary transactions only; the enforcer's templates additionally carry the BIP300/301 coinbase data (sidechain proposal and withdrawal-bundle ACKs, BMM commitments), so blocks built from them earn BIP301 blind-merged-mining fees and participate in sidechain governance. This is the stack the official software (BitWindow, simplepool) is built around.

### 1. Run the node

A running, synced eCash node ([01](01-node-setup.md)) with the RPC server, credentials, and ZMQ enabled in `drivechain-ecash.conf`:

```ini
server=1
rpcuser=user
rpcpassword=pass
zmqpubsequence=tcp://127.0.0.1:29000
```

Two node patches matter here: `getblocktemplate` works with no peers and during IBD, and the fork block itself must be minimum-difficulty by consensus.

### 2. Create a payout address

```sh
bitcoin-cli -datadir=./drynet3 createwallet mine
bitcoin-cli -datadir=./drynet3 getnewaddress
```

### 3. Run the enforcer

```sh
bip300301_enforcer \
  --node-rpc-addr=localhost:8332 \
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

Any GBT-compatible SHA-256d miner, aimed at **8122** instead of the node's 8332, e.g. cpuminer:

```sh
minerd -a sha256d -o http://127.0.0.1:8122 --coinbase-addr=<your-address>
```

Or run a stratum pool against it so ordinary ASIC/CPU miners can connect without touching GBT; see the simplepool section below (`bitcoind_url = http://127.0.0.1:8122`, no auth keys).

### Fallbacks

- **Smoke test without a real miner:** `generatetoaddress` is available on this fork (a hidden RPC; it does not show in `help` but works):
  ```sh
  bitcoin-cli -datadir=./drynet3 generatetoaddress 1 <your-address> 2000000000
  ```
  At difficulty 1 a block takes ~4.3 billion hash attempts on average and `maxtries` caps at ~2.1 billion per call, so an empty `[]` result is normal; loop until it prints a block hash. Single-threaded, so a proof-of-life only.
- **Node templates directly:** point the GBT miner at the node instead (`-o http://127.0.0.1:8332 -O user:pass`). Blocks are valid but contain no drivechain coinbase data, so no BMM fees and no ACK participation.
- **Mine into someone else's pool:** `stratum+tcp://<pool-host>:3334`, username = your ECX payout address (`<address>` or `<address>.<rig_label>`), any password.

### Good to know

- **Coinbase maturity:** block rewards are spendable after 100 confirmations, the standard Bitcoin rule. With fast post-fork blocks that is quicker in wall-clock time than on Bitcoin, but also more reorg-prone; see the next point.
- **Reorg risk:** during difficulty re-equilibration, blocks arrive erratically and reorgs are more likely than on Bitcoin. Do not treat freshly mined rewards as final.
- **Replay:** coinbase outputs are new post-fork coins and cannot be replayed. Later spends of them are eCash-only too; setting `nLockTime = 499999999` anyway costs nothing ([03](03-replay-protection-and-coin-splitting.md)).
- **Why mine at launch:** the team expects difficulty to find an equilibrium proportional to the USD value of the block reward. The first retarget periods are the window where small miners matter, including for ACKing sidechain proposals ([05](05-sidechains-and-l2s.md)).
- The official per-network walkthrough lives in [09-drynets/DRYNET-3.md](09-drynets/DRYNET-3.md#mining).

## Running a pool: simplepool

[simplepool](https://github.com/LayerTwo-Labs/simplepool) is a single-binary stratum server in C11. Miners connect on TCP **3334** with their payout address as the username; each block's coinbase pays the finding miner directly, minus an operator fee, so the pool never custodies funds in solo mode. Shares are recorded in SQLite; optional Redis pub/sub feeds a dashboard.

```
miners (stratum, port 3334)
        |
   simplepool  ── getblocktemplate / submitblock ──>  bip300301_enforcer GBT server (8122)
                                                              |
                                                       eCash bitcoind (RPC + ZMQ)

   (fallback: simplepool can also talk to eCash bitcoind directly on 8332)
```

simplepool has no network-specific configuration; it only speaks JSON-RPC to whatever template backend you point it at. Connecting to the eCash network is entirely the node's job (datadir, pinned peers, fork-hash check, all per [01](01-node-setup.md)).

### Build

```sh
git clone https://github.com/LayerTwo-Labs/simplepool
cd simplepool
# macOS: brew install sqlite curl        Debian/Ubuntu: apt install build-essential libsqlite3-dev libcurl4-openssl-dev
make
mkdir -p data && sqlite3 data/shares.db < schema.sql
cp proxy.conf.example proxy.conf
```

### Configure (`proxy.conf`)

Pointing at the enforcer's template server (recommended; run it as in "How to mine" above):

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

Fallback, pointing at the node directly (plain templates, no drivechain coinbase data):

```ini
bitcoind_url = http://127.0.0.1:8332
bitcoind_user = <rpcuser>          # required for stock bitcoind;
bitcoind_pass = <rpcpassword>      # cookie auth is NOT supported
```

Then run:

```sh
./build/simplepool proxy.conf
```

Vardiff is on by default (`vardiff_target_spm = 12`).

### Pool modes

| Mode | Coinbase pays | Notes |
|---|---|---|
| `solo` (default) | The finding miner's address, minus `fee_bps` to `operator_address` | No off-chain accounting; if the fee would be below dust (~546 sats) it is dropped and the miner gets the full reward |
| `pps` | A BIP300 deposit output to a Thunder (sidechain 9) reserve address | **Broken by design for now:** the enforcer does not credit coinbase-source deposits, so the output is well-formed but effectively unspendable. Kept for shape-validation only |
| `pps-classic` | A pool L1 address (`pool_btc_address`) | Miners authorize with a Thunder address and accrue `pps_credits` at `pps_sats_per_diff`; the operator batches funds into Thunder and a separate payout service settles on the sidechain |

For a straightforward ECX pool, use `solo` (per-block direct payout) or `pps-classic` (conventional PPS with sidechain settlement).

### Deployment

`scripts/deploy-to-server.sh` provisions a full host (systemd units for pool + dashboard, nginx, UFW opening 80/443/3334). A Docker compose setup exists under `deploy/docker/` for the stratum proxy, dashboard, and payout worker; it expects bitcoind, Thunder, the enforcer, and electrs running on the host. Integration test: `tests/test_integration.sh` against a local regtest node.

### What simplepool does not cover (and where it is covered)

- **Connecting the node to eCash/drynet3:** datadir, `connect=drynet3.drivechain.dev:8337`, fork-hash verification, assumeutxo bootstrap. See [01](01-node-setup.md).
- **RPC auth:** set `rpcuser`/`rpcpassword` in `drivechain-ecash.conf`; simplepool cannot use the `.cookie` file.
- **Difficulty churn at launch:** template staleness matters during the first retarget periods; keep `bitcoind_poll_interval_ms` low and expect frequent `mining.notify` job changes.
