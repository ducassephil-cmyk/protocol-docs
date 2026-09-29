# GreyValley Protocol — Tokenomics & Features
> Version: 2026-07-01 | APR Model V3 + Formal Supply + vUSD Institutional Track + Guild tiers

---

> ⚠️⚠️ **REAL CORRECTION 2026-08-30 — Koywe is NOT an active partner.**
> Everything this document describes about Koywe (integration, delegated
> KYC/AML, "current Phase 1", PISP accreditation, etc.) is the **strategy
> design** for when such a fiat partner exists — today there is no real
> relationship with Koywe, not even commercial contact. The technical
> webhook on the GreyValley side is built and ready, but it does not point
> to any confirmed partner yet. Read the sections below as a reference
> plan, not as current operational status.

## 1. Ecosystem Tokens

### PXRM (Puranium) — Main Token
| Parameter | Value |
|-----------|-------|
| Ledger canister | `q7nmw-diaaa-aaaah-quy4a-cai` (MAINNET) |
| Standard | ICRC-1 + ICRC-2 |
| Decimals | 8 (1 PXRM = 100,000,000 e8s) |
| Canister name | Puranium / PXRM (correct symbol and name since init) |
| Model | **Deflationary** — burned on PXRM→ICP swap + enterprise boost burn |
| **Total supply** | **5,000,509.85 PXRM** as of 2026-08-30 (real `icrc1_total_supply`, not fixed) |
| Use | Vault yield (boost), welcome minipack, swap, staking rewards, Guild rewards |

> Historical note: the original ledger (`5zqoe-hqaaa-aaaaj-qrupa-cai`) was compromised
> (lost minting key, ~20M phantom supply) and was replaced on 2026-08-09 by the ledger
> above. Full detail in `GREYVALLEY_MASTER_STATE.md` §29.

> ⚠️ **Real correction 2026-08-30**: this doc used to say "fixed, never increases" —
> that is no longer true. The founder minted 500 new PXRM (real, on-chain) as a
> welcome fund for the closed tester pilot. It is a deliberate, small exception
> (0.01% of supply), not a change to the model.

**Supply distribution — Vesting Schedule (2026-08-01):**

Locked tokens — released linearly after the cliff. Formal vesting supply: 5,000,000 PXRM (does not include the tester fund above).

| Group | Tokens (PXRM) | Cliff | Vesting | TGE Unlock |
|---|---|---|---|---|
| Protocol Treasury | 2,000,000 | — | Governance-controlled | 5% |
| Team / Founders | 1,000,000 | 12 months | 36 months linear | 0% |
| Ecosystem / Guilds | 1,000,000 | — | 36 months (emission) | 10% |
| Staking Rewards | 500,000 | — | Per epoch / APR | 0% |
| DAO Governance | 500,000 | — | Community vote | 0% |

> Note on Treasury: "Governance-controlled" is the general mechanism — within that, the portion earmarked for PXRM Base APR (+ Guild Multiplier for Track B) on vaults follows the semi-annual Sunset Clause schedule (`GREYVALLEY_APR_MODEL.md` §10: Year 1 boost 100%, Year 2 drops to 50% if real yield exceeds 5% and ODL exceeds $500K/month, Year 3 can reach 0%). The 5% TGE Unlock is separate from that schedule — initial liquidity available from day 1. **The 2,000,000 PXRM is not a guaranteed expense** — it is a ceiling that shrinks if the protocol generates enough real fees ahead of schedule; with an early-triggered sunset clause, a large part of that 40% of supply goes unspent.
> Note on Staking Rewards (500K PXRM): this is a supply pool separate from the real fee split that stakers receive in ICP/ckUSDC/ckBTC/PXRM (PxrmStakers bucket of the `fee_splitter`) — it doesn't replace it, it's an additional reinforcement in PXRM. **Real since 2026-09-08** ("PXRM Staker Boost", `staking/main.mo`): a real 6-day timer distributes proportionally to `pxrmStaked × lock-based rate` (1% Flex / 4% 90d / 6% 180d / 10% 365d annualized) among all active stakers, funded from the `\04` subaccount. Manual Sunset Clause, with longer triggers than vaults (120d stable + 6-month corridor, annual review). Funded with 10,000 real PXRM on launch day, ~390,000 PXRM remain undeployed in the subaccount.
>
> ⚠️ **Team/Founders (1M PXRM, 20%) and Ecosystem/Guilds (1M PXRM, 20% —
> outside the Genesis Round reserve below) still have no real on-chain
> mechanism**, unlike Treasury and Staking Rewards (real subaccounts
> above). Verified 2026-09-09: no subaccount, lock, cliff, or vesting
> contract exists for either — the Team/Founders 12-month cliff and the
> Ecosystem/Guilds 36-month schedule are, today, only the intent
> documented here. The only piece of Ecosystem/Guilds with real code is
> the Genesis Round reserve (`genesis_registry`, real linear vesting per
> co-founder, see below) — the rest of the bucket and 100% of
> Team/Founders remain unassigned to any subaccount or contract to date.

**Launch price (v2, 2026-07-31):**
- Fixed parity: **1 PXRM = ICP / 10** — real mechanism in `swapPXRMtoICP` (backend/main.mo) and mirrored in the oracle (`oracle.getPxrmPegRatio()`, adjustable without redeploy via `setPxrmPegRatio`, admin-only). There is no AMM yet, so this fixed swap is the only real price.
- Example with ICP at today's real price (~$2.07): 1 PXRM ≈ **$0.207** → launch FDV ≈ **$1.03M** (5,000,000 PXRM × $0.207)
- The PXRM price moves 1:1 with ICP's real price (via the XRC oracle) — it is not a fixed USD number, it scales with the market.

**Target FDV (adoption, unchanged):**
- LatAm adoption: $100/PXRM → $500M FDV
- Banking adoption: $500/PXRM → $2.5B FDV

> Correction 2026-07-31: the previous version of this section said "1 PXRM = 1 ICP (~$12) → $60M FDV" — outdated parity and ICP price. The real parity changed to ICP/10 (previously 1:1) and ICP's price today is ~$2.07, not ~$12 — the real launch FDV is ~60x lower than what was documented before.

---

