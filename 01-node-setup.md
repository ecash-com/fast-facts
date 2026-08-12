# Full Node Setup

The eCash L1 is Bitcoin Core v31.1 plus a ~12-commit patch stack; operationally it behaves exactly like Bitcoin Core.

> The production chain forks Bitcoin at block ~963,648 (~August 22, 2026). Until then, integrate against **drynet4**, the live dry-run network that forked Bitcoin mainnet at block 961,632 with the launch mechanics (difficulty reset, replay protection, coin repurposing) plus, new in this run, eCash's own network magic, ports, and datadir. The launch network parameters will be announced at https://drivechain.info/dev.txt. The official per-network docs (drynet1-4, including what changed between runs) are mirrored in [09-drynets](09-drynets/README.md).

## Where to get the software

| Source | Location |
|---|---|
| Source code | https://github.com/ecash-com/bitcoin, branch **`drynet4`** (launch branch TBA) |
| Binaries | https://releases.drivechain.info/ under `L1-ecash-bitcoin-drynet4-<platform>.zip` (macOS arm64/x86_64, Linux x86_64, Windows x86_64) |
| Integrity | SHA-256 hashes at https://releases.drivechain.info/hashes.json (no PGP signatures published yet, see [06](06-contacts-and-resources.md)) |
| Docker | `ghcr.io/ecash-com/bitcoin:drynet4` |
| GUI / activation client | BitWindow, https://layertwolabs.com/download |

Building from source works exactly like upstream Bitcoin Core v31 (CMake); see the repo's `README.md` and `INSTALL.md`.

## Differences from stock Bitcoin Core

The complete patch stack on top of v31.1:

1. Fork/activation height (`EcashHeight`): 961,632 on drynet4 (= 477 × 2016, a retarget boundary). Difficulty resets to 1 at this height (consensus rule: the fork block must have minimum difficulty).
2. Replay protection via magic `nLockTime = 499999999`. See [03](03-replay-protection-and-coin-splitting.md).
3. `setRepurposeTx`: hard-coded list of 220 txids (`src/repo_txns.h`) whose input-script checks are skipped (the Patoshi coin reassignment; expanded from drynet3's 122).
4. `OP_DRIVECHAIN` (repurposes `OP_NOP5`, 0xb4) added and made standard.
5. OP_RETURN limits removed.
6. **Own network magic and ports** (new in drynet4): message-start bytes `0xeca5d404` on mainnet and P2P/RPC ports 8533/8532, so an eCash node cannot handshake with Bitcoin Core peers and both can run side by side on default ports.
7. Default datadir `~/.ecash` (macOS: `~/Library/Application Support/ecash`, Windows: `AppData\Local\ecash`), config file `ecash.conf`.
8. DNS seeds replaced with `drynet4.drivechain.dev`.
9. A BTC→ECX P2P bridge mode: a special bridge node can peer with Bitcoin peers and relay data to eCash peers with translated network magic (how pre-fork history reaches the isolated network).
10. `CLIENT_NAME` identifies as eCash in the user agent.

Note a drynet3 patch that was **dropped**: `getblocktemplate` again requires a connected, synced node (stock Bitcoin Core behavior), so no more mining with no peers or during IBD.

Everything else (RPC, wallet, ZMQ, REST, P2P) is stock v31.1.

## Network identity

Drynet4 fixes the biggest footgun of earlier drynets: it has **its own network magic (`0xeca5d404`) and its own default ports (P2P 8533, RPC 8532)**, so it cannot peer with Bitcoin Core nodes or earlier drynets, and it won't collide with a Bitcoin node's default ports or datadir on the same machine.

Two cautions remain:

- `getblockchaininfo` still reports `chain: main` (the fork identifies as mainnet). Monitoring must verify the chain by block hash at or after the fork height, not by chain name:

```sh
bitcoin-cli -datadir=./drynet4 getblockhash 961632
# must return: 00000000001e6f522e6b954cba44bf8c36d59c52beb5cf61158f44e39d0a76ae
```

- A dedicated `-datadir` per network is still good practice (the default `~/.ecash` would be reused by any future drynet or the production network).

## Quick start (drynet4)

```sh
mkdir drynet4
bitcoind -datadir=./drynet4 -addnode=drynet4.drivechain.dev:8533
```

Production-style config (`drynet4/ecash.conf`):

```ini
server=1
txindex=1                              # if you index by txid; omit if pruning
# prune=2000                           # pruned mode works
connect=drynet4.drivechain.dev:8533    # exclusive peering to a known eCash node
listen=0
rpcuser=<user>
rpcpassword=<password>
zmqpubsequence=tcp://127.0.0.1:29000   # real-time deposit detection
rest=1
```

`connect=` (exclusive) pins your peering; with drynet4's own network magic this is less critical than on earlier drynets, but it keeps the peer set known. `drynet4.drivechain.dev` is also baked in as the DNS seed, so a plain `bitcoind -datadir=./drynet4` finds the network on its own.

## Bootstrapping: assumeutxo vs full sync

Full sync from genesis is ~850 GB (all of Bitcoin's history up to the fork, then the post-fork chain). The fast path is the official assumeutxo snapshot at the fork point (published 2026-08-12):

```sh
curl -O https://data.drivechain.dev/drynet4/utxo-961632.dat     # ~9.5 GB
bitcoind -datadir=./drynet4 -addnode=drynet4.drivechain.dev:8533
bitcoin-cli -datadir=./drynet4 loadtxoutset utxo-961632.dat
```

- Snapshot parameters are consensus-pinned in the node (`hash_serialized` `bc468e130a3dcf9f09583d6b8955cb21fb6ee2629f1e43b6113d6d282728ab7a` at height 961,632); a tampered snapshot is rejected.
- The pin landed on the `drynet4` branch on 2026-08-12; older builds reject the snapshot as unknown, so use a node built from that commit or later.
- The node is usable at the tip within minutes; a background chainstate re-validates history afterward.
- Pruned mode (`prune=2000`) works with this flow if you don't need `txindex`.

## Ports

| Service | Port |
|---|---|
| P2P | **8533** (Tor: 8534) |
| RPC | **8532** |
| ZMQ (convention) | 28332 / 29000 |
| BIP300/301 enforcer gRPC (optional) | 50051 |

Testnet3/testnet4/signet/regtest use 18533/48533/38533/18644 (P2P) and 18532/48532/38532/18643 (RPC).

## Mining

Difficulty reset to 1 at the fork block and is re-equilibrating (~16,000 as of 2026-08-11, blocks about every minute). Mining goes through the public pool or an external getblocktemplate miner; [08-mining.md](08-mining.md) covers the pool, solo, and run-your-own-pool setups. On drynet4, `getblocktemplate` requires a connected, fully synced node.

## Operational notes

- `getblockchaininfo` reports `chain: main`, and RPC amounts display as `BTC` (`CURRENCY_UNIT` unchanged). Handle the ECX ticker at your application layer.
- Halving schedule, 21M cap, and block interval are inherited from Bitcoin; the ledger and supply schedule continue from the fork point.
- Expect fast, erratic blocks and elevated reorg risk while difficulty re-equilibrates.
