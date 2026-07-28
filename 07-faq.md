# FAQ

## The fork itself

**What exactly is eCash (ECX)?**
A hard fork of Bitcoin activating at BTC block ~963,648 (on or around August 22, 2026, 15:00 UTC target). It copies Bitcoin's entire ledger: every address holding BTC at the fork block is credited an equal ECX balance, with no claim or registration. The fork's purpose is to activate drivechains (BIP300/301). It is its own L1 blockchain, not a token on another chain.

**Does anything happen to BTC?**
No. BTC is untouched; eCash is a new chain that starts from Bitcoin's ledger.

**What are the "repurposed" (Patoshi) coins?**
The most controversial feature: on the eCash chain, a hard-coded list of early Satoshi-era coinbase transactions is made spendable without the original keys (script checks for those txids are skipped, via `setRepurposeTx` in `src/repo_txns.h`; drynet3 rehearses this with 122 txids). Only the eCash chain is affected; Satoshi's actual BTC is untouched. Expect user questions and media coverage.

**What is the mining situation at launch?**
Same SHA-256d PoW as Bitcoin, but difficulty resets to minimum at the fork block. For the first ~2,016 blocks the chain is CPU-mineable and blocks arrive erratically; difficulty then re-equilibrates and blocks settle to roughly 10 minutes.

**What if BTC activates BIP300/301 itself?**
The team has stated they would abandon the project in that case (unless eCash's market cap already exceeds BTC's).

## Exchange operations

**How much work is the integration?**
The L1 is a Bitcoin Core clone (v31.1 base). RPC, wallet formats, address formats, transaction formats, and the indexing stack are identical to Bitcoin; reuse your BTC pipeline with a separate node and datadir.

**Do we have to credit users ECX?**
Users holding BTC on your exchange at the fork block implicitly own the corresponding ECX. The project's position: crediting users more or less requires listing, and not splitting your coins risks accidentally sending ECX along with BTC withdrawals. Legal obligations vary by jurisdiction, but the operational risk is real either way.

**What is the replay risk, concretely?**
A transaction spending pre-fork UTXOs is valid on both networks unless it opts into replay protection. If you do nothing, your BTC withdrawals will in most cases also be broadcastable on eCash, sending customers ECX you didn't intend to send. Split your coins before treating the ledgers independently: [03-replay-protection-and-coin-splitting.md](03-replay-protection-and-coin-splitting.md).

**How many confirmations should we require for ECX deposits?**
During the post-fork difficulty re-equilibration, treat it like a young PoW network: significantly deeper confirmations than Bitcoin (dozens, not 3-6) and reorg monitoring. Revisit once hashrate stabilizes. There is no official recommendation as of this writing.

**Can ECX be acquired without an exchange?**
Yes, via cross-chain atomic swaps (the CoinShift sidechain) between ECX and BTC, LTC, XMR, USDT. Context for liquidity expectations, not something an exchange must integrate.

**Is there a testnet we can integrate against today?**
Yes: **drynet3**, a live full-scale dry run (fork of Bitcoin mainnet at block 957,600 with the launch mechanics: difficulty reset, drivechains, replay protection, coin repurposing). Public node, explorer, Esplora API, and Electrum server are up; see [01-node-setup.md](01-node-setup.md). Drynets get replaced as launch approaches (drynet1 is already retired); the official per-network docs are in [09-drynets](09-drynets/README.md).

**Where do we get launch-day software?**
Binaries at https://releases.drivechain.info/ (hashes in `hashes.json`), source at https://github.com/ecash-com/bitcoin. The production branch/tag will be announced via https://drivechain.info/dev.txt, https://ecash.com, and the DcInsiders Telegram.

## Wallets & custody

**Do our existing HD wallet and cold-storage procedures work?**
Yes. Same secp256k1, BIP32/39/44/49/84/86, address formats, PSBT flow, descriptor wallets, and watch-only support. Existing BTC keys already control the matching ECX. See [02-wallets-keys-addresses.md](02-wallets-keys-addresses.md).

**Should we use the same seed for BTC and ECX?**
Pre-fork keys control both automatically. For post-fork operations, use a separate seed for eCash wallets so a compromise on one chain doesn't affect the other, and accounting stays clean.

**Is there an official end-user wallet?**
BitWindow (desktop GUI, also the activation client): https://layertwolabs.com/download. Mobile: https://github.com/ecash-com/ecash-wallet-mobile (iOS TestFlight: https://testflight.apple.com/join/KfTBarUr).

## Sidechains

**Do we need to run sidechain software?**
No. L1-only custody requires only the L1 node. Sidechains matter only for L2 assets or fast-withdrawal services: [05-sidechains-and-l2s.md](05-sidechains-and-l2s.md).

**Are drivechain withdrawals a risk to L1 funds we custody?**
No. BIP300 only governs coins users explicitly deposit into sidechain escrows. Plain L1 UTXOs are ordinary Bitcoin-style UTXOs.

## Not covered here?

- Fast-info file (always current): https://drivechain.info/dev.txt
- Official FAQs: https://ecash.com and https://www.drivechain.info/faq/index.html
- Developer contact: dev@layertwolabs.com, Telegram https://t.me/DcInsiders