## 1b. Genesis Round — Pre-TGE Co-Founders (2026-08-03)

The Genesis Round is the TVL-raising program before the TGE. Participants are **external co-founders** — actors who deposit real capital (ckUSDC) into the vaults and receive a PXRM allocation with vesting in return. **They are not the GreyValley team** (that is the Team/Founders bucket).

### Where does the PXRM for the Genesis Round come from?

From the **Ecosystem / Guilds bucket (1,000,000 PXRM)** — that is exactly its function: early adopters and co-builders of the protocol. The Team/Founders bucket (1M PXRM) is exclusive to the internal team (12-month cliff) and is not touched for external participants.

- **Genesis Round reserve:** ~200–350K PXRM (20–35% of the Ecosystem bucket)
- **Remaining for post-TGE Guilds / ecosystem:** ~650–800K PXRM

> **Second track (2026-09-12) — "Investors Guild"**: a separate mechanism
> (own canister in the future, not `genesis_registry`) that also draws
> from the Ecosystem/Guilds bucket — flat 2x return (no tiers), $500
> minimum, 9-month vesting, 200K PXRM cap (regulable). Added to the first
> track's cap (300K), the real total reserved is around ~500K PXRM, not
> the ~650-800K implied by the invariant above — that invariant
> (≤350K/≥650K) applies ONLY to the first track (co-founders). 100%
> design, zero canister built yet.

### Co-founder tiers

| Tier | Ideal count | Min TVL each | PXRM allocated each | Vesting | Additional benefits |
|------|---------------|----------------|-------------------|---------|------------------------|
| 🔵 **Angel** | 3–4 | $10K–$25K ckUSDC | 15K–25K PXRM | 6m post-TGE linear | Early access, Corridor Guild priority |
| 🟣 **Seed** | 2–3 | $25K–$75K ckUSDC | 40K–70K PXRM | 9m post-TGE linear | Active Corridor Guild, co-branding |
| 🩷 **Strategic** | 1–2 | $75K–$200K ckUSDC | 80K–150K PXRM | 12m post-TGE linear | Advisory seat, Protocol Treasury voting, dedicated API |

> **Invariant rule:** no Genesis co-founder has a TGE unlock — all allocated PXRM has post-TGE vesting. This eliminates sell pressure on launch day.

> ⚠️ **The table above is the original design at large TGE scale.** The
> real model deployed on mainnet today (`genesis_registry`, updated
> 2026-08-31) is different: **10 slots** (Angel 4 / Seed 3 / Strategic 3
> — the 3rd Strategic slot reserved for whoever also contributes real
> liquidity to the Corridor and tests it live), **cap of 300,000 PXRM**
> (30% of the bucket, within the already-approved 20-35% range above),
> and a mechanism of **% capital guarantee at PXRM's live oracle price
> at the moment of registration** (80/75/70/65% by order of entry, safety
> floor 1.0 PXRM/$ minimum) instead of the fixed PXRM/$ rate from the
> table above. Any operational loss on the special liquidity slot is
> reimbursed in PXRM via a dedicated admin function, separate from
> vesting.

### Fundraising scenarios

| Scenario | Composition | Genesis TVL | PXRM allocated | % supply | TVL/FDV |
|-----------|------------|-------------|---------------|----------|---------|
| **Minimum viable** | 2 Angel + 1 Seed | ~$70K | ~70K PXRM | 1.4% | 6.8% |
| **Credible launch** ✅ | 3 Angel + 2 Seed + 1 Strategic | ~$295K | ~310K PXRM | 6.2% | 28.5% |
| **Fast scale** | 4 Angel + 2 Seed + 1 Strategic | ~$320K | ~345K PXRM | 6.9% | ~31% |

**Recommended scenario: Credible launch.** Activates the full ODL corridor (Vault Exaltite ≥$50K ckUSDC), TVL/FDV = 28.5% (excellent for DeFi early stage), PXRM genesis ≤ 8% supply.

> "Fast scale" is deliberately close to the ceiling of the invariants below (7 of a maximum 8 co-founders, 345K of a maximum 350K PXRM) — there's no room to go more aggressive without breaking the concentration/coordination limit. If real demand appears in practice for more than 8 co-founders or more than 350K PXRM, that requires revisiting the invariants themselves (a founder decision), not forcing a fourth scenario that breaks them.

### Healthy TGE Criteria

1. **Day 1 TVL ≥ $105K** — minimum for real corridor capacity (Vault Exaltite needs $50K ckUSDC/ckEURC). See `corredor-mechanics.md` §5.
2. **PXRM Genesis ≤ 8% of supply** — above this it starts pressuring price post-vesting.
3. **TGE float: ~200K circulating PXRM** (4% supply) — only Ecosystem TGE unlock (100K) + Treasury (100K). Initial market cap ~$41K vs TVL $295K → 7x ratio.
4. **Mandatory vesting for all Genesis participants** — no TGE unlock, no cliff dump.
5. **Number of co-founders: 5–8** — fewer is risky concentration; more than 8 makes pre-TGE coordination impractical.

### The Strategic co-founder: the ODL pivot

A single actor with $100K+ ckUSDC activates the full ODL Corridor and gives the protocol immediate credibility. The ROI for them: ~2.5% savings on SWIFT fees on that capital (~$2,500/month in active use) + PXRM exposure with upside. Ideal profile: a Chilean importer/exporter already moving that monthly figure in international payments.

### Genesis Round invariants

```
PXRM Genesis ≤ 350K  →  Ecosystem bucket retains ≥ 650K for post-TGE
TVL Genesis ≥ $105K  →  ODL viable on day 1
Minimum vesting: 6m   →  Zero dump at TGE
Number: 5–8          →  Distribution + viable coordination
```

### Genesis Pool Sizing — ICPSwap liquidity and price protection

The PXRM/ICP pool on ICPSwap is the token's price infrastructure. Its depth determines how much sell pressure the market can absorb before the price drops.

**Origin of each pool component:**

