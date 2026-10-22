# GreyValley in 3 minutes

> Draft · 2026-10-05, updated 2026-10-13 and 2026-10-22 · Describes what exists today and promises no returns. It is not legal or financial advice.

## What it is
GreyValley is an **open-source interface** to protocols deployed on Internet Computer. You operate with **your own wallet**: you sign every action and assets are controlled by public smart contracts, not by GreyValley. The core asset is **ckUSDC**, a digital dollar.

## What you can do (the simplified app, `/app`)
| Feature | What it does | How it works |
|---|---|---|
| **Convert and Swap** | Exchange tokens for ckUSDC and for each other | Compares ICPSwap with GreyValley's pools and discards routes with unreasonable prices. ICP goes through ICPSwap; ckEURC and ckUSDT, through GreyValley's pools; everything else, through the best route. When the route goes through ICPSwap, GreyValley charges a **0.22% fee** on the amount as a separate transfer, shown before you sign. It is only charged when it exceeds 3 times the token's network fee. If you arrived through a partner's link, half goes to the partner (since 2026-10-16) |
| **Limit and Auto** | Buy or sell orders at a price (Limit) and recurring purchases (Auto) | The order's funds stay in the orders contract until it executes or you cancel it. The contract checks orders every 5 minutes |
| **Lend** | Supply ckUSDC to a credit pool | Other users borrow from that pool; the interest they pay is shared among lenders. You can only withdraw what is not lent out at that moment; the rest is freed when borrowers repay. Each withdrawal pays the ckUSDC network fee (0.01). The rate rises with pool usage: between 0.33% and 3.30% per month for borrowers, as of 2026-10-22. 15% of the interest first goes to a reserve that covers losses, up to 5% of the pool, and then to the GreyValley treasury; 85% goes to lenders. The app shows the aggregate health of all loans |
| **Borrow** | Receive ckUSDC without selling your crypto | You post ckBTC or ckETH as collateral. The rules (minimum collateral, liquidation, bonus) are read from the contract. As of 2026-10-22: 200% to borrow, liquidation below 140%, a 10% liquidation bonus and up to half of the debt per liquidation. Minimum 1 ckUSDC per loan, and a total cap of 10,000 ckUSDC of debt per collateral type. A keeper liquidates positions below the minimum and sells the collateral through the best route (ICPSwap or GreyValley's pools) |
| **Encrypted documents** (prototype) | Store notes and files encrypted on-chain and share them with whoever you choose | Encrypted with Internet Computer vetKeys: only you and the people you authorize can open them; title and reference are public. Currently using the network test key |
| **Pools and vaults** | Provide a token to swap pools | Returns come from swap fees; impermanent loss applies. In the app they sit in three tabs: Swap Vaults, ICP Staking Vault and Corridor Vaults |
| **Corridor** | Supply inventory to a CLP ↔ USD/EUR payments corridor | In test mode; depends on ramp partners |
| **Send and withdraw** | Move assets to another wallet or to Ethereum | ckETH and ckUSDC can be withdrawn to an EVM wallet |

Before each signature there is a **plain-language summary** (what you authorize, what risks exist and who can change what).

**Prices.** They come from the GreyValley oracle: Internet Computer's XRC for ICP, BTC and ETH; CoinGecko for LINK; mindicador.cl, with a fallback, for CLP and EUR. Every contract that uses a price (Lend and Borrow, GreyValley's pools, the CDP) checks the age of that asset's price and does not operate with one older than 25 minutes: it first asks the oracle for a new one. During this testing stage the oracle refreshes only every few hours and on demand from operations, to spend less (since 2026-10-19).

## Who it is for
- **Users:** convert, save in digital dollars, or borrow against crypto.
- **Partners and businesses — GV District** (`/distrito`): each business has its **Riverside** inside the app (`/app/ribera`), where it embeds the app on its site with its name and logo and earns a commission on its users' operations. Each partner registers on-chain the sites where it may be embedded (up to 5); on any other site the app does not show, so a third party cannot use it to trick users (see `partnerships/market-maker-guide.md` and `architecture/corredor-mechanics.md`).

## What it is NOT
- It is not a bank or a deposit: **there is no insurance or guarantee** that you recover what you supply.
- It is not an investment offer: returns are **not guaranteed** and can be low or zero.
- It is not a mature product: **early stage**, own pools with little liquidity, no external code audit, and a credit pool without CMF legal confirmation.

## The full app
A full version also exists with yield vaults, staking, the PXRM token, vUSD and collateral positions (CDP). It is explained in `finance/tokenomics.md`. The simplified app does not show PXRM or vUSD; to see them, connect to the full app with the same wallet. New vUSD cannot be minted today (`vusdMint = false`, verified 2026-10-22).

## More information
`architecture/tech-overview.md` · `architecture/corredor-mechanics.md` · `legal/regulatory.md` · `finance/tokenomics.md`
