# Block Explorers & APIs

Public infrastructure currently runs against **drynet3**; expect the same services under new hostnames at launch (check https://drivechain.info/dev.txt). Equivalent endpoints for drynet2 (still live) are listed in [09-drynets/DRYNET-2.md](09-drynets/DRYNET-2.md).

## Block explorer

| Service | URL |
|---|---|
| Block explorer (mempool.space instance) | https://explorer.drynet3.drivechain.dev |
| Network info hub | https://drynet3.drivechain.dev/info |

## Esplora REST API

Blockstream-Esplora-compatible, at `https://esplora.drynet3.drivechain.dev`:

```sh
curl https://esplora.drynet3.drivechain.dev/blocks/tip/height
curl https://esplora.drynet3.drivechain.dev/address/<address>
curl https://esplora.drynet3.drivechain.dev/tx/<txid>
curl -X POST -d <rawtx-hex> https://esplora.drynet3.drivechain.dev/tx
```

Endpoint reference: https://github.com/Blockstream/esplora/blob/master/API.md

## Electrum server

```
ssl://drynet3.drivechain.dev:50012
```

Standard Electrum protocol over TLS; works with electrs-compatible clients and libraries (BDK's Electrum backend, Electrum wallet with `--server`). Useful for light-client balance tracking without a full node.

## Your own infrastructure

The standard Bitcoin indexing stack works unmodified against an eCash node:

- **mempool.space** self-hosted (what the official explorer runs)
- **electrs** / **Blockstream electrs** (Electrum + Esplora API)
- Core's REST interface (`rest=1`) and RPC (`getblock`, `gettransaction`, `scantxoutset`)
- ZMQ notifications (`zmqpubsequence`, `zmqpubhashblock`) for real-time deposit detection

Caveat: your indexer must ingest the post-fork chain from an eCash node. Pre-fork history is identical to Bitcoin, so an existing Bitcoin index could in principle be reused up to the fork height, but a dedicated index per chain is the simplest correct approach.

## Other endpoints (drynet3)

| Service | Address |
|---|---|
| Public P2P node | `drynet3.drivechain.dev:8337` |
| UTXO snapshot (assumeutxo bootstrap) | https://data.drivechain.dev/drynet3/utxo-957600.dat (~9 GB) |
| Fast-withdrawal server | `fw1.drynet3.drivechain.dev` |