| Component | Where it comes from | Required capital |
|---|---|---|
| PXRM in the pool | Treasury (Protocol Treasury bucket) | $0 additional — already exists |
| ICP in the pool | Founder (personal) + Genesis round (co-founders' ICP) | Real capital |

**The treasury's PXRM never costs additional capital.** The ICP does — it has to come out of a real pocket. That's why the correct model is mixed: the founder seeds the minimum viable amount, the genesis round deepens it.

**Recommended pool model:**

```
Phase 0 — Pre-genesis (founder only):
  50,000 PXRM (treasury) + 2,500–5,000 ICP (founder personal)
  Pool value: ~$20K–$40K | Pool/monthly-emission ratio: 8–16×

Phase 1 — Post-genesis (founder + co-founders):
  100,000 PXRM (treasury) + 10,000 ICP (combined)
  Pool value: ~$80K | Pool/monthly-emission ratio: 33×  ← stable target
```

**Why not put in $80K of the founder's own ICP:**
The founder as sole LP assumes impermanent loss if PXRM and ICP diverge. If PXRM falls 50% relative to ICP, the LP position is worth ~30% less than holding pure ICP. That capital is better used as an operational reserve or as a deposit in Vault Exaltium (earns yield and signals confidence to the market).

**Narrative for the genesis round:**
*"Your contribution not only gives you PXRM with upside — it also funds the market liquidity that protects the value of that PXRM."* Co-founders have a direct incentive in the pool being deep because that stabilizes the price of their own vesting position.

**Pool launch price:**
1 PXRM = 0.1 ICP at seeding. **This is not a peg** — it's the initial seeding ratio. From the moment the pool launches, the ICPSwap AMM determines the free-market price based on supply and demand. The protocol's internal `swapPXRMtoICP` mechanism keeps the 1/10 ICP peg only for that specific swap (with a 2%/day cap) — it does not determine the overall market price.

**Price projection without protection vs. with a deep pool:**

| Month | $20K Pool (founder only) | $80K Pool (founder + genesis) |
|---|---|---|
| 0 | 0.100 ICP | 0.100 ICP |
| 1 | 0.086 ICP | 0.096 ICP |
| 3 | 0.072 ICP | 0.088 ICP |
| 6 | 0.059 ICP | 0.078 ICP |

*Assumption: $75K avg TVL, 39% mixed APR, 60% of yield recipients sell. No external catalysts.*

**Additional protection mechanism — active sunset clause:**
If the price falls to 0.080 ICP → reduce Flexible bps by 20% → less PXRM issued → less sell pressure. This adjustment is manual (founder) until governance is active. Document each adjustment on-chain as a governance proposal for the CMF (Chile's financial markets regulator) record.

---

### LUNX (Luminox) — GreyValley's External Multichain Token

**Canonical dual-token design:** PXRM is the protocol's internal token (ICP, staking, epoch tiers). LUNX is the external, multichain representation of GreyValley's value — the real market token.

| Parameter | V1 Beta (today) | V2 (roadmap) |
|-----------|--------------|--------------|
| Type | Local `lunxCredits` credit in canister | ERC-20 on Ethereum / SPL on Solana |
| Transferable | No (soulbound) | Yes — tradeable on Uniswap, CEXes |
| Accumulation | 1:1 with PXRM yield on **harvest** | Same + bridgeable to Ethereum |
| Storage | `Map<Principal, Nat>` in main.mo | ERC-20 smart contract on Ethereum |
| Bridge | ❌ Not active | ✅ Via Chain Fusion (threshold ECDSA) |
| Price | Internal (mirrors PXRM) | Price discovery on the Ethereum market |
| Supply | Floating (mirror of bridged PXRM) | No cap — mint/burn per bridge |

**Why LUNX and not just PXRM for everything:**
- PXRM staked in tiers T1-T5 **must not leave the protocol** — that's what generates TVL retention
- LUNX lets the user have external liquidity and market price **without breaking staking**
- The "LUNX / Luminox" ticker has more resonance in external markets than "PXRM / Puranium"

**V1 use (today):** participation history, governance vote, APR boost (+1% per 100K LUNX)
**V2 use:** Ethereum bridge, Uniswap V3 liquidity, CEX listing, collateral in EVM protocols

**See full technical architecture:** `GREYVALLEY_INTEGRATIONS.md` §11

> **Real bug fixed 2026-09-08:** the PXRM staking that the app uses today (canister `staking-vault`,
> separate from the legacy system that lived inside `backend.mo`) never had a way to credit
> LUNX — any real user who staked PXRM and claimed rewards earned real PXRM/hard assets
> but **zero LUNX**, with no visible error. Real fix: `staking-vault.claimRewards()` now
> calls `backend.creditLunxFromStakingVault(caller, pxrmYield)` 1:1 with the claimed PXRM
> yield — except when the `caller` is the protocol's own position (`epoch_pool`,
> flywheel Flow A), which is deliberately excluded because LUNX is a reward for
> people, not for the protocol's internal seed capital.

### Other ecosystem tokens
| Token | Role |
|-------|-------|
| **vUSD** | Overcollateralized synthetic stablecoin. Issued from the CDP. **Live since 2026-08-02** (`ENABLE_VUSD_MINT = true`, `vusd_ledger` deployed). |
| **sCLP** | Synthetic Chilean peso, 1:1. Minted by Chanfusion upon receiving bank CLP. Requires CMF approval. |

---

## 2. Vault APR — V3 Model with Epoch Tiers

### Anatomy of the APR (3 layers always visible in the UI)

```
TOTAL APR = Base Real Yield + (PXRM Base APR × Guild Multiplier) + Epoch Tier Bonus
```

**Layer 1 — Base Real Yield:** comes from the protocol's real work (NNS staking, ODL fees, AMM). Payout token varies by vault.
**Layer 2 — PXRM Base APR (Treasury Incentive):** base APR in PXRM funded by the treasury (2M PXRM). It is not a "boost on top of" another base — it IS the PXRM base APR. Semi-annual review (Sunset Clause). Multiplied by Guild Multiplier for Track B: ×1.3 Institutional, ×2.0 Apex. Source: same treasury bucket for PXRM Base APR and the extra Guild Multiplier.
**Layer 3 — Epoch Tier Bonus (T1–T5):** reward for continuous holding, streamed on each harvest. **Updated (2026-09-19):** in addition to streaming on the APR at each harvest, the Epoch Retention Pool bucket of the fee split accumulates in a dedicated `epoch_pool` subaccount and is distributed in cycles to tiered holders (T1+), weighted by their bonus (since 2026-09-14). Flows A and B no longer see it. See §3.

### Vault Exaltium (ICP)
- **Capital on withdrawal:** the exact ICP deposited
- **Base Real Yield:** ~3.5%/year in ICP (via NNS neuron staking / WaterNeuron)
- **PXRM Base APR:** 12%–24% USD in PXRM from treasury (depending on lock period; recalibrated 2026-08-15, same target for all 3 vaults)
- **Guild Multiplier:** ×1.3 Institutional / ×2.0 Apex on the PXRM Base APR
- **+ Epoch Tier Bonus per §3**

### Vault Crypto (ckBTC, ckETH)
- **Capital on withdrawal:** the exact ckBTC/ckETH deposited
- **Base Real Yield:** ~0% in beta (AMM fees activatable in V2)
- **PXRM Base APR:** PXRM accumulation — the main attractor of this vault
- **Guild Multiplier:** ×1.3 Institutional / ×2.0 Apex on the PXRM Base APR
- **+ Epoch Tier Bonus**
- ⚠️ Minimal USD APR at current prices (BTC>>PXRM in price). An accumulation vault for PXRM, not a USD-yield vault.

### Vault Exaltite (ckUSDC, ckUSDT, ckEURC)
- **Capital on withdrawal:** the exact ckUSDC deposited
- **Base Real Yield:** 0%→2%/year variable in ckUSDC — **only exists with ODL volume**. In bootstrap = "--".
- **PXRM Base APR:** 12%–24% USD in PXRM from treasury (the main attractor during bootstrap; recalibrated 2026-08-15, same target for all 3 vaults)
- **Guild Multiplier:** ×1.3 Institutional / ×2.0 Apex on the PXRM Base APR
- **+ Epoch Tier Bonus**
- The deposited ckUSDC/ckEURC acts as corridor inventory (see `corredor-mechanics.md`) — ckUSDC for CLP↔USD, ckEURC for CLP↔EUR (integrated 2026-09-04), separate custody

---

## 3. Epoch Tiers — Holding System

There is no forced lock. Capital is always withdrawable. The tier rewards permanence.

| Tier | Name | Days since deposit | Yield type | Bonus APR |
|------|--------|---------------------|--------------|-----------|
| Grace | — | 0–14d | no tier | 0% |
| T1 | Navegante | 15–44d | live streaming ⟳ | +1% |
| T2 | Explorador | 45–89d | 30d epoch | +2% |
| T3 | Vórtice | 90–179d | 30d epoch | +4% |
| T4 | Singularidad | 180–464d | 30d epoch | +6% |
| T5 | El Heraldo | 465d+ | 30d epoch | +9% |

> V3 (2026-07-17): the grace period (0-14d) was added to eliminate free-riders, T4 was extended ~100d, and T5 moved from 365d to 465d — a full year is no longer enough to reach T5. Canonical in `GREYVALLEY_APR_MODEL.md` §4.1, implemented in `epoch_pool/main.mo`.

**5% Rule:**
- Withdrawal ≤5% of current capital (including reinvestments) → tier is preserved
- Withdrawal >5% of current capital → tier resets to T1
- The withdrawal is NEVER blocked — it always proceeds, only the tier changes

**T1 Streaming:** T1's 1% runs in real time, visible in the frontend. There's no need to wait for an epoch to collect it.

**T2-T5 Epoch:** Bonuses accumulate and are available for harvest at any time — the 30-day window is only the tier counter (active days), not a payment date. The real bonus is paid from the PXRM Base APR (treasury), not from a pool distributed every 30 days.

**Harvest:** Does not reset the tier or depositTimeNanos. Can be done at any time.

---

## 4. Fee Split — Protocol Fee Distribution (V1 — active in `fee_splitter/main.mo`)

> **Real revision 2026-09-08**: the split is **NO LONGER a single 35/25/23/10/7 table** — every
> real fee carries a `SourceCategory` that decides which FIXED table it is split with. The 3 tables always
> add up to 10,000 bps (100%), with no runtime rescaling.

### 4.1 — Standard category `#PxrmDenominated` (trading fees: AMM swaps and marketplace, in any token; the code's name is historical)

Routes: AMM swap (any pair), Marketplace redemption. (Since 2026-09-19 the PXRM→ICP swap is no longer split: its 0.33% fee stays entirely in the backend, it's the buy-back of PXRM with the protocol's own ICP.)

| Destination | % | Description |
|---------|---|-------------|
| PXRM Stakers | 35% | Fee pool in hard assets (ICP/ckUSDC/ckBTC) — not in PXRM |
| LP AMM providers | 25% | Proportional to liquidity contributed in `amm` |
| Treasury | 23% | Funds PXRM Base APR + Guild Multiplier |
| Epoch Retention Pool | 10% | Pays the T1–T5 Tier Bonus: accumulates in a dedicated `epoch_pool` subaccount and is distributed to tiered holders in cycles, weighted by their bonus (since 2026-09-14) — separate from Flows A/B |
| Volume Guilds | 7% | Only qualified Track A depositors (Exaltite), proportional to their ckUSDC/ckUSDT/ckEURC |
| **Total** | **100%** | |

### 4.2 — `#VaultBacked` (the fee comes out of vault capital, not PXRM)

Routes: CDP interest, CDP liquidation (ICP/ckBTC/ckETH collateral), corridor fee (`bridge_odl`, 0.66%).

| Destination | % | Description |
|---------|---|-------------|
| PXRM Stakers | 2% | Real remainder from rounding the other 4 to whole numbers (previously 0%) |
| LP AMM providers | 40% | Corridor → real LPs of `liquidity_pool` · CDP → LPs of `amm` |
| Treasury | 33% | Funds PXRM Base APR + Guild Multiplier |
| Epoch Retention Pool | 15% | Pays the T1–T5 Tier Bonus: accumulates in a dedicated `epoch_pool` subaccount and is distributed to tiered holders in cycles, weighted by their bonus (since 2026-09-14) — separate from Flows A/B |
| Volume Guilds | 10% | Only qualified Track A (Exaltite) — hard assets from corridor and CDP fees |
| **Total** | **100%** | |

**Specialized split (2026-09-19):** this table serves two specialties of the app and is shown
as two separate cards on /tokenomics:
- **Mint (CDP, liquidations, and crypto bridge):** the fee is paid in the collateral token. The 40% LP share goes
  only to LPs of pools that **contain that token** (`amm.receiveAllocationForToken`), weighted
  by each pool's value and by LP: a fee in ckBTC is collected by pools with ckBTC, not by a
  vUSD/PXRM LP (which only collects the symbolic 2% as a PXRM staker).
- **Corridor:** fees are only ckUSDC and ckEURC and go only to LPs who contributed to the corridors
  (`liquidity_pool`, per pair CLP_USD/CLP_EUR) — a BTC/vUSD LP does not collect from the corridor.
- **Design pending:** Volume Guilds pooling assets to buy ckUSDC/ckEURC. Epoch Pool stays as
  is (distributes per token to all tiered holders): rewarding by vault asset was ruled out as too odd.

### 4.3 — `#PxrmLiquidation` (new, 100% PXRM liquidated collateral)

Route: CDP liquidation when the collateral was PXRM — weighted higher than a generic swap because
the lost collateral IS real PXRM capital from the borrower.

| Destination | % | Description |
|---------|---|-------------|
| PXRM Stakers | 63% | The real lost PXRM collateral goes to those who bet on PXRM |
| LP AMM providers | 10% | To all LPs across all pools, regardless of pair (weighted by value): extra PXRM for the stakers' indirect LPs |
| Treasury | 20% | Funds PXRM Base APR + Guild Multiplier |
| Reward bucket | 7% | Since the fee is 100% PXRM, it goes directly to the Staking Rewards subaccount and ends up with stakers via the PXRM Staker Boost |
| Volume Guilds | 0% | No guilds: the fee is 100% PXRM; Volume Guild is only Track A with hard assets from corridor/CDP |
| **Total** | **100%** | |

### 4.4 — Real routing of the LpAmm bucket (fix 2026-09-08)

The LpAmm bucket from `#VaultBacked` **corridor** sources (`bridge_odl:<pair>`) goes to
`liquidity_pool.receiveAllocation()` — 100% of the amount goes ONLY to the LPs of that specific
pair (CLP_USD, CLP_EUR, etc.), not split 1/N among the corridor's 4 pairs
(previously ~half was lost on pairs with no real LPs, like CLP_BRL/CLP_ARS). `cdp_interest` and
`cdp_liquidation` (non-PXRM collateral) still go to `amm.receiveAllocation()` unchanged.

### 4.5 — The protocol's two real flows (unified naming 2026-09-08)

- **The protocol's own PXRM position flow** (~100,000 PXRM staked, owned by
  `epoch_pool`): 100%/100% categorical split by token, no percentages. Hard assets
  (ICP/ckBTC/ckETH/ckUSDC) refill the weakest AMM pool; the PXRM from the reward
  returns entirely to Staking Rewards (real fix 2026-09-08 — previously it was left
  orphaned). Never buys BTC.
