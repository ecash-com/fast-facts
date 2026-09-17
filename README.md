## eCash (ECX) Integration Guide

Last updated: **2026-09-16**, during the **alphanet** stage.

eCash rolls out in three stages, each a fresh fork of Bitcoin mainnet: **alphanet** (live since 2026-08-23), **betanet** (fork block 967,680, expected ~2026-09-19), and **mainnet** (fork block ~973,728, ~2026-10-31). 

## Quick facts

| | |
|---|---|
| Asset | eCash, ticker **ECX** |
| What it is | Hard fork of Bitcoin by LayerTwo Labs, activating drivechains (BIP300/301). Every BTC address is credited ECX 1:1 at the fork block; BTC itself is untouched |
| Fork points | alphanet **963,648** (2026-08-23, live), betanet **967,680** (~2026-09-19), mainnet **~973,728** (~2026-10-31, ~15:00 UTC) |
| Consensus | SHA-256d PoW, one-time difficulty reset at the fork, then normal retargeting |
| Node software | Fork of **Bitcoin Core v31.1**: [github.com/ecash-com/bitcoin](https://github.com/ecash-com/bitcoin); binaries at [releases.ecash.com](https://releases.ecash.com/) (mirror: [releases.drivechain.info](https://releases.drivechain.info/)) |
| Address/key formats | Identical to Bitcoin (`1...`/`3...`/`bc1...`, same WIF/xpub/xprv, secp256k1) |
| Network identity | Own network magic per stage (alphanet `0xeca5a104`, betanet `0xeca5b104`) and ports **8533/8532**, so nodes can't cross-connect with Bitcoin Core; `getblockchaininfo` still reports `chain=main` ([details](01-node-setup.md)) |
| Replay protection | Opt-in: set `nLockTime = 499999999` on eCash transactions ([details](03-replay-protection-and-coin-splitting.md)) |
| Consensus quirk | Whitelisted "repurpose" transactions reassign Satoshi-era (Patoshi) coins without signatures |
| Mining | SHA-256d, any Bitcoin miner works. Public pool `stratum+tcp://pool.alpha.bip300.xyz:3333` ([details](08-mining.md)) |
| Sidechains | 7 drivechain L2s at launch (Thunder, zSide, BitNames, BitAssets, Truthcoin, Photon, CoinShift) |
| Live network | **alphanet**: DNS seeds `seed.alpha.ecash.ninja` etc. (port 8533), [explorer](https://explorer.alpha.ecash.ninja), [Esplora API](https://esplora.alpha.ecash.ninja/blocks/tip/height), Electrum `ssl://explorer.alpha.ecash.ninja:50002` |
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

## Integration checklist

1. Run a node against alphanet now, move to the betanet branch with a fresh datadir when it forks (~2026-09-19), and to the mainnet branch when announced. Dedicated datadir per stage, verify the fork-block hash.
2. Reuse your Bitcoin pipeline (RPC, ZMQ, descriptors, electrs/mempool.space) pointed at the eCash node. Key balances by `(chain, address)`.
3. Fork week: freeze withdrawals at the fork block, split coins (eCash side first, `nLockTime = 499999999`), resume with deep confirmation requirements while difficulty re-equilibrates.
4. Set the magic nLockTime on every ECX withdrawal, permanently.
5. Decide your crediting policy for customer BTC held at the mainnet fork block; prepare comms about ECX and the Satoshi-coin reassignment.

## Primary sources

- [ecash.com](https://ecash.com), official site, stage dates and heights, FAQ
- [github.com/ecash-com/bitcoin](https://github.com/ecash-com/bitcoin) (branch READMEs carry each stage's parameters), [github.com/LayerTwo-Labs](https://github.com/LayerTwo-Labs)
- [explorer.alpha.ecash.ninja](https://explorer.alpha.ecash.ninja), live alphanet explorer
- [pool.drivechain.info](https://pool.drivechain.info), mining pool registry
- [BIP300](https://github.com/bitcoin/bips/blob/master/bip-0300.mediawiki), [BIP301](https://github.com/bitcoin/bips/blob/master/bip-0301.mediawiki)

## Support Groups

- [Telegram](https://t.me/eCashHangout)
- [Discord](https://discord.gg/swyE78UPw)
  
Feedback Appreciated!
