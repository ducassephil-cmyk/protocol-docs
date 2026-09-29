# GREYVALLEY — Corredor: Mechanics, Comparison, and Partner Model
> Version: 2026-06-28 | Complements TOKENOMICS.md §10 (Corredor) | Updated 2026-09-04

---

## 1. WHAT THE CORREDOR IS AND WHY IT MATTERS

GreyValley's corredor (corridor) is the mechanism that moves value between currencies/countries in seconds using a digital asset as a bridge, instead of pre-funding nostro accounts at every correspondent bank — the same "on-demand liquidity" principle used by the remittance industry, applied with chain-key assets instead of an intermediary banking partner.

At GreyValley, the corredor connects **Chilean CLP ↔ ckUSDC/ckEURC ↔ destination currencies** using ICP as the settlement layer (~2 seconds of finality). Two corridors have real custody today: **CLP↔USD** (ckUSDC) and **CLP↔EUR** (ckEURC, integrated 2026-09-04) — each with an independent pool in `liquidity_pool`, real pricing via `oracle.getClpPerUsd()`/`getClpPerEur()` (mindicador.cl).

The Exaltite Vault (ckUSDC/ckUSDT/ckEURC) **IS the corredor's liquidity pool**. It is not a bank account or a custodian — it is the inventory of ckUSDC/ckEURC available to execute transactions instantly, one per corridor (CLP↔USD and CLP↔EUR do not share custody).

---

## 2. HOW A PAYMENT THROUGH THE GREYVALLEY CORREDOR WORKS

> ⚠️ **Real correction (updated 2026-09-04, see §6 below for the
> full detail):** the ORIGINAL design of this section (a
> `koywe_bridge` canister with an EVM wallet and its own minting)
> was never built and `src/koywe_bridge/main.mo` **does not exist**. But
> that does NOT mean there is no real code — the path that does exist and
> is deployed on mainnet (`api_gateway` → `settlement` → `bridge` → `liquidity_pool`)
> is different and simpler, with no EVM wallet or minting. Today it is
> **switched off via feature flag** (`bridge=false`) and Koywe is not yet
> registered as an institution — not for lack of code, but because commercial
> KYB is still pending
> (`INSTRUCCIONES_FOUNDER.md` §3). Do not use the word "active" for the
> corredor with Koywe until KYB is approved and the flag is turned on.
> Native sCLP (Pegasus SpA + Fintoc, §7) remains the only variant
> blocked by CMF (not by code) — it runs in sandbox.

### Remittance flow (Chile → abroad) — real code deployed, switched off until Koywe approves KYB

```
1. User deposits CLP with Koywe (Koywe's own custody and KYB — does NOT
   touch any GreyValley canister; CLP has no on-chain representation
   without sCLP)
2. The Exaltite Vault (ckUSDC pool funded by depositors) releases the
   equivalent ckUSDC instantly on-chain — it does not wait for the CLP
   from step 1 to be "converted"; inventory-first pattern: the pool pays
   with existing inventory
3. ckUSDC travels on the ICP chain (~2 seconds)
4. On arrival: Koywe redeems the ckUSDC (ICP's official minter → real USDC,
   or its own rails) and settles into local currency (USD, MXN, EUR, etc.)
5. Recipient receives funds
6. The CLP from step 1 (minus fees) is what eventually rebalances
   GreyValley's ckUSDC pool — there is no atomic CLP→ckUSDC conversion
   per transaction

GreyValley fee: 0.66% + ~1% ramp (total to user: ~1.7%)
Equivalent SWIFT fee: ~2.5–3.5%
```

### The pool doesn't "run dry"

The vault's ckUSDC is not consumed by each transaction — it is used as **instant liquidity collateral**. The real flow:

- ckUSDC enters the pool when there are payments in the reverse direction (inbound to Chile)
- ckUSDC leaves when there are payments headed abroad
- If the flow is **bidirectional**: the pool self-replenishes, capital stays intact permanently
- If the flow is **unidirectional**: the pool can become unbalanced → the protocol rebalances using accumulated fees (0.66% × volume)