- **The protocol's general fee flow** (Treasury bucket, 23%/35%/18% depending on category —
  distinct from the Protocol Treasury in §1, which is 40% of SUPPLY): proportional split
  3%/10%/10% over the real accumulated balance in the `VAELIX_TREASURY_V1` subaccount of
  `neo-protocol-backend` (naming predates the rebrand to GreyValley, 2026-08-29 — the
  real on-chain identifier of the subaccount has not been cleaned up yet, this is not a
  typo in this document) (POL — Protocol-Owned-Liquidity): 3% opex/maintenance (liquid,
  stays there), 10% Positions (moved to `epoch_pool`, which rebalances pools + buys real
  ckBTC via **ICPSwap**), 10% Flexible reserve (moved to a separate subaccount,
  `TREASURY_FLEX_RESERVE_V1`, "other assets not committed to BTC"). The 3:10:10 ratio is
  proportional to the real mixed balance, not a fixed amount — if more money comes in from
  the 35% category, each tranche scales proportionally larger.

### 4.6 — Real audit 2026-09-08

- LUNX was never credited in real PXRM staking (`staking-vault`, distinct from the orphaned
  legacy system inside `backend.mo`) — real fix, `claimRewards()` now credits 1:1,
  excluding the protocol's own position.
- 6 real routes blocked against burning the PXRM ledger's minting principal (which
  coincides with the founder's Plug identity) — `unstake`/`claimRewards`/`adminForceUnstake`
  in `staking-vault`, `claimUnpaidYield`/`claimStakeRewards`/`unstakePXRM` in `backend.mo`.
