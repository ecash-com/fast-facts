# Contacts & Resources

eCash L1 and the launch sidechains are developed by **LayerTwo Labs** (founded by Paul Sztorc, author of BIP300/301).

## Development team

| Contact | Address / Handle |
|---|---|
| Developer email (integration questions) | dev@layertwolabs.com |
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

As of 2026-07-27, no eCash/LayerTwo Labs PGP release-signing key has been published (no `.asc` signatures on the binary server, none in `dev.txt`). Binary integrity is verified via SHA-256 hashes in `hashes.json` on the release server. If you need signed binaries or a PGP channel for sensitive disclosure, request one via dev@layertwolabs.com, and re-check `https://drivechain.info/dev.txt` for newly published keys.

## Official sites

| Site | Purpose |
|---|---|
| https://ecash.com | Official eCash site (fork countdown, FAQ, downloads) |
| https://layertwolabs.com | Company site; `/download` for BitWindow installers |
| https://drivechain.info | Drivechain reference (literature, FAQ, misinformation rebuttals) |
| https://drivechain.info/dev.txt | Canonical fast-info file, updated by Paul directly. Check this first |
| https://www.truthcoin.info/blog/drivechain/ | Original Drivechain design writing |
| https://bip300cusf.com/ | BIP300 CUSF (activation client) info |
| http://bip300.xyz | Command-line install script |

## Source code (GitHub)

| Repo | What it is |
|---|---|
| https://github.com/ecash-com/bitcoin | eCash L1 full node (Bitcoin Core fork). Branches: `drynet1`/`drynet2`/`drynet3` (dry-run networks), `master` (tracks upstream) |
| https://github.com/LayerTwo-Labs | Main org: enforcer, sidechains, frontends |
| https://github.com/LayerTwo-Labs/bip300301_enforcer | BIP300/301 validator/wallet daemon (gRPC) |
| https://github.com/LayerTwo-Labs/drivechain-frontends | BitWindow (GUI wallet / activation client) |
| https://github.com/ecash-com/ecash-wallet-mobile | Official mobile wallet (iOS TestFlight: https://testflight.apple.com/join/KfTBarUr) |
| https://github.com/LayerTwo-Labs/simplepool | Stratum mining pool implementation (see [08-mining-pool-setup.md](08-mining-pool-setup.md)) |

Sidechain repos are listed in [05-sidechains-and-l2s.md](05-sidechains-and-l2s.md).

## Binaries

| Resource | URL |
|---|---|
| Binary server (all platforms) | https://releases.drivechain.info/ |
| SHA-256 hashes | https://releases.drivechain.info/hashes.json |
| Version manifest | https://releases.drivechain.info/versions.json |
| Download page | https://layertwolabs.com/download |
| BitWindow one-line installer | `curl -fsSL https://raw.githubusercontent.com/LayerTwo-Labs/drivechain-frontends/refs/heads/master/install/install-bitwindow.sh \| bash` |
| Docker (L1 node, drynet3) | `ghcr.io/ecash-com/bitcoin:drynet3` |

Dry-run L1 binaries follow the pattern `L1-ecash-bitcoin-drynet3-<target>.zip` (macOS Apple Silicon and Intel, Linux x86_64, Windows x86_64).

## Community channels

| Channel | Link |
|---|---|
| Telegram, DcInsiders (main dev/insider group) | https://t.me/DcInsiders |
| Telegram, eCash Hangout | https://t.me/eCashHangout |
| Telegram, eCash official announcements | https://t.me/ecashcom_official |
| Discord | https://discord.gg/swyE78UPw |
| Reddit | r/Drivechains |
| YouTube | https://youtube.com/@LayerTwoLabs |
| Community blog (not an L2L property) | https://ecashblog.com/ |
| News aggregator | https://news.ecash.com |
| Bug bounty / hackathon | https://drivechain.info/blog/bug-contest/ |

No official Signal group has been published; Telegram is the primary real-time channel, with Discord as a secondary option.
