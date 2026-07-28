# Drynet 1

**Status: retired.** The public infrastructure (`drynet1.drivechain.dev`) is no
longer online and drynet1 is not listed in the networks JSON. This README is
kept for the historical record; use [drynet3](DRYNET-3.md) for the
current network.

## Overview

First eCash/Drivechain dry run: a fork of Bitcoin mainnet with BIP300/BIP301
activated and PoW difficulty reset to 1 at the fork block.

| | |
|---|---|
| Fork height | **955,584** (= 474 × 2016, a retarget boundary) |
| Base version | Bitcoin Core v31.1 |
| Code | [`ecash-com/bitcoin`, branch `drynet1`](https://github.com/ecash-com/bitcoin/tree/drynet1) |
| Currency | eCash (ECX) |
| Chain type | `main` (mainnet fork) |

## Ports

| Service | Port |
|---|---|
| P2P (default, same as Bitcoin Core) | 8333 |
| RPC | 8332 |

Data directory: `~/.drivechain-ecash` by default (config file
`drivechain-ecash.conf`), but always run with an explicit `-datadir` to avoid
mixing state with other networks.

## Consensus / policy changes vs Bitcoin Core v31.1

- `OP_DRIVECHAIN` (BIP300/301) added and made standard.
- Difficulty reset to minimum at fork height 955,584.
- IBD and peer requirements removed from `getblocktemplate` (solo mining works).
- OP_RETURN limits removed.
- **Optional replay protection via magic transaction version**: a transaction
  with `version = 12566463` (`0x00BFBF3F`) is serialized with an extra marker
  byte (`0x3f`). Stock Bitcoin Core cannot parse that serialization, so an
  opted-in transaction cannot be replayed on mainnet.
- assumeutxo snapshot data for the fork point (height 955,584).
- Bitcoin Core DNS seeds removed; `seed.bip300.xyz` added.

Unlike drynet2/3, drynet1 has **no repurpose-transaction set**: no coins were
reassigned at the fork.

## How it was run and mined

The public drynet1 node is offline, so there is nothing to connect to anymore.
For the record, the procedure was the same as the later drynets:

```sh
# connect (the drynet1 public node no longer exists)
bitcoind -datadir=./drynet1 -addnode=<drynet1-node-host:port>

# mine: create a wallet/address, then grind blocks with the built-in miner
# or point a getblocktemplate miner at your node
bitcoin-cli -datadir=./drynet1 createwallet mine
bitcoin-cli -datadir=./drynet1 getnewaddress
bitcoin-cli -datadir=./drynet1 generatetoaddress 1 <your-address> 2000000000
```

Difficulty restarts at 1 from the fork block, so blocks were CPU-mineable. See
[DRYNET-3.md](DRYNET-3.md#mining) for the current, detailed instructions on the
live network.

## Notes

- Superseded by drynet2, which moved the fork point one retarget period later
  (957,600) and introduced the repurpose-transaction mechanism.
- Network magic and default port were left identical to Bitcoin mainnet, so
  cross-connection with mainnet/other-drynet peers was possible, one of the
  operational lessons carried into later runs (always `-addnode` the right
  host and use a dedicated datadir).
