# Sidechains (L2s)

Exchanges only strictly need the L1 (ECX deposits and withdrawals work exactly like BTC). The L2 layer matters because coins move between L1 and L2s, and some products (fast withdrawals, the zSide privacy chain) may be relevant to your users.

**Where the rules live:** the `ecash-com/bitcoin` node contains only minimal drivechain plumbing (`OP_DRIVECHAIN` made standard, OP_RETURN limits removed, the fork-height difficulty reset). The BIP300/301 state machine (sidechain proposals, deposit escrows, withdrawal-bundle ACK counting, blind merged mining) is validated by the companion daemon [`bip300301_enforcer`](https://github.com/LayerTwo-Labs/bip300301_enforcer) running alongside the node. Plain L1 nodes without the enforcer still follow the chain.

## The seven launch sidechains

Each is its own chain with its own node software, secured by eCash miners via BIP300 (no federation, no multisig custodians).

| Slot | Sidechain | Purpose | Repo |
|---|---|---|---|
| 2 | BitNames | Identity / DNS | https://github.com/LayerTwo-Labs/plain-bitnames |
| 4 | BitAssets | Tokens / NFTs | https://github.com/LayerTwo-Labs/plain-bitassets |
| 9 | Thunder | Large-block scaling | https://github.com/LayerTwo-Labs/thunder-rust |
| 13 | Truthcoin | Prediction markets | https://github.com/LayerTwo-Labs/truthcoin-dc |
| 98 | zSide | Privacy (Zcash-style) | https://github.com/iwakura-rein/thunder-orchard |
| 99 | Photon | Quantum resistance | https://github.com/LayerTwo-Labs/photon |
| 255 | CoinShift | Cross-coin atomic swaps | https://github.com/LayerTwo-Labs/coinshift-rs |

Default ports follow the pattern `40<slot>` (P2P), `60<slot>` (RPC), `280<slot>` (ZMQ); e.g. Thunder (slot 9) uses P2P 4009, RPC 6009. Full table: https://drivechain.info/dev.txt

## How coins move between L1 and L2

- **Deposit (L1 to L2):** an M5 deposit transaction sends ECX into the sidechain's BIP300 escrow UTXO. Wallet software (BitWindow, sidechain wallets) handles this.
- **Withdrawal (L2 to L1):** withdrawals are batched into **bundles** (up to 6,000 per bundle). A bundle needs a work score of **13,150 miner ACKs** within a **26,300-block window** (roughly 3-6 months) to pay out on L1. The slow path is the security model.
- **Fast withdrawals:** a service that atomically swaps L2 coins for L1 coins immediately (a third party fronts the L1 coins and collects the bundle payout later). Drynet3 runs one at `fw1.drynet3.drivechain.dev`. Convenience service, not consensus-critical.
- The escrow uses no cryptographic signatures (a hashrate escrow: anyone-can-spend UTXO protected by consensus rules), which is why the team describes the peg as quantum-proof.

## Proposing a new sidechain

Anyone can propose one; miners decide activation (BIP300 M1/M2):

1. **Write the sidechain software.** The practical path is forking an existing L2 (Thunder is the usual template) and changing its slot number and rules; the LayerTwo Labs sidechains build on the CUSF sidechain protocol (`cusf_sidechain_proto`).
2. **M1, propose:** a miner includes an M1 "Propose Sidechain" message in a coinbase (slot number, title, description, hash of the sidechain software/genesis). Submit via the enforcer:
   ```
   cusf.mainchain.v1.WalletService/CreateSidechainProposal   (bip300301_enforcer gRPC)
   ```
   BitWindow also exposes this in its GUI.
3. **M2, ACK:** miners ACK the proposal in subsequent coinbases.
   - Unused slot: activates unless it accumulates 1,008 fails (non-ACKing blocks) within 2,016 blocks, i.e. sustained ~50% hashrate approval for about two weeks.
   - Overwriting a used slot: requires the longer 26,300-block / 13,150-fail threshold.
4. Once active, the sidechain's escrow exists at L1 and deposits can begin.

eCash launches with home-mineable difficulty and BIP301 blind merged mining, so proposing and ACKing during the early network is unusually accessible. To socialize a proposal, post in the DcInsiders Telegram ([06](06-contacts-and-resources.md)).

Specs: [BIP300](https://github.com/bitcoin/bips/blob/master/bip-0300.mediawiki), [BIP301](https://github.com/bitcoin/bips/blob/master/bip-0301.mediawiki).

## BIP301: blind merged mining

Sidechain block producers pay L1 miners (in ECX) to commit to sidechain block hashes in L1 coinbases, so miners earn sidechain revenue without running sidechain nodes. No exchange action needed.

## The enforcer

[`bip300301_enforcer`](https://github.com/LayerTwo-Labs/bip300301_enforcer) validates BIP300/301 messages alongside the L1 node and provides the gRPC wallet/validator API used by BitWindow and the sidechains (gRPC 50051; getblocktemplate on 8122 for miners). Run it only if you interact with sidechains or mine; plain L1 custody does not require it.
