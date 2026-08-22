# Mining

> **This page now covers `alphanet`, which supersedes drynet4.** eCash is rolling out in three
> stages — Alpha, Beta, Mainnet — each one its own fork of Bitcoin mainnet at its own height.
> Drynet4 still runs, but it is no longer where new mining effort should go. The rest of this
> repo has not been updated for the staged rollout yet; treat this page as the current one for
> mining. Verified 2026-08-22.

## Fast facts

| | |
|---|---|
| Algorithm | **SHA-256d**, identical to Bitcoin. Any Bitcoin miner works unmodified: ASICs, home miners (Bitaxe, NerdAxe), cpuminer |
| Network | **alphanet**, stage 1 of 3. Forks Bitcoin mainnet at height **963,648**; branch [`alphanet`](https://github.com/ecash-com/bitcoin/tree/alphanet) |
| Rollout | Alpha **963,648** (23 Aug 2026) → Beta **967,680** (20 Sep 2026) → Mainnet **973,728** (~31 Oct 2026). Each stage is a separate fork, so miners re-point at each one ([below](#the-three-stage-rollout)) |
| Difficulty | Resets to **1** at each stage's fork block, then normal 2,016-block retargeting capped at 4× per period |
| Block reward | **3.125 ECX** + fees (height 963,648 is in the 4th halving epoch); spendable after 100 confirmations |
| Block templates | Must come from [`bip300301_enforcer`](https://github.com/LayerTwo-Labs/bip300301_enforcer). The node's own `getblocktemplate` **now refuses every other caller** ([below](#templates-must-come-from-the-enforcer)) |
| Public pools | Three on alphanet, all stratum port **3334**. Registry: [LayerTwo-Labs/mining-pools](https://github.com/LayerTwo-Labs/mining-pools) ([list](#public-pools)) |
| Pool login | Username = a **BTC address** on `solo` pools, a **Thunder address** on `pps-classic` pools. Password ignored |
| Solo mining | Point a `getblocktemplate` miner at your own enforcer ([below](#solo-mining)) |
| Pool software | [simplepool](https://github.com/LayerTwo-Labs/simplepool), one-line installer ([below](#running-your-own-pool-simplepool)) |

## The three-stage rollout

eCash is no longer a single fork event. Per [ecash.com](https://ecash.com):

| Stage | Fork height | Date | Notes |
|---|---|---|---|
| **Alpha** | 963,648 | 23 August 2026 | Live now. `alphanet` branch. This is what the public pools mine |
| **Beta** | 967,680 | 20 September 2026 | Branch not published yet |
| **Mainnet** | 973,728 | ~31 October 2026 (15:00 UTC target) | The production chain |

All three heights are exact multiples of 2,016 (478, 480 and 483 retarget periods), which is
what makes the difficulty reset land cleanly — the reset in `pow.cpp` only fires on a retarget
boundary:

```cpp
// eCash fork activation difficulty reset
if (pindexLast->nHeight + 1 == params.EcashHeight)
    bnNew = bnPowLimit;
```

**Two consequences worth internalising before you point hashrate anywhere:**

1. **Before its fork height, an alphanet node is following Bitcoin mainnet** — same blocks, same
   full mainnet difficulty. A pool pointed at it in that window is handing miners real Bitcoin
   work that a small ASIC will never solve. At 20:46 UTC on 2026-08-22 the public alphanet pools
   were at tip 963,632, sixteen blocks short of the fork. Only at 963,648 does the chain diverge
   and difficulty drop to 1.
2. **Each stage is a fresh chain from a fresh fork height.** Alpha coins are not Beta coins and
   neither are Mainnet coins. Expect to re-point miners, and expect pool operators to stand up new
   instances, at each stage. The pool registry has a `chain` field for exactly this reason.

## Public pools

From [`pools.json`](https://github.com/LayerTwo-Labs/mining-pools/blob/master/pools.json), the
registry behind pool attribution in the explorer (`pools.ecash.com` is the announced front-end;
it does not resolve yet). All three run simplepool on port **3334**:

| Pool | Mode | Fee | Coinbase tag | Stratum | Username must be |
|---|---|---|---|---|---|
| [bip300.xyz](https://pool.alpha.bip300.xyz) | `solo` | 1% | `/bip300xyz/` | `stratum+tcp://pool.alpha.bip300.xyz:3334` | your **BTC** address |
| [avonpool](https://pool.alpha.avonpool.xyz) | `pps-classic` | 1% | `/avonpool/` | `stratum+tcp://pool.alpha.avonpool.xyz:3334` | your **Thunder** address |
| [ecashpool.tech](https://ecashpool.tech) | `proportional` (operator's term; PPLNS) | 0% | `/ecashpool-alpha/` | `stratum+tcp://stratum.ecashpool.tech:3334` | your **BTC** address |

Each exposes a JSON status endpoint (`/api/status`, or `/api/pool/health` on ecashpool.tech) with
live tip height, hashrate, share counts and health checks — useful for monitoring, and for
confirming a pool is actually past the fork height before you connect.

```sh
curl -s https://pool.alpha.bip300.xyz/api/status | jq '.pool.mode, .node.tip_height'
```

### Which address goes in the username

This is the one thing that silently costs you money, and it differs by pool mode:

- **`solo`** — the coinbase of a found block pays the finder directly, so the username is a
  **Bitcoin-format address** (`bc1…` bech32 P2WPKH, or base58 `1…`/`3…`). Optionally
  `<address>.<rig_label>` to split rigs in the leaderboard. If your miner doesn't find the block,
  nobody on that pool earns for that height.
- **`pps-classic`** — the coinbase pays the pool; every accepted share accrues a credit that is
  settled to you **on the Thunder sidechain**. The username must be a **bare base58 Thunder
  address** (`thunder-cli get-new-address`, or the Thunder wallet in BitWindow). The deposit-format
  wrapper `s9_<base58>_<hex6>` is **rejected** — Thunder doesn't recognise it at the byte level.
  An L1 address is rejected too, and no shares accrue.

  Payouts run as a daily batch: everyone over the operator's `PAYOUT_MIN_SATS` goes out in one
  Thunder transaction every 24h. The credit rate is derived per-template as
  `(coinbasevalue / network_difficulty) × (1 − fee_bps/1e4)` unless the operator pins a fixed
  `pps_sats_per_diff`, which simplepool now warns against because it drifts as difficulty moves.

Vardiff is on by default on all three (target ~12 shares/minute), so low-hashrate miners still
accrue credit.

### Getting your pool listed, and why `coinbase_tag` matters

Open a PR against [LayerTwo-Labs/mining-pools](https://github.com/LayerTwo-Labs/mining-pools)
appending one object to the `pools` array in `pools.json`. Required fields: `name`, `operator`,
`chain`, `mode`, `fee_bps`, `coinbase_tag`, `stratum_url`, `operator_address`. Optional:
`dashboard_url`, `status_url`, `pool_btc_address`, `payout`, `software`, `contact`.

`coinbase_tag` is the field to get right. It is the string your pool stamps into the coinbase
scriptSig of every block it finds, and it is the only thing an explorer can use to attribute a
block to you. Left at simplepool's `/simplepool/` default, your blocks are credited to someone
else and **nothing anywhere reports an error**. Read it from the running pool, not from memory:

```sh
grep coinbase_tag /path/to/simplepool/proxy.conf
```

On simplepool the installer restores its saved answers from `/etc/simplepool/install.env` on every
run, so a hand-edited `proxy.conf` is silently reverted on the next upgrade. If the two disagree,
fix the installer answer as well.

## Solo mining

### Templates must come from the enforcer

New on alphanet, and the single biggest change from the drynet4 instructions: the node's
`getblocktemplate` now **refuses to serve anyone but the enforcer**.

```
getblocktemplate only serves templates to bip300301_enforcer: the template is missing the
BIP300/BIP301 commitments and is not safe to mine on as-is. Miners must fetch templates from
bip300301_enforcer.
```

A plain node template carries ordinary transactions only. The enforcer's templates additionally
carry the BIP300/301 coinbase data — sidechain proposal and withdrawal-bundle ACKs, BMM
commitments — so blocks built from them earn blind-merged-mining fees and participate in sidechain
governance. Mining a plain template orphans sidechain activity for that block.

The RPC now takes a `bip300301_enforcer: true` acknowledgement, which the enforcer sets and you
should not. The node-wide escape hatch for testing is `-deprecatedrpc=getblocktemplate`; there is
no good reason to use it on a chain with live sidechains.

### 1. Run the node

A synced alphanet node ([01](01-node-setup.md)) with RPC, credentials and ZMQ enabled in
`ecash.conf`:

```ini
server=1
rpcuser=user
rpcpassword=pass
zmqpubsequence=tcp://127.0.0.1:29000
```

Alphanet ships four working DNS seeds, so no `addnode` is needed:
`seed.alpha.ecash.ninja`, `seed.alpha.bip300.xyz`, `seed.alpha.ecash.drivecha.in`,
`seed.alpha.ecash.zuexeuz.net`.

Network identity differs from drynet4 — message-start bytes are **`0xeca5a104`** (drynet4 used
`0xeca5d404`); P2P/RPC ports stay **8533/8532**, datadir `~/.ecash`, config `ecash.conf`. Binaries
are published as `L1-ecash-bitcoin-alphanet-<platform>.zip` at
[releases.drivechain.info](https://releases.drivechain.info/).

As on drynet4, `getblocktemplate` requires a connected, synced node — the drynet3 patch that let it
run with no peers or during IBD is still gone.

### 2. Create a payout address

```sh
bitcoin-cli -datadir=./alphanet createwallet mine
bitcoin-cli -datadir=./alphanet getnewaddress
```

### 3. Run the enforcer

```sh
bip300301_enforcer \
  --node-rpc-addr=localhost:8532 \
  --node-rpc-user=user --node-rpc-pass=pass \
  --node-zmq-addr-sequence=tcp://127.0.0.1:29000 \
  --enable-mempool \
  --enable-block-template-server \
  --coinbase-recipient=<your-address>
```

`--enable-block-template-server` is now an explicit flag and requires `--enable-mempool`; enabling
the wallet is no longer what turns templates on. The server works without a wallet, in which case
`--coinbase-recipient` is required (with `--enable-wallet` and no recipient set, the reward goes to
a fresh wallet address). It listens on `127.0.0.1:8122` by default (`--serve-rpc-addr`) with no
authentication; `--gbt-cache-lifetime-s` controls template caching. Sanity check:

```sh
curl -s --data '{"method":"getblocktemplate","params":[{"rules":["segwit"]}]}' http://127.0.0.1:8122
```

### 4. Point a miner at the enforcer

Any GBT-compatible SHA-256d miner, aimed at **8122** rather than the node's 8532:

```sh
minerd -a sha256d -o http://127.0.0.1:8122 --coinbase-addr=<your-address>
```

Or run a stratum server against it so ordinary ASICs can connect without speaking GBT; see
[running your own pool](#running-your-own-pool-simplepool).

### `generatetoaddress`

Still present as a hidden RPC (absent from `help`, but it works). Viable only in the first minutes
after a fork block while difficulty is genuinely 1 — `maxtries` caps at ~2.1 billion hashes per
call, so it stops being practical as soon as difficulty starts climbing.

## Running your own pool: simplepool

[simplepool](https://github.com/LayerTwo-Labs/simplepool) is a single-binary stratum v1 server in
C11: it accepts miners on TCP `:3334`, builds templates via `getblocktemplate`, submits blocks via
`submitblock`, and records every accepted share in SQLite (WAL, the proxy is the only writer). A
read-only Node dashboard and a separate Thunder payout worker read from that file. Nothing in the
stratum path depends on either being up.

```
miners (stratum :3334)
        |
   simplepool  ── getblocktemplate / submitblock ──>  bip300301_enforcer GBT server (8122)
        |                                                      |
   SQLite shares.db                                     eCash bitcoind (RPC 8532 + ZMQ)
        |
   dashboard (read-only) + payout worker (pps-classic, Thunder)
```

simplepool has no network-specific configuration — it speaks JSON-RPC to whatever template backend
you point it at. Connecting to alphanet is entirely the node's job ([01](01-node-setup.md)).

### Install

On a fresh Ubuntu/Debian box:

```sh
curl -fsSL https://raw.githubusercontent.com/LayerTwo-Labs/simplepool/main/scripts/install.sh | sudo bash
```

It downloads the published build for the architecture, verifies it against the release
`SHA256SUMS`, then interviews you for pool mode, bitcoind RPC, operator address, dashboard domain,
nginx and TLS, and leaves a running pool behind nginx. `--from-source` builds instead. Answers are
saved to `/etc/simplepool/install.env` and re-running is how you change them.

```sh
simplepoolctl status      # services, ports, version, ledger totals
simplepoolctl doctor      # checks the things that actually break in production
simplepoolctl logs -f     # follow every service
simplepoolctl upgrade     # next release, then restart
```

To build by hand: deps are `sqlite3`, `libcurl`, `libhiredis`, pthreads and a C11 compiler
(`brew install sqlite curl hiredis` / `apt install build-essential libsqlite3-dev
libcurl4-openssl-dev libhiredis-dev`), then `make`; the binary lands at `build/simplepool`.

### Minimal `proxy.conf`

Pointing at the enforcer's template server, which is the recommended backend:

```ini
listen_addr = 0.0.0.0
listen_port = 3334

bitcoind_url = http://127.0.0.1:8122
# omit bitcoind_user and bitcoind_pass entirely — the enforcer's GBT endpoint
# takes no basic-auth, and simplepool then sends none
bitcoind_poll_interval_ms = 10000

operator_address = <your bc1... address>
fee_bps = 100                      # 1% operator fee, range 0..1000
coinbase_tag = /yourpool/          # CHANGE THIS — see above

pool_mode = solo
db_path = ./data/shares.db
```

`bitcoind_poll_interval_ms` is also the worst-case delay before a sidechain's BMM request can reach
a stratum job: a sidechain publishes its BMM transaction only after it sees the new tip, so a block
found within one poll of that cannot carry the commitment. The shipped default is 30000; lower it
to 5000–10000 on a drivechain pool, at the cost of one extra `getblocktemplate` call per interval.

Vardiff is on by default (`vardiff_target_spm = 12`). `fee_bps = 0` drops the fee output entirely,
as does a computed fee below the ~546 sat dust threshold. Optional `redis_url` mirrors shares,
rejects, blocks, tip changes and credits to Redis pub/sub for the dashboard; SQLite stays the
source of truth.

### Pool modes

- **`solo`** (default) — each block's coinbase pays the finding miner directly, minus `fee_bps` to
  `operator_address`. Each connection gets its own coinbase rendered against its own address; no
  off-chain accounting, no inter-miner sharing. The simplest thing to run.
- **`pps-classic`** — the coinbase pays a single pool-owned `pool_btc_address` as a normal output;
  each accepted share credits `pps_credits.accrued_sats`. The operator batches accumulated BTC into
  the pool's Thunder reserve from the admin dashboard, and the separate `payout/` worker issues
  Thunder transactions to drain those credits daily. The pool carries the variance.
- **`pps`** — **removed.** It put a BIP300 deposit output directly in the coinbase so the pool would
  never custody BTC; both regtest and forknet showed the enforcer does not credit coinbase-sourced
  deposits, so the block confirms but the sidechain Ctip never moves and the reward is stranded.
  The evidence and the design that replaced it are in
  [`CLASSIC_PAYOUTS.md`](https://github.com/LayerTwo-Labs/simplepool/blob/main/CLASSIC_PAYOUTS.md).

In both surviving modes the operator fee stays in BTC and is paid to `operator_address` out of the
same coinbase.

### Deployment

`scripts/install.sh` brings a box up from nothing; `scripts/deploy-to-server.sh` drives an
already-installed box from your workstation while iterating on unreleased code. A Docker compose
setup under `deploy/docker/` covers the stratum proxy, dashboard and payout worker — the drivechain
daemons (bitcoind, Thunder, the enforcer, electrs) are deliberately **not** containerised and are
expected on the host. Stratum is raw TCP and does not go through nginx: point your stratum hostname
straight at the box and open 3334. Integration, end-to-end and payout tests run against a local
regtest node (`tests/`), and per-mode verification checklists are in `VERIFY.md`.

## Good to know

- **The low-difficulty window is short but not instant.** Difficulty resets to 1 at the fork block
  and the first retarget is 2,016 blocks later (965,664 on alphanet). Because a retarget can raise
  difficulty at most 4×, it takes several retarget periods for it to climb to equilibrium — on
  drynet4 that meant roughly 16,000 within days. That window is when small miners matter, including
  for ACKing sidechain proposals ([05](05-sidechains-and-l2s.md)).
- **Reorg risk.** While difficulty is re-equilibrating, blocks arrive fast and erratically and
  reorgs are far more likely than on Bitcoin. Do not treat freshly mined rewards as final.
- **Replay.** Coinbase outputs are new post-fork coins and cannot be replayed. Later spends of them
  are eCash-only too; setting `nLockTime = 499999999` anyway costs nothing
  ([03](03-replay-protection-and-coin-splitting.md)).
- **`chain` reports `main`.** Every one of these networks is a mainnet fork, so `getblockchaininfo`
  returning `chain=main` is correct and not a misconfiguration. Verify which chain you are on by
  block hash at or after the fork height, never by chain name.
- **Drynet4** is superseded but still running, including its pool on the old port 3333
  ([09-drynets/DRYNET-4.md](09-drynets/DRYNET-4.md#mining)). Nothing mined there carries over.
