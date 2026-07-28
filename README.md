## eCash (ECX) Integration Guide

Last updated: **2026-07-27**, pre-launch.

Technical parameters come from the live **drynet3** dry-run network and the `drynet3` branch of [`ecash-com/bitcoin`](https://github.com/ecash-com/bitcoin). Final launch parameters (fork height/hash, branch/tag, replay scheme) will be published at [drivechain.info/dev.txt](https://drivechain.info/dev.txt) and [ecash.com](https://ecash.com). Re-verify anything marked *drynet3* before go-live.

## Quick facts

| | |
|---|---|
| Asset | eCash, ticker **ECX** |
| What it is | Hard fork of Bitcoin by LayerTwo Labs, activating drivechains (BIP300/301). Every BTC address is credited ECX 1:1 at the fork block; BTC itself is untouched |
| Fork point | BTC block **~963,648**, targeted **August 22, 2026** (~15:00 UTC) |
| Consensus | SHA-256d PoW, one-time difficulty reset to minimum at fork, then normal retargeting |
| Node software | Fork of **Bitcoin Core v31.1**: [github.com/ecash-com/bitcoin](https://github.com/ecash-com/bitcoin); binaries at [releases.drivechain.info](https://releases.drivechain.info/) |
| Address/key formats | Identical to Bitcoin (`1...`/`3...`/`bc1...`, same WIF/xpub/xprv, secp256k1) |
| Network identity | Keeps Bitcoin's network magic, `chain=main`, and ports 8333/8332. Isolate nodes carefully ([details](01-node-setup.md)) |
| Replay protection | Opt-in: set `nLockTime = 499999999` on eCash transactions ([details](03-replay-protection-and-coin-splitting.md)) |
| Consensus quirk | 122 whitelisted "repurpose" transactions reassign Satoshi-era (Patoshi) coins without signatures |
| Sidechains | 7 drivechain L2s at launch (Thunder, zSide, BitNames, BitAssets, Truthcoin, Photon, CoinShift) |
| Test network | **drynet3** (live): node `drynet3.drivechain.dev:8337`, [explorer](https://explorer.drynet3.drivechain.dev), [info hub](https://drynet3.drivechain.dev/info) |
| Dev contact | dev@layertwolabs.com, [t.me/DcInsiders](https://t.me/DcInsiders) |

## Docs

1. [Full Node Setup](01-node-setup.md)
2. [Wallets, Keys & Addresses](02-wallets-keys-addresses.md)
3. [Replay & Coin Splitting](03-replay-protection-and-coin-splitting.md) (read first)
4. [Explorers & APIs](04-explorers-and-apis.md)
5. [Sidechains & L2s](05-sidechains-and-l2s.md) (includes proposing new L2s)
6. [Contacts & Resources](06-contacts-and-resources.md)
7. [FAQ](07-faq.md)
8. [Mining Pool Setup](08-mining-pool-setup.md)
9. [Drynet Dry-Run Networks](09-drynets/README.md) (official per-network docs: drynet1/2/3, connect and mine walkthroughs, what changed between runs)

## Integration checklist

1. Run a node against drynet3 now; swap to the launch branch when announced. Dedicated datadir, pinned peers, verify the fork-block hash.
2. Reuse your Bitcoin pipeline (RPC, ZMQ, descriptors, electrs/mempool.space) pointed at the eCash node. Key balances by `(chain, address)`.
3. Fork week: freeze withdrawals at the fork block, split coins (eCash side first, `nLockTime = 499999999`), resume with deep confirmation requirements while difficulty re-equilibrates.
4. Set the magic nLockTime on every ECX withdrawal, permanently.
5. Decide your crediting policy for customer BTC held at the fork block; prepare comms about ECX and the Satoshi-coin reassignment.

## Primary sources

- [drivechain.info/dev.txt](https://drivechain.info/dev.txt), the canonical fast-info file, updated frequently
- [ecash.com](https://ecash.com), official site and FAQ
- [drynet3.drivechain.dev/info](https://drynet3.drivechain.dev/info), live test network hub
- [github.com/ecash-com/bitcoin](https://github.com/ecash-com/bitcoin), [github.com/LayerTwo-Labs](https://github.com/LayerTwo-Labs)
- [BIP300](https://github.com/bitcoin/bips/blob/master/bip-0300.mediawiki), [BIP301](https://github.com/bitcoin/bips/blob/master/bip-0301.mediawiki)

## Support Groups

- [Telegram](https://t.me/eCashHangout)
- [Discord](https://discord.gg/swyE78UPw)
  
Feedback Appreciated!