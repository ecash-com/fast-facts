# FAQ

## The fork itself

**What exactly is eCash (ECX)?**
A hard fork of Bitcoin by LayerTwo Labs, created to activate drivechains (BIP300/301). It is its own L1 blockchain, not a token on another chain. The mainnet fork copies Bitcoin's entire ledger as of block 973,727: every address holding BTC then has an equal ECX balance on the new chain, with no claim or registration. Alphanet and betanet were earlier rehearsals with practice coins ([testnets/](testnets/README.md)); only mainnet ECX is permanent.

**When is the fork?**
At Bitcoin block **973,728**, Bitcoin's next difficulty adjustment. It is a height, not a date. As of 2026-10-27 the estimate is about 2026-10-29 15:00 UTC; the announced target was 2026-10-31. Live ETA: https://mempool.space/api/v1/difficulty-adjustment.

**Does anything happen to BTC?**
No. BTC is untouched; eCash is a new chain that starts from Bitcoin's ledger.

**What are the "repurposed" (Patoshi) coins?**
The most controversial feature. On the eCash chain, 232 hard-coded transactions spend early Satoshi-era coins without the original keys: script checks for those txids are skipped (`setRepurposeTx` in `src/repo_txns.h`). Only the eCash chain is affected; Satoshi's actual BTC is untouched. Expect user questions and media coverage.

