# FAQ

## The fork itself

**What exactly is eCash (ECX)?**
A hard fork of Bitcoin by LayerTwo Labs, created to activate drivechains (BIP300/301). It is its own L1 blockchain, not a token on another chain. The mainnet fork copies Bitcoin's entire ledger as of block 973,727: every address holding BTC then has an equal ECX balance on the new chain, with no claim or registration. Alphanet and betanet were earlier rehearsals with practice coins ([testnets/](testnets/README.md)); only mainnet ECX is permanent.

**When was the fork?**
At Bitcoin block **973,728**, mined on **2026-10-29 18:59:55 UTC**. The first eCash block's hash is `0000000000000001441a6a4ae49d679d4a30581db3e3eaeeb2fba5da6dc146ed`. Blocks up to 973,727 are shared with Bitcoin.

**Does anything happen to BTC?**
No. BTC is untouched; eCash is a new chain that starts from Bitcoin's ledger.

**What are the "repurposed" (Patoshi) coins?**
The most controversial feature. On the eCash chain, 232 hard-coded transactions spend early Satoshi-era coins without the original keys: script checks for those txids are skipped (`setRepurposeTx` in `src/repo_txns.h`). Only the eCash chain is affected; Satoshi's actual BTC is untouched. Expect user questions and media coverage.

