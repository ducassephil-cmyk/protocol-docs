# ODL Bridge — Mechanics, Comparisons, and Partner Model
> On-Demand Liquidity architecture · 2026-08-26

---

## What is ODL and Why it Matters

ODL (On-Demand Liquidity) is the mechanism that allows moving value between currencies/countries in seconds using a digital asset as the bridge, instead of pre-funding nostro accounts at each correspondent bank.

The ODL bridge connects **CLP ↔ ckUSDC ↔ destination currencies** using a blockchain layer with ~2 second finality.

The stablecoin vault (Exaltite) **IS the bridge liquidity pool**. It is not a bank account or custodian — it is the ckUSDC inventory available to execute transactions instantly.

---

## How an ODL Payment Works

### Remittance flow (Chile → international)

```
1. User deposits CLP via regulated fiat partner (KYB handled by partner)
2. The stablecoin vault (ckUSDC pool funded by depositors) releases the
   equivalent ckUSDC on-chain instantly — this is classic ODL: the pool
   pays from existing inventory, not waiting for the CLP to "convert"
3. ckUSDC travels on-chain (~2 seconds finality)
4. At destination: fiat partner redeems ckUSDC and settles in local
   currency (USD, MXN, EUR, etc.)
5. Recipient receives funds
6. The CLP from step 1 (minus fees) eventually rebalances the ckUSDC pool

Protocol fee: 0.5% + fiat ramp ~1% (total user cost: ~1.5%)
SWIFT equivalent: ~2.5–3.5%
```

### The pool does not get "drained"

The ckUSDC in the vault is not consumed per transaction — it serves as **instant liquidity guarantee**. The actual flow:

- ckUSDC enters the pool when there are payments in the reverse direction (into Chile)
- ckUSDC exits when there are payments outbound
- If the flow is **bidirectional**: pool self-replenishes, capital permanently intact
- If the flow is **unidirectional**: pool can become unbalanced → protocol rebalances with accumulated fees (0.5% × volume)

**The depositor's capital never disappears.** If the pool becomes extremely unbalanced, the bridge pauses — but capital remains withdrawable.

### Capacity vs. TVL

```
$20K ckUSDC in vault → $20K of instant capacity (in simultaneous flight)
But the chain settles in ~2 seconds:
→ $20K pool can process $500K–$1M+ monthly volume
   (if no more than $20K is in flight simultaneously)

Real risk: extreme unidirectional flow, not pool size
```

---

## Comparison: Protocol vs. XRP/Ripple vs. SWIFT

### The Ripple/XRP model

Ripple uses XRP as a bridge asset between market makers (Bitso, SBI Remit, etc.):

```
US company → USD → MM buys XRP → XRP travels → MM Mexico sells XRP → MXN → recipient
```

Market makers maintain **XRP inventory on both sides of the corridor**. That inventory IS their ODL liquidity. They earn the spread + fees per transaction.

**The problem:** XRP is volatile. If XRP drops 40% while the MM holds inventory, it loses on principal. Only large entities with tolerance for that volatility become serious MMs.

### Comparison table

| Dimension | SWIFT | XRP/Ripple | This protocol |
|-----------|-------|-----------|---------------|
| Bridge asset | None (direct nostro) | XRP (volatile) | ckUSDC (stable $1) |
| Speed | 1–5 business days | 3–5 seconds | ~2 seconds |
| Typical user fee | 2.5–3.5% | 0.3–0.5% | ~0.5–0.7% |
| MM risk | Low (fiat) | High (XRP volatility) | Very low (ckUSDC stable) |
| New MM entry | Private Ripple agreement | High barrier | Deposit ckUSDC in vault |
| Governance | Banking / bilateral | Ripple Inc. centralizes | On-chain, transparent |
| Initial corridor | Global banking | USD↔MXN, USD↔PHP | CLP↔USD (LATAM first) |
| Yield for MM | None additional | Only spread/fees | Vault APR + Volume Guild |

### Why this architecture is better positioned than XRP for ODL

XRP was born as a speculative asset and adapted as a bridge. This infrastructure was born as distributed computing with:

