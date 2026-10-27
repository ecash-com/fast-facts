# Wallets, Keys & Addresses

Key and address formats are identical to Bitcoin. Existing Bitcoin tooling can be reused, but format compatibility alone does not establish wallet support. Verify the network connection, signing workflow, replay-protection locktime and input sequence settings ([03](03-replay-protection-and-coin-splitting.md)).

## Address & key formats

| Parameter | Value | Same as BTC? |
|---|---|---|
| P2PKH prefix | `0x00`, addresses start `1...` | Yes |
| P2SH prefix | `0x05`, addresses start `3...` | Yes |
| Bech32 HRP | `bc`, so `bc1q...` (segwit v0) and `bc1p...` (taproot) | Yes |
| WIF prefix | `128` (0x80) | Yes |
| BIP32 xpub | `0x0488B21E` (`xpub...`) | Yes |
| BIP32 xprv | `0x0488ADE4` (`xprv...`) | Yes |
| Curve / signatures | secp256k1, ECDSA + Schnorr (taproot) | Yes |
| Unit | 1 ECX = 100,000,000 sats (8 decimals) | Yes |

A BTC address is an ECX address and vice versa; nothing in the string identifies the chain. Deposit UX must make the target chain explicit, and internal systems must key balances by `(chain, address)`, never by address alone.

## Deriving new addresses

**Bitcoin Core wallet (recommended for exchanges):**

```sh
bitcoin-cli -datadir=<dir> createwallet "deposits"
bitcoin-cli -datadir=<dir> getnewaddress "" bech32        # bc1q...
bitcoin-cli -datadir=<dir> getnewaddress "" bech32m       # bc1p... (taproot)

# Stateless derivation from a descriptor
bitcoin-cli -datadir=<dir> deriveaddresses \
  "wpkh([fingerprint/84h/0h/0h]xpub.../0/*)#checksum" "[0,999]"
```

**External libraries:** any Bitcoin library (BDK, bitcoinjs-lib, btcd, libwally) produces correct eCash addresses with its Bitcoin-mainnet settings. The official mobile wallet uses BDK with `Network.bitcoin`. Standard paths apply: BIP84 (`m/84'/0'/0'`) for P2WPKH, BIP86 for taproot.

No separate SLIP-44 coin type is registered for ECX; software treats it as coin type `0'` (Bitcoin). For key separation within one seed use a distinct account index, but a separate seed is cleaner (below).

## Storing seed randomness

Identical best practice to Bitcoin:

- Generate seeds from a high-quality entropy source (OS CSPRNG or hardware RNG on an HSM or air-gapped signer), 128-256 bits, as a BIP39 mnemonic or raw BIP32 seed.
- Keep signing keys offline. Bitcoin hardware wallets and HSMs use compatible signing formats, but support depends on the device, firmware and companion application. Confirm that the complete workflow preserves `nLockTime` and a non-final input `nSequence`, then inspect the final signed transaction. Do not export a hardware-wallet seed merely to work around a missing feature.
- Standard cold-storage hygiene: distributed backups, metal mnemonic backup, multisig (`wsh(multi(...))`) for treasury, periodic restore drills.
- Core descriptor wallets store keys in encrypted SQLite (`walletpassphrase` unchanged). Back up via `listdescriptors true` or file-level wallet backups.

Fork-specific: any seed that held BTC at block 973,727 automatically controls the matching ECX. For post-fork operational wallets, generate a fresh eCash-dedicated seed. That way a hot-wallet compromise on one chain doesn't expose the other, and accounting stays unambiguous. Every eCash stage derives the same addresses from one seed, so a seed used on betanet is also a real mainnet ECX seed.

## Watching-only wallets

Full support via descriptor wallets (keys offline, watch-only node online):

