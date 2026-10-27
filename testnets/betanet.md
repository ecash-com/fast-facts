# Betanet

The second eCash rehearsal. It forked Bitcoin at block **967,680** on 2026-09-19 and is still running as of 2026-10-27. No sunset date has been announced: the pool registry says it will retire "once mainnet is up", and the mobile wallet team says it keeps running beside mainnet. Betanet ECX (pECX) is a test-network asset, not native mainnet ECX; any separate redemption programme has its own terms ([overview](README.md#redemption-programmes)). **For mainnet values, use the [main docs](../README.md).**

## Parameters

| | |
|---|---|
| Fork height | 967,680 (= 2016 × 480); last shared block 967,679 |
| Fork block hash | `00000000000000030101ba5cfea54b22becc79f95dc6040beb76e01dd9d04042` |
| Difficulty at fork | reset to ~1e9 (`EcashForkBits = 0x19044b7e`), then Bitcoin's normal 2,016-block retargeting. The first retarget, at 969,696, hit the +300% cap |
| Replay `nLockTime` | `499999999` (`LOCKTIME_THRESHOLD - 1`) |
| Network magic | `0xeca5b104` (testnet3/4/regtest `0xeca5b114`/`24`/`34`) |
| Ports | P2P 8533 (Tor 8534), RPC 8532 |
| Datadir / config | `~/.ecash`, macOS `~/Library/Application Support/ecash`, Windows `AppData\Local\ecash`; `ecash.conf` |
| DNS seeds | `seed.beta.ecash.ninja`, `seed.beta.bip300.xyz`, `seed.beta.ecash.drivecha.in`, `seed.beta.ecash.zuexeuz.net` (no fixed seeds) |
| Repurposed (Patoshi) txids | 232 in `src/repo_txns.h` (not the same set as mainnet) |
| `OP_DRIVECHAIN` | `OP_NOP8` (0xb7) |
| `CLIENT_NAME` | `Bitcoin Core (eCash betanet)` |
| Source | `betanet` branch of https://github.com/ecash-com/bitcoin |
| Binaries | https://releases.ecash.com/ `L1-ecash-bitcoin/betanet/<commit>/` (`index.json` lists the latest build) |
| Docker | `ghcr.io/ecash-com/bitcoin:betanet` |
| assumeutxo | no fork-point snapshot. The branch pins only upstream heights (840,000, 880,000, 910,000, 935,000); load a height-935,000 snapshot produced by any synced Bitcoin Core v31 node (`dumptxoutset utxo-935000.dat rollback=935000`) |
| BIP300 thresholds | as mainnet, except unused-slot activation needs more than 1,008 of 2,016 blocks (mainnet 1,815) |
| Sidechains | FreeBank (slot 130, https://ecxfreebank.com) proposed at 968,020 and activated at 969,029 with 1,009 ACKs. The hub lists the LayerTwo Labs sidechains |

## Endpoints

| Service | Address |
|---|---|
| Hub (L1 tip, sidechain status) | https://beta.ecash.ninja |
| Explorer (mempool.space fork) | https://explorer.beta.ecash.ninja (API under `/api/`) |
| Esplora REST | https://esplora.beta.ecash.ninja |
| Electrum (Fulcrum 2.1.2) | `ssl://explorer.beta.ecash.ninja:50002` |
| Blockbook | https://blockbook.beta.ecash.ninja |
| Hosted enforcer | https://enforcer.beta.ecash.ninja |
| Fast withdrawal | https://fastwithdraw.betanet.ecash.ninja |
| Thunder (slot 9) P2P | `seed.beta.ecash.ninja:4009` |
| `explorer.ecash.com` | still 302-redirects here as of 2026-10-27 |

Machine-readable: the `betanet` entry in https://drivechain.dev/config.

## Quick start

```sh
mkdir betanet
bitcoind -datadir=./betanet            # betanet-branch binary
bitcoin-cli -datadir=./betanet getblockhash 967680
# 00000000000000030101ba5cfea54b22becc79f95dc6040beb76e01dd9d04042
```

Use a datadir that no mainnet node will ever open (see the datadir note in the [overview](README.md)).

## Mining on betanet

| | |
|---|---|
| L2L public pool | `stratum+tcp://stratum.beta.bip300.xyz:3334` (`pool.beta.bip300.xyz:3334` also answers), [dashboard](https://pool.beta.bip300.xyz), status https://pool.beta.bip300.xyz/api/status |
| Mode / fee | `pplns-coinbase`, 1%: every miner in the PPLNS window is paid in the block's own coinbase. Shares under 546 sats, or beyond the coinbase's output slots, roll to the front of the next block |
| Username | `<bitcoin-address>.<rig>`, any password |
| Ports | 3334 vardiff from 1 (individual miners), 3335 from 1,048,576 (Braiins-style rented hashrate), 3336 from 2,000,000 (NiceHash-style) |
| Other pools | `networks/betanet/pools.json` in [LayerTwo-Labs/mining-pools](https://github.com/LayerTwo-Labs/mining-pools): bip300.xyz, eCPool.tech, ePool, avonpool, Pow.re (private), FreeBank (private solo) |
| Mining stats | https://explorer.beta.ecash.ninja/mining |

Solo mining uses the same stack as mainnet ([08](../08-mining.md#solo-mining)), with the enforcer started as:

```sh
bip300301_enforcer --network-preset=betanet \
  --node-rpc-addr=127.0.0.1:8532 --node-rpc-user=user --node-rpc-pass=pass \
  --node-zmq-addr-sequence=tcp://127.0.0.1:29000 \
  --enable-wallet --enable-mempool       # serves getblocktemplate on 127.0.0.1:8122
```

## Test funds

There is no faucet. The fork copied Bitcoin's UTXOs at block 967,679, but the keys controlling those coins may still control real BTC or mainnet ECX. **Do not import production private keys or seeds for testing.** To inspect balances, use public descriptors in a watch-only wallet. For signing drills, use isolated regtest keys or dedicated betanet test funds. You can also mine to a fresh test wallet at a betanet pool (coinbase outputs mature after 100 blocks).

## Replay drill

1. In an isolated regtest setup using the betanet binary, construct a spend with `nLockTime = 499999999` and at least one input `nSequence < 0xffffffff`. Use dedicated test keys and the same test UTXO on both nodes.
2. Compare `testmempoolaccept` results on that node and stock Bitcoin Core regtest. The otherwise-valid transaction should be accepted on the betanet-binary node and rejected as `non-final` on Bitcoin Core. Do not broadcast a rehearsal to Bitcoin mainnet; a missing-input rejection alone proves nothing about replay protection.
3. The betanet branch ships `test/functional/feature_replay_protection.py` for the betanet value. Mainnet's `master` does not include that test.
4. **Before mainnet, change the constant to `499999998`** and run the regtest check in [09](../09-mainnet-wallet-and-exchange-support.md#2-transaction-rules-every-ecx-send).
