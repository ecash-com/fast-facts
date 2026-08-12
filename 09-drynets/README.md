# eCash Drivechain Dry Runs

Documentation for each eCash/Drivechain dry-run network ("drynet").
One file per drynet (`DRYNETN.md`) covering how to connect, how to mine, ports,
fork heights, and what changed relative to the previous run.

| | [drynet1](DRYNET-1.md) | [drynet2](DRYNET-2.md) | [drynet3](DRYNET-3.md) | [drynet4](DRYNET-4.md) |
|---|---|---|---|---|
| **Status** | Retired (infra offline) | Retired (explorer offline) | Live (superseded) | **Live (current network)** |
| **Fork height** | 955,584 | 957,600 | 957,600 | 961,632 |
| **Base version** | Bitcoin Core v31.1 | Bitcoin Core v31.1 | Bitcoin Core v31.1 | Bitcoin Core v31.1 |
| **Branch** | [`drynet1`](https://github.com/ecash-com/bitcoin/tree/drynet1) | [`drynet2`](https://github.com/ecash-com/bitcoin/tree/drynet2) | [`drynet3`](https://github.com/ecash-com/bitcoin/tree/drynet3) | [`drynet4`](https://github.com/ecash-com/bitcoin/tree/drynet4) |
| **Network identity** | Bitcoin's magic + ports | Bitcoin's magic + ports | Bitcoin's magic + ports | Own magic `0xeca5d404`, P2P 8533 / RPC 8532 |
| **Public node** | n/a | `drynet2.drivechain.dev:8335` | `drynet3.drivechain.dev:8337` | `drynet4.drivechain.dev:8533` |
| **Replay protection** | Opt-in, magic tx version | Opt-in, magic tx version | Opt-in, magic nLockTime | Opt-in, magic nLockTime |
| **Repurpose txs** | none | 16 (test set) | 122 (Patoshi reassignment) | 220 (expanded set) |
| **Mining pool** | n/a | n/a | n/a | `pool.drynet4.drivechain.dev:3333` |
| **Hub / info page** | n/a | [drynet2.drivechain.dev/info](https://drynet2.drivechain.dev/info) | [drynet3.drivechain.dev/info](https://drynet3.drivechain.dev/info) | [drynet4.drivechain.dev/info](https://drynet4.drivechain.dev/info) |
| **Explorer** | n/a | offline | [explorer.drynet3.drivechain.dev](https://explorer.drynet3.drivechain.dev) | [explorer.drynet4.drivechain.dev](https://explorer.drynet4.drivechain.dev) |

## Quick start: connect and mine

Full step-by-step instructions live in each network's file; the short version
for the current network (drynet4) is:

```sh
# 1. Get the software: build the drynet4 branch, pull the Docker image
#    ghcr.io/ecash-com/bitcoin:drynet4, or download a binary from
#    releases.drivechain.info

# 2. Connect (drynet4 has its own network magic; -addnode is optional
#    since drynet4.drivechain.dev is the built-in DNS seed)
bitcoind -datadir=./drynet4 -addnode=drynet4.drivechain.dev:8533

# 3. Mine: point a stratum miner at the public pool
#    stratum+tcp://pool.drynet4.drivechain.dev:3333
#    (username = a Thunder sidechain address, any password)
```

Or point an external `getblocktemplate` miner at your own node's RPC port
(8532); [08-mining.md](../08-mining.md) and [DRYNET-4.md](DRYNET-4.md#mining)
show the exact setups. Note the drynet1-3 trick of grinding blocks with
`generatetoaddress` is impractical on drynet4; difficulty has already
climbed beyond CPU solo mining.

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
  the minimum difficulty (1), making blocks CPU-mineable at first. All fork
  heights chosen so far (955,584 = 474 × 2016, 957,600 = 475 × 2016,
  961,632 = 477 × 2016) sit on difficulty-retarget boundaries.
- **Own data directory** (never `~/.bitcoin`): `~/.drivechain-ecash` with
  `drivechain-ecash.conf` on drynet1-3; renamed to `~/.ecash` with
  `ecash.conf` on drynet4. In practice everyone passes an explicit
  `-datadir=./drynetN` anyway; see the per-network READMEs.
- **Seed nodes**: Bitcoin Core's DNS seeds removed (`seed.bip300.xyz` on
  drynet1-3, `drynet4.drivechain.dev` on drynet4).
- **OP_RETURN limits removed**.
- **Replay protection** (opt-in, scheme varies per drynet; see below).
- **CI builds**: Docker images at `ghcr.io/ecash-com/bitcoin:<branch>` (e.g.
  `:drynet4`) and binaries uploaded to `releases.drivechain.info`.
- **assumeutxo at the fork point**: a UTXO snapshot (~9.5 GB) lets a node
  validate the tip within minutes while history syncs in the background
  (`data.drivechain.dev/drynetN/utxo-<forkheight>.dat`; drynet4's arrived
  2026-08-12). A full historical sync is ~850 GB.

Properties that changed in drynet4 (see [DRYNET-4.md](DRYNET-4.md)):

- **Solo mining relaxation dropped**: drynet1-3 removed the IBD and
  connected-peer requirements from `getblocktemplate` so a lone node could
  mine immediately; drynet4 restored the stock checks.

**Network-identity caveat (drynet1-3 only)**: those drynets keep Bitcoin
mainnet's network magic (`0xf9beb4d9`) and default P2P port 8333, so nodes on
different drynets (or mainnet) can connect to each other and only sort
themselves out via chain divergence after the fork height; always
`-addnode`/`-connect` to the correct network's public node and use a dedicated
`-datadir`. **drynet4 fixes this**: it has its own network magic
(`0xeca5d404`) and ports (P2P 8533, RPC 8532) and cannot handshake with
Bitcoin Core or earlier drynets.

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
- **drynet3 → drynet4**: new fork point (961,632); eCash gets its own network
  identity (magic `0xeca5d404`, ports 8533/8532, datadir `~/.ecash`, config
  `ecash.conf`, DNS seed `drynet4.drivechain.dev`, eCash client name) plus a
  BTC→ECX bridge-node mode to relay pre-fork data into the isolated network;
  repurpose set expanded to 220 txids; `getblocktemplate` peer/IBD relaxation
  dropped; first run with a public mining pool
  (`pool.drynet4.drivechain.dev:3333`). See [DRYNET-4.md](DRYNET-4.md).

## Sources of truth

- **Live network info**: the hub `/info` pages
  ([drynet4](https://drynet4.drivechain.dev/info),
  [drynet3](https://drynet3.drivechain.dev/info)) and the JSON endpoint served
  from the `server-config` repo (`config.json`). The plan is for that JSON to
  become the single source of truth, with `/info` as a frontend rendering of
  it. These READMEs are the human-friendly narrative layer on top.

### Information not in config file

Fields present in these READMEs (or needed by node operators) that the current
`config.json` does not expose:

1. **Current-network entry**: the config seen so far lists only `drynet2` and
   `mainnet`, neither drynet3 nor drynet4.
2. **P2P addnode endpoint** (`host:port`, e.g. `drynet4.drivechain.dev:8533`).
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