- Outdated info corrected on `/analytics` and `/vaults` (old fixed bucket %, mislabeling
  "Treasury 23%" vs. "Treasury 40%").

> **Unimplemented proposal (V1.5, requires decision + code change):** reassign points
> from the Treasury bucket to a dedicated "vUSD Institutional Pool" to reinforce the
> institutional track in §5b. Full V1.5/V2 progression by ODL volume: see
> `GREYVALLEY_APR_MODEL.md` §6.

---

### 4.5 — Fees charged in PXRM (special card, 2026-09-19)

Applies to any fee whose token is PXRM and is not a CDP liquidation (which has its
own table, §4.3): in practice, an AMM swap on the `vUSD_PXRM` pair in the
PXRM→vUSD direction. The backend's PXRM→ICP swap does not fall in here: its fee stays
entirely in the backend.

| Destination | % | Description |
|---------|---|-------------|
| PXRM Stakers | 30% | The only case where stakers receive PXRM from a fee — it's recycled, going back into the reward |
| LP AMM providers | 30% | Only to LPs of the `vUSD_PXRM` pair, proportional to their LP (positions with vUSD against PXRM and vUSD stakers who provide liquidity) |
| Treasury | 23% | Accumulates in PXRM in the Treasury (final destination to be defined) |
| Reward bucket | 17% | 10% from Epoch Pool + 7% from Volume Guilds recycled to the Staking Rewards subaccount, from which the PXRM Staker Boost distributes it to stakers |
| **Total** | **100%** | |

Reason: PXRM must be recycled and cared for. Neither tiers nor guilds are paid in PXRM.