**What is the mining situation at launch?**
Same SHA-256d PoW as Bitcoin, but difficulty resets to ~2.6e9 at the fork block, so ASIC hashrate is needed from the first block. After that, difficulty retargets every 288 blocks and can rise at most ~12% per retarget, so blocks may be much faster than 10 minutes until it catches up with the hashrate ([01](01-node-setup.md#difficulty-after-the-fork)). No public mainnet pool has been announced yet ([08](08-mining.md)).

**What if BTC activates BIP300/301 itself?**
The team has stated they would abandon the project in that case (unless eCash's market cap already exceeds BTC's).

## Exchange operations

**How much work is the integration?**
The L1 is a Bitcoin Core clone (v31.1 base). RPC, wallet formats, address formats, transaction formats and the indexing stack are identical to Bitcoin. Reuse your BTC pipeline with a separate node and datadir, and set the replay marker on sends. Step-by-step: [09](09-mainnet-wallet-and-exchange-support.md).

**Do we have to credit users ECX?**
Users holding BTC on your exchange at the fork block implicitly own the corresponding ECX. The project's position is that crediting users more or less requires listing, and that not splitting your coins risks accidentally sending ECX along with BTC withdrawals. ecash.com runs a public [exchange listing campaign](https://ecash.com/list-ecx/) and cites the 2017 Bitcoin Cash precedent: some exchanges credited within hours, others reversed course months later. Legal and tax obligations vary by jurisdiction, but the operational risk is real either way.

**What will users ask us?**
ecash.com tells users to ask their exchange three questions ([ecash.com/join](https://ecash.com/join/)). Will you credit ECX for BTC held at the mainnet snapshot? Will ECX withdrawals be supported, and when? Will BTC withdrawals pause around the fork? Publish answers before the fork.

**What is the replay risk, concretely?**
A transaction spending pre-fork UTXOs is valid on both networks unless it opts into replay protection (`nLockTime = 499999998` plus a non-final input sequence). If you do nothing, your BTC withdrawals will in most cases also be broadcastable on eCash, sending customers ECX you didn't intend to send. Split your coins before treating the ledgers independently: [03-replay-protection-and-coin-splitting.md](03-replay-protection-and-coin-splitting.md).

**How many confirmations should we require for ECX deposits?**
During the post-fork difficulty re-equilibration, treat it like a young PoW network: significantly deeper confirmations than Bitcoin (dozens, not 3-6) and reorg monitoring. Revisit once hashrate stabilizes. There is no official recommendation as of this writing.

**Can ECX be acquired without an exchange?**
Holding BTC in self-custody at block 973,727 is the main route. Others are cross-chain atomic swaps once the CoinShift sidechain is live on mainnet, and community reward programmes. Context for liquidity expectations, not something an exchange must integrate.

**Is there a testnet we can integrate against?**
Betanet is still live and has all the launch mechanics, but with a different replay marker, magic and difficulty rule ([testnets/betanet.md](testnets/betanet.md)). The mainnet binary's `-regtest` uses the mainnet replay marker. The mainnet endpoints (`*.ecash.ninja`) are already up and serve the shared chain until the fork ([04](04-explorers-and-apis.md)).

**Is there a faucet?**
No, on any eCash network. To test with real-shaped funds on betanet, import a key that held BTC at block 967,679, or mine.

**Where do we get launch-day software?**
L1 node: `ecash-v1.1.0-core31.1` at https://releases.ecash.com/L1-ecash-bitcoin/mainnet/latest/release.json, Docker `ghcr.io/ecash-com/bitcoin:latest`, source on `master` of https://github.com/ecash-com/bitcoin. Everything else (BitWindow, sidechains, enforcer) is at https://releases.drivechain.info/. Upgrade any v1.0.0 node before block 974,016.

## Wallets & custody

**Do our existing HD wallet and cold-storage procedures work?**
Yes. Same secp256k1, BIP32/39/44/49/84/86, address formats, PSBT flow, descriptor wallets and watch-only support. Existing BTC keys already control the matching ECX. See [02-wallets-keys-addresses.md](02-wallets-keys-addresses.md).

**Should we use the same seed for BTC and ECX?**
Pre-fork keys control both automatically. For post-fork operations, use a separate seed for eCash wallets, so a compromise on one chain doesn't affect the other and accounting stays clean.

**Is there an official end-user wallet?**
Mobile: the eCash wallet, v1.4.6 or later, which supports mainnet and has a built-in split (Android APK at https://github.com/ecash-com/ecash-wallet-mobile/releases/latest, iOS via https://testflight.apple.com/join/PxZjPxrK). Desktop: BitWindow, https://ecash.com/download.

## ECX, XEC, wECX and Solana tokens

**Is ECX the same as eCash (XEC)?**
No. XEC (https://e.cash) is an unrelated project that shares the name. CoinGecko's id for ECX is `ecash-com` (a preview listing with no price as of 2026-10-27); `ecash` is XEC. Get tickers, logos and price feeds right.

**What are wECX, wbECX and the Solana "eCash" token?**
None of them is native ECX:
- **wECX** is a wrapped token on Solana (mint `6ombVv3YMs9qpx1QkycD9XW8HKSfv7vGrCFX3cxCYMVY`), traded on Orca and referenced from https://buy.ecash.com. It is a pre-trading market, separate from native ECX on the eCash chain.
- **wbECX** is the older beta wrapped token. It is not redeemable for real ECX after 2026-10-31.
- The **pump.fun "eCash" memecoin** is an independent community token, with its own limited top-15-holder redemption programme ([ecash.com/sol-token](https://ecash.com/sol-token/)).

Deposits of any of these are Solana SPL transfers, not ECX transactions.

## Sidechains

**Do we need to run sidechain software?**
No. L1-only custody requires only the L1 node. Sidechains matter only for L2 assets or fast-withdrawal services: [05-sidechains-and-l2s.md](05-sidechains-and-l2s.md). No sidechain is active at the mainnet fork; each must be activated by miners first.

**Are drivechain withdrawals a risk to L1 funds we custody?**
No. BIP300 only governs coins users explicitly deposit into sidechain escrows. Plain L1 UTXOs are ordinary Bitcoin-style UTXOs.

## Not covered here?

- Official FAQs: https://ecash.com/faq/ and https://www.drivechain.info/faq/index.html
- Developer contact: dev@layertwolabs.com, Telegram https://t.me/DcInsiders