**The depositor's capital NEVER disappears.** If the pool becomes severely unbalanced, the bridge pauses — but the capital remains accessible for withdrawal.

### Real capacity vs. TVL

```
$20K ckUSDC in the vault → $20K of instant capacity (simultaneously in flight)
But ICP settles in ~2 seconds:
→ a $20K pool can process $500K-$1M+ in monthly volume
   (as long as no more than $20K is in flight at the same instant)

The real risk: extreme unidirectional flow, not the size of the pool
```

### 2.1 Designed destination variants (post-CMF / Chanfusion Satellite)

Migrated from `GREYVALLEY_LIVING_SPEC.md` (deleted 2026-08-07). These diagrams describe the
route with sCLP via Chanfusion Satellite — **out of scope while sCLP remains blocked
by CMF**, not the active Koywe flow from section 2. Kept as a reference design
for when that corridor is enabled.

> **Do not confuse with the real CLP/EUR corridor (2026-09-04).** The "Chile →
> Europe" diagram below describes a FUTURE route via sCLP (blocked
> by CMF, with no sCLP↔ckUSDC↔EUR code built). The CLP/EUR corridor
> that DOES exist today (`liquidity_pool` CLP_EUR pair, `bridge_canister`
> `depositEurcToCkEurc()`) is a different, simpler path: real
> ckEURC (Circle EUR, DFINITY ledger) enters directly via tECDSA from an
> EVM wallet, without going through sCLP or Chanfusion. Both paths can
> coexist in the future (sCLP for the custodied CLP side, ckEURC for the
> already-resolved EUR side), but today only the second one has real
> custody.

**Chile → Europe (future design, sCLP — blocked by CMF)**
```
1. CLP → Fintoc Webhook → Chanfusion
2. Canister mints sCLP 1:1
3. AMM: sCLP → ckUSDC (mindicador.cl oracle)
4. Chain Fusion: native USDC on Ethereum
5. European partner: USDC → EUR → bank
6. Burn sCLP on-chain — closed cycle in <60s
```

**Chile → LATAM**
```
1. CLP → Khipu → Chanfusion → sCLP
2. AMM: sCLP → ckUSDC
3. Koywe API: USDC → COP/PEN/ARS/BRL
4. Bank deposit in destination country
5. Burn sCLP
```

---

## 3. COMPARISON: GREYVALLEY vs XRP/RIPPLE vs SWIFT

### The Ripple/XRP model

Ripple uses XRP as a bridge asset between market makers (Bitso, SBI Remit, etc.):

```
US company → USD → MM buys XRP → XRP travels → Mexican MM sells XRP → MXN → recipient
```

Market makers hold **XRP inventory on both sides of the corridor**. That inventory IS their corridor liquidity. They earn the spread + a per-transaction fee.

**The problem:** XRP is volatile. If XRP drops 40% while the MM holds the inventory, it loses on principal. Only large entities with tolerance for that volatility become serious MMs.

### Comparison table

| Dimension | SWIFT | XRP/Ripple | GreyValley |
|-----------|-------|-----------|--------|
| Bridge asset | None (direct nostro) | XRP (volatile) | ckUSDC (stable $1) |
| Speed | 1-5 business days | 3-5 seconds | ~2 seconds |
| Typical user fee | 2.5-3.5% | 0.3-0.5% | ~0.2-0.4% |
| MM risk | Low (fiat) | High (XRP volatility) | Very low (stable ckUSDC) |
| Entry for new MMs | Private Ripple agreement | High barrier | Deposit ckUSDC in the vault |
| Governance | Banking / bilateral | Centralized by Ripple Inc. | On-chain, transparent |
| Initial corridor | Global banking | USD↔MXN, USD↔PHP, etc. | CLP↔USD (LATAM first) |
| Yield for the MM | None extra | Spread/fees only | Vault APR + Volume Guild |

### Why ICP is better positioned than XRP for a global corridor

XRP was born as a speculative asset and was adapted as a bridge. ICP was born as distributed compute infrastructure with:

1. **~2-second finality** (vs. 3-5s for XRP, with no reversals)
2. **Native HTTPS outcalls** — the canister can call banking APIs directly (Koywe, Fintoc, SWIFT MX, etc.) with no external middleware
3. **Stable compute cycles** — the cost of processing a corridor transaction doesn't fluctuate with the price of ICP
4. **Native ckAssets** (ckBTC, ckUSDC, ckETH) — 1:1 representations of real assets within ICP, with no additional bridging
5. **Canisters = code that cannot be switched off** — the corridor cannot be deactivated by a central bank or regulator

The analogy: XRP is a private highway built on borrowed land. ICP is a public highway that also serves as a decentralized internet layer — the corridor is just one of its use cases.

---

## 4. THE BUSINESS-PARTNER-AS-MARKET-MAKER MODEL

The key insight of this document: **GreyValley's business partners are not "investors" — they are market makers of the CLP corridor**.

### What a market maker does on Ripple

- Bitso (Mexico) deposits XRP on both sides of the USA↔MX corridor
- When someone sends $1,000 USA→MX, Bitso executes the conversion instantly
- Bitso earns the spread (XRP buy/sell difference) + a commission
- The size of its inventory determines its volume capacity

### What a business partner does on GreyValley

- The company deposits ckUSDC into the Crypto Vault
- That ckUSDC is the CLP↔USD corridor's inventory
- When someone sends CLP to Mexico/USA, the ckUSDC pool executes instantly
- The partner earns: PXRM Base APR from the vault (paid in PXRM, not ckUSDC) + AMM/corridor fees proportional to its liquidity (variable, based on real volume) + Volume Guild if >$25K/month through the corridor

### The double advantage of a partner that also uses the app for its own payments

A company that deposits ckUSDC AND routes its own international payments through the app gets
up to 3 income sources — **none of them is a fixed guaranteed amount, and all but one
depend on the real volume passing through the corridor**:

```
Income 1 (semi-fixed, treasury-subsidized):
  PXRM Base APR — 12-24% depending on lock, PAID IN PXRM, not ckUSDC.
  Depends on the treasury (bootstrap) until real fees can sustain it alone.

Income 2 (variable, depends on real volume):
  AMM fees on the pooled ckUSDC pair (feeBps=33 today, ~0.33%/swap)
  — 0 if there's no volume through that pool.

Income 3 (variable, depends on real corridor volume):
  Proportional share of your ckUSDC/TVL over the corridor's 0.66%
  (Volume Guilds, 7% of the fee pool) — ONLY if the corridor processes
  >$25K/month in real volume. Today the corridor has ~0 volume (see
  GREYVALLEY_AUDIT_LIVE.md); this income is illustrative of the
  MECHANIC, not a projection of what is charged today.

Savings: your own payments through the corridor cost 0.66% instead of
  2.5-3.5% SWIFT — this one IS real and immediate, it does not depend
  on third parties.
```

