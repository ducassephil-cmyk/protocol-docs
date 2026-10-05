# Partner guide: being the "market maker" of the CLP corridor

> Draft · 2026-10-05 · Commercial summary of `architecture/corredor-mechanics.md` §4, without technical jargon. It promises no returns and is not legal advice.

## The idea in one sentence
In international payment corridors, whoever supplies **inventory** (money ready on both sides) lets each payment execute instantly and captures part of the value of each transaction. In GreyValley, that inventory is **ckUSDC**, a stable digital dollar.

## How it works
1. A business supplies ckUSDC (or ckEURC, for the euro corridor) to the corridor pool.
2. When someone sends Chilean pesos abroad (or receives them), the pool executes the exchange instantly at a price taken from an oracle.
3. The supplier receives a proportional share of the fees from the real payment volume that goes through the corridor.

## What it is NOT
- **It is not a fixed interest rate.** Income depends on real payment volume, which is currently low or nil. Any figure is illustrative of the mechanics, not a projection.
- **It is not an insured deposit.** There is no insurance or third-party guarantee.
- **Corridor fees are locked** until the protocol's governance is active, and the corridor runs **in test mode**.

## What it does offer (per the mechanics described)
- **Stable inventory:** being in digital dollars, it carries no price risk against the dollar; exchange-rate risk between the peso, the dollar and the euro does exist.
- **Own savings:** if the business also processes its international payments through the corridor, the cost of those payments is the corridor fee (currently 0.66%, verified on-chain) instead of traditional international-transfer fees.
- **Variable participation:** a share of the fees and, if monthly volume exceeds the defined threshold, an additional distribution among the partners with the most liquidity.

## Main risks
- Low volume: income can be zero.
- Dependence on ramp partners and on regulation allowing it.
- Smart-contract and oracle risk; no external audit yet.
- Funds sit in contracts currently controlled by a single principal (see `legal/regulatory.md` §11.3).

## What you need
An Internet Computer wallet, ckUSDC or ckEURC, and to read the plain-language agreement before signing. To take part as an operator with its own integration, the model requires a separate agreement (see `partnerships/odl-agreement.md`, a template pending the signing entity).
