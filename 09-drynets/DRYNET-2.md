# Drynet 2

**Status: live.** Second eCash/Drivechain dry run: a fork of Bitcoin mainnet
with BIP300/BIP301 activated, PoW difficulty reset to 1 at the fork block, and
a first (test-sized) set of "repurpose" transactions.

| | |
|---|---|
| Fork height | **957,600** (= 475 × 2016, a retarget boundary) |
| Base version | Bitcoin Core v31.1 |
| Code | [`ecash-com/bitcoin`, branch `drynet2`](https://github.com/ecash-com/bitcoin/tree/drynet2) |
| Currency | eCash (ECX) |
| Chain type | `main` (mainnet fork) |
| Current height | see the [explorer](https://explorer.drynet2.drivechain.dev) or [/info](https://drynet2.drivechain.dev/info) |

## Endpoints & ports

| Service | Address |
|---|---|
| Public P2P node | `drynet2.drivechain.dev:8335` |
| Hub / faucet / info | https://drynet2.drivechain.dev ([/info](https://drynet2.drivechain.dev/info)) |
| Block explorer | https://explorer.drynet2.drivechain.dev |
| Esplora REST API | https://esplora.drynet2.drivechain.dev |
| Electrum | `drynet2.drivechain.dev` (TCP 50011, SSL 50012) |
| Fast withdrawal server | `fw1.drynet2.drivechain.dev` |
| UTXO snapshot | https://data.drivechain.dev/drynet2/utxo-957600.dat |

Local node ports are unchanged from Bitcoin Core: **P2P 8333, RPC 8332**.
(The `:8335` above is just the public node's listen port.)

## Running a node

Build the `drynet2` branch (see the branch README for dependencies), or use the
Docker image `ghcr.io/ecash-com/bitcoin:drynet2`, or grab a binary from
`releases.drivechain.info`. Then:

```sh
bitcoind -datadir=./drynet2 -addnode=drynet2.drivechain.dev:8335
```

`-datadir` is not optional in practice: the drynets share Bitcoin mainnet's
network magic and default ports, so running without a dedicated datadir will
mix state with (or connect you to) the wrong network.

**Fast bootstrap**: the client ships an assumeutxo commitment for the fork
block (height 957,600). Loading the UTXO snapshot (~9 GB) gets you validating
at the tip within minutes while historical blocks download in the background.
A full historical sync is ~850 GB.

## Mining

Difficulty restarted at 1 from the fork block, so blocks are CPU-mineable, and
`getblocktemplate` works without peers or completed IBD (both requirements were
patched out), so you can mine solo against your own node.

First create a wallet and a payout address:

```sh
bitcoin-cli -datadir=./drynet2 createwallet mine
bitcoin-cli -datadir=./drynet2 getnewaddress
```

### Option A: built-in miner (easiest)

`generatetoaddress` is available on this fork (it is a hidden RPC, so it does
not show in `help`, but it works). With low difficulty it will actually find
blocks:

```sh
bitcoin-cli -datadir=./drynet2 generatetoaddress 1 <your-address> 2000000000
```

At difficulty 1 a block takes about 4.3 billion hash attempts on average, and
`maxtries` is capped at ~2.1 billion per call, so a call will often return an
empty array (`[]`). That is not an error; just run the command again (in a
loop) until it prints a block hash. The built-in grinder is single-threaded
and slow, so treat this as the "prove it works" option; for sustained mining
use Option B. Normal difficulty retargeting also resumes after the fork, so
as network hashrate grows the built-in miner becomes impractical.

### Option B: external getblocktemplate miner

Enable the RPC server with credentials in `./drynet2/drivechain-ecash.conf`:

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

## Changes vs drynet1

- **Fork height moved** from 955,584 → 957,600 (one retarget period later),
  with new assumeutxo data for the new fork point.
- **Repurpose transactions introduced**: `src/repo_txns.h` hard-codes a set of
  16 transaction ids (`setRepurposeTx`); `CheckInputScripts` skips signature
  validation for them, allowing those transactions to spend coins without
  valid signatures. On drynet2 this is a small *test set* proving out the
  mechanism (the real Patoshi-coin list landed in drynet3).

Everything else (OP_DRIVECHAIN, difficulty reset, OP_RETURN limits removed,
GBT relaxation, seed nodes, datadir naming, Docker/binary CI) is carried over
unchanged from drynet1.

## Replay protection

Same opt-in scheme as drynet1: a transaction with `version = 12566463`
(`0x00BFBF3F`) is serialized with an extra marker byte (`0x3f`), which stock
Bitcoin Core cannot parse, so an opted-in transaction cannot be replayed on
mainnet. (Replaced with a magic-`nLockTime` scheme in drynet3.)

## Notes

- drynet3 forks at the **same height with the same assumeutxo snapshot**: the
  two networks share history up to 957,600 and diverge afterwards. Being
  connected to the wrong network's peers is a real footgun; always
  `-addnode` explicitly.
- This is the network currently listed in the `server-config` networks JSON
  (`id: drynet2`).
