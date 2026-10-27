# eCash (ECX) Integration Guide

Last updated: **2026-10-27**, before the mainnet fork (Bitcoin tip 973,441, 287 blocks to go). This guide covers **mainnet**. Alphanet and betanet are covered in [testnets/](testnets/README.md).

eCash mainnet forks Bitcoin at block **973,728**, Bitcoin's next difficulty adjustment. The fork is set by height, not by date. As of 2026-10-27, mempool.space puts it at about **2026-10-29 15:00 UTC**; the earlier announced target was 2026-10-31. Live ETA: https://mempool.space/api/v1/difficulty-adjustment.

## Holder quick links

| What do you need? | Guide |
|---|---|
| I hold BTC and want to understand ECX access | [Bitcoin-holder guide](https://ecash.com/how-to-claim/) |
| I need to separate BTC and ECX safely | [Coin splitting: wallet apps, hardware wallets and exchange accounts](https://ecash.com/split/) |
| I want markets or my exchange's fork policy | [Markets & exchange support](10-markets-and-exchange-support.md) |
| I need wallet software | [Official downloads](https://ecash.com/download/) — check the network and release |
| I operate a wallet or exchange | [Mainnet integration playbook](09-mainnet-wallet-and-exchange-support.md) |

There is no pre-claim registration. Never enter a seed phrase on a website. A betanet split does not split mainnet coins, and an exchange's fork-support announcement does not necessarily mean ECX trading or withdrawals are available.

## Quick facts

| | |
|---|---|
| Asset | eCash, ticker **ECX**, 8 decimals. Not XEC (e.cash), and not a Solana token ([FAQ](07-faq.md#ecx-xec-wecx-and-solana-tokens)) |
| What it is | Hard fork of Bitcoin by LayerTwo Labs that activates drivechains (BIP300/301). Every address holding BTC at block 973,727 has the same amount of ECX on the new chain. BTC is untouched |
| Fork | Blocks 0 to **973,727** are shared with Bitcoin. **973,728** is the first block mined by eCash miners, so its hash is known only after it is mined |
| Node software | **`ecash-v1.1.0-core31.1`**, a fork of Bitcoin Core v31.1 (`master` of [github.com/ecash-com/bitcoin](https://github.com/ecash-com/bitcoin)). Binaries at [releases.ecash.com](https://releases.ecash.com/L1-ecash-bitcoin/mainnet/latest/release.json), Docker `ghcr.io/ecash-com/bitcoin:latest`. **v1.0.0 nodes split off the chain at block 974,016: upgrade them** |
| Consensus | SHA-256d PoW. Difficulty resets to ~2.6e9 at the fork, then retargets **every 288 blocks** with a dampened rule ([details](01-node-setup.md#difficulty-after-the-fork)) |
| Address/key formats | Identical to Bitcoin (`1...`/`3...`/`bc1q...`/`bc1p...`, same WIF/xpub/xprv, coin type `0'`) |
| Network identity | Magic `0xeca5dcb7`, ports **8533/8532**, datadir `~/.ecash`, `ecash.conf`. Cannot peer with Bitcoin Core. `getblockchaininfo` still reports `chain=main` |
| Seed nodes | `seed.ecash.ninja`, `seed.bip300.xyz`, `seed.ecash.drivecha.in`, `seed.ecash.zuexeuz.net`, `seed.ecash.eu.com`, all on port 8533. Only the first and fourth resolve as of 2026-10-27 |
| Replay protection | Opt-in: **`nLockTime = 499999998`** plus at least one input with `nSequence < 0xffffffff`. Betanet's `499999999` does not work on mainnet ([details](03-replay-protection-and-coin-splitting.md)) |
| Consensus quirk | 232 whitelisted "repurpose" transactions move Satoshi-era (Patoshi) coins without signatures |
| Public endpoints | Explorer https://explorer.ecash.ninja, Esplora https://esplora.ecash.ninja, Electrum `ssl://explorer.ecash.ninja:50002`, Blockbook https://blockbook.ecash.ninja. All are live and follow the shared chain until the fork ([04](04-explorers-and-apis.md)) |
| Fast bootstrap | assumeutxo snapshot at height 965,000, pinned in the node ([01](01-node-setup.md#bootstrapping-assumeutxo)) |
| Mining | Any SHA-256d miner. No public mainnet pool has been announced yet ([08](08-mining.md)) |
| Sidechains | None is active at the fork. Each slot must be proposed and activated by miners ([05](05-sidechains-and-l2s.md)) |
| Dev contact | dev@layertwolabs.com, [t.me/DcInsiders](https://t.me/DcInsiders) |

## Docs

1. [Full Node Setup](01-node-setup.md)
2. [Wallets, Keys & Addresses](02-wallets-keys-addresses.md)
3. [Replay & Coin Splitting](03-replay-protection-and-coin-splitting.md) (read first)
4. [Explorers & APIs](04-explorers-and-apis.md)
5. [Sidechains & L2s](05-sidechains-and-l2s.md) (includes proposing new L2s)
6. [Contacts & Resources](06-contacts-and-resources.md)
7. [FAQ](07-faq.md)
8. [Mining / Mining Pool](08-mining.md)
9. [Mainnet Wallet & Exchange Support](09-mainnet-wallet-and-exchange-support.md) (step-by-step integration playbook)
10. [Markets & Exchange Support](10-markets-and-exchange-support.md) (wrapped-token markets and exchange distribution policies)
11. [Test networks: alphanet & betanet](testnets/README.md)

## Integration checklist

1. Run `ecash-v1.1.0-core31.1` (or later) with its own datadir, bootstrapped from the 965,000 snapshot. Make sure it is running v1.1.0 before block 974,016.
2. Reuse your Bitcoin pipeline (RPC, ZMQ, descriptors, electrs/mempool.space) pointed at the eCash node. Key balances by `(chain, address)`.
3. Set **`nLockTime = 499999998`** on every ECX transaction, with at least one input **`nSequence < 0xffffffff`**. Verify both fields in the signed transaction. If you tested on betanet, change the locktime constant.
4. Fork window: snapshot balances at block 973,727 and freeze withdrawals by height. Split coins on the eCash side first, then resume with deep confirmation requirements while difficulty re-equilibrates.
5. Decide your crediting policy for customer BTC held at block 973,727, and publish it. Distinguish distribution, trading, deposits and withdrawals. Explain the Satoshi-coin reassignment and the differences between native ECX, wrapped wECX and the unrelated XEC ([markets guide](10-markets-and-exchange-support.md)).
6. Verify the fork block's hash against the explorer once it is mined, and alert on any tip mismatch.

## Primary sources

- [ecash.com](https://ecash.com): official site (fork height, [coin splitting](https://ecash.com/split/), [FAQ](https://ecash.com/faq/), [downloads](https://ecash.com/download/))
- [github.com/ecash-com/bitcoin](https://github.com/ecash-com/bitcoin) `master` / tag `ecash-v1.1.0-core31.1`: consensus parameters in `src/kernel/chainparams.cpp`
- [drivechain.dev/config](https://drivechain.dev/config): machine-readable network registry (backends, seeds, snapshot) used by the official wallets. The mainnet entry has id `ecash`
- [releases.ecash.com/index.json](https://releases.ecash.com/index.json): node builds per network, with SHA-256s
- [LayerTwo-Labs/mining-pools](https://github.com/LayerTwo-Labs/mining-pools) (rendered at [pool.drivechain.info](https://pool.drivechain.info)): pool registry
- [BIP300](https://github.com/bitcoin/bips/blob/master/bip-0300.mediawiki), [BIP301](https://github.com/bitcoin/bips/blob/master/bip-0301.mediawiki)

## Support Groups

- [Telegram, official announcements](https://t.me/ecashcom_official)
- [Telegram, eCash Hangout](https://t.me/eCashHangout)
- [Discord](https://discord.gg/swyE78UPw)

Feedback Appreciated!
