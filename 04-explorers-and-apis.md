# Block Explorers & APIs

Public infrastructure currently runs against **alphanet**. Expect the same services under `*.beta.ecash.ninja` hostnames when betanet forks, and under new names again at mainnet. Check [ecash.com](https://ecash.com) for the current set.

## Block explorer

| Service | URL |
|---|---|
| Block explorer (mempool.space instance) | https://explorer.alpha.ecash.ninja |
| Network info hub | https://alpha.ecash.ninja |
| Mining pool list | https://pool.drivechain.info ([08](08-mining.md)) |

## Esplora REST API

Blockstream-Esplora-compatible, at `https://esplora.alpha.ecash.ninja`:

```sh
curl https://esplora.alpha.ecash.ninja/blocks/tip/height
curl https://esplora.alpha.ecash.ninja/address/<address>
curl https://esplora.alpha.ecash.ninja/tx/<txid>
curl -X POST -d <rawtx-hex> https://esplora.alpha.ecash.ninja/tx
```

The explorer's own mempool.space API is also available under `https://explorer.alpha.ecash.ninja/api/` (e.g. `/api/blocks/tip/height`, `/api/v1/difficulty-adjustment`, `/api/v1/mining/pools/1w`).

Endpoint reference: https://github.com/Blockstream/esplora/blob/master/API.md

## Electrum server

```
ssl://explorer.alpha.ecash.ninja:50002
```

A Fulcrum instance. Standard Electrum protocol over TLS. Works with electrs-compatible clients and libraries (BDK's Electrum backend, Electrum wallet with `--server`). Useful for light-client balance tracking without a full node.

## Your own infrastructure

The standard Bitcoin indexing stack works unmodified against an eCash node:

- **mempool.space** self-hosted (what the official explorer runs, the eCash fork is at https://github.com/ecash-com/mempool)
- **electrs** / **Blockstream electrs** (Electrum + Esplora API)
- Core's REST interface (`rest=1`) and RPC (`getblock`, `gettransaction`, `scantxoutset`)
- ZMQ notifications (`zmqpubsequence`, `zmqpubhashblock`) for real-time deposit detection

Caveat: your indexer must ingest the post-fork chain from an eCash node. Pre-fork history is identical to Bitcoin, so an existing Bitcoin index could in principle be reused up to the fork height, but a dedicated index per chain is the simplest correct approach.

## Other endpoints (alphanet)

| Service | Address |
|---|---|
| Public P2P nodes / DNS seeds | `seed.alpha.ecash.ninja`, `seed.alpha.bip300.xyz`, `seed.alpha.ecash.drivecha.in`, `seed.alpha.ecash.zuexeuz.net` (port 8533) |
| Mining pools (stratum) | third-party, listed in [08](08-mining.md#mining-at-a-public-pool) |
| UTXO snapshot (assumeutxo bootstrap) | https://data.drivechain.dev/alphanet/utxo-963648.dat (~9.5 GB, [01](01-node-setup.md)) |
| Node binaries | https://releases.ecash.com/ (`index.json` for the latest build per branch) |
