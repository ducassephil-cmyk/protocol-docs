# GreyValley in 3 minutes

> Draft · 2026-10-05 · Describes what exists today and promises no returns. It is not legal or financial advice.

## What it is
GreyValley is an **open-source interface** to protocols deployed on Internet Computer. You operate with **your own wallet**: you sign every action and assets are controlled by public smart contracts, not by GreyValley. The core asset is **ckUSDC**, a digital dollar.

## What you can do (the simplified app, `/app`)
| Feature | What it does | How it works |
|---|---|---|
| **Convert and Swap** | Exchange tokens for ckUSDC and for each other | Compares ICPSwap with GreyValley's pools and discards routes with unreasonable prices |
| **Lend** | Supply ckUSDC to a credit pool | Other users borrow from that pool; the interest they pay is shared among lenders |
| **Borrow** | Receive ckUSDC without selling your crypto | You post ckBTC or ckETH as collateral. The rules (minimum collateral, liquidation, bonus) are read from the contract; as of 2026-10-05: 200% to borrow, liquidation below 140% |
| **Pools and vaults** | Provide a token to swap pools | Returns come from swap fees; impermanent loss applies |
| **Corridor** | Supply inventory to a CLP ↔ USD/EUR payments corridor | In test mode; depends on ramp partners |
| **Send and withdraw** | Move assets to another wallet or to Ethereum | ckETH and ckUSDC can be withdrawn to an EVM wallet |

Before each signature there is a **plain-language summary** (what you authorize, what risks exist and who can change what).

## Who it is for
- **Users:** convert, save in digital dollars, or borrow against crypto.
- **Partners and businesses:** embed the app on their site with their name and logo and earn a commission on their users' operations (see `partnerships/market-maker-guide.md` and `architecture/corredor-mechanics.md`).

## What it is NOT
- It is not a bank or a deposit: **there is no insurance or guarantee** that you recover what you supply.
- It is not an investment offer: returns are **not guaranteed** and can be low or zero.
- It is not a mature product: **early stage**, own pools with little liquidity, no external code audit, and a credit pool without CMF legal confirmation.

## The full app
A full version also exists with yield vaults, staking, the PXRM token, vUSD and collateral positions (CDP). It is explained in `finance/tokenomics.md`. The simplified app does not show PXRM or vUSD; to see them, connect to the full app with the same wallet.

## More information
`architecture/tech-overview.md` · `architecture/corredor-mechanics.md` · `legal/regulatory.md` · `finance/tokenomics.md`