**What is the mining situation?**
Same SHA-256d PoW as Bitcoin. Difficulty reset to ~2.6e9 at the fork block and retargets every 288 blocks, rising at most ~12% per retarget. Blocks were much faster than 10 minutes in the first days: about 10,500 blocks were mined in the first 15 days, and difficulty was ~8.0e10 on 2026-11-13 ([01](01-node-setup.md#difficulty-after-the-fork)). Six pools are in the mainnet registry ([08](08-mining.md)).

**What if BTC activates BIP300/301 itself?**
The team has stated they would abandon the project in that case (unless eCash's market cap already exceeds BTC's).

## Exchange operations

**How much work is the integration?**
The L1 is a Bitcoin Core clone (v32.1 base). RPC, wallet formats, address formats, transaction formats and the indexing stack are identical to Bitcoin. Reuse your BTC pipeline with a separate node and datadir, and set the replay marker on sends. Step-by-step: [09](09-mainnet-wallet-and-exchange-support.md).

**Do we have to credit users ECX?**
The matching coins are controlled by the exchange's keys, not directly by the customer's account. Publish your eligibility, snapshot, distribution and withdrawal policy; customers should not assume a credit until you confirm it. Distribution does not by itself require spot trading: providers may announce ECX distribution, a conditional cash distribution, or other arrangements. See the sourced [markets & exchange-support directory](10-markets-and-exchange-support.md). Applicable obligations depend on your terms and jurisdiction. Regardless of your distribution policy, unsplit coins expose operations to replay risk ([03](03-replay-protection-and-coin-splitting.md)).

**Where can users find markets and exchange announcements?**
Use [Markets & Exchange Support](10-markets-and-exchange-support.md) for SideShift's native ECX and wECX swaps, Orca's wECX pool, the ECX/wECX bridge, and the published CoinW and GMO Coin fork policies. The page identifies the asset, network, source and verification date. Wrapped wECX trading is not native ECX trading, and a distribution announcement is not confirmation of deposits or withdrawals. Self-custody holders should start with the [holder guide](https://ecash.com/how-to-claim/) and [coin-splitting guidance](https://ecash.com/split/).

**What will users ask us?**
ecash.com tells users to ask their exchange three questions ([ecash.com/join](https://ecash.com/join/)). Will you credit ECX for BTC held at the mainnet snapshot? Will ECX withdrawals be supported, and when? Will BTC withdrawals pause around the fork? If you have not published answers, do so now.

**What is the replay risk, concretely?**
A transaction spending pre-fork UTXOs is valid on both networks unless it opts into replay protection (`nLockTime = 499999998` plus a non-final input sequence). If you do nothing, your BTC withdrawals will in most cases also be broadcastable on eCash, sending customers ECX you didn't intend to send. Split your coins before treating the ledgers independently: [03-replay-protection-and-coin-splitting.md](03-replay-protection-and-coin-splitting.md).

**How many confirmations should we require for ECX deposits?**
During the post-fork difficulty re-equilibration, treat it like a young PoW network: significantly deeper confirmations than Bitcoin (dozens, not 3-6) and reorg monitoring. Revisit once hashrate stabilizes. There is no official recommendation as of this writing.

**Can ECX be acquired without an exchange?**
Holding BTC in self-custody at block 973,727 was the main route. Others are mining, non-custodial swap services such as SideShift ([markets](10-markets-and-exchange-support.md)), unwrapping wECX, and cross-chain atomic swaps if the CoinShift sidechain is activated on mainnet. Context for liquidity expectations, not something an exchange must integrate.

**Is there a testnet we can integrate against?**
Use regtest: the mainnet binary's `-regtest` uses the mainnet replay marker. Betanet has a different replay marker, magic and difficulty rule, and its public explorer and Esplora did not respond on 2026-11-13 ([testnets/betanet.md](testnets/betanet.md)). The mainnet endpoints (`*.ecash.ninja`) are live for read-path testing ([04](04-explorers-and-apis.md)).

**Is there a faucet?**
No, on any eCash network. Use regtest with dedicated test keys for signing and replay checks. For betanet, mine to a fresh test wallet or use dedicated betanet test funds. To inspect a pre-fork balance, import a public descriptor into a watch-only wallet, not the private key: that key may still control real BTC or mainnet ECX. See [betanet testing](testnets/betanet.md#test-funds).

**Where do we get the software?**
L1 node: `ecash-v1.2.0-core32.1`, listed in https://releases.ecash.com/L1-ecash-bitcoin/mainnet/latest/release.json (take the file path from it; [01](01-node-setup.md#where-to-get-the-software)), Docker `ghcr.io/ecash-com/bitcoin:latest`, source on `master` of https://github.com/ecash-com/bitcoin. Enforcer v1.0.1 is in https://releases.ecash.com/index.json. BitWindow and the sidechains are at https://releases.drivechain.info/. A node older than v1.1.0 left the chain at block 974,016.

## Wallets & custody

**Do our existing HD wallet and cold-storage procedures work?**
The key and address formats, PSBT flow, descriptor wallets and watch-only support are compatible with Bitcoin. Verify your particular signer, firmware and companion application's network and replay-marker support before using it. Existing BTC keys already control the matching ECX; do not export a hardware-wallet seed to work around missing support. See [02-wallets-keys-addresses.md](02-wallets-keys-addresses.md).

**Should we use the same seed for BTC and ECX?**
Pre-fork keys control both automatically. For post-fork operations, use a separate seed for eCash wallets, so a compromise on one chain doesn't affect the other and accounting stays clean.

**Is there an official end-user wallet?**
Mobile: the eCash wallet (v1.5.1 as of 2026-11-11), which supports mainnet and has a built-in split (Android APK at https://github.com/ecash-com/ecash-wallet-mobile/releases/latest; store links at https://ecash.com/wallets/). Desktop: BitWindow, https://ecash.com/download. ecash.com/wallets also lists the third-party Edge and EcashJr wallets.

## ECX, XEC, wECX and Solana tokens

**Is ECX the same as eCash (XEC)?**
No. XEC (https://e.cash) is an unrelated project that shares the name. CoinGecko's id for ECX is `ecash-com` (still a preview listing with no price as of 2026-11-13); `ecash` is XEC. Get tickers, logos and price feeds right.

**What are wECX, wbECX and the Solana "eCash" token?**
None of them is native ECX:

- **wECX** is a wrapped token on Solana (mint `6ombVv3YMs9qpx1QkycD9XW8HKSfv7vGrCFX3cxCYMVY`), traded on Orca and referenced from https://buy.ecash.com. A custodial bridge at https://wrap.ecash.com converts between wECX and native ECX. It remains a separate asset from native ECX on the eCash chain.
- **wbECX** is the older beta wrapped token. Per the official notice it is no longer redeemable for real ECX (the cut-off was 2026-10-31).
- The **pump.fun "eCash" memecoin** is an independent community token, unrelated to the official wECX mint. The page that described its redemption programme (`ecash.com/sol-token`) no longer exists as of 2026-11-13.

Deposits of any of these are Solana SPL transfers, not ECX transactions.

Verify the mint against the [official token reference](https://ecash.com/wrap_info.txt). Current market links and exchange policies are maintained on [Markets & Exchange Support](10-markets-and-exchange-support.md).

## Sidechains

**Do we need to run sidechain software?**
No. L1-only custody requires only the L1 node. Sidechains matter only for L2 assets or fast-withdrawal services: [05-sidechains-and-l2s.md](05-sidechains-and-l2s.md). Five slots were activated on mainnet by 2026-11-12, but only Thunder and Flush show recorded activity; check [05](05-sidechains-and-l2s.md) before supporting one.

**Are drivechain withdrawals a risk to L1 funds we custody?**
No. BIP300 only governs coins users explicitly deposit into sidechain escrows. Plain L1 UTXOs are ordinary Bitcoin-style UTXOs.

## Not covered here?

- Official FAQs: https://ecash.com/faq/ and https://www.drivechain.info/faq/index.html
- Developer contact: dev@layertwolabs.com, Telegram https://t.me/DcInsiders
