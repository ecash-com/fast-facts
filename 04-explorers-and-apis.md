# Block Explorers & APIs

Public infrastructure currently runs against **betanet** under `*.beta.ecash.ninja` hostnames. The alphanet `*.alpha.*` services are retired on 2026-09-24. Expect new hostnames again at mainnet; check [ecash.com](https://ecash.com) and the [betanet hub](https://beta.ecash.ninja) for the current set.

## Block explorer

| Service | URL |
|---|---|
| Block explorer (mempool.space instance) | https://explorer.beta.ecash.ninja |
| Network info hub (L1 tip, sidechain status) | https://beta.ecash.ninja |
| Mining pool list | https://pool.drivechain.info ([08](08-mining.md)) |

## Esplora REST API

Blockstream-Esplora-compatible, at `https://esplora.beta.ecash.ninja`:

```sh
curl https://esplora.beta.ecash.ninja/blocks/tip/height
curl https://esplora.beta.ecash.ninja/block-height/967680      # fork block: 00000000000000030101ba5cfea54b22becc79f95dc6040beb76e01dd9d04042
curl https://esplora.beta.ecash.ninja/address/<address>
curl https://esplora.beta.ecash.ninja/tx/<txid>
curl -X POST -d <rawtx-hex> https://esplora.beta.ecash.ninja/tx
```

The explorer's own mempool.space API is also available under `https://explorer.beta.ecash.ninja/api/` (e.g. `/api/blocks/tip/height`, `/api/v1/difficulty-adjustment`, `/api/v1/mining/pools/1w`).

Endpoint reference: https://github.com/Blockstream/esplora/blob/master/API.md

## Electrum server

```
ssl://explorer.beta.ecash.ninja:50002
```

A Fulcrum instance (2.1.x). Standard Electrum protocol over TLS. Works with electrs-compatible clients and libraries (BDK's Electrum backend, Electrum wallet with `--server`). Useful for light-client balance tracking without a full node.

## Your own infrastructure

The standard Bitcoin indexing stack works unmodified against an eCash node:

- **mempool.space** self-hosted (what the official explorer runs, the eCash fork is at https://github.com/ecash-com/mempool)
- **electrs** / **Blockstream electrs** (Electrum + Esplora API)
- Core's REST interface (`rest=1`) and RPC (`getblock`, `gettransaction`, `scantxoutset`)
- ZMQ notifications (`zmqpubsequence`, `zmqpubhashblock`) for real-time deposit detection

Caveat: your indexer must ingest the post-fork chain from an eCash node. Pre-fork history is identical to Bitcoin, so an existing Bitcoin index could in principle be reused up to the fork height, but a dedicated index per chain is the simplest correct approach.

## Other endpoints (betanet)

| Service | Address |
|---|---|
| Public P2P nodes / DNS seeds | `seed.beta.ecash.ninja`, `seed.beta.bip300.xyz`, `seed.beta.ecash.drivecha.in`, `seed.beta.ecash.zuexeuz.net` (port 8533) |
| Mining pools (stratum) | third-party, listed in [08](08-mining.md#mining-at-the-public-pool) |
| UTXO snapshot (assumeutxo bootstrap) | none published for betanet; the branch pins upstream height 935,000, see [01](01-node-setup.md) |
| Node binaries | https://releases.ecash.com/ (`index.json` for the latest build per branch) |
