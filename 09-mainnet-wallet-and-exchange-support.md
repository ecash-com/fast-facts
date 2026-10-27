# Mainnet Wallet & Exchange Support

Last updated: **2026-10-27**. Verified against the mainnet node release `ecash-v1.1.0-core31.1` (`master`, commit `5bd8690fab`), the live mainnet endpoints, and the network registry at https://drivechain.dev/config. Anything not yet public is marked **TBA** with where it will be announced.

Looking for places to trade or your exchange's distribution policy? Use [Markets & Exchange Support](10-markets-and-exchange-support.md). This page is the technical integration playbook.

> **Breaking change from betanet:** the replay-protection locktime on mainnet is **`499999998`**, not the `499999999` used on alphanet and betanet. With a non-final input sequence, a mainnet send stamped with `499999999` is rejected as non-final by both eCash and Bitcoin nodes. All-final input sequences instead disable Bitcoin's locktime check and provide no replay protection. Update the constant and verify the sequences before cutover. Other differences from betanet: [testnets/](testnets/README.md#what-changes-between-betanet-and-mainnet).

> **Required node upgrade:** `ecash-v1.1.0-core31.1` (2026-10-26) introduced a new post-fork difficulty rule. Nodes on `ecash-v1.0.0-core31.1` split off the chain at block **974,016**. Upgrade every node, indexer backend and explorer before then.

## 1. Mainnet parameters

