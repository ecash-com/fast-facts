# Mining Pool Setup

At launch, difficulty resets to 1 and eCash is open to CPU and small-scale SHA-256d mining, so pools are relevant from day one. Solo mining with a single node is covered in [01](01-node-setup.md#mining) and, in more detail, in the official drynet docs ([09-drynets/DRYNET-3.md](09-drynets/DRYNET-3.md#mining)); this doc covers running a stratum pool with [simplepool](https://github.com/LayerTwo-Labs/simplepool), and getting block templates either from the eCash node directly or from the BIP300/301 enforcer.

## Architecture

```
miners (stratum, port 3334)
        |
   simplepool  ── getblocktemplate / submitblock ──>  eCash bitcoind (RPC 8332)
                                        or        ──>  bip300301_enforcer GBT server (8122)
                                                              |
                                                       eCash bitcoind (RPC + ZMQ)
```

simplepool itself has no network-specific configuration. It only speaks JSON-RPC to whatever template backend you point it at; connecting to the eCash network is entirely the node's job (datadir, pinned peers, fork-hash check, all per [01](01-node-setup.md)).

## Block template source: node vs enforcer

**Option A, eCash bitcoind directly.** The node's `getblocktemplate` is patched to work with no peers and during IBD, so it can serve templates on a young or quiet network. Templates contain ordinary transactions only.

**Option B, bip300301_enforcer (recommended for drivechain-aware blocks).** The enforcer wraps the node and serves its own `getblocktemplate` with BIP300/301 coinbase data included (sidechain proposal ACKs, withdrawal-bundle ACKs, BMM commitments). Blocks built from these templates are what earn the BIP301 merged-mining fees and keep the pool participating in sidechain governance.

```sh
bip300301_enforcer \
  --node-rpc-addr=localhost:8332 \
  --node-rpc-user=<user> --node-rpc-pass=<pass> \
  --node-zmq-addr-sequence=tcp://127.0.0.1:29000 \
  --enable-wallet \
  --enable-mempool        # maintains a mempool; with the wallet enabled, serves getblocktemplate
```

- GBT is served on `127.0.0.1:8122` by default (`--serve-rpc-addr`), with **no authentication**.
- `--gbt-cache-lifetime-s <n>` controls template caching.
- The node needs `server=1`, RPC credentials, and `zmqpubsequence` (see the config in [01](01-node-setup.md)).

## simplepool

A single-binary stratum server in C11. Miners connect on TCP **3334** with their payout address as the stratum username (`<address>` or `<address>.<rig_label>`, password ignored). Each block's coinbase pays the finding miner directly, minus an operator fee, so the pool never custodies funds in solo mode. Shares are recorded in SQLite; optional Redis pub/sub feeds a dashboard.

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

Pointing at the node directly:

```ini
listen_addr = 0.0.0.0
listen_port = 3334
bitcoind_url = http://127.0.0.1:8332
bitcoind_user = <rpcuser>          # required for stock bitcoind;
bitcoind_pass = <rpcpassword>      # cookie auth is NOT supported
bitcoind_poll_interval_ms = 30000

operator_address = <your bc1... address>
fee_bps = 100                      # 1% operator fee (0..1000)
coinbase_tag = /yourpool/

pool_mode = solo
db_path = ./data/shares.db
```

Pointing at the enforcer instead: change the backend lines to

```ini
bitcoind_url = http://127.0.0.1:8122
# omit bitcoind_user and bitcoind_pass entirely; the enforcer's GBT
# endpoint takes no basic-auth, and simplepool then sends none
```

Then run:

```sh
./build/simplepool proxy.conf
```

Miners connect with `stratum+tcp://<host>:3334`, username = their ECX payout address. Vardiff is on by default (`vardiff_target_spm = 12`).

### Pool modes

| Mode | Coinbase pays | Notes |
|---|---|---|
| `solo` (default) | The finding miner's address, minus `fee_bps` to `operator_address` | No off-chain accounting; if the fee would be below dust (~546 sats) it is dropped and the miner gets the full reward |
| `pps` | A BIP300 deposit output to a Thunder (sidechain 9) reserve address | **Broken by design for now:** the enforcer does not credit coinbase-source deposits, so the output is well-formed but effectively unspendable. Kept for shape-validation only |
| `pps-classic` | A pool L1 address (`pool_btc_address`) | Miners authorize with a Thunder address and accrue `pps_credits` at `pps_sats_per_diff`; the operator batches funds into Thunder and a separate payout service settles on the sidechain |

For a straightforward ECX pool, use `solo` (per-block direct payout) or `pps-classic` (conventional PPS with sidechain settlement).

### Deployment

`scripts/deploy-to-server.sh` provisions a full host (systemd units for pool + dashboard, nginx, UFW opening 80/443/3334). A Docker compose setup exists under `deploy/docker/` for the stratum proxy, dashboard, and payout worker; it expects bitcoind, Thunder, the enforcer, and electrs running on the host. Integration test: `tests/test_integration.sh` against a local regtest node.

## What simplepool does not cover (and where it is covered)

The simplepool README assumes a working node and never mentions eCash network parameters. The missing pieces:

- **Connecting the node to eCash/drynet3:** datadir, `connect=drynet3.drivechain.dev:8337`, fork-hash verification, assumeutxo bootstrap. See [01](01-node-setup.md).
- **RPC auth:** set `rpcuser`/`rpcpassword` in `drivechain-ecash.conf`; simplepool cannot use the `.cookie` file.
- **Replay safety for pool payouts:** coinbase outputs are new post-fork coins, so block rewards themselves cannot be replayed. Any subsequent pool-wallet transactions should still set `nLockTime = 499999999` per [03](03-replay-protection-and-coin-splitting.md).
- **Difficulty churn at launch:** during the first retarget periods, template staleness matters; keep `bitcoind_poll_interval_ms` low and expect frequent `mining.notify` job changes.
