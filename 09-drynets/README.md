# eCash Drivechain Dry Runs

Documentation for each eCash/Drivechain dry-run network ("drynet").
One file per drynet (`DRYNETN.md`) covering how to connect, how to mine, ports,
fork heights, and what changed relative to the previous run.

| | [drynet1](DRYNET-1.md) | [drynet2](DRYNET-2.md) | [drynet3](DRYNET-3.md) |
|---|---|---|---|
| **Status** | Retired (infra offline) | Live | Live (current network) |
| **Fork height** | 955,584 | 957,600 | 957,600 |
| **Base version** | Bitcoin Core v31.1 | Bitcoin Core v31.1 | Bitcoin Core v31.1 |
| **Branch** | [`drynet1`](https://github.com/ecash-com/bitcoin/tree/drynet1) | [`drynet2`](https://github.com/ecash-com/bitcoin/tree/drynet2) | [`drynet3`](https://github.com/ecash-com/bitcoin/tree/drynet3) |
| **Public node** | n/a | `drynet2.drivechain.dev:8335` | `drynet3.drivechain.dev:8337` |
| **Replay protection** | Opt-in, magic tx version | Opt-in, magic tx version | Opt-in, magic nLockTime |
| **Repurpose txs** | none | 16 (test set) | 122 (Patoshi reassignment) |
| **Hub / info page** | n/a | [drynet2.drivechain.dev/info](https://drynet2.drivechain.dev/info) | [drynet3.drivechain.dev/info](https://drynet3.drivechain.dev/info) |
| **Explorer** | n/a | [explorer.drynet2.drivechain.dev](https://explorer.drynet2.drivechain.dev) | [explorer.drynet3.drivechain.dev](https://explorer.drynet3.drivechain.dev) |

## Quick start: connect and mine

Full step-by-step instructions live in each network's file
([DRYNET-2.md](DRYNET-2.md), [DRYNET-3.md](DRYNET-3.md)); the short version for
the current network (drynet3) is:

```sh
# 1. Get the software: build the branch, or pull the Docker image
#    ghcr.io/ecash-com/bitcoin:drynet3

# 2. Connect (a dedicated -datadir is mandatory, see the caveat below)
bitcoind -datadir=./drynet3 -addnode=drynet3.drivechain.dev:8337

# 3. Mine: create a wallet and payout address, then grind blocks
bitcoin-cli -datadir=./drynet3 createwallet mine
bitcoin-cli -datadir=./drynet3 getnewaddress
bitcoin-cli -datadir=./drynet3 generatetoaddress 1 <your-address> 2000000000
# an empty [] result just means no block found this round; run it again
```

Alternatively point any external `getblocktemplate` miner at your node's RPC
port (8332) with your payout address; each network file shows the exact setup.
For drynet2, use `-addnode=drynet2.drivechain.dev:8335` and datadir
`./drynet2`.

## What a drynet is

Each drynet is a fork of Bitcoin mainnet that activates BIP300/BIP301
(Drivechain) rules at a chosen "fork height" and resets proof-of-work
difficulty to 1 at that block, so anyone can CPU-mine. The code lives in the
[`ecash-com/bitcoin`](https://github.com/ecash-com/bitcoin) repo as one branch
per drynet (`drynet1`, `drynet2`, `drynet3`, etc.). Each branch is a small patch
stack rebased on top of Bitcoin Core; diffing two drynet branches shows
exactly what changed between runs.

## Common properties (all drynets)

Every drynet branch applies the same base patch set on top of Bitcoin Core v31.1:

- **BIP300/BIP301**: `OP_DRIVECHAIN` added and made standard.
- **Difficulty reset**: at the fork height the next-work calculation returns
  the minimum difficulty (1), making blocks CPU-mineable. Note both fork
  heights chosen so far (955,584 = 474 × 2016 and 957,600 = 475 × 2016) sit on
  difficulty-retarget boundaries.
- **Solo mining enabled**: the IBD and connected-peer requirements are removed
  from the `getblocktemplate` RPC, so a lone node can mine immediately.
- **Own data directory**: defaults to `~/.drivechain-ecash` (macOS:
  `~/Library/Application Support/drivechain-ecash`), config file
  `drivechain-ecash.conf`. In practice everyone passes an explicit
  `-datadir=./drynetN` anyway; see the per-network READMEs.
- **Seed nodes**: Bitcoin Core's DNS seeds removed, replaced with
  `seed.bip300.xyz`.
- **OP_RETURN limits removed**.
- **Replay protection** (opt-in, scheme varies per drynet; see below).
- **assumeutxo at the fork point**: a UTXO snapshot lets a node validate the
  tip within minutes (~9 GB download) while history syncs in the background.
  A full historical sync is ~850 GB.
- **CI builds**: Docker images at `ghcr.io/ecash-com/bitcoin:<branch>` (e.g.
  `:drynet3`) and binaries uploaded to `releases.drivechain.info`.

**Network-identity caveat**: the drynets keep Bitcoin mainnet's network
magic (`0xf9beb4d9`) and default P2P port 8333. Nodes on different drynets (or
mainnet) can connect to each other and will only sort themselves out via chain
divergence after the fork height. Always `-addnode`/`-connect` to the correct
network's public node, and always use a dedicated `-datadir`.

## What changed in each run

- **drynet1 → drynet2**: fork height moved from 955,584 to 957,600 (one
  retarget period later); introduced `setRepurposeTx`, a hard-coded set of
  transaction ids whose input-script checks are skipped, allowing coins to be
  reassigned ("repurposed") without valid signatures. drynet2 shipped a 16-tx
  test set.
- **drynet2 → drynet3**: same fork point (957,600, identical assumeutxo
  snapshot); replay protection scheme replaced (magic serialization byte →
  magic `nLockTime`, see [DRYNET-3.md](DRYNET-3.md)); repurpose
  set replaced with the real 122-tx list reassigning Patoshi-era coins.

## Sources of truth

- **Live network info**: the hub `/info` pages
  ([drynet2](https://drynet2.drivechain.dev/info),
  [drynet3](https://drynet3.drivechain.dev/info)) and the JSON endpoint served
  from the `server-config` repo (`config.json`). The plan is for that JSON to
  become the single source of truth, with `/info` as a frontend rendering of
  it. These READMEs are the human-friendly narrative layer on top.

### Information not in config file

Fields present in these READMEs (or needed by node operators) that the current
`config.json` does not expose:

1. **drynet3 entry**: the config seen so far lists only `drynet2` and
   `mainnet`.
2. **P2P addnode endpoint** (`host:port`, e.g. `drynet3.drivechain.dev:8337`).
   Currently only faucet/api/explorer/electrum are listed, but the addnode
   address is the first thing a node operator needs.
3. **Fork height and fork block hash** (`DrivechainHeight`).
4. **UTXO snapshot URL + assumeutxo height/hash** (e.g.
   `https://data.drivechain.dev/drynet2/utxo-957600.dat`).
5. **Source pointer**: repo, branch name, and Docker image tag
   (`ghcr.io/ecash-com/bitcoin:drynetN`).
6. **Replay-protection scheme identifier** (wallets need to know which
   mechanism the network uses).
7. **Status field** (`active` / `retired`) and launch date, so retired
   networks like drynet1 stay documented.
8. **Fast-withdrawal server** (`fw1.drynetN.drivechain.dev`): shown on /info
   but absent from the JSON.