| Parameter | Mainnet value |
|---|---|
| Fork height | **973,728** (= 2016 × 483, a retarget boundary). Blocks 0 to **973,727** are shared with Bitcoin; 973,728 is the first eCash-mined block |
| Fork date | **est. 2026-10-29 ~15:00 UTC** as of 2026-10-27 (287 blocks to go; the announced target was 2026-10-31). It is a height, not a date. Live ETA: https://mempool.space/api/v1/difficulty-adjustment |
| Fork block hash | **TBA.** Block 973,728 is mined by eCash miners at the reset difficulty, so its hash differs from Bitcoin's block 973,728 and exists only once mined. Verify it as in §6 |
| Difficulty | reset to ~2.6e9 at the fork (`nBits = 0x1901a6e1`), then a retarget **every 288 blocks**: `new = old × (6T + measured) / 7T`, at most ~12% up or ~30% down per window. First retarget at **974,016** ([01](01-node-setup.md#difficulty-after-the-fork)) |
| Replay locktime | **`499999998`** (`LOCKTIME_THRESHOLD - 2`) |
| Network magic | `0xeca5dcb7` |
| Ports (P2P / RPC) | **8533 / 8532** (Tor 8534) |
| DNS seeds | `seed.ecash.ninja` and `seed.ecash.zuexeuz.net` (live); `seed.bip300.xyz`, `seed.ecash.drivecha.in`, `seed.ecash.eu.com` (in the node, not resolving yet). No fixed seeds; fallback `addnode=seed.ecash.ninja:8533` |
| Datadir / config | `~/.ecash`, macOS `~/Library/Application Support/eCash`, Windows `%LOCALAPPDATA%\eCash`; `ecash.conf` |
| User agent | `/eCash Satoshi:31.1.0/` |
| Source | `master` branch of https://github.com/ecash-com/bitcoin, tag `ecash-v1.1.0-core31.1` |
| Binaries | https://releases.ecash.com/L1-ecash-bitcoin/mainnet/latest/release.json (macOS arm64/x86_64, Linux x86_64, Windows x86_64) |
| Docker | `ghcr.io/ecash-com/bitcoin:latest` (= `:sha-5bd8690`; pin the sha tag) |
| assumeutxo snapshot | https://releases.ecash.com/assumeutxo/mainnet/utxo-965000.dat (9.47 GB, height 965,000, pinned in the node) |
| Ticker / unit | **ECX**, 8 decimals (1 ECX = 100,000,000 sats). RPC still prints `BTC`; label at the application layer |
| Repurposed (Patoshi) txids | 232 (mainnet's own list in `src/repo_txns.h`; not the betanet set) |
| `OP_DRIVECHAIN` | `0xb7` (`OP_NOP8`) |
| Registry id | `ecash` in https://drivechain.dev/config (family `ecash`, display name "eCash") |

Everything else (RPC, REST, ZMQ, wallet, PSBT, descriptors) is stock Bitcoin Core v31.1.

## 2. Transaction rules (every ECX send)

1. **`nLockTime = 499999998`.** eCash treats it as final. With rule 2, Bitcoin treats it as non-final until that far-future block height is passed. Verify both locktime and sequence fields in the final signed transaction; do not assume a wallet preserves them. Bitcoin Core's `sendtoaddress`/`sendmany` cannot set the marker, so use `send`, `sendall`, `walletcreatefundedpsbt` or `createrawtransaction` with an explicit locktime ([02](02-wallets-keys-addresses.md#withdrawal-construction-one-ecash-specific-rule)).
2. **At least one input with `nSequence < 0xffffffff`.** Otherwise the locktime is ignored and the transaction is replayable. `0xfffffffd` (RBF) and `0xfffffffe` are both fine.
3. **Do not combine the marker with `OP_CHECKLOCKTIMEVERIFY` scripts.** A transaction carrying `499999998` fails any CLTV check (`src/script/interpreter.cpp`). HTLC-style scripts need a different design on ECX.
4. **Fees and dust follow Bitcoin Core defaults:** min relay 100 sat/kvB (0.1 sat/vB); dust 294 sats P2WPKH, 330 P2WSH/P2TR, 546 P2PKH. Zero-fee transactions do not relay.
5. **Keys and addresses are identical to Bitcoin:** `1…`/`3…`/`bc1q…`/`bc1p…`, same WIF/xpub/xprv, coin type `0'`. Any BTC address is an ECX address. Label the chain in every receive UI and key balances by `(chain, address)`.

Verify before the fork. The mainnet binary's `-regtest` enforces the same marker. Use isolated test keys and make the same test UTXO available on both regtest nodes; a missing-input rejection is not proof of replay protection. Do not broadcast a rehearsal to Bitcoin mainnet:

```sh
bitcoind -regtest -datadir=./rt -daemon                # eCash mainnet binary
bitcoin-cli -regtest -datadir=./rt testmempoolaccept '["<hex, nLockTime=499999998, nSequence=0xfffffffd>"]'   # allowed: true
# the same hex on stock Bitcoin Core regtest: allowed: false, reject-reason "non-final"
```

## 3. Endpoints

All live on 2026-10-27. Until block 973,728 they serve the shared chain (their tip equals Bitcoin's). The authoritative list is the `ecash` entry in https://drivechain.dev/config.

| Service | Mainnet |
|---|---|
| Esplora REST | `https://esplora.ecash.ninja` |
| Electrum (Fulcrum 2.1.4) | `ssl://explorer.ecash.ninja:50002` |
| Explorer (mempool.space fork) | `https://explorer.ecash.ninja` (`/tx/{txid}`, `/address/{address}`, `/block/{hash}`) |
| Explorer API | `https://explorer.ecash.ninja/api/` |
| Blockbook | `https://blockbook.ecash.ninja` (still in initial sync on 2026-10-27) |
| P2P | `seed.ecash.ninja:8533` |
| Binaries, snapshot | `https://releases.ecash.com` |
| Status dashboard | `https://dashboard.ecash.com/?network=mainnet` |
| Stable explorer entry point | `https://explorer.ecash.com` **still redirects to betanet** (root path only; not an API base or URL template). Use `explorer.ecash.ninja` |
| Hosted enforcer | `https://seed.ecash.eu.com/enforcer` per the config, but **not resolving yet** (2026-10-27) |
| Mining pool, fast withdrawal, faucet | **TBA** (empty in the config). There will be no faucet |

Make every URL configurable, or read them from drivechain.dev/config, and plan a config push at cutover. Announcement channels: https://ecash.com, https://t.me/ecashcom_official, https://t.me/DcInsiders, and this repo.

### Esplora: what is supported

The public Esplora is a Blockstream-compatible instance. The endpoints below are the supported integration surface:

| Need | Endpoint |
|---|---|
| Balance (confirmed vs mempool) | `GET /address/{addr}` → `chain_stats`, `mempool_stats` |
| UTXOs | `GET /address/{addr}/utxo` |
| History (confirmed + unconfirmed) | `GET /address/{addr}/txs` (unconfirmed first), `/txs/mempool`, `/txs/chain/{last_txid}` for paging |
| Transaction | `GET /tx/{txid}`, `GET /tx/{txid}/status`, `GET /tx/{txid}/hex` |
| Fees | `GET /fee-estimates` (sat/vB by target block). Returns `{}` as of 2026-10-27 |
| Tip | `GET /blocks/tip/height`, `GET /blocks/tip/hash`, `GET /block-height/{h}` |
| Broadcast | `POST /tx` with raw hex as `text/plain`. Returns the txid, or HTTP 400 with the node's `sendrawtransaction` error |

Full reference: https://github.com/Blockstream/esplora/blob/master/API.md

Terms: **no auth, no API key, no published rate limit, no SLA, no geo restriction.** It is best effort for the community. There is no webhook, push or websocket API. Confirmation status is what `/tx/{txid}/status` returns; count confirmations as `tip_height - block_height + 1`.

**Production policy:** a wallet or exchange moving customer funds must not depend solely on the public endpoint. Run your own node (§4) and treat public Esplora as a fallback, or run your own Esplora/Electrum against your node and point clients at it.

## 4. Your own infrastructure (recommended for both wallets and exchanges)

```sh
# 1. Install and verify (targets: aarch64-apple-darwin, x86_64-apple-darwin, x86_64-unknown-linux-gnu, x86_64-w64-msvc)
B=https://releases.ecash.com/L1-ecash-bitcoin/mainnet/latest
curl -O $B/release.json -O $B/L1-ecash-bitcoin-x86_64-unknown-linux-gnu.zip
grep -o '"tag": "[^"]*"' release.json                         # "ecash-v1.1.0-core31.1" (or later)
sha256sum L1-ecash-bitcoin-x86_64-unknown-linux-gnu.zip      # must match "sha256" in release.json
gh attestation verify L1-ecash-bitcoin-x86_64-unknown-linux-gnu.zip --repo ecash-com/bitcoin   # optional, needs GitHub CLI
unzip L1-ecash-bitcoin-x86_64-unknown-linux-gnu.zip           # -> L1-ecash-bitcoin-31.1.0-5bd8690fab60-x86_64-unknown-linux-gnu/{bitcoind,bitcoin-cli,bitcoin-util,qt/bitcoin-qt}

# or Docker: ghcr.io/ecash-com/bitcoin:sha-5bd8690

# 2. ./ecash-mainnet/ecash.conf (mainnet is the default network; no -chain flag)
server=1
txindex=1                              # for /tx lookups by txid; omit if pruning
rpcuser=<user>
rpcpassword=<password>
zmqpubsequence=tcp://127.0.0.1:29000   # deposit detection
zmqpubhashblock=tcp://127.0.0.1:28332
rest=1

# 3. Run with a mainnet-only datadir (never reuse a betanet datadir)
bitcoind -datadir=./ecash-mainnet
```

Bootstrap with the published snapshot (usable in minutes; the node re-validates history in the background):

```sh
curl -O https://releases.ecash.com/assumeutxo/mainnet/utxo-965000.dat
curl -O https://releases.ecash.com/assumeutxo/mainnet/SHA256SUMS
sha256sum -c SHA256SUMS            # 89351f7106051c0685880f9c64026e993335308de63ae6980e94ba3fafdc7dda
# wait until `getblockchaininfo` shows headers past 965,000, then:
bitcoin-cli -datadir=./ecash-mainnet -rpcclienttimeout=0 loadtxoutset utxo-965000.dat
```

The snapshot is Bitcoin's UTXO set at height 965,000 (block `00000000000000000001595977e6000ce56129f5c9b4073e31ccc30b90b97da9`). Its `hash_serialized` `4a8d7943…77ec2` is consensus-pinned in `src/kernel/chainparams.cpp`, so a tampered file is rejected. The node then syncs the remaining pre-fork blocks and the post-fork chain. A full sync from genesis is ~860 GB.

Deposit detection, pick one:

| Approach | Fit | Notes |
|---|---|---|
| Own node + ZMQ `sequence` + descriptor watch-only wallet | exchanges, custody | authoritative; `listsinceblock` for reconciliation |
| Own node + electrs/Fulcrum/mempool-electrs/Blockbook | wallets at scale | same API as the public services, under your control |
| Public Electrum `blockchain.scripthash.subscribe` | light wallets | push notifications per script; the server is Fulcrum 2.1.4 |
| Poll public Esplora | prototypes, fallback | works, no guarantees |

Broadcast: the canonical path is `sendrawtransaction` on your own node. `POST /tx` on Esplora is the same call, proxied. Client-side vs backend broadcast is your choice; a backend proxy lets you add fallbacks (own node, then public Esplora) and log rejections. There is no published list of fallback nodes; the DNS seeds are for P2P only.

## 5. Step-by-step: non-custodial wallet

1. **Config:** make Esplora/Electrum/explorer URLs remote-configurable per network, or read the `ecash` entry from drivechain.dev/config. Add a new mainnet network id; never remap betanet wallets onto it (a wallet's network is fixed at creation).
2. **Send:** set `nLockTime = 499999998`, at least one input `nSequence < 0xffffffff`, and Bitcoin Core dust/fee rules. Use `/fee-estimates`, but expect it to be empty or unreliable in the first hours after the fork; fall back to a floor of 1 sat/vB.
3. **Receive:** label the chain ("ECX") on every address display. An address identical to the BTC one is expected and correct.
4. **Balances:** key by `(chain, address)`. Confirmed = `chain_stats`; pending = `mempool_stats`. Show unconfirmed history from `/address/{addr}/txs`.
5. **Confirmations:** post-fork block times are erratic, and difficulty can climb only ~12% per 288-block window (first retarget at 974,016). Require deeper confirmations than on Bitcoin (tens, not six) for the first weeks; there is no official number.
6. **Pre-fork coins:** any UTXO unspent as of block 973,727 starts on both chains. Spending it on Bitcoin without first splitting may also move the ECX if the transaction is replayed. Offer a "split" action that sweeps selected pre-fork UTXOs to a fresh address with the marker (§6 steps 4 and 5). Verify separation per UTXO, including later receipts; one split is not a permanent wallet-wide status. Reference implementations: the official mobile wallet (https://github.com/ecash-com/ecash-wallet-mobile, BDK-based; see `docs/coin-splitting.md` and `docs/real-ecash-fork-transition.md`), which flags a coin for splitting only while Bitcoin's explorer shows it unspent, and BitWindow's post-fork split.
7. **Treat it as real money:** the same send confirmations and warnings you use for BTC. Keep it distinct in the UI from betanet's practice coins.
8. **Price:** CoinGecko id **`ecash-com`** (symbol ECX) is correct, but it is a preview listing with no price or tickers today. Do **not** use `ecash`; that is XEC, an unrelated project. Do not use the Solana wECX pre-trading quote as an ECX price feed without saying so. No official oracle exists; show no fiat value until a market exists.
9. **Cutover day:** wait until the chain passes 973,728, verify the fork hash (§6), then enable mainnet as the default (the mobile wallet does exactly this).

## 6. Step-by-step: exchange

1. **Node and index:** own node per §4 on **v1.1.0 or later**, dedicated datadir, `txindex=1`, watch-only descriptor wallet or your own indexer fed by ZMQ. Verify the chain by hash, not by `chain=main`:
   ```sh
   bitcoin-cli -datadir=./ecash-mainnet getblockhash 973728   # eCash-mined block; must match explorer.ecash.ninja and differ from Bitcoin's
   ```
2. **Snapshot:** record every customer BTC balance and every UTXO you control as of block **973,727**, the last block shared with Bitcoin. Decide and announce your ECX crediting policy before then. ecash.com tells users to ask whether you will credit ECX, when ECX withdrawals open, and whether BTC withdrawals pause around the fork.
3. **Freeze** BTC and ECX withdrawals shortly before 973,728. Key the freeze on height, not clock time; the ETA drifts with Bitcoin's block rate (about two days ahead of the announced 2026-10-31 as of 2026-10-27).
4. **Split ECX first:** sweep every pre-fork UTXO into fresh addresses (new ECX-dedicated seed) with `nLockTime = 499999998` and at least one input `nSequence < 0xffffffff`. Verify both fields after signing. These confirm only on eCash under current chain conditions.
5. **Wait** for deep confirmation on eCash. Difficulty is re-equilibrating, and a reorg past the split re-exposes you.
6. **Check both chains, then resume BTC** withdrawals for verified split UTXOs: their ECX side must be spent and deeply confirmed, while the BTC side remains unspent. Omitted or newly received UTXOs need their own check, and a reorg can undo a split.
7. **Resume ECX** deposits and withdrawals. Set the marker on every ECX withdrawal permanently. Credit ECX deposits only from your eCash node, never inferred from a BTC transaction.
8. **Confirmations:** require dozens for ECX deposits until hashrate stabilises; monitor reorgs.
9. **Patoshi coins:** 232 whitelisted transactions spend Satoshi-era coins without signatures. Coins descending from them are valid ECX; expect user questions.
10. **Look-alikes:** ECX is not XEC (e.cash), and native ECX is not the Solana tokens (wECX, wbECX, the pump.fun "eCash" memecoin). Make sure listing pages, tickers and deposit networks say so ([07](07-faq.md#ecx-xec-wecx-and-solana-tokens)).
11. **Monitoring:** alert on `getblockchaininfo` tip age, on a mismatch between your node and the public explorer tip, and on the node version (anything below v1.1.0 forks off at 974,016).

## 7. Test funds and dry run

- **Faucet:** none, on any eCash network.
- **Mainnet dry run:** there is no pre-fork mainnet window; mainnet does not exist before block 973,728. Re-run your send path on regtest with the mainnet binary (`-regtest` uses the mainnet marker). The mainnet endpoints (§3) already work for read-path testing against the shared chain.
- **Betanet:** still live for end-to-end drills, with marker `499999999` and different parameters: [testnets/betanet.md](testnets/betanet.md).

## 8. Out of scope / not available

- **Swaps and bridges:** not required for a wallet or exchange. CoinShift atomic swaps are a separate sidechain product, not active on mainnet at the fork. Keep "convert" features off unless you integrate them deliberately.
- **Sidechain deposits:** no sidechain is active at the fork; enable deposits only for a sidechain once it is activated on mainnet ([05](05-sidechains-and-l2s.md)).
- **Push/webhook deposit API, authenticated Esplora, SLA:** none. Run your own stack (§4).
- **Dedicated integration channel:** none. Use dev@layertwolabs.com and DcInsiders (below).

Contacts and escalation: [06-contacts-and-resources.md](06-contacts-and-resources.md). Integration questions: dev@layertwolabs.com; real-time and cutover coordination: https://t.me/DcInsiders.
