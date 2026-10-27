# Transaction Replay & Coin Splitting

## Why replay happens

The fork starts from Bitcoin's UTXO set at block 973,727, and the transaction formats are identical. A transaction whose inputs remain spendable on both networks can be **replayed**: broadcast on one chain, then copied by anyone onto the other. A post-fork transaction date alone does not establish that its coins are separated.

For an exchange this is dangerous in both directions:

- You send a customer a BTC withdrawal; it replays on eCash and the customer also receives matching ECX from your reserves.
- You sweep ECX deposits; the sweep replays on Bitcoin and moves your BTC without authorization from your BTC accounting system.

## The protection mechanism: magic nLockTime (opt-in)

> A transaction with **`nLockTime = 499999998`** is treated as **final** by eCash mainnet nodes. With **at least one input `nSequence < 0xffffffff`**, Bitcoin enforces that far-future height lock and rejects it as non-final. If every input has the final sequence, Bitcoin ignores the locktime. Normal signature, UTXO and script checks still apply on eCash.

| Requirement | Value |
|---|---|
| `nLockTime` | **`499999998`** (`LOCKTIME_THRESHOLD - 2`; `LOCKTIME_THRESHOLD` is `500000000`) |
| `nSequence` | at least one input below `0xffffffff`. If every input is `0xffffffff`, Bitcoin ignores the locktime and the transaction is replayable. `0xfffffffd` (RBF) and `0xfffffffe` both work |
| Never use | `500000000` or higher: those values are read as a timestamp |

### ⚠️ Betanet used a different value

Alphanet and betanet used `499999999` (`LOCKTIME_THRESHOLD - 1`). On mainnet that value is an ordinary far-future height lock: **with a non-final input sequence, a transaction built with `499999999` is non-final on both eCash and Bitcoin**. With all-final sequences, Bitcoin ignores the lock and it provides no replay protection. Signing code tested on betanet must change the constant at cutover. Parameterize it per network. Other test-network differences: [testnets/](testnets/README.md).

- **Opt-in:** transactions without the marker may remain valid on both chains if their inputs and other validity conditions permit it.
- **One-directional:** it protects eCash transactions from replaying onto Bitcoin. No marker makes a Bitcoin transaction invalid on eCash; the BTC side is protected by the order of splitting (below).
- **Signing support must be checked:** serialization is unchanged, but not every wallet, hardware device or companion application exposes or preserves these fields. Bitcoin Core's `sendtoaddress`/`sendmany` use ordinary anti-fee-sniping locktimes, not the marker. Use `send`/`sendall`/`walletcreatefundedpsbt` with an explicit locktime and verify the final input sequences ([02](02-wallets-keys-addresses.md#withdrawal-construction-one-ecash-specific-rule)).
- **Not compatible with CLTV:** the magic value is excluded from the `OP_CHECKLOCKTIMEVERIFY` height comparison, so a transaction carrying it fails every CLTV check. Inputs with script-level timelocks (HTLCs, some vault and Lightning scripts) need a procedure designed for that script.

### Implementation

Reference commit on mainnet `master`: [`b9df0c5` "Add replay protection via magic nLockTime"](https://github.com/ecash-com/bitcoin/commit/b9df0c5fe1edc0f1ed37c96ee032c2c595136a9c). It makes two one-line consensus changes:

- `IsFinalTx` in `src/consensus/tx_verify.cpp` short-circuits to final when `tx.nLockTime == LOCKTIME_THRESHOLD - 2`, alongside the existing `nLockTime == 0` case.
- `CheckLockTime` in `src/script/interpreter.cpp` adds the matching CLTV exclusion.

`master` ships no functional test for it; the betanet branch's `feature_replay_protection.py` tests the betanet value. Test the mainnet value on regtest with the mainnet binary. The check is network-independent, so `-regtest` enforces `499999998` too:

```sh
bitcoind -regtest -datadir=./rt -daemon                 # eCash mainnet binary (v1.1.0)
# fund a wallet, then build a tx with nLockTime=499999998 and one input nSequence=0xfffffffd
bitcoin-cli -regtest -datadir=./rt testmempoolaccept '["<hex>"]'
# eCash: "allowed": true      stock Bitcoin Core regtest: "allowed": false, "reject-reason": "non-final"
```

## How to split coins (exchange procedure)

1. **Snapshot and freeze.** Record every balance and UTXO as of block **973,727**, the last block shared with Bitcoin. Freeze BTC (and ECX) withdrawals shortly before block 973,728. Key the freeze on height, not clock time.
2. **Do not move pre-fork BTC UTXOs** until the eCash side is split. Order matters.
3. **Split on the eCash side first:** sweep every pre-fork UTXO you control into fresh addresses (ideally from a new eCash-dedicated seed) with `nLockTime = 499999998` and a non-final input sequence. These sweeps confirm only on eCash and cannot touch your BTC.
4. **Wait for deep confirmation on eCash.** The post-fork period has a reset difficulty and elevated reorg risk; a reorg past your split transactions would re-expose you to replay. Count confirmations in blocks, and remember that blocks may arrive much faster than every 10 minutes at first ([01](01-node-setup.md#difficulty-after-the-fork)).
5. **Check both chains.** Each split transaction should be confirmed on eCash, and the corresponding BTC still unspent on Bitcoin.
6. **Resume BTC operations for verified split UTXOs:** those outputs are already spent on eCash, so a Bitcoin transaction spending them has nothing to replay against, unless a reorg reverses the split. Do not assume omitted or newly received UTXOs are also separated.
7. **Keep setting the magic `nLockTime` on all future ECX transactions.** After the split it is redundant for your own coins, but it guards against any unsplit UTXO that later lands in your wallets (e.g. a customer depositing pre-fork coins that were never split).

Only the outputs included in a successful split are covered. Other accounts, omitted outputs and later receipts can still be shared. A post-fork timestamp or block height on its own does not prove that a coin is separated.

For end users, the official mobile wallet ("Check for splittable coins") and BitWindow perform the split for them. See [ecash.com/split](https://ecash.com/split/) for the user-facing guidance per wallet type.

Never enter a recovery phrase on a website or import production BTC keys into a test wallet. A hardware-wallet seed should stay on the device; use a supported signing workflow.

## Deposit-side cautions

- A customer's ECX deposit may be a replay of their BTC transaction (or vice versa). Crediting is fine, but credit each chain only from that chain's own node and index. Never infer a deposit on one chain from a transaction seen on the other.
- The "same" transaction can confirm at different times, or on only one chain, ever. Treat the ledgers as fully independent from block 973,728.
- **Repurposed (Patoshi) coins:** 232 hard-coded transactions (`src/repo_txns.h` on `master`) spend Satoshi-era coins without signatures. Coins descending from them are valid ECX by consensus. Mainnet's list is not the betanet list.

## Rehearsing

Mainnet does not exist before block 973,728, so there is no pre-fork mainnet window. Options:

- **Regtest with the mainnet binary** (above): exercises the exact mainnet marker. Use isolated test keys and make the same test UTXO available on both regtest nodes before comparing `testmempoolaccept`; a missing-input rejection does not prove replay protection.
- **Betanet** ([testnets/betanet.md](testnets/betanet.md)): a live fork for end-to-end drills, but with marker `499999999`. Use dedicated test funds and keys, not production BTC keys. Switch the constant before promoting the drill to mainnet.