**LP bucket split (fix 2026-09-19):** the 25% of a swap fee goes only to the LPs of the pair
where it occurred (`amm.receiveAllocationForPair`), proportional to their LP. General fees
(e.g., CDP interest) are split among all pools weighted by each pool's value
(previously 1/n equally, which gave a $1 pool the same as a $1M pool).

## 5b. vUSD Institutional Track (NEW — 2026-07-01)

> ⚠️ **Regulatory risk note (2026-08-07):** "Does not require CMF approval" below is a
> product claim, not a legal conclusion — this track (pooled deposit, explicit yield %,
> aimed at "investors/companies/banks") has security-like characteristics that have not
> yet been formally analyzed. Different from vUSD minted via CDP to pay in the
> Marketplace (individual, without pooling), which is properly framed as a means of
> payment. See `GREYVALLEY_REGULATORY_PLANB.md` §6 before scaling this track or
> presenting it to real institutional investors.

Stable-yield track for classic investors, companies, and banks. **Does not require CMF approval** *(claim pending validation — see note above)*.

| Parameter | Value |
|-----------|-------|
| APR | 4–6% stable annual in USD — **design target, not real today** (see base source) |
| Yield token | vUSD (1:1 USD synthetic stablecoin) |
| Base source | ⚠️ **0% real today** — the original design targeted "2% ODL fees (Vault Exaltite real base)", but Exaltite has 0% real base yield (the external hook was removed 2026-08-07, never replaced) and the ODL corridor has no real volume yet |
| Supplementary source | Additional PXRM boost via Guild Multiplier (Treasury, §2) — exists as a classification in `staking-vault` since 2026-08-13, but not yet wired to any real payout |
| Collateral | User's ckUSDC (internal CDP, 110% ratio) |
| On exit | vUSD burned, ckUSDC returned in full |
| Activation | ✅ Mint live since 2026-08-02 (`ENABLE_VUSD_MINT = true`) — but vUSD has no real use beyond closing the CDP itself: it cannot be sent, cannot be swapped, the Marketplace is UI-only with no canister (see `GREYVALLEY_MASTER_STATE.md` §32) |

**Access by Guild tier — corrected 2026-08-14, permissionless in all three:**

| Guild | Requirement | vUSD Boost | Cap |
|-------|-----------|-----------|-----|
| Explorador | ICP wallet only, $5K combined | +1.5% target APR — no real source today | $50K |
| Institucional | ICP wallet only, $15K combined | +2.5% target APR — no real source today | unlimited |
| Apex | ICP wallet only, $50K combined | +3.5% target APR — no real source today | unlimited |

---

## 5c. Guild Program (NEW — 2026-07-01, corrected 2026-08-14)

Three Track A (Volume Guild) tiers. **Permissionless in all three — never requires KYC
or KYB.** The tier is calculated automatically, live, from your real position in the
vaults (`useGuildTier.ts`, shown in the Wallet) — no registration, no form. (Track B,
the API operator program, is the one that does require KYB at all its tiers — see
`docs/CORREDOR_GUILD_OPERATOR_GUIDE.md`.)

### Guild Explorador (individual)
- ICP wallet + combined deposit ≥ $5,000 USD (min. 50% in ckUSDC/Exaltite)
- No company, no KYC

### Guild Institucional
- ICP wallet + combined deposit ≥ $15,000 USD
- 7% fee pool in PXRM (VolumeGuilds bucket) — still no real destination or distribution

### Guild Apex
- ICP wallet + combined deposit ≥ $50,000 USD
- Same benefits as Institucional, with no additional identity requirements

### Protocol fee table
| Operation | Fee |
|-----------|-----|
| ODL bridge | 0.5% of each transaction |
| Volatile AMM swap | 0.3% |
| Stable AMM swap | 0.1% |
| CLP fiat ramp (on top of ~1% Koywe fee) | +0.2% |
| CDP loan | No interest fee — that mechanism does not exist in `cdp/main.mo` |
| CDP liquidation | 13% of collateral (10% liquidator / 3% treasury / 87% to owner) — real, confirmed in code |
| PXRM→ICP swap | 0.3% (25 PXRM cap) |

---

## 5. PXRM Staking

### Hybrid model: Lock multiplier + Epoch Tier bonus

Unlike the vaults (which use only Epoch Tiers with no lock), PXRM staking keeps a **forced lock** because:
- PXRM stakers participate in governance → the lock creates real alignment
- The lock reduces PXRM's circulating supply → stabilizes the token
- Analogous to NNS neurons: dissolve delay = commitment

Staking APR comes from the PxrmStakers bucket of the `fee_splitter` (not from the treasury) — the
real % varies depending on where each fee came from (revised 2026-09-08, see §4): 35% from
swaps/marketplace, 2% from CDP interest/corridor, 50% from liquidations with 100% PXRM
collateral. Paid in ICP, ckUSDC, ckBTC, or real PXRM, proportional to each fee's volume.
Additionally, since 2026-09-08 there's a dedicated
"PXRM Staker Boost" (bootstrap, funded from the Staking Rewards subaccount, real 6-day
timer, 1%/4%/6%/10% annual rates by lock) — separate from the real yield, never mixed into
the same number.

### Lock multiplier (base)

| Lock | Multiplier | Example with 8% base APR |
|------|--------------|------------------------|
| Flexible | ×1.0 | 8% |
| 3 months | ×1.3 | 10.4% |
| 6 months | ×1.6 | 12.8% |
| 12 months | ×2.0 | 16% |

### Epoch Tier bonus (on top of the lock multiplier)

Continuous time in staking (without withdrawing >5%) adds a bonus on top of the already-multiplied APR:

| Tier | Name | Days staked | Bonus |
|------|--------|-----------------|-------|
| T1 | Navegante | 0–30d | +0% |
| T2 | Explorador | 30–90d | +5% |
| T3 | Vórtice | 90–180d | +10% |
| T4 | Singularidad | 180–365d | +15% |
| T5 | El Heraldo | 365d+ | +20% |

**5% Rule:** Withdrawing >5% of the total position (capital + accumulated rewards) resets the Epoch Tier to T1. The lock period keeps running independently — the lock has its own early-exit penalty.

### Self-dilution of the protocol's own position (NEW — 2026-09-09)

