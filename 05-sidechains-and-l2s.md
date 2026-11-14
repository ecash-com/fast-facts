# Sidechains (L2s)

Exchanges only strictly need the L1: ECX deposits and withdrawals work exactly like BTC. The L2 layer matters because coins move between L1 and L2s, and some products (fast withdrawals, the zSide privacy chain) may be relevant to your users.

**Where the rules live:** the `ecash-com/bitcoin` node contains only minimal drivechain plumbing (`OP_DRIVECHAIN` = `OP_NOP8` made standard, the `getblocktemplate` `bip300301` rule, the fork-height difficulty reset). The BIP300/301 state machine (sidechain proposals, deposit escrows, withdrawal-bundle ACK counting, blind merged mining) is validated by the companion daemon [`bip300301_enforcer`](https://github.com/LayerTwo-Labs/bip300301_enforcer), which runs alongside the node. Plain L1 nodes without the enforcer still follow the chain.

## Mainnet sidechain status

Sidechain slots did not carry over from betanet: at block 973,728 every slot was empty, and each sidechain becomes usable only after miners propose and activate it on mainnet ([below](#proposing-a-new-sidechain)). As reported on [ecash.com/sidechains](https://ecash.com/sidechains/) (observation of 2026-11-12, ECX block 983,381):

| Slot | Sidechain | Mainnet status |
|---|---|---|
| 9 | Thunder | activated at block 978,324; block commitments recorded on L1 |
| 46 | Flush | activated at block 978,331; block commitments recorded on L1 (community project) |
| 24 | Elements+ | activated at block 978,708; no commitments recorded, availability unconfirmed |
| 130 | FreeBank | activated at block 978,341; no commitments recorded, release described as dormant |
| 131 | Tally | activated at block 978,342; specification only, no software. Do not deposit |
| 8, 47, 88, 98 | Solana, Shuttle, Sidepot, zSide | proposed, awaiting miner ACKs |

An activated slot is not a usable sidechain: activation creates the L1 escrow, but running software and block production are separate. Recheck the live page before enabling anything. ecash.com takes these figures from a third-party explorer, `https://explorer.blockfab.org/api/v1/drivechain/summary`, which on 2026-11-13 (block 984,270) showed the same five slots, about 11.56 ECX deposited into Thunder and 10.41 ECX into Flush, and no completed withdrawals yet. The `ecash` entry in [drivechain.dev/config](https://drivechain.dev/config) still lists no sidechains, so wallets that read it show none.

## The seven LayerTwo Labs sidechains

Each is its own chain with its own node software, secured by eCash miners via BIP300 (no federation, no multisig custodians). The "Software status" column says whether the software ran on the test networks; mainnet status is in the table above. As of 2026-11-12 only Thunder is activated on mainnet, zSide is proposed, and the other five are not activated:

| Slot | Sidechain | Purpose | Software status | Repo |
|---|---|---|---|---|
| 2 | BitNames | Identity / DNS | live | https://github.com/LayerTwo-Labs/plain-bitnames |
| 4 | BitAssets | Tokens / NFTs | live | https://github.com/LayerTwo-Labs/plain-bitassets |
| 9 | Thunder | Large-block payments | live | https://github.com/LayerTwo-Labs/thunder (Rust node: `thunder-rust`) |
| 13 | Truthcoin | Prediction markets | live | https://github.com/LayerTwo-Labs/truthcoin-dc |
| 98 | zSide | Privacy (Zcash-style) | live | https://github.com/LayerTwo-Labs/zebra (also `iwakura-rein/thunder-orchard`) |
| 99 | Photon | Quantum resistance | in progress | https://github.com/LayerTwo-Labs/photon |
| 255 | CoinShift | Cross-coin atomic swaps | in progress | https://github.com/LayerTwo-Labs/coinshift-rs |

The slot numbers are the ones these sidechains use on the test networks; on mainnet a slot is whatever the M1 proposal claims. Default ports follow the pattern `40<slot>` (P2P), `60<slot>` (RPC), `280<slot>` (ZMQ); for example, Thunder (slot 9) uses P2P 4009 and RPC 6009. Full table: https://drivechain.info/dev.txt

Photon's existence does not make L1 ECX transactions quantum resistant: the base chain uses Bitcoin's keys and signatures.

**Community sidechains:** slots are open to anyone, and four of the five activated mainnet slots are community projects (Flush, Elements+, FreeBank, Tally). FreeBank (https://ecxfreebank.com) holds slot 130 on betanet and mainnet and has a solo-mining entry in the mainnet pool registry ([08](08-mining.md)).

## How coins move between L1 and L2

- **Deposit (L1 to L2):** an M5 deposit transaction sends ECX into the sidechain's BIP300 escrow UTXO. Wallet software (BitWindow, sidechain wallets, the mobile wallet) handles this.
- **Withdrawal (L2 to L1):** withdrawals are batched into **bundles** (up to 6,000 per bundle). On eCash mainnet a bundle needs **more than 6,048 miner ACKs** within a **12,096-block window** to pay out on L1 (the enforcer's `ECX_MAINNET` thresholds; BIP300 as written uses 13,150 of 26,300). That is at least 6 weeks and at most 12 weeks at 10-minute blocks. The slow path is the security model.
- **Fast withdrawals:** a service that atomically swaps L2 coins for L1 coins immediately (a third party fronts the L1 coins and collects the bundle payout later). Convenience service, not consensus-critical. None is listed for mainnet in drivechain.dev/config as of 2026-11-13.
- The escrow uses no cryptographic signatures (a hashrate escrow: an anyone-can-spend UTXO protected by consensus rules), which is why the team describes the peg as quantum-proof.

All BIP300 windows are counted in **blocks**. With the post-fork difficulty algorithm, blocks have come much faster than every 10 minutes at times ([01](01-node-setup.md#difficulty-after-the-fork)), so windows can take far less wall-clock time than these estimates.

## Proposing a new sidechain

Anyone can propose one; miners decide activation (BIP300 M1/M2):

1. **Write the sidechain software.** The practical path is forking an existing L2 (Thunder is the usual template) and changing its slot number and rules. The LayerTwo Labs sidechains build on the CUSF sidechain protocol (`cusf_sidechain_proto`).
2. **M1, propose:** a miner includes an M1 "Propose Sidechain" message in a coinbase (slot number, title, description, hash of the sidechain software/genesis). Submit via the enforcer:
   ```
   cusf.mainchain.v1.BlockProducerService/CreateSidechainProposal   (bip300301_enforcer gRPC)
   ```
   BitWindow also exposes this in its GUI.
3. **M2, ACK:** miners ACK the proposal in subsequent coinbases.
   - Unused slot, mainnet: needs **more than 1,815 ACKs within 2,016 blocks** (ecash.com shows it as 1,816 approvals), i.e. sustained ~90% hashrate approval for one 2,016-block period. (Betanet trialled a lower 1,008 threshold.)
   - Overwriting a used slot: requires the longer withdrawal-bundle threshold, more than 6,048 ACKs within 12,096 blocks on eCash mainnet.
4. Once active, the sidechain's escrow exists at L1 and deposits can begin.

To socialize a proposal, post in the DcInsiders Telegram ([06](06-contacts-and-resources.md)).

Specs: [BIP300](https://github.com/bitcoin/bips/blob/master/bip-0300.mediawiki), [BIP301](https://github.com/bitcoin/bips/blob/master/bip-0301.mediawiki).

## BIP301: blind merged mining

Sidechain block producers pay L1 miners (in ECX) to commit to sidechain block hashes in L1 coinbases, so miners earn sidechain revenue without running sidechain nodes. No exchange action needed.

## The enforcer

[`bip300301_enforcer`](https://github.com/LayerTwo-Labs/bip300301_enforcer) validates BIP300/301 messages alongside the L1 node. It provides the gRPC validator / block producer / wallet API used by BitWindow and the sidechains (gRPC 50051; `getblocktemplate` on 8122 for miners). Run it only if you interact with sidechains or mine. Plain L1 custody does not require it.

**Mainnet status (2026-11-13):** mainnet support has shipped. [PR #666](https://github.com/LayerTwo-Labs/bip300301_enforcer/pull/666) was merged on 2026-10-27 and released in v1.0.0; the current release is **v1.0.1** (2026-10-29, `bip300301-enforcer/stable` on [releases.ecash.com](https://releases.ecash.com/index.json)). It replaced `--network-preset` with `--fork bitcoin|ecash`, detects eCash from the node's user agent, and applies activation height 973,728, `OP_NOP8`, the eCash mainnet thresholds (unused slot: more than 1,815 of 2,016; withdrawal bundle and used slot: more than 6,048 of 12,096) and the replay `nLockTime` on its wallet sends. It supports Bitcoin Core majors 30 to 32, so it runs against node v1.2.0. **Do not run a pre-1.0 enforcer on mainnet**: without eCash parameters it falls back to `OP_NOP5` and activation height 0.
