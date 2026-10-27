# Sidechains (L2s)

Exchanges only strictly need the L1: ECX deposits and withdrawals work exactly like BTC. The L2 layer matters because coins move between L1 and L2s, and some products (fast withdrawals, the zSide privacy chain) may be relevant to your users.

**Where the rules live:** the `ecash-com/bitcoin` node contains only minimal drivechain plumbing (`OP_DRIVECHAIN` = `OP_NOP8` made standard, the `getblocktemplate` `bip300301` rule, the fork-height difficulty reset). The BIP300/301 state machine (sidechain proposals, deposit escrows, withdrawal-bundle ACK counting, blind merged mining) is validated by the companion daemon [`bip300301_enforcer`](https://github.com/LayerTwo-Labs/bip300301_enforcer), which runs alongside the node. Plain L1 nodes without the enforcer still follow the chain.

## Mainnet starts with no active sidechains

Sidechain slots do not carry over from betanet. At block 973,728 every slot is empty (the mainnet entry in [drivechain.dev/config](https://drivechain.dev/config) lists no sidechains). Each sidechain, including the LayerTwo Labs ones, becomes usable only after miners propose and activate it on mainnet ([below](#proposing-a-new-sidechain)). Until then, no deposit escrow exists and there is nothing to deposit into.

## The seven LayerTwo Labs sidechains

Each is its own chain with its own node software, secured by eCash miners via BIP300 (no federation, no multisig custodians). Status as listed on [ecash.com/sidechains](https://ecash.com/sidechains/) ("live" here means the software runs on the test networks; none is active on mainnet yet):

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

**Community sidechains:** slots are open to anyone. FreeBank (https://ecxfreebank.com) activated in slot 130 on betanet and has a solo-mining entry in the mainnet pool registry ([08](08-mining.md)).

## How coins move between L1 and L2

- **Deposit (L1 to L2):** an M5 deposit transaction sends ECX into the sidechain's BIP300 escrow UTXO. Wallet software (BitWindow, sidechain wallets, the mobile wallet) handles this.
- **Withdrawal (L2 to L1):** withdrawals are batched into **bundles** (up to 6,000 per bundle). A bundle needs a work score of **13,150 miner ACKs** within a **26,300-block window** to pay out on L1, roughly 3-6 months at 10-minute blocks. The slow path is the security model.
- **Fast withdrawals:** a service that atomically swaps L2 coins for L1 coins immediately (a third party fronts the L1 coins and collects the bundle payout later). Convenience service, not consensus-critical. None has been published for mainnet yet.
- The escrow uses no cryptographic signatures (a hashrate escrow: an anyone-can-spend UTXO protected by consensus rules), which is why the team describes the peg as quantum-proof.

All BIP300 windows are counted in **blocks**. With the post-fork difficulty algorithm, early blocks may come much faster than every 10 minutes ([01](01-node-setup.md#difficulty-after-the-fork)), so the first windows may take less wall-clock time than these estimates.

## Proposing a new sidechain

Anyone can propose one; miners decide activation (BIP300 M1/M2):

1. **Write the sidechain software.** The practical path is forking an existing L2 (Thunder is the usual template) and changing its slot number and rules. The LayerTwo Labs sidechains build on the CUSF sidechain protocol (`cusf_sidechain_proto`).
2. **M1, propose:** a miner includes an M1 "Propose Sidechain" message in a coinbase (slot number, title, description, hash of the sidechain software/genesis). Submit via the enforcer:
   ```
   cusf.mainchain.v1.BlockProducerService/CreateSidechainProposal   (bip300301_enforcer gRPC)
   ```
   BitWindow also exposes this in its GUI.
3. **M2, ACK:** miners ACK the proposal in subsequent coinbases.
   - Unused slot, mainnet: needs **more than 1,815 ACKs within 2,016 blocks**, i.e. sustained ~90% hashrate approval for one 2,016-block period. (Betanet trialled a lower 1,008 threshold.)
   - Overwriting a used slot: requires the longer 26,300-block / 13,150-ACK threshold.
4. Once active, the sidechain's escrow exists at L1 and deposits can begin.

To socialize a proposal, post in the DcInsiders Telegram ([06](06-contacts-and-resources.md)).

Specs: [BIP300](https://github.com/bitcoin/bips/blob/master/bip-0300.mediawiki), [BIP301](https://github.com/bitcoin/bips/blob/master/bip-0301.mediawiki).

## BIP301: blind merged mining

Sidechain block producers pay L1 miners (in ECX) to commit to sidechain block hashes in L1 coinbases, so miners earn sidechain revenue without running sidechain nodes. No exchange action needed.

## The enforcer

[`bip300301_enforcer`](https://github.com/LayerTwo-Labs/bip300301_enforcer) validates BIP300/301 messages alongside the L1 node. It provides the gRPC validator / block producer / wallet API used by BitWindow and the sidechains (gRPC 50051; `getblocktemplate` on 8122 for miners). Run it only if you interact with sidechains or mine. Plain L1 custody does not require it.

**Mainnet status (2026-10-27):** enforcer `master` only knows `--network-preset=alphanet|betanet`. Mainnet support is in open [PR #666](https://github.com/LayerTwo-Labs/bip300301_enforcer/pull/666). That PR replaces `--network-preset` with `--fork bitcoin|ecash-alphanet|ecash-betanet|ecash`, detects eCash from the node's user agent, applies mainnet magic `eca5dcb7`, activation height 973,728, `OP_NOP8` and the mainnet thresholds, and sets the replay `nLockTime` on its wallet sends. BitWindow 0.2.251 already starts the enforcer with `--fork`. **Do not run an enforcer on mainnet without mainnet parameters**: with no preset it falls back to `OP_NOP5`, Bitcoin's magic and activation height 0.
