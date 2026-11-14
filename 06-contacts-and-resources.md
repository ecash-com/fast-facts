# Contacts & Resources

eCash L1 and the launch sidechains are developed by **LayerTwo Labs** (founded by Paul Sztorc, author of BIP300/301). ecash.com is operated by Blockchain Emulation Services Ltd.

## Development team

| Contact | Address / Handle |
|---|---|
| Developer email (integration questions) | dev@layertwolabs.com |
| eCash.com enquiries | admin@ecash.com ([ecash.com/contact](https://ecash.com/contact/)) |
| Contact form | https://layertwolabs.com/contact |
| Paul Sztorc, Telegram (direct) | https://t.me/psztorc |
| Paul Sztorc, X/Twitter | https://x.com/Truthcoin |
| LayerTwo Labs, X/Twitter | https://x.com/layertwolabs |

### Emergency / incident contact

For time-sensitive issues (chain split, consensus bug, security incident):

1. Email **dev@layertwolabs.com** with `[URGENT]` in the subject.
2. Post in the **DcInsiders** Telegram group (developers are active daily): https://t.me/DcInsiders
3. DM Paul Sztorc on Telegram: https://t.me/psztorc

> The `SECURITY.md` in `ecash-com/bitcoin` is inherited from upstream and points to `security@bitcoincore.org`. Do not report eCash issues there; report to LayerTwo Labs via the contacts above.

### PGP keys

As of 2026-09-16, no eCash/LayerTwo Labs PGP release-signing key has been published. L1 node binaries on releases.ecash.com carry GitHub build-provenance attestations plus SHA-256 hashes in `index.json` / `release.json`. If you need a PGP channel for sensitive disclosure, request one via dev@layertwolabs.com.

## Official sites

For holders: [accessing ECX](https://ecash.com/how-to-claim/), [coin splitting](https://ecash.com/split/), and the [markets & exchange-support directory](10-markets-and-exchange-support.md). The directory separates wrapped-token markets from exchange distribution policies.

| Site | Purpose |
|---|---|
| https://ecash.com | Official eCash site: fork height, [coin splitting](https://ecash.com/split/), [FAQ](https://ecash.com/faq/), [what to expect](https://ecash.com/what-to-expect/), [downloads](https://ecash.com/download/), [buy / swap page](https://ecash.com/buy/) (`/list-ecx/` now shows the same page), [wallets](https://ecash.com/wallets/), [sidechains](https://ecash.com/sidechains/) |
| https://news.ecash.com | Project news and launch updates |
| https://dashboard.ecash.com | Market and network dashboard (`?network=mainnet`) |
| https://archive.ecash.com | Archive |
| https://buy.ecash.com | Swap page: native ECX via SideShift, wECX (wrapped, Solana) via Orca. wECX is **not native ECX** ([FAQ](07-faq.md#ecx-xec-wecx-and-solana-tokens)) |
| https://wrap.ecash.com | Custodial ECX ↔ wECX bridge ([markets](10-markets-and-exchange-support.md)) |
| https://ecash.com/wallets/ | Wallet directory (eCash Wallet, Edge, EcashJr, BitWindow) |
| https://ecash.com/sidechains/ | Live mainnet sidechain status ([05](05-sidechains-and-l2s.md)) |
| https://explorer.ecash.ninja | Mainnet explorer ([04](04-explorers-and-apis.md)); mirrored at https://explorer.ecash.com |
| https://drivechain.dev/config | Machine-readable network registry (endpoints, seeds, snapshot) |
| https://layertwolabs.com | Company site; `/download` for BitWindow installers |
| https://drivechain.info | Drivechain reference (literature, FAQ, misinformation rebuttals) |
| https://www.truthcoin.info/blog/drivechain/ | Original Drivechain design writing |
| https://bip300cusf.com/ | BIP300 CUSF (activation client) info |
| http://bip300.xyz | Command-line install script |

## Source code (GitHub)

| Repo | What it is |
|---|---|
| https://github.com/ecash-com/bitcoin | eCash L1 full node (Bitcoin Core fork). **`master`** = mainnet (tags `ecash-v1.2.0-core32.1` current, `ecash-v1.1.0-core31.1` minimum, `ecash-v1.0.0-core31.1` off the chain since block 974,016). Also `betanet`, `alphanet`, `drynet1`-`drynet4` (test networks, [testnets/](testnets/README.md)) and `*-bridge` branches (BTC-to-ECX P2P relay, including `mainnet-bridge`) |
| https://github.com/ecash-com/mempool | eCash fork of the mempool.space explorer |
| https://github.com/ecash-com/ecash-wallet-mobile | Official mobile wallet, v1.5.1 as of 2026-11-11 (Android APK in releases; store links on https://ecash.com/wallets/) |
| https://github.com/ecash-com/fast-facts | This guide |
| https://github.com/LayerTwo-Labs | Main org: enforcer, sidechains, frontends |
| https://github.com/LayerTwo-Labs/bip300301_enforcer | BIP300/301 validator/wallet daemon (gRPC). Mainnet support since v1.0.0; current v1.0.1 ([08](08-mining.md#solo-mining)) |
| https://github.com/LayerTwo-Labs/drivechain-frontends | BitWindow (GUI wallet / activation client) and the sidechain orchestrator |
| https://github.com/LayerTwo-Labs/simplepool | Stratum mining pool implementation (see [08-mining.md](08-mining.md)) |
| https://github.com/LayerTwo-Labs/mining-pools | Pool registry behind pool.drivechain.info: per-network `networks/<net>/pools.json`, with stage lifecycle dates in `networks.json` |

Sidechain repos are listed in [05-sidechains-and-l2s.md](05-sidechains-and-l2s.md).

## Binaries

| Resource | URL |
|---|---|
| L1 node, mainnet | https://releases.ecash.com/L1-ecash-bitcoin/mainnet/latest/release.json (tag `ecash-v1.2.0-core32.1`, commit `c8474e0b4f2b`; files under `.../mainnet/1.2.0-core32.1/`) |
| L1 node, all networks | https://releases.ecash.com/index.json |
| UTXO snapshot | https://releases.ecash.com/assumeutxo/mainnet/utxo-965000.dat (+ `SHA256SUMS`) |
| Docker (L1 node) | `ghcr.io/ecash-com/bitcoin:latest` (= `:sha-c8474e0`, mainnet v1.2.0) |
| Enforcer | `bip300301-enforcer/stable` in https://releases.ecash.com/index.json (v1.0.1) |
| Docker (enforcer) | `ghcr.io/layertwo-labs/bip300301_enforcer` (use a v1.0.0-or-later build for mainnet) |
| Mobile wallet | Android APK: https://github.com/ecash-com/ecash-wallet-mobile/releases/latest; Google Play and App Store links: https://ecash.com/wallets/ |
| BitWindow | macOS https://releases.drivechain.info/BitWindow-latest-x86_64-apple-darwin.dmg, Windows https://releases.drivechain.info/BitWindow-latest-x86_64-windows.exe, Linux `curl -sSL https://raw.githubusercontent.com/LayerTwo-Labs/drivechain-frontends/refs/heads/master/install/install-bitwindow.sh \| bash` |
| Everything else (sidechains, enforcer) | https://releases.drivechain.info/ (version manifest `versions.json`, hashes `hashes.json`) |

## Community channels

| Channel | Link |
|---|---|
| X, official eCash account | https://x.com/BTCdrivechains |
| Telegram, eCash official announcements | https://t.me/ecashcom_official |
| Telegram, DcInsiders (main dev/insider group) | https://t.me/DcInsiders |
| Telegram, eCash Hangout | https://t.me/eCashHangout |
| YouTube | https://youtube.com/@ecashdotcom |
| Reddit | r/Drivechains |
| Community blog (not an L2L property) | https://ecashblog.com/ |
| Bug bounty / hackathon | rules at https://drivechain.info/blog/bug-contest/ (`ecash.com/bounty` now serves the downloads page) |

No official Signal group has been published. Telegram is the primary real-time channel. The Discord invite previously listed here has expired, and [ecash.com/contact](https://ecash.com/contact/) lists no Discord. Announcements (node, pool and enforcer releases) come via https://ecash.com, https://t.me/ecashcom_official, https://t.me/DcInsiders and this repo.