```sh
# 1. Create a watch-only wallet (no private keys can ever enter it)
bitcoin-cli -datadir=<dir> createwallet "watch" true true    # disable_private_keys=true, blank=true

# 2. Import public descriptors exported from the offline signer
bitcoin-cli -datadir=<dir> -rpcwallet=watch importdescriptors '[
  {"desc":"wpkh([f00dbabe/84h/0h/0h]xpub6.../0/*)#xxxxxxxx","timestamp":"now","active":true,"range":[0,9999],"internal":false},
  {"desc":"wpkh([f00dbabe/84h/0h/0h]xpub6.../1/*)#yyyyyyyy","timestamp":"now","active":true,"internal":true}
]'

# 3. Detect deposits
bitcoin-cli -datadir=<dir> -rpcwallet=watch listunspent
# ...or subscribe to ZMQ (zmqpubsequence) and use gettransaction / listsinceblock

# 4. Withdrawals: build PSBT online (set locktime=499999998), sign offline, broadcast online
bitcoin-cli -datadir=<dir> -rpcwallet=watch walletcreatefundedpsbt '[]' '[{"bc1q...":0.1}]' 499999998 '{"replaceable":true}'
bitcoin-cli -datadir=<dir> sendrawtransaction <hex>
```

- `"timestamp":"now"` avoids rescans. Use a real timestamp or height to pick up historical deposits. Pre-fork timestamps need non-pruned block data, so plan around assumeutxo and pruning for deep rescans.
- The same xpub imported into a Bitcoin node and an eCash node yields the same addresses watching two different ledgers. That is how you observe both sides of the fork with one key set.
- Lighter alternative: the public Electrum server (`ssl://explorer.ecash.ninja:50002`) or the Esplora API (`https://esplora.ecash.ninja`, [04](04-explorers-and-apis.md)). For custody-grade detection, run your own node plus an Electrum server.
- `dumpprivkey`/`importprivkey` only apply to legacy (BDB) wallets, which can no longer be created in v31. Use descriptors.

## Withdrawal construction: one eCash-specific rule

Set `nLockTime = 499999998` on every ECX withdrawal, with at least one input `nSequence < 0xffffffff`. This makes the transaction final on eCash but non-final on Bitcoin until that far-future block height is passed. Normal signature, UTXO and script checks still apply. Serialization is unchanged, but the wallet or signing workflow must let you set and preserve both fields. Details: [03](03-replay-protection-and-coin-splitting.md).

**Bitcoin Core wallet RPCs:** `sendtoaddress` and `sendmany` have no locktime parameter and use ordinary anti-fee-sniping locktimes, not the eCash marker. Their transactions may replay if the inputs are spendable on both chains. Use an RPC that takes a locktime:

| RPC | How to set the marker |
|---|---|
| `send` | `-named send outputs='{"bc1q...":0.1}' options='{"locktime":499999998,"replaceable":true}'` |
| `sendall` | `options='{"locktime":499999998}'` |
| `walletcreatefundedpsbt` | third positional argument `499999998` (above) |
| `createrawtransaction` | third positional argument `499999998`; set input `sequence` below `0xffffffff` |

Decode the final signed transaction (`decoderawtransaction`) and check `locktime` and every input's `sequence` before broadcasting.

## End-user wallets

| Wallet | Mainnet status (2026-10-27) |
|---|---|
| eCash mobile wallet ([ecash-com/ecash-wallet-mobile](https://github.com/ecash-com/ecash-wallet-mobile)) | v1.4.6 (2026-10-26) lists **eCash (ECX)** as a network, configured from drivechain.dev/config (id `ecash`). New and imported wallets switch to it by default once the chain passes block 973,728. It has a built-in split ("Check for splittable coins"), which flags a coin only while Bitcoin's explorer still shows it unspent. Android APK from [GitHub releases](https://github.com/ecash-com/ecash-wallet-mobile/releases/latest); iOS via [TestFlight](https://testflight.apple.com/join/PxZjPxrK) |
| BitWindow (desktop; also the activation client) | build 0.2.251 (2026-10-26) includes the eCash mainnet network. https://ecash.com/download |
| Other Bitcoin wallets (Sparrow, Electrum, hardware wallets) | Formats are compatible, but they need to connect to an eCash server and must set the replay marker. Compatibility alone is not support ([ecash.com/split](https://ecash.com/split/)) |
