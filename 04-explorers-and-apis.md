# Block Explorers & APIs

Mainnet public infrastructure runs under `*.ecash.ninja`. The authoritative, machine-readable list is the `ecash` entry in **https://drivechain.dev/config**, which the official wallets read at runtime. Everything below was checked live on 2026-10-27.

**Before the fork these services follow the shared chain**, so their tip equals Bitcoin's (973,441 on 2026-10-27). From block 973,728 they follow eCash only. Test-network endpoints (`*.beta.ecash.ninja`) are in [testnets/betanet.md](testnets/betanet.md).

## Endpoints

| Service | URL | Notes |
|---|---|---|
| Block explorer (mempool.space fork) | https://explorer.ecash.ninja | tx `/tx/{txid}`, address `/address/{address}`, block `/block/{hash}` |
| Explorer API (mempool.space API) | https://explorer.ecash.ninja/api/ | e.g. `/api/blocks/tip/height`, `/api/v1/difficulty-adjustment`, `/api/v1/mining/pools/1w` |
| Esplora REST | https://esplora.ecash.ninja | Blockstream-compatible ([below](#esplora-rest-api)) |
| Electrum (Fulcrum 2.1.4) | `ssl://explorer.ecash.ninja:50002` | TLS only (no plaintext 50001) |
| Blockbook | https://blockbook.ecash.ninja | Trezor Blockbook `0.6.0-ecx`, coin `ECX`. **Still in initial sync** on 2026-10-27 (indexed to 898,549), so don't depend on it yet |
| P2P | `seed.ecash.ninja:8533` | see [01](01-node-setup.md#peers-and-seed-nodes) |
| UTXO snapshot | https://releases.ecash.com/assumeutxo/mainnet/utxo-965000.dat | SHA-256 `89351f71…7dda`, [01](01-node-setup.md#bootstrapping-assumeutxo) |
| Network dashboard | https://dashboard.ecash.com/?network=mainnet | |
| `explorer.ecash.com` | stable entry point | **still 302-redirects to the betanet explorer** as of 2026-10-27. It only redirects the root path, so it is not an API base or URL template. Use `explorer.ecash.ninja` directly |

Not live for mainnet yet: a hosted enforcer (the config lists `https://seed.ecash.eu.com/enforcer`, but the hostname does not resolve as of 2026-10-27), a public mining pool and fast-withdrawal services (`services.mining_pool` and `fast_withdrawal` are empty for `ecash`). There is no faucet and there will not be one.

## Esplora REST API

```sh
curl https://esplora.ecash.ninja/blocks/tip/height
curl https://esplora.ecash.ninja/block-height/973728      # the fork block, once mined
curl https://esplora.ecash.ninja/address/<address>
curl https://esplora.ecash.ninja/tx/<txid>
curl -X POST -H 'Content-Type: text/plain' -d <rawtx-hex> https://esplora.ecash.ninja/tx
```

| Need | Endpoint |
|---|---|
| Balance (confirmed vs mempool) | `GET /address/{addr}` → `chain_stats`, `mempool_stats` |
| UTXOs | `GET /address/{addr}/utxo` |
| History | `GET /address/{addr}/txs` (unconfirmed first), `/txs/mempool`, `/txs/chain/{last_txid}` for paging |
| Transaction | `GET /tx/{txid}`, `/tx/{txid}/status`, `/tx/{txid}/hex` |
| Fees | `GET /fee-estimates` (sat/vB by target). **Returns `{}` as of 2026-10-27**; expect it to be empty or unreliable in the first hours after the fork. Fall back to a floor of 1 sat/vB |
| Tip | `GET /blocks/tip/height`, `/blocks/tip/hash`, `/block-height/{h}` |
| Broadcast | `POST /tx`, raw hex as `text/plain`. Returns the txid, or HTTP 400 with the node's `sendrawtransaction` error |

Endpoint reference: https://github.com/Blockstream/esplora/blob/master/API.md

Terms: **no auth, no API key, no published rate limit, no SLA**. The service is best effort for the community. There is no webhook, push or websocket API on Esplora; for push notifications use Electrum `blockchain.scripthash.subscribe`. Count confirmations as `tip_height - block_height + 1`.

## Electrum server

```
ssl://explorer.ecash.ninja:50002
```

Fulcrum 2.1.4, Electrum protocol up to 1.6. `server.features` reports Bitcoin's genesis hash (`000000000019d6…e26f`), as every eCash stage does, so the genesis hash cannot tell eCash apart from Bitcoin. Check the tip or the hash at height 973,728 instead. Works with BDK's Electrum backend, the Electrum wallet (`--server explorer.ecash.ninja:50002:s --oneserver`), and other electrs-compatible clients.

## Your own infrastructure

The standard Bitcoin indexing stack works unmodified against an eCash node:

- **mempool.space** self-hosted (what the official explorer runs; the eCash fork is https://github.com/ecash-com/mempool)
- **electrs** / **Blockstream electrs** / **mempool-electrs** (Electrum + Esplora API), **Fulcrum**
- **Blockbook** (the public instance runs an `-ecx` build)
- Core's REST interface (`rest=1`) and RPC (`getblock`, `gettransaction`, `scantxoutset`)
- ZMQ notifications (`zmqpubsequence`, `zmqpubhashblock`) for real-time deposit detection

Caveat: your indexer must ingest the post-fork chain from an eCash node. Pre-fork history is identical to Bitcoin, so an existing Bitcoin index could in principle be reused up to block 973,727, but a dedicated index per chain is the simplest correct approach.

**Production policy:** a wallet or exchange moving customer funds must not depend solely on the public endpoints. Run your own node and indexer, and treat the public services as a fallback. Make every URL configurable.
