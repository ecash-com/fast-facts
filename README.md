## eCash (ECX) Integration Guide

Last updated: **2026-09-22**, during the **betanet** stage.

eCash rolls out in three stages, each a fresh fork of Bitcoin mainnet: **alphanet** (forked 2026-08-23, retired 2026-09-24), **betanet** (live since 2026-09-19, fork block 967,680), and **mainnet** (fork block ~973,728, ~2026-10-31).

## Quick facts

| | |
|---|---|
| Asset | eCash, ticker **ECX** |
| What it is | Hard fork of Bitcoin by LayerTwo Labs, activating drivechains (BIP300/301). Every BTC address is credited ECX 1:1 at the fork block; BTC itself is untouched |
| Fork points | betanet **967,680** (2026-09-19, live, hash `00000000000000030101ba5cfea54b22becc79f95dc6040beb76e01dd9d04042`), mainnet **~973,728** (~2026-10-31, ~15:00 UTC); alphanet **963,648** (2026-08-23, retired 2026-09-24) |
| Consensus | SHA-256d PoW, one-time difficulty reset at the fork (betanet: to ~1e9), then normal 2,016-block retargeting |
| Node software | Fork of **Bitcoin Core v31.1**: [github.com/ecash-com/bitcoin](https://github.com/ecash-com/bitcoin); binaries at [releases.ecash.com](https://releases.ecash.com/) (mirror: [releases.drivechain.info](https://releases.drivechain.info/)) |
| Address/key formats | Identical to Bitcoin (`1...`/`3...`/`bc1...`, same WIF/xpub/xprv, secp256k1) |
| Network identity | Own network magic per stage (betanet `0xeca5b104`, alphanet `0xeca5a104`) and ports **8533/8532**, so nodes can't cross-connect with Bitcoin Core; `getblockchaininfo` still reports `chain=main` ([details](01-node-setup.md)) |
| Replay protection | Opt-in: set `nLockTime = 499999999` on eCash transactions ([details](03-replay-protection-and-coin-splitting.md)) |
| Consensus quirk | Whitelisted "repurpose" transactions reassign Satoshi-era (Patoshi) coins without signatures (232 txids on betanet) |
| Mining | SHA-256d, any Bitcoin miner works. Public pool `stratum+tcp://stratum.beta.bip300.xyz:3334` (PPLNS paid in the coinbase, 1% fee; [details](08-mining.md)) |
| Sidechains | 7 LayerTwo Labs L2s (Thunder, zSide, BitNames, BitAssets, Truthcoin, Photon, CoinShift) plus community proposals; betanet activated **FreeBank** (slot 130) at height 969,029 |
| Live network | **betanet**: DNS seeds `seed.beta.ecash.ninja` etc. (port 8533), [hub](https://beta.ecash.ninja), [explorer](https://explorer.beta.ecash.ninja), [Esplora API](https://esplora.beta.ecash.ninja/blocks/tip/height), Electrum `ssl://explorer.beta.ecash.ninja:50002` |
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

1. Run a node against betanet now (`betanet` branch; alphanet is retired 2026-09-24), and move to the mainnet branch with a fresh datadir when announced. Dedicated datadir per stage, verify the fork-block hash.
2. Reuse your Bitcoin pipeline (RPC, ZMQ, descriptors, electrs/mempool.space) pointed at the eCash node. Key balances by `(chain, address)`.
3. Fork week: freeze withdrawals at the fork block, split coins (eCash side first, `nLockTime = 499999999`), resume with deep confirmation requirements while difficulty re-equilibrates.
4. Set the magic nLockTime on every ECX withdrawal, permanently.
5. Decide your crediting policy for customer BTC held at the mainnet fork block; prepare comms about ECX and the Satoshi-coin reassignment.

## Primary sources

- [ecash.com](https://ecash.com), official site, stage dates and heights, FAQ
- [github.com/ecash-com/bitcoin](https://github.com/ecash-com/bitcoin) (branch READMEs carry each stage's parameters), [github.com/LayerTwo-Labs](https://github.com/LayerTwo-Labs)
- [beta.ecash.ninja](https://beta.ecash.ninja), betanet hub (L1 tip, sidechain status); [explorer.beta.ecash.ninja](https://explorer.beta.ecash.ninja), live betanet explorer
- [pool.drivechain.info](https://pool.drivechain.info), mining pool registry
- [BIP300](https://github.com/bitcoin/bips/blob/master/bip-0300.mediawiki), [BIP301](https://github.com/bitcoin/bips/blob/master/bip-0301.mediawiki)

## Support Groups

- [Telegram](https://t.me/eCashHangout)
- [Discord](https://discord.gg/swyE78UPw)
  
Feedback Appreciated!
