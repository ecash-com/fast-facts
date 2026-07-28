# Transaction Replay & Coin Splitting

## Why replay happens

Every UTXO that existed at the fork block exists on both chains, and the transaction formats are identical. A transaction spending pre-fork UTXOs is therefore valid on both networks and can be **replayed**: broadcast on one chain, copied by anyone onto the other. Per the team, even after the split "99% of the new txns will be replayable."

For an exchange this is dangerous in both directions:

- You send a customer a BTC withdrawal; it replays on eCash and the customer also receives matching ECX from your reserves.
- You sweep ECX deposits; the sweep replays on Bitcoin and moves your BTC without authorization from your BTC accounting system.

## The protection mechanism: magic nLockTime (opt-in)

> A transaction with `nLockTime == 499999999` (`LOCKTIME_THRESHOLD - 1`) is treated as **final** by eCash nodes. Stock Bitcoin Core interprets that value as a block height ~500 million blocks in the future and rejects the transaction as non-final, so it can never be relayed or mined on Bitcoin.

- **Opt-in:** transactions with any other `nLockTime` remain valid on both chains.
- **One-directional:** it protects eCash transactions from replaying onto Bitcoin. There is no marker to make a Bitcoin transaction invalid on eCash; the BTC side is protected by splitting order (below).
- **No tooling impact:** serialization is unchanged; any wallet, library, or HSM can set the field (with sequence numbers below `0xffffffff` so locktime is enforced, as usual).
- Implemented in `IsFinalTx` (`src/consensus/tx_verify.cpp`) with a matching CLTV adjustment; functional test `test/functional/feature_replay_protection.py`.

> **Scheme history, verify before launch:** drynet1 and drynet2 used a magic transaction version (`12566463` / `0x00BFBF3F`) instead; per-network details are in [09-drynets](09-drynets/README.md). Confirm the final scheme against the launch branch and https://drivechain.info/dev.txt before going live.

## How to split coins (exchange procedure)

1. **Freeze** BTC (and ECX) withdrawals shortly before the fork block. Record your balance snapshot at the fork height.
2. **Do not move pre-fork BTC UTXOs** until the eCash side is split. Order matters.
3. **Split on the eCash side first:** sweep every pre-fork UTXO you control into fresh addresses (ideally from a new eCash-dedicated seed) with `nLockTime = 499999999`. These sweeps confirm only on eCash and cannot touch your BTC.
4. **Wait for deep confirmation on eCash.** The post-fork period has minimum difficulty and elevated reorg risk; a reorg past your split transactions would re-expose you to replay.
5. **The BTC side is now automatically safe:** your pre-fork UTXOs are already spent on eCash, so a Bitcoin transaction spending them has nothing to replay against. Resume BTC operations.
6. **Keep setting `nLockTime = 499999999` on all future ECX transactions.** Post-split it is redundant, but it guards against any not-yet-split UTXO that later lands in your wallets (e.g. a customer depositing pre-fork, never-split coins).

For end users, BitWindow performs the split as an automatic one-time step after the fork.

## Deposit-side cautions

- A customer's ECX deposit may be a replay of their BTC transaction (or vice versa). Crediting is fine, but credit each chain only from that chain's own node and index; never infer a deposit on one chain from a transaction seen on the other.
- The "same" transaction can confirm at different times, or on only one chain, ever. Treat the ledgers as fully independent from the first post-fork block.
- **Repurposed (Patoshi) coins:** 122 hard-coded transactions spend Satoshi-era coins without signatures (whitelisted in `src/repo_txns.h`). Coins descending from them are valid ECX by consensus; flag them only if your compliance policy cares about provenance.

## Test it on drynet3

Practice the split on drynet3, then attempt to replay your own transactions across drynet3 and a regtest or mainnet-following node to verify your pipeline refuses them. `feature_replay_protection.py` shows the exact expected node behavior.