1. **~2 second finality** (vs 3–5s for XRP, with no reversals)
2. **Native HTTP outcalls** — canisters can call banking APIs directly (Koywe, Fintoc, SWIFT MX, etc.) without external middleware
3. **Stable compute costs** — the cost of processing an ODL transaction does not fluctuate with the native token price
4. **Native chain-key assets** (ckBTC, ckUSDC, ckETH) — 1:1 representations of real assets, no additional bridging
5. **Unstoppable protocol** — the ODL code cannot be deactivated by a central bank or regulator acting unilaterally

---

## Business Partner Model — Partners as Market Makers

The key insight: **business partners are not "investors" — they are market makers of the CLP corridor.**

### What a Ripple market maker does

- Bitso (Mexico) deposits XRP on both sides of the USA↔MX corridor
- When someone sends $1,000 USA→MX, Bitso executes the conversion instantly
- Bitso earns the spread (XRP buy/sell difference) + commission
- The size of their inventory determines their volume capacity

### What a business partner does in this protocol

- Company deposits ckUSDC in the stablecoin vault
- That ckUSDC is the inventory for the CLP↔USD corridor
- When someone sends CLP to Mexico/USA, the ckUSDC pool executes instantly
- The partner earns: vault APR + proportional ODL fees + Volume Guild if >$25K/month

### The dual advantage of a partner who also uses the app for their own payments

A company that deposits ckUSDC AND processes its own international payments through the app gets:

```
Income 1:  APR on deposited capital (~8–10%/year in ckUSDC + PXRM Base APR)
Income 2:  Share of the 0.5% fee on each transaction flowing through their liquidity
Savings:   Their own payments go out at 0.5% instead of 2.5–3.5% SWIFT
Guild:     If >$25K/month in ODL → 7% of the fee pool from ALL users
```

**The right pitch:** Not "invest in DeFi" — it's "be the Bitso of Chile. Your inventory in ckUSDC (stable, no price risk) gives you APR + fees from every payment that passes through, and your own payments go out 5× cheaper."

---

## Bootstrap Capital Requirements

### Minimum viable (founder only)

| Vault | Demo minimum | Credible launch |
|-------|-------------|----------------|
| ICP vault (Exaltium) | $15K | $40K |
| Crypto vault (ckBTC/ckETH) | $5K | $15K |
| **Stablecoin vault (Exaltite)** ← critical | **$20K** | **$50K** |
| Total TVL | $40K | $105K |

**The stablecoin vault is the bottleneck:** without it there is no ODL capacity and the stablecoin vault APRs remain at `"--"`.

### With business partners (scalable)

| Partner profile | Own payments/month | Suggested vault TVL | SWIFT savings | Guild |
|----------------|-------------------|--------------------|--------------|----|
| Small (~$15K payments) | $15K | $10–25K | ~$375/month | No |
| Medium (~$30K payments) | $30K | $25–75K | ~$750/month | No |
| Large (~$75K payments) | $75K | $75K–300K | ~$1,875/month | ✓ Yes |

**Suggested Phase 1 captation structure:**
- 3 Angel partners ($15K TVL each) → $45K external TVL
- 1 Seed partner ($40K TVL) → $40K TVL
- 1 Strategic partner ($100K TVL) → $100K TVL + active Guild
- Total external: ~$185K TVL → protocol can self-sustain PXRM Base APR

### Revenue projection

| Monthly ODL | ODL Fees | NNS Yield | Swap+CDP | Total/month |
|------------|---------|-----------|---------|------------|
| $50K | $250 | $600 | $250 | ~$1,100 |
| $200K | $1,000 | $1,200 | $500 | ~$2,700 |
| $500K | $2,500 | $2,000 | $1,200 | ~$5,700 |
| $1M | $5,000 | $3,000 | $2,500 | ~$10,500 |
| $2M+ | $10,000 | $4,000 | $5,000 | ~$19,000 |

---

## Koywe Bridge — CLP → ckUSDC Technical Architecture

### Key resolved question: does Koywe need to install any blockchain code?

**No.** Koywe is pure web2. It installs nothing, integrates no blockchain SDK. The technical integration lives 100% on the protocol side:

