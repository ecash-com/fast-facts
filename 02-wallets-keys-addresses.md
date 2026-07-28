# Wallets, Keys & Addresses

Key management is byte-for-byte identical to Bitcoin (a stated design goal of the fork); your existing BTC key-management stack works unchanged.

## Address & key formats

Verified against `src/kernel/chainparams.cpp` on the `drynet3` branch:

| Parameter | Value | Same as BTC? |
|---|---|---|
| P2PKH prefix | `0x00`, addresses start `1...` | Yes |
| P2SH prefix | `0x05`, addresses start `3...` | Yes |
| Bech32 HRP | `bc`, so `bc1q...` (segwit v0) and `bc1p...` (taproot) | Yes |
| WIF prefix | `128` (0x80) | Yes |
| BIP32 xpub | `0x0488B21E` (`xpub...`) | Yes |
| BIP32 xprv | `0x0488ADE4` (`xprv...`) | Yes |
| Curve / signatures | secp256k1, ECDSA + Schnorr (taproot) | Yes |

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

**External libraries:** any Bitcoin library (BDK, bitcoinjs-lib, btcd, libwally) produces correct eCash addresses with its Bitcoin-mainnet settings. Standard paths apply: BIP84 (`m/84'/0'/0'`) for P2WPKH, BIP86 for taproot.

No separate SLIP-44 coin type is registered for ECX; software treats it as coin type `0'` (Bitcoin). For key separation within one seed use a distinct account index, but a separate seed is cleaner (below).

## Storing seed randomness

Identical best practice to Bitcoin:

- Generate seeds from a high-quality entropy source (OS CSPRNG or hardware RNG on an HSM or air-gapped signer), 128-256 bits, as a BIP39 mnemonic or raw BIP32 seed.
- Keep signing keys offline. Bitcoin hardware wallets and HSMs work as-is; they will sign eCash transactions since they can't tell the difference (see [03](03-replay-protection-and-coin-splitting.md) for why that matters).
- Standard cold-storage hygiene: distributed backups, metal mnemonic backup, multisig (`wsh(multi(...))`) for treasury, periodic restore drills.
- Core descriptor wallets store keys in encrypted SQLite (`walletpassphrase` unchanged). Back up via `listdescriptors true` or file-level wallet backups.

Fork-specific: any seed that held BTC at the fork block automatically controls the matching ECX. For post-fork operational wallets, generate a fresh eCash-dedicated seed so a hot-wallet compromise on one chain doesn't expose the other, and accounting stays unambiguous.

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

# 4. Withdrawals: build PSBT online, sign offline, broadcast online
bitcoin-cli -datadir=<dir> -rpcwallet=watch walletcreatefundedpsbt ...
bitcoin-cli -datadir=<dir> sendrawtransaction <hex>
```

- `"timestamp":"now"` avoids rescans; use a real timestamp/height to pick up historical deposits (pre-fork timestamps need non-pruned block data, so plan around assumeutxo/prune for deep rescans).
- The same xpub imported into a Bitcoin node and an eCash node yields the same addresses watching two different ledgers; that is how you observe both sides of the fork with one key set.
- Lighter alternative: the Electrum server (`ssl://drynet3.drivechain.dev:50012`) or Esplora API. For custody-grade detection run your own node plus electrs.
- `dumpprivkey`/`importprivkey` only apply to legacy (BDB) wallets, which can no longer be created in v31. Use descriptors.

## Withdrawal construction: one eCash-specific rule

Set `nLockTime = 499999999` on every ECX withdrawal. This is the opt-in replay protection marker: final on eCash, permanently invalid on Bitcoin. Serialization is unchanged, so any wallet, library, or HSM can produce protected transactions by setting that field. Details: [03](03-replay-protection-and-coin-splitting.md).
