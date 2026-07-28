# Drynet 3

**Status: live (current network).** Third eCash/Drivechain dry run: same fork
point as drynet2 but with a new replay-protection scheme and the full
Patoshi-coin reassignment list.


| | |
|---|---|
| Fork height | **957,600** (= 475 × 2016, a retarget boundary; same fork point and assumeutxo snapshot as drynet2) |
| Base version | Bitcoin Core v31.1 |
| Code | [`ecash-com/bitcoin`, branch `drynet3`](https://github.com/ecash-com/bitcoin/tree/drynet3) |
| Currency | eCash (ECX) |
| Chain type | `main` (mainnet fork) |
| Current height | see the [explorer](https://explorer.drynet3.drivechain.dev) or [/info](https://drynet3.drivechain.dev/info) |

## Endpoints & ports

| Service | Address |
|---|---|
| Public P2P node | `drynet3.drivechain.dev:8337` |
| Hub / info | https://drynet3.drivechain.dev ([/info](https://drynet3.drivechain.dev/info)) |
| Block explorer | https://explorer.drynet3.drivechain.dev |
| Esplora REST API | https://esplora.drynet3.drivechain.dev |
| Electrum | `ssl://drynet3.drivechain.dev:50012` |
| Fast withdrawal server | `fw1.drynet3.drivechain.dev` |

Local node ports are unchanged from Bitcoin Core: **P2P 8333, RPC 8332**.
(The `:8337` above is just the public node's listen port.)

## Running a node

Build the `drynet3` branch (see the branch README for dependencies) or use the
Docker image `ghcr.io/ecash-com/bitcoin:drynet3`. Then:

```sh
bitcoind -datadir=./drynet3 -addnode=drynet3.drivechain.dev:8337
```

`-datadir` is not optional in practice: the drynets share Bitcoin mainnet's
network magic and default ports; omitting it writes into your real Bitcoin
datadir, and drynet2/drynet3 share all history up to block 957,600, so a node
peered with the wrong network can follow the wrong chain. Always pass
`-datadir` and `-addnode` explicitly.

**Fast bootstrap**: an assumeutxo commitment for the fork block (957,600) lets
you load a ~9 GB UTXO snapshot and validate at the tip within minutes while
history downloads in the background. Full historical sync is ~850 GB.

## Mining

Difficulty restarted at 1 from the fork block, so blocks are CPU-mineable, and
`getblocktemplate` works without peers or completed IBD (both requirements were
patched out), so you can mine solo against your own node.

First create a wallet and a payout address:

```sh
bitcoin-cli -datadir=./drynet3 createwallet mine
bitcoin-cli -datadir=./drynet3 getnewaddress
```

### Option A: built-in miner (easiest)

`generatetoaddress` is available on this fork (it is a hidden RPC, so it does
not show in `help`, but it works). With low difficulty it will actually find
blocks:

```sh
bitcoin-cli -datadir=./drynet3 generatetoaddress 1 <your-address> 2000000000
```

At difficulty 1 a block takes about 4.3 billion hash attempts on average, and
`maxtries` is capped at ~2.1 billion per call, so a call will often return an
empty array (`[]`). That is not an error; just run the command again (in a
loop) until it prints a block hash. The built-in grinder is single-threaded
and slow, so treat this as the "prove it works" option; for sustained mining
use Option B. Normal difficulty retargeting also resumes after the fork, so
as network hashrate grows the built-in miner becomes impractical.

### Option B: external getblocktemplate miner

Enable the RPC server with credentials in `./drynet3/drivechain-ecash.conf`:

```
server=1
rpcuser=user
rpcpassword=pass
```

Then point any GBT-compatible miner at your node with your payout address,
e.g. with cpuminer:

```sh
minerd -a sha256d -o http://127.0.0.1:8332 -O user:pass \
  --coinbase-addr=<your-address>
```

## Changes vs drynet2

1. **New replay-protection scheme: magic `nLockTime`** (replaces the
   magic-version serialization scheme):
   - A transaction with `nLockTime = 499999999` (`LOCKTIME_THRESHOLD - 1`) is
     treated as **final** on drynet3.
   - Stock Bitcoin Core reads that value as a block-height locktime ~500
     million blocks in the future and rejects the transaction as non-final,
     so an opted-in transaction confirms here but cannot replay onto Bitcoin.
   - Unlike drynet2's scheme, this keeps the standard transaction
     serialization, so unmodified wallets/libraries can construct protected
     transactions just by setting `nLockTime`.
   - Covered by unit tests and `test/functional/feature_replay_protection.py`.
2. **Patoshi coin reassignment**: `setRepurposeTx` replaced with the real list
   of **122 transaction ids** that reassign Patoshi-era coins (drynet2 carried
   only a 16-tx test set). Input-script checks are skipped for these
   transactions, letting them move the coins without the original keys.

Everything else (OP_DRIVECHAIN/BIP300/301, difficulty reset, OP_RETURN limits
removed, GBT relaxation, seed node `seed.bip300.xyz`, `drivechain-ecash`
datadir naming, Docker/binary CI) is carried over from drynet2.

## Notes

- As of this writing, `drynet3` is **not yet in the networks JSON**
  (`config.json` lists only `drynet2` and `mainnet`)
- drynet3 binaries are now uploading to `releases.drivechain.info`
  (`L1-ecash-bitcoin-drynet3-<target>.zip`, verified 2026-07-27)
- The drynet3 UTXO snapshot is published at
  `https://data.drivechain.dev/drynet3/utxo-957600.dat` (~9 GB; same fork
  point and snapshot content as drynet2's)