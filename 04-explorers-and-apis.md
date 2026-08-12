# Block Explorers & APIs

Public infrastructure currently runs against **drynet4**; expect the same services under new hostnames at launch (check https://drivechain.info/dev.txt). The drynet3 equivalents (still live) are listed in [09-drynets/DRYNET-3.md](09-drynets/DRYNET-3.md).

## Block explorer

| Service | URL |
|---|---|
| Block explorer (mempool.space instance) | https://explorer.drynet4.drivechain.dev |
| Network info hub | https://drynet4.drivechain.dev/info |
| Mining pool dashboard | https://pool.drynet4.drivechain.dev |

## Esplora REST API

Blockstream-Esplora-compatible, at `https://esplora.drynet4.drivechain.dev`:

```sh
curl https://esplora.drynet4.drivechain.dev/blocks/tip/height
curl https://esplora.drynet4.drivechain.dev/address/<address>
curl https://esplora.drynet4.drivechain.dev/tx/<txid>
curl -X POST -d <rawtx-hex> https://esplora.drynet4.drivechain.dev/tx
```

Endpoint reference: https://github.com/Blockstream/esplora/blob/master/API.md

## Electrum server

```
ssl://drynet4.drivechain.dev:50002
```

Standard Electrum protocol over TLS; works with electrs-compatible clients and libraries (BDK's Electrum backend, Electrum wallet with `--server`). Useful for light-client balance tracking without a full node.

## Your own infrastructure

The standard Bitcoin indexing stack works unmodified against an eCash node:

- **mempool.space** self-hosted (what the official explorer runs)
- **electrs** / **Blockstream electrs** (Electrum + Esplora API)
- Core's REST interface (`rest=1`) and RPC (`getblock`, `gettransaction`, `scantxoutset`)
- ZMQ notifications (`zmqpubsequence`, `zmqpubhashblock`) for real-time deposit detection

Caveat: your indexer must ingest the post-fork chain from an eCash node. Pre-fork history is identical to Bitcoin, so an existing Bitcoin index could in principle be reused up to the fork height, but a dedicated index per chain is the simplest correct approach.

## Other endpoints (drynet4)

| Service | Address |
|---|---|
| Public P2P node | `drynet4.drivechain.dev:8533` |
| Mining pool (stratum) | `stratum+tcp://pool.drynet4.drivechain.dev:3333` ([08](08-mining.md)) |
| UTXO snapshot (assumeutxo bootstrap) | https://data.drivechain.dev/drynet4/utxo-961632.dat (~9.5 GB, [01](01-node-setup.md)) |
| Fast-withdrawal server | `fw1.drynet4.drivechain.dev` |