```
Koywe:    web2 REST API (PAYIN / ONRAMP / OFFRAMP / PAYOUT)
Protocol: koywe_bridge canister that calls Koywe's API via HTTPS Outcall
```

Koywe only needs:
1. A **webhook URL** to notify when a payment confirms
2. An **EVM address** (Ethereum/Polygon) to send USDC to

Both are provided by the `koywe_bridge` canister.

### Koywe API — relevant endpoints

| Endpoint | Function |
|----------|---------|
| `POST /v3/deals` | Create ONRAMP order (CLP → USDC). Params: amount, fromCurrency, toCurrency, network, destinationAddress |
| `GET /v3/deals/{id}` | Check order status |
| `POST /v3/payouts` | OFFRAMP — convert USDC to CLP and send to bank account |
| Webhook `POST [url]/koywe-hook` | Koywe notifies when deal confirms |

### The Custody Triangle

```
┌──────────────────────────────────────────────────────────────┐
│                     CUSTODY TRIANGLE                         │
│                                                              │
│  [Koywe]           [EVM custody wallet]    [On-chain Ledger] │
│  CLP custody       USDC custody            ckUSDC            │
│  ~500 CLP          ~0.55 USDC              0.55 ckUSDC        │
│  (stays w/Koywe)   (canister address)      (user_principal)  │
│                                                              │
│  Koywe holds CLP → USDC arrives at EVM → Bridge mints ck    │
│                    custody wallet          1:1 on-chain      │
└──────────────────────────────────────────────────────────────┘

Invariant: ckUSDC in circulation = USDC locked in EVM custody wallet (1:1)
```

### Canister architecture

```
koywe_bridge canister responsibilities:
  1. Expose webhook endpoint via api_gateway
  2. Store: orderId → icp_principal (stable storage)
  3. Verify USDC on EVM via HTTPS Outcall → EVM RPC
  4. Call ckusdc_ledger.icrc1_mint(user_principal, amount)
  5. For off-ramp: call Koywe PAYOUT API via HTTPS Outcall

EVM key:
  - The EVM custody wallet uses Threshold ECDSA (tECDSA)
  - The private key NEVER exists on any server
  - It is fragmented across subnet nodes
  - Canister requests signature from runtime → network signs → tx sent to EVM RPC
```

### Complete on-ramp flow

```
Step 1:  User connects wallet → user_principal = "abc12-xyz34-..."
Step 2:  Frontend creates Koywe order with metadata { icp_principal: "abc12-xyz34..." }
Step 3:  koywe_bridge.registerOrder(koywe_order_id, user_principal) → stable map
Step 4:  User pays via Khipu → CLP received by Koywe
Step 5:  Koywe sends USDC to canister's EVM custody wallet
Step 6:  Koywe sends POST webhook → api_gateway
Step 7:  koywe_bridge verifies tx via EVM RPC HTTPS Outcall
Step 8:  koywe_bridge retrieves user_principal from stable map (by koywe_order_id)
Step 9:  koywe_bridge.icrc1_mint({ to: {owner: user_principal}, amount }) on ckusdc_ledger
Step 10: Wallet updates balance (icrc1_balance_of queries the ledger)
```

---

## Multi-Currency Expansion Roadmap

| Currency | Open Banking coverage | What's needed | Viability |
|----------|----------------------|---------------|-----------|
| **sCLP** | Fintoc (Chile) | CMF approval under Ley 21.521 | Code complete, regulatory pending |
| **ckMXN** | Fintoc (Mexico) | Real bank account in Mexico via local entity | Technically straightforward — same pattern as sCLP |
| **ckBRL** | Belvo/Pluggy (Brazil) | Brazilian Open Banking provider + BACEN registration | Higher regulatory cost |
| **ckARS** | Not viable | Argentine exchange controls (cepo) block any viable flow | Deferred until regulatory environment changes |

The canister-side code (`mint/burn/reconcile` pattern) is reusable as a template per currency — only the Open Banking provider and the banking custodian behind it need to change.

---

*ODL Bridge Mechanics · 2026-08-26*
