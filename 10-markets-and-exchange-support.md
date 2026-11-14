# Markets & Exchange Support

**Last checked: 2026-11-13.** This page distinguishes native ECX swaps, wrapped-token trading and exchange fork-distribution policies. Availability, supported networks and account restrictions can change; recheck the provider before sending funds. Inclusion is not an endorsement or a guarantee of liquidity, redemption or customer eligibility.

Mainnet forked at Bitcoin block **973,728** on 2026-10-29 (UTC). Block **973,727** is the last shared Bitcoin block. See [launch information](https://ecash.com/what-to-expect/) and [holder coin-splitting guidance](https://ecash.com/split/). Operators should use the separate [mainnet integration playbook](09-mainnet-wallet-and-exchange-support.md).

## Identify the asset first

| Asset | Network / meaning |
|---|---|
| **ECX** | Native eCash Mainnet asset. Its Bitcoin-style addresses do not make it interchangeable with BTC. |
| **wECX / WECX** | Wrapped eCash on **Solana**, used by some venues below. Check the exact mint, not just the name. |
| **wbECX** | The older wrapped **Beta** token on Solana. It has a different mint and separate redemption terms; it is not wECX. |
| **XEC** | A different project also named eCash. It is not ECX or wECX. |

The separate community-created Solana token is also not the official wECX mint. BTC-holder entitlement, wrapped-token redemption and earlier burn programmes are different processes; do not infer one programme's eligibility or deadline from another.

## Swap and wrapped-token markets

| Venue | Asset / network | Verified status | Last checked |
|---|---|---|---|
| [SideShift](https://sideshift.ai) | **ECX — native eCash network** | Its public asset API lists coin `ECX`, "eCash (ECX)", on network `ecx`, with deposits and settlements not marked offline. The [ecash.com buy page](https://ecash.com/buy/) creates native ECX swaps through SideShift. Availability varies by location. | 2026-11-13 |
| [SideShift](https://sideshift.ai/from/wecx-solana) | **WECX — Solana** | Its public asset API lists the official mint below, with deposits and settlements not marked offline. | 2026-11-13 |
| [Orca pool](https://www.orca.so/pools/7T4xAj1ENEuPckhWqz6nJP9ZkvDnFpVF7nLbUAvSwm43) | **wECX — Solana** | Pool identified in the project's official token reference, and used by the ecash.com buy page for USDC → wECX. Check the current pool quote, liquidity and transaction details on Orca. | 2026-11-13 |
| [wrap.ecash.com](https://wrap.ecash.com) | **ECX ↔ wECX bridge** | Described on ecash.com as a **custodial** bridge, 1 ECX ↔ 1 wECX before fees, with a minimum. Follow the order's exact payment instructions; a plain wECX transfer will not match an order. | 2026-11-13 |

**Official wECX Solana mint:**

```text
6ombVv3YMs9qpx1QkycD9XW8HKSfv7vGrCFX3cxCYMVY
```

Verification sources: [public eCash token/pool reference](https://ecash.com/wrap_info.txt), [SideShift's live coin list](https://sideshift.ai/api/v2/coins), and [SideShift's API field definitions](https://docs.sideshift.ai/endpoints/v2/coins/). SideShift also lists the unrelated **XEC**, so select **ECX** (network `ecx`) for native eCash or **WECX on Solana** for the wrapped asset. An API availability flag is not a completed swap test.

Before using a swap or wrapped-token venue:

- Check the mint and the network for both sides of the transaction. Never send native ECX or BTC to a Solana token deposit route, and never send BTC to an ECX deposit address: the address formats are identical.
- Read the wrapper's custody and redemption terms. A token address or pool listing is not proof of reserves or a guarantee of redemption.
- Check fees, slippage, price impact and the amount you would receive. Do not rely on a quote copied into a guide.
- Never enter a recovery phrase or private key on a swap website.

The older Beta token's mint is `EVHqNdzjCupKi4rQkbuYw52sa1m8A7jeUAMP23S9AVVq`. The [published Beta notice](https://ecash.com/wrap_info.txt) says it is no longer redeemable for real ECX after 31 October; that date has passed.

## Exchange fork and distribution policies

**Fork support is not the same as a trading listing.** Crediting balances, distributing ECX or cash, opening deposits, opening withdrawals and enabling spot trading are separate decisions.

| Exchange | Announced policy | ECX trading / transfers | Source and last checked |
|---|---|---|---|
| **CoinW** | Plans a snapshot of eligible credited BTC balances at the fork and a **1 BTC : 1 ECX** distribution. | The cited notice covers snapshot/distribution only. Trading, deposits and withdrawals require separate announcements. | [19 October notice](https://www.coinw.com/help-center/announcement-center/more/2026-10-19/coinw-supports-the-ecash-ecx-bitcoin-hard-fork/8164); last read 2026-10-27. The page could not be re-fetched on 2026-11-13 (bot challenge), so check CoinW for post-fork updates |
| **GMO Coin — Japan** | Plans ECX **or Japanese yen** distribution for eligible BTC holdings, including eligible lending balances. The form, method, reference time and distribution timing remain to be announced. If ECX cannot be sold, the notice says a yen distribution will not be made. | The cited notice says GMO Coin **does not handle ECX**. Do not send ECX to its BTC deposit address. The notice suspended BTC deposits and withdrawals from 2026-10-29 16:00 JST, with resumption planned for 2026-11-02 12:00 JST; confirm the current state with GMO Coin. | [October notice](https://coin.z.com/jp/news/2026/10/16670/); checked 2026-11-13 |

These entries describe the cited notices, not a promise that every customer qualifies. Check your own exchange's current policy. Older announcements may contain an outdated calendar estimate for the fork; it happened at Bitcoin block **973,728** on 2026-10-29 (UTC). No CoinGecko price or exchange ticker was listed for ECX (`ecash-com`) on 2026-11-13.

## For Bitcoin holders

- **Self-custody:** start with [accessing ECX](https://ecash.com/how-to-claim/) and [coin splitting](https://ecash.com/split/). No pre-claim registration is required; protect the keys that may control both BTC and ECX.
- **Exchange custody:** check the exchange's eligibility, snapshot and distribution policy. An exchange announcement does not require you to expose a seed phrase or use a third-party claim form.
- **Network data:** use the [dashboard](https://dashboard.ecash.com/) and [explorer/API guide](04-explorers-and-apis.md), checking the selected network. Beta observations are not Mainnet observations.

## Keeping this page current

Add or update a venue only with an official source identifying the asset, network and service status. Record the source date and verification date. Change a distribution entry to a trading listing only when the provider explicitly announces or exposes that service. Label paused or discontinued services rather than leaving old availability claims in place.

[Back to Fast Facts](README.md)
