# Drynet 4

**Status: live (current network).** Fourth eCash/Drivechain dry run
| | |
|---|---|
| Fork height | **961,632** (= 477 × 2016, a retarget boundary) |
| Fork block hash | `00000000001e6f522e6b954cba44bf8c36d59c52beb5cf61158f44e39d0a76ae` |
| Base version | Bitcoin Core v31.1 |
| Code | [`ecash-com/bitcoin`, branch `drynet4`](https://github.com/ecash-com/bitcoin/tree/drynet4) |
| Currency | eCash (ECX) |
| Chain type | `main` (mainnet fork) |
| Network magic | `0xeca5d404` (own; cannot peer with Bitcoin Core or earlier drynets) |
| Default ports | P2P **8533**, RPC **8532** (Tor: P2P + 1) |
| Datadir / config | `~/.ecash` (macOS `~/Library/Application Support/ecash`), `ecash.conf` |
| Current height | see the [explorer](https://explorer.drynet4.drivechain.dev) or [/info](https://drynet4.drivechain.dev/info) |

## Endpoints & ports

| Service | Address |
|---|---|
| Public P2P node | `drynet4.drivechain.dev:8533` (also the built-in DNS seed) |
| Hub / info | https://drynet4.drivechain.dev ([/info](https://drynet4.drivechain.dev/info)) |
| Block explorer | https://explorer.drynet4.drivechain.dev |
| Esplora REST API | https://esplora.drynet4.drivechain.dev |
| Electrum | `ssl://drynet4.drivechain.dev:50002` |
| Mining pool (stratum) | `stratum+tcp://pool.drynet4.drivechain.dev:3333`, [dashboard](https://pool.drynet4.drivechain.dev) |
| Fast withdrawal server | `fw1.drynet4.drivechain.dev` |

## Running a node

Build the `drynet4` branch (see the branch README for dependencies), use the
Docker image `ghcr.io/ecash-com/bitcoin:drynet4`, or download a binary from
[releases.drivechain.info](https://releases.drivechain.info/)
(`L1-ecash-bitcoin-drynet4-<target>.zip`). Then:

```sh
bitcoind -datadir=./drynet4 -addnode=drynet4.drivechain.dev:8533
```

Drynet4's own network magic and ports rule out handshaking with Bitcoin Core
or earlier drynets, and the default datadir (`~/.ecash`) does not collide with
`~/.bitcoin`, so the explicit `-datadir` is convention rather than a safety
requirement. `drynet4.drivechain.dev` is the built-in DNS seed, so the node
finds the network even without `-addnode`.

**Fast bootstrap**: an assumeutxo snapshot for the fork block is published at
`https://data.drivechain.dev/drynet4/utxo-961632.dat` (~9.5 GB; pinned in the
node at `hash_serialized`
`bc468e130a3dcf9f09583d6b8955cb21fb6ee2629f1e43b6113d6d282728ab7a`). Load it
with `bitcoin-cli -datadir=./drynet4 loadtxoutset utxo-961632.dat` to validate
at the tip within minutes; the pin landed on the branch 2026-08-12, so it
needs a build from that commit or later. A full historical sync is ~850 GB.

## Mining

Difficulty restarted at 1 from the fork block and retargets normally
(~16,000 a few days in, blocks roughly every minute; check the explorer).
Standard SHA-256d, so any Bitcoin miner hardware works.

- **Public pool (easiest):** `stratum+tcp://pool.drynet4.drivechain.dev:3333`,
  username = a Thunder sidechain address (optionally `.<rig_label>`), any
  password. It is a simplepool PPS instance paying out on Thunder; details and
  caveats in [08-mining.md](../08-mining.md).
- **Solo:** point a `getblocktemplate` miner at your own node (via the
  BIP300/301 enforcer for drivechain-aware templates). Full walkthrough in
  [08-mining.md](../08-mining.md).
- Note: drynet3's patch letting `getblocktemplate` run with no peers / during
  IBD was dropped; the node must be connected and synced. `generatetoaddress`
  still exists as a hidden RPC but is impractical at current difficulty.

## Changes vs drynet3

1. **Own network identity**:
   - Message-start bytes `0xeca5d404` (mainnet; each chain type has its own),
     so eCash nodes cannot connect to Bitcoin Core peers at all.
   - Own default ports: P2P 8533 / RPC 8532 on mainnet (Bitcoin +200), with
     matching testnet/signet/regtest ports; Tor = P2P + 1.
   - Default datadir `~/.ecash` and config `ecash.conf` (was
     `~/.drivechain-ecash` / `drivechain-ecash.conf`).
   - DNS seed replaced with `drynet4.drivechain.dev` (was `seed.bip300.xyz`).
   - `CLIENT_NAME` identifies as eCash.
2. **New fork point**: 961,632 (drynet2/3 forked at 957,600). Consensus
   parameter renamed `DrivechainHeight` → `EcashHeight`.
3. **BTC-to-ECX P2P bridge**: a bridge node mode that peers with Bitcoin
   peers and relays data to eCash peers with translated network magic (branch
   `drynet4-bridge` builds the bridge image); this is how pre-fork mainnet
   data reaches the magic-isolated network.
4. **Repurpose set expanded**: 220 txids in `setRepurposeTx` (was 122).
5. **GBT relaxation dropped**: stock `getblocktemplate` behavior is back
   (requires ≥1 peer and completed IBD).
6. **Public mining pool**: first drynet with an official pool
   (`pool.drynet4.drivechain.dev`, simplepool in PPS mode paying on Thunder).

Carried over from drynet3: OP_DRIVECHAIN/BIP300/301, difficulty reset at the
fork height, OP_RETURN limits removed, magic-`nLockTime` replay protection
(`499999999`), Docker/binary CI.

## Notes

- Chain still reports `chain: main` in `getblockchaininfo`; verify by fork
  block hash (above), not chain name.
- drynet4 binaries are on `releases.drivechain.info` but not yet listed in
  `hashes.json` (verified 2026-08-11).
- drynet3 infrastructure remained live as of 2026-08-11; drynet2's explorer
  and Electrum endpoints were already unreachable.