**Real problem evaluated this session:** the PxrmStakers bucket split is purely
relative (`weightOf = pxrmStaked × multiplier`) — if there's only one active staker, they
take 100% of the bucket regardless of how small the amount is. With $10M in total fees, a
single $100 staker would take the full $3.5M of the bucket (35%). `epoch_pool`'s own
position (~100,000 PXRM, Flexible) already acted as ballast to prevent this, but since it
was fixed, it dilutes evenly forever regardless of how much real adoption there is.

The founder evaluated and explicitly rejected a per-staker cap ("we won't limit it to a %
of the bucket, that's what our own position is for") — the chosen solution is for the
protocol's own position to yield space rather than limiting third parties:

- Every external `stake()` (not `epoch_pool` itself) subtracts that same amount from the
  protocol's own position, **floor-guarded** — it never drops below
  `pxrmSelfDilutionFloor` (default 25,000 PXRM, ~US$6K at the current oracle price,
  adjustable via `setPxrmSelfDilutionFloor` under the same manual-review criteria as
  `boostSunsetMultiplier`).
- The subtracted PXRM is returned for real to the Staking Rewards subaccount (same
  destination as `epoch_pool`'s `returnUnusedPxrmToStakingRewards()`) — never burned,
  never lost.
- Once the protocol's own position reaches the floor, it stops dropping — from then on,
  the bucket's total weight really grows with each new staker, instead of just being
  reshuffled.

### vUSD Staking Pool — real TVL without needing PXRM (NEW — 2026-09-09)

**Real problem**: a PXRM staker does not contribute TVL to the AMM pools unless they also have vUSD to manually pair in `vUSD_PXRM` — almost nobody did this, so that pair stayed nearly empty despite existing since 2026-08-28 (auto-pooled only from Marketplace fees, 0.12 vUSD per listing).

**Solution**: `stakeVusd(amount)`/`unstakeVusd(amount)` in `staking-vault` — pure vUSD deposit, no PXRM required. Mechanics:

- The deposited vUSD is automatically paired with the protocol's PXRM position (`autoPoolPxrm()`, a mechanism already existing since 2026-08-24, reused as-is) via a real `add_liquidity` on the `vUSD_PXRM` pair.
- Reward: real LP of that pair — the LpAmm bucket of the `fee_splitter` when there are swaps, **not** the PxrmStakers bucket (35%).
- **Fast-withdrawal buffer (5%)**: a portion of the deposited vUSD is kept liquid, unpooled, to instantly pay small withdrawals without touching the real pool.
- **Post-withdrawal PXRM buffer (adjustable window, default 30 min)**: if a large withdrawal needs to pull real liquidity from the pool (`remove_liquidity`), the PXRM returned from that operation **is not immediately re-staked** — it waits in case someone else deposits vUSD soon (it's reused directly, without the pointless restake→unstake round trip). If nobody deposits within that window, a timer re-credits it to the protocol's own position.
- The protocol's own position (`pxrmStaked` in its `positions` entry) **does not decrease** when its PXRM is pooled — that field is the claim/right for fee distribution (`weightOf`), not the physical location of the token.

**Verified real on mainnet (2026-09-09)**: a test deposit of 0.10 vUSD → `staking-vault` became the **second real LP provider** of `vUSD_PXRM` — reserves rose from $1.08 vUSD/~4.47 PXRM to $1.18 vUSD/~4.88 PXRM.

### The full real exit chain for PXRM: PXRM → vUSD → ckUSDC (NEW — 2026-09-09)

PXRM (a proprietary token with no external market) has no real exit to something stable except by selling it against `vUSD_PXRM` — and from there, vUSD also has no real exit to ckUSDC (chain-key USDC, redeemable 1:1 against real USDC) except by selling it against `vUSD_ckUSDC`. Only the first link had a mechanism deepening it — `autoPoolVusdToCkusdc()` was added (same pattern as `autoPoolVusdToPxrm()`) so the second one also grows on its own, from the same real source (Marketplace fees, split ~50/50 between both pools).

Real ICP replenishment for `swapPXRMtoICP` (2026-09-09): after evaluating ICPSwap (ruled out, no real PXRM/ICP pool) and the PXRM→vUSD→ckUSDC→ICP chain (ruled out for now, intermediate pools too small), the chosen approach was taking an adjustable 4% of the real ICP already held in the Flexible Reserve (the Treasury's second 10%) — same token, no pool involved, zero price impact. Runs on the backend's 8h tick.

### Combined example (6-month Lock · T3 Vórtice)

```
Base fee-pool APR:    8%
× 6-month Lock:       ×1.6  → 12.8%
+ T3 Epoch bonus:     +10%  → 22.8% effective
Paid in:              ICP + ckUSDC + ckBTC (proportional to fees)
LUNX credited:        1:1 with rewards on claimRewards() (staking-vault, real since 2026-09-08)
```

### APR by protocol volume

The base APR (8% in the example) is for reference only — it depends on real volume:

| Monthly ODL volume | Monthly fee pool (35%) | Est. PXRM staking APR |
|--------------------|----------------------|----------------------|
| $100K | ~$175 | ~2–4% |
| $1M | ~$1,750 | ~8–12% |
| $10M | ~$17,500 | ~20–30% |

Lock and epoch multipliers apply to the real APR, not the estimated one.

---

## 6. Minipack (Welcome Program)

- **Amount:** 10 PXRM (`1_000_000_000` e8s) on the first deposit in any vault
- **Condition:** Only on each wallet's first-ever deposit
- **Execution:** Best-effort — if it fails, it queues with a max of 10 retries
- **Timer:** Processes up to 5 pending per tick (every 300s)
- **Critical rule:** NEVER block the deposit for the minipack

---

## 7. PXRM → ICP Swap

| Parameter | Value |
|-----------|-------|
| Cap | 25 PXRM per transaction |
| Protocol fee | 0.3% |
| Rate | 1 PXRM = ICP / 10 (correction 2026-07-31 — see §1) |
| PXRM mechanism | Burn via `icrc2_transfer_from` with memo `"burn"` |
| ICP mechanism | Comes out of the treasury |
| Rollback | If the ICP transfer fails → PXRM returned to the user |

---

## 8. CDP (Collateralized Debt Position)

Required collateralization ratios (**corrected 2026-09-12** — the previous table had ICP at 175% instead of 150%, and listed `ckUSDC` as collateral, which does not exist as a real type in `cdp/main.mo`):
| Asset | Minimum ratio | Liquidation penalty |
|--------|-------------|----------------|
| ckBTC | 130% | 13.13% (10% bounty + 3% treasury) |
| ICP | 150% | 13.13% |
| ckETH | 150% | 13.13% |
| PXRM | 200% | 13.13% (or 50% of the fee split if the liquidated collateral is 100% PXRM — see §4.3) |

**Interest (stability fee), deployed 2026-08-16:** 0.3% monthly on the minted vUSD,
linear (not compounded, same "simple interest" criteria as the rest of the protocol —
PXRM Base APR, vault.mo yield). Computed on-the-fly from `createdAt` (no timer, no new
field on the CDP — `accruedInterestE2s()` in `cdp/main.mo`) and charged at `close_cdp()`:
the owner must burn `vusdMinted + accrued interest`, not just the principal. The real
interest goes to `fee_splitter` (minting the same extra burned amount → supply-neutral
burn+mint, with no net vUSD inflation) as the `cdp_interest` source, `#VaultBacked`
category — same route as `cdp_liquidation`. The `getAccruedInterest(owner)` query lets
the frontend show the exact amount before closing (`CdpPage.tsx`). The only other real
penalty remains liquidation (13%, one-time, only if the ratio falls below the minimum) —
interest is not charged there, it accrues but is not explicitly demanded (the borrower
already loses 13% of the collateral).

### 8.1 Leverage Loop UX — decision made

The x2/x3/x4 buttons on CdpPage are **shortcuts that prefill the Leverage Simulator, they do
not execute directly**. The simulator always shows, before any confirmation: total
exposure, liquidation price, current ratio, and 30-day liquidation probability.
Confirmation is explicit — never a single click from the shortcut buttons.

### 8.2 Safe Haven — quick exit on market downturn

A 3-click flow to exit crypto exposure into stable assets, available from
any vault or CDP position:
1. Exit the vault (e.g., Exaltium/ICP) → receive the underlying liquid ck-token.
2. Swap the ck-token → vUSD (freezes the value in dollars).
3. Portfolio in "Capital Preservation" mode → rebalances the rest toward vUSD.

---

## 9. Price Oracle

**Updated 2026-09-09** — real intervals:

| Source | Data | Frequency |
|--------|-------|-----------|
| XRC (system canister) | ICP, BTC, ETH in USD | Every 8h, adjustable |
| Derived | PXRM = ICP × fixed peg (1 PXRM = 0.1 ICP) | Same tick as ICP |
| CoinGecko HTTP | ckLINK in USD | Every 8h |
| mindicador.cl + fallback | CLP/USD, CLP/EUR | Every 8h, separate own timer (reference) |
| Fallback | Previous cache | If the request fails |

**Rules:**
- CLP: **never divide by 100** — it comes as an integer (e.g., 950 = 950 CLP per 1 USD)
- If there's no cache: show `"--"` in the UI — NEVER make up data
- Mandatory transform function on all HTTP outcalls
- Real fix 2026-09-09: LINK's price was losing decimals when parsing
  (truncated after the decimal point) — corrected, now preserves 8 decimals.

### Server-side swap impact guard + pool recalibration (NEW 2026-09-09)

`amm.swap()` now has its own guard (not just client-side): it compares
the real USD value of what goes in vs. what comes out, adjustable 11%
threshold, fails closed if the oracle doesn't respond. Verified real:
rejected a swap with 2.17M% impact against a broken pool. Also, a new
admin function `adminRecalibratePool()` to reseed pools with no real
LPs safely (returns the broken reserves before reseeding).

---

## 10. ODL Bridge

Vault Exaltite (ckUSDC/ckEURC) acts as the corridor's liquidity inventory:
- $1 deposited in Vault Exaltite = $1 of instant ODL capacity
- ICP settles in ~2 seconds → the same pool can process much more monthly volume
- The depositor's capital is always intact — the pool is never "consumed"
- See **`corredor-mechanics.md`** for full detail

---

## 11. Governance

```
Development → Sandbox → Restricted → Live
                              ↕
                           Paused
```

| Status | Deposits | Harvest | Withdraw | Staking |
|--------|-----------|---------|---------|---------|
| Development | ❌ | ✅ | ✅ | ❌ |
| Sandbox | ❌ | ✅ | ✅ | ❌ |
| Restricted | ✅ (whitelist) | ✅ | ✅ | ✅ |
| Live | ✅ | ✅ | ✅ | ✅ |
| Paused | ❌ | ✅ | ✅ | ❌ |

Only `controllerPrincipal` can change the status.

---

## 12. Feature Flags

| Flag | Status (2026-08-03) | Enables |
|------|---------------------|--------|
| `ENABLE_SCLP_BRIDGE` | false | Cross-chain sCLP bridge |
| `ENABLE_CRYPTO_BRIDGE` | false | Native ETH/BTC bridge |
| `ENABLE_VUSD_MINT` | ✅ true | vUSD stablecoin (CDP) |
| `ENABLE_SCLP_MINT` | false | sCLP minting |
| `ENABLE_ODL_PENALTIES` | false | ODL corridor penalties |
| `ENABLE_API_GATEWAY` | false | API access (Corporate/Apex tier) |

---

## 13. Tokenomics Roadmap

### V1.5 — LUNX Governance + ReFi
- Accumulated LUNX grants voting rights on protocol parameters
- Activate ReFi (0% → X%) via T3+ vote
- APR boost: 100,000 LUNX → +1% APR on any vault

### V2.0 — RFRY Index Token
- Token representing a portfolio of 40+ thesis tokens
- Automatic rebalancing every 30 days
- Yield distributed proportionally to holders

### V2.1 — Vault Bundles (Slice ETF)
- 4 themed vaults (Brain / Interchain / Hardware / Consumer)
- User deposits ckUSDC → weighted exposure to the slice
- Yield in PXRM

---

*Tokenomics v2.0 | GreyValley Protocol | 2026-06-28*