**None of the numbers above is "deposit $100, get $X/year guaranteed."** The only income
with a semi-predictable floor is the PXRM Base APR (paid in PXRM, treasury-subsidized
while real fees don't yet sustain it alone) — everything else (Income 2, Income 3) is
strictly proportional to the real volume passing through the AMM/corridor, which is
low or zero today. The right pitch is not "we pay you a fixed X%" — it's "be the Bitso of
Chile: your ckUSDC inventory (stable, no price risk) captures value from EVERY real
transaction that passes through, and grows with volume, not with a promise of fixed
returns."

---

## 5. BOOTSTRAP CAPITAL — HOW MUCH IS NEEDED

### Minimum viable (founder only)

| Vault | Minimum demo | Credible launch |
|-------|------------|---------------------|
| Exaltium (ICP) | $15K | $40K |
| Crypto (ckBTC/ckETH) | $5K | $15K |
| **Exaltite (ckUSDC/ckEURC)** ← critical | **$20K** | **$50K** |
| Total personal TVL | $40K | $105K |

> Naming corrected 2026-08-03: Puranium is the PXRM staking canister, separate from these 3 vaults — it is not "the ICP vault." See `GREYVALLEY_APR_MODEL.md` §12 for the full mapping.

**Treasury for PXRM Base APR (6 months, minimum TVL):**
- Gross cost: ~$2,000 in PXRM value
- NNS staking of the ICP generates ~$500-700/month (reduces the real cost)
- Effective net cost: ~$1,000-1,500 for 6 months

**ckUSDC/ckEURC is the bottleneck:** without it there is no corridor capacity and the Exaltite Vault's APRs stay at `"--"`.

### With business partners (scalable)

| Partner profile | Own payments/month | Suggested vault TVL | Savings vs. SWIFT | Guild |
|-------------|------------------|-----------------------|----------------|-------|
| Small (~$15K company payments) | $15K | $10-25K | ~$375/month | No |
| Medium (~$30K company payments) | $30K | $25-75K | ~$750/month | No |
| Large (~$75K company payments) | $75K | $75K-300K | ~$1,875/month | ✓ Yes |

**Suggested fundraising structure, phase 1:**
- 3 Angel partners ($15K TVL each) → $45K external TVL
- 1 Seed partner ($40K TVL) → $40K TVL
- 1 Strategic partner ($100K TVL) → $100K TVL + active Guild
- Total external: ~$185K TVL → the protocol can self-sustain the PXRM Base APR

### Revenue projection (from APR Model V3)

| Monthly Corridor | Corridor Fees | NNS Yield | Swap+CDP | Total/month |
|-------------|---------|-----------|---------|----------|
| $50K | $250 | $600 | $250 | ~$1,100 |
| $200K | $1,000 | $1,200 | $500 | ~$2,700 |
| $500K | $2,500 | $2,000 | $1,200 | ~$5,700 |
| $1M | $5,000 | $3,000 | $2,500 | ~$10,500 |
| $2M+ | $10,000 | $4,000 | $5,000 | ~$19,000 |

---

## 6. KOYWE — Real technical architecture of the integration

> **Updated 2026-09-04** — the original design of this section (session
> 2026-08-02) described a `koywe_bridge` canister with its own EVM wallet
> and ckUSDC minting via tECDSA. **That design was never built and is no
> longer the real path** — `src/koywe_bridge/main.mo` does not exist, 0 lines of
> code, no canister deployed (confirmed on the repo's filesystem).
> The real integration that IS deployed and working is simpler:
> Koywe (or any registered institution) calls an already-built generic
> HTTP endpoint — `api_gateway` → `settlement` → `bridge` →
> `liquidity_pool` — with no canister dedicated to Koywe, no EVM wallet,
> no new minting. Rewritten to reflect this.

### Key question: does Koywe need to install ICP code?

**NO.** Koywe is pure web2 (REST API). It installs nothing, integrates no
ICP SDK. All the integration lives on GreyValley's side, in
canisters that are **already deployed on mainnet**:

```
Koywe:      its own web2 REST API (PAYIN / ONRAMP / OFFRAMP / PAYOUT)
GreyValley: api_gateway (webhook + institution registry)
              → settlement (payment tracking)
              → bridge (execute_bridge — real engine: oracle price + FIFO queue)
              → liquidity_pool (swap() against the already-pooled ckUSDC/ckEURC inventory)
```

The four canisters above are **already deployed and in production**
(see `GREYVALLEY_MASTER_STATE.md` §1) — there's nothing technical
left to build for Koywe to start sending payments, all that's missing is:
1. Koywe's KYB approval (documents §3 of `INSTRUCCIONES_FOUNDER.md`).
2. Registering Koywe as an `Institution` in `api_gateway` (`registerInstitution()`,
   admin-only) — API key, authorized corridors (e.g. `CLP_USD`, `CLP_EUR`),
   daily limit in CLP.
3. Turning on the `bridge` feature flag (currently `false` — see `getFeatureFlags()`).

### Koywe API — relevant endpoints (Koywe side)

| Endpoint | Function |
|----------|---------|
| `POST /v3/deals` | Create an ONRAMP order (CLP → USD). Parameters: amount, fromCurrency, toCurrency, network, destinationAddress |
| `GET /v3/deals/{id}` | Query order status |
| `POST /v3/payouts` | OFFRAMP / PAYOUT — convert to CLP and send to a destination bank account |

Contact: `business@koywe.com` (BD and technical agreements) | [koywe.com/partners](https://koywe.com/partners)

### Real endpoint on the GreyValley side (already built)

```
POST api_gateway.raw.ic0.app/v1/payments

Headers: Authorization with the institution's API key (SHA256 hash
         compared server-side — the key is never stored in plain text)
Body:    { institutionId, sourceAmount, sourceCurrency, destCurrency,
           destAccount, destInstitution }

Real internal flow (api_gateway/main.mo, line ~250 onward):
  1. Validates the `bridge` feature flag (backend.getFeatureFlags()) — if
     it's off, responds 503 "Corridor disabled."
  2. Validates the API key against the institution's stored hash (401 if it fails).
  3. Validates that the institution is authorized for THAT specific
     corridor (source_dest, e.g. "CLP_USD") — 403 if not.
  4. Validates the daily CLP limit (429 if exceeded; automatic reset on
     calendar day change).
  5. Registers the payment in `settlement.initiate_payment()` (real
     tracking, not a placeholder).
  6. Calls `bridge.execute_bridge()` — this IS the real engine: it uses
     the oracle price (mindicador.cl) + the same inventory model and
     FIFO queue (`#Queued`) already running in `liquidity_pool.swap()`
     for retail users. Responds #Success, #Queued, #InsufficientLiquidity,
     #UnsupportedCorridor, #SystemNotLive, or #Error — never fabricates
     a result.
```

**There is no new ckUSDC/ckEURC minting in this flow.** The corridor uses
the inventory that ALREADY exists — ckUSDC/ckEURC pooled by the Exaltite
Vault's depositors (see §2 of this document, "inventory model"). Koywe
custodies the CLP and delivers the destination currency on its own side
(its own off-ramp, outside of ICP) — GreyValley never touches that leg
or the CLP.

### Real status (verified against the filesystem and code, 2026-09-04)

| Component | Status |
|------------|--------|
| `api_gateway` (webhook + institution registry) | ✅ **Deployed on mainnet**, `ua27v-4yaaa-aaaah-quyfa-cai` |
| `settlement` (payment tracking) | ✅ **Deployed on mainnet**, `yknil-6yaaa-aaaah-quzja-cai` |
| `bridge` (real corridor engine) | ✅ **Deployed on mainnet**, `us4im-qiaaa-aaaah-quyga-cai` — feature flag `bridge=false` today |
| `liquidity_pool` (swap against real inventory) | ✅ **Deployed on mainnet**, `uv5oy-5qaaa-aaaah-quygq-cai` |
| `src/koywe_bridge/main.mo` (original 2026-08-02 design) | ❌ **Does not exist — never built, no longer the plan** |
| Koywe registered as a real `Institution` | ❌ Pending — depends on KYB (§3 `INSTRUCCIONES_FOUNDER.md`) |
| Koywe KYB (documents, sandbox, production) | ⏳ Pending |

> For the technical implementation of HTTPS Outcalls, tECDSA, and ICRC-1 (used
> in OTHER canisters of the protocol, not in this flow): see `GREYVALLEY_ICP_TECH.md`

### 6b. Two different routes — do not confuse them

Two possible routes were designed for Koywe-type partners. Only one has
real code today.

**Route A — Individual on/off-ramp with direct mint/burn to the user**
(2026-08-02 design, original §6 section of this document):
would require an EVM wallet custodied via tECDSA that receives real USDC
and mints ckUSDC 1:1 to the user only after verifying arrival via an HTTPS
Outcall. **It has no code — it's an unbuilt design**, different from
the one that actually runs today.

**Route B — Remittance via corridor with already-pooled liquidity (the real
one, the one described in §6 above):** it mints nothing new — it uses the
ckUSDC/ckEURC already pooled by Exaltite depositors as the "destination"
side of the transaction, using the same inventory + oracle model as
`liquidity_pool.swap()`. The partner (Koywe) receives the client's CLP on
its own side and delivers the destination currency directly — there is
no replenishment back to a "GreyValley EVM wallet" because that wallet
does not exist in this design; the cycle simply closes with the CLP that
Koywe keeps as its own business margin.

**What happens if the pool runs out of inventory**: a real FIFO queue
already exists in `liquidity_pool.swap()` (`#queued`, `drainQueue()`) for
this — it never executes at a worse price nor rejects, it waits. **Real gap
identified 2026-09-01, no code yet**: today the pool drains literally to
`foreignReserve = 0` before queuing — there is no reserve floor
(unlike the vaults, which do have `checkBufferFloor` at 5% of
NAV). Design pending: add a configurable floor (10-15% of the
pool) that starts queuing BEFORE hitting real zero, so that
the staker who put in the first ckUSDC never sees the pool completely empty.

**What happens if the partner is slow to replenish (steps 5-6)**: the pool
can run dry before the replenishment arrives. A real FIFO queue
already exists in `liquidity_pool.swap()` (`#queued`, `drainQueue()`) for
this — it never executes at a worse price nor rejects, it waits. **Real gap
identified 2026-09-01, no code yet**: today the pool drains literally to
`foreignReserve = 0` before queuing — there is no reserve floor
(unlike the vaults, which do have `checkBufferFloor` at 5% of
NAV). Design pending: add a configurable floor (10-15% of the
pool) that starts queuing BEFORE hitting real zero, so that
the staker who put in the first ckUSDC never sees the pool completely empty.

**Self-funding as an additional mitigant** (does not replace the floor
above, it complements it): the real minimum already documented in
`TOKENOMICS.md` (Exaltite Vault ≥$50K ckUSDC for day-1 corridor capacity)
makes running dry rare in practice — but it is idle money, not a
code-level guarantee.

**Future variant of Route B (v3, with CMF approval)**: instead of
the partner handling the CLP off-chain (as Koywe does today), the CLP
side could come from **sCLP → ckUSDC/ckEURC**, with GreyValley custodying
the real CLP (via Pegasus SpA + Fintoc, see §7 below) instead of
depending on an external partner for that leg. This is strictly subsequent
to CMF approval of the regulatory sandbox — today Route B runs 100% with
an external partner (Koywe), with GreyValley never touching CLP at any point.

**Why Route B is regulatorily lighter than sCLP alone**:
GreyValley never custodies CLP in this route — this matches exactly the
"Pillar 3" of the current regulatory defense (`GREYVALLEY_REGULATORY.md`):
"GreyValley never touches Chilean pesos." The v3/CMF variant above would
cross that line — which is why it is explicitly marked as future and
conditional on approval, not part of the current design.

**Standalone sCLP** (§7 below) — blocked by CMF. Its own CLP custody
with no dependence on any partner — it's the piece a future variant
of Route B would use once it exists, but it's a separate development,
already documented in §7.

---

---

## 7. MULTI-FIAT CUSTODY — sCLP / ckBRL / ckARS / ckMXN (2026-08-15)

Founder's question: to scale the corridor beyond CLP, do we look for new
partners per country, or build our own custody (GreyValley/founder) in each
fiat? This section records the analysis and the recommendation.

### What already exists (sCLP, the only corridor with real code)

`src/sclp_ledger/main.mo` + `src/sclp_treasury/main.mo` implement the full
pattern: CLP deposit into **Pegasus SpA's** bank account (self-custody,
the founder's Chilean company) → detected via **Fintoc** (Chilean Open Banking
API, HTTPS outcall polling every 60s) → 1:1 sCLP mint → reconciliation every
10 min against the real bank balance (circuit breaker: `reconciliationOk`).
Requires CMF approval (Ley 21.521) before handling real money; the code can
exist and be tested in sandbox without that approval (see §6 of this
document on the Koywe/CMF pattern, and `GREYVALLEY_CORREDOR_GREYVALLEY_SANDBOX.md`
if documented separately).

This is viable because Pegasus SpA **is already a real Chilean entity** —
Fintoc only operates over Chilean and Mexican banks, so the missing piece
for CLP is regulatory (CMF), not banking or code.

**Real gap found 2026-08-24 — off-ramp (withdrawal) with no real payout:**
`sclp_treasury.requestRedeem()` burns the user's sCLP and logs the
request, but never triggers the wire transfer (TEF) back to the bank — the code
only had the mint (on-ramp) side complete. The real Fintoc endpoint
(`POST /v2/transfers`) was investigated to complete this: it requires a
**per-request JWS signature** (JSON Web Signature), a mechanism different
from and more complex than the `Authorization: apiKey` already used by
the movement/balance polling. Needs the founder to generate the signing
keys in his Fintoc dashboard before this can be implemented — see
`INSTRUCCIONES_FOUNDER.md` §7.2 for the full detail and next step.

### What would be needed for BRL / ARS / MXN

| Fiat | Fintoc coverage | What self-custody requires | Viability |
|------|------------------|-------------------------------|------------|
| **ckMXN** | Yes (Fintoc covers Mexico) | A real bank account in Mexico under an entity's name — the founder's or a Mexican partner's | Technically the shortest of the three: same pattern as sCLP, same provider (Fintoc), but requires opening a bank account/entity in Mexico — not just deploying a canister |
| **ckBRL** | No — Fintoc doesn't cover Brazil | A Brazilian Open Banking provider (e.g. Belvo, Pluggy — both with Pix support) + a bank account in Brazil + registration with BACEN if operating as a payment institution | Self-custody implies full Brazilian regulatory compliance — high cost to operate solo |
| **ckARS** | No viable equivalent | Any USD/ARS custody runs into BCRA exchange controls (*cepo cambiario*) | The hardest of the three — not recommended as a next step while the *cepo* persists |

### Recommendation

**Fully self-custody (founder/Pegasus SpA) is only realistic for CLP** — it's
already built and is the natural extension of a company the founder already
controls. For BRL and MXN, the fast path is not to rebuild the same custody
triangle in 3 more countries (own bank + license + local compliance in each),
but to **look for already-licensed local partners** — EMIs or PSPs with a
bank account and their own regulatory authorization in their country — playing
the same role Pegasus SpA plays for CLP. The canister side
(`sclp_ledger`/`sclp_treasury`) is reusable as a template per fiat (same
mint/burn/reconcile pattern), only changing the Open Banking provider and the
bank custodian behind it.

ARS stays off the near-term roadmap until exchange restrictions change —
this is not an architecture problem, it's an external regulatory problem
that no technical integration solves.

---

## 8. PENDING TECHNICAL QUESTIONS (V1)

1. **NNS Neuron staking on behalf of the user:** can a GreyValley canister stake the ICP deposited into the NNS on behalf of the user? Or does the canister "own" the neuron and manually distribute the yield? → Investigate custody.

2. **T1 streaming implementation:** T1 yield accrues second by second. In Motoko, this implies either a very frequent timer or lazy calculation at harvest time. Which is more efficient in ICP cycles?

3. **Automatic unidirectional rebalancing:** can the protocol execute an automatic ckUSDC→ICP→ckUSDC swap to rebalance without manual intervention, using accumulated fees?

4. **ckUSDC on ICP vs. real USDC:** ckUSDC on ICP is a 1:1 representation of USDC on Ethereum via Chain Fusion. Is there any delay or depegging risk under extreme market conditions?

---

*Document: corredor-mechanics.md (formerly GREYVALLEY_ODL_MECHANICS.md) | 2026-06-28 · Updated: 2026-09-04 (renamed, "ODL" references removed, real fee 0.66% (verified on-chain 2026-09-18), CLP/EUR corridor added as real)*
*Related: GREYVALLEY_APR_MODEL.md, TOKENOMICS.md §10, GREYVALLEY_ICP_TECH.md (Principal/HTTPS Outcalls/tECDSA mechanics)*
