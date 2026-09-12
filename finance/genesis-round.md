# Genesis Round — Co-Founders Pre-TGE
> Protocol bootstrapping program · 2026-08-15
> Normativa: Ley Fintech 21.521 · estructura no-dilutiva para el protocolo

---

> ⚠️ **This document describes the original large-scale design (2026-08-15).
> The model actually deployed on mainnet (`genesis_registry`, updated
> 2026-08-31) is different — see `TOKENOMICS.md` §1b for the current,
> authoritative model.** Kept here as reference for the eventual larger
> TGE scale; do not use the numbers below as current state.
>
> **Real model live today**: 10 co-founder slots (Angel 4 / Seed 3 /
> Strategic 3, the 3rd Strategic slot reserved for a co-founder who also
> provides real liquidity to the Corridor and tests it live once active).
> Cap: 300,000 PXRM (30% of the Ecosystem/Guilds bucket — within the
> 20-35% range this same document already approved below). Mechanism:
> each slot guarantees a % of capital value in PXRM (80/75/70/65% by
> entry order, same across all three tiers), computed using PXRM's
> **live oracle price at the exact moment of registration** — not a
> fixed PXRM/$ rate. Safety floor: never less than 1.0 PXRM per $1.
> A dedicated `creditGenesisBonus()` admin function reimburses any
> operational loss on the special Strategic slot's Corridor-liquidity
> capital, separate from the vesting allocation.
>
> **Second track — "Investors Guild"**: a SEPARATE mechanism (own future
> canister, own tracking, not `genesis_registry`) pulling from the SAME
> Ecosystem/Guilds bucket — flat 2x return (no tiers), $500 minimum,
> 9-month vesting, 200K PXRM cap (regulable). 100% design as of now, zero
> code/canister built. Do not confuse the two — this document describes
> only the FIRST track (co-founder slots above).

---

## What is the Genesis Round?

The Genesis Round is the TVL bootstrapping program before the Token Generation Event (TGE). Participants are **external co-founders** — actors who deposit real capital (stablecoins) into the protocol vaults and receive a PXRM allocation with vesting in return.

This is not a traditional fundraise. No equity, no SAFEs, no dilution. Co-founders earn yield on their deposited capital from day one while their PXRM vests post-TGE — aligned with the protocol's growth.

---

## Why a Genesis Round?

A DeFi protocol's credibility at launch is determined by its TVL. Without an initial pool of real capital, the ODL corridor has no inventory, the stablecoin vault has no depth, and the market has no confidence anchor.

The Genesis Round solves this by turning early capital into protocol co-ownership. Each co-founder simultaneously:
- Provides TVL that activates core protocol functionality
- Seeds the DEX liquidity pool that protects token price stability
- Earns real yield from the protocol treasury while doing so
- Holds a PXRM position with asymmetric upside

---

## Co-Founder Tiers

| Tier | Slots | Min TVL | PXRM Allocation | Vesting | Benefits |
|------|-------|---------|-----------------|---------|----------|
| 🔵 **Angel** | 3–4 | $10K–$25K (stablecoin) | 15K–25K PXRM | 6m post-TGE linear | Early access, priority operator tier |
| 🟣 **Seed** | 2–3 | $25K–$75K (stablecoin) | 40K–70K PXRM | 9m post-TGE linear | Active operator status, co-branding |
| 🩷 **Strategic** | 1–2 | $75K–$200K (stablecoin) | 80K–150K PXRM | 12m post-TGE linear | Advisory seat, Treasury voting, dedicated API |

> **Invariant rule:** no Genesis co-founder has a TGE unlock. All allocated PXRM vests post-TGE. This eliminates sell pressure on launch day.

---

## Captation Scenarios

| Scenario | Composition | Genesis TVL | PXRM Allocated | % Supply | TVL/FDV |
|----------|------------|-------------|----------------|----------|---------|
| **Minimum viable** | 2 Angel + 1 Seed | ~$70K | ~70K PXRM | 1.4% | 6.8% |
| **Credible launch** ✅ | 3 Angel + 2 Seed + 1 Strategic | ~$295K | ~310K PXRM | 6.2% | 28.5% |
| **Fast scale** | 4 Angel + 2 Seed + 1 Strategic | ~$320K | ~345K PXRM | 6.9% | ~31% |

**Recommended: Credible launch.** Activates the full ODL corridor (stablecoin vault ≥$50K), achieves TVL/FDV = 28.5% — excellent for DeFi early stage — with PXRM Genesis ≤ 8% of supply.

---

## What Co-Founders Actually Earn

Capital deposited in the stablecoin vault (Exaltite) earns from two sources simultaneously:

**Source 1 — PXRM Base APR (treasury, active from day 1)**
- Flexible: ~8% USD equivalent/year in PXRM
- 60 days lock: ~10% USD equivalent/year
- 180 days lock: ~14% USD equivalent/year
- 365 days lock: ~16% USD equivalent/year

**Source 2 — ODL fees (active once corridor is live)**
- Real yield in stablecoin from on/off-ramp volume
- Target: 0%→2%/year in stablecoin (grows with corridor volume)

**Epoch Tier Bonus (on top of both)**
- Rewards continuous holding without forced lock
- +1% to +9% depending on time held (T1 Navegante to T5 El Heraldo)
- Capital is always withdrawable — tier rewards permanence, not lock-in

---

## Token Allocation Source

Genesis PXRM comes from the **Ecosystem / Guilds bucket** (1,000,000 PXRM total) — that is its exact function: early adopters and protocol co-builders.

- **Genesis Round reserve:** ~200K–350K PXRM (20–35% of Ecosystem bucket)
- **Remaining for Guilds / ecosystem post-TGE:** ~650–800K PXRM

The Team/Founders bucket (1M PXRM, 12-month cliff) is separate and untouched for external co-founders.

---

## Liquidity Pool — Why Co-Founders Benefit From Seeding It

The PXRM/native-token DEX pool is the price infrastructure of the token. Its depth determines how much sell pressure the market can absorb before the price falls.

**Pool composition:**
| Component | Source | Capital required |
|-----------|--------|-----------------|
| PXRM side | Protocol treasury | $0 additional — already exists |
| Native token side | Founder (personal) + Genesis co-founders | Real capital |

**Recommended pool model:**
```
Phase 0 — Pre-genesis (founder only):
  50,000 PXRM (treasury) + 2,500–5,000 native tokens (founder personal)
  Pool value: ~$20K–$40K

Phase 1 — Post-genesis (founder + co-founders):
  100,000 PXRM (treasury) + 10,000 native tokens (combined)
  Pool value: ~$80K | Pool/monthly emission ratio: 33× ← stable target
```

**The narrative for co-founders:**
*"Your contribution doesn't just give you PXRM with upside — it founds the market liquidity that protects the value of that PXRM."* Co-founders have a direct incentive in a deep pool because it stabilizes the price of their own vesting position.

**Launch price:** 1 PXRM = 0.1 native token at seeding (initial seeding ratio, not a fixed peg — the DEX AMM determines market price from launch).

---

## Price Stability Projections

Without deep pool vs. with genesis-funded pool (assumptions: $75K TVL avg, 39% mixed APR, 60% of yield recipients sell):

| Month | Pool ~$20K (founder only) | Pool ~$80K (founder + genesis) |
|-------|--------------------------|-------------------------------|
| 0 | 0.100 ratio | 0.100 ratio |
| 1 | 0.086 | 0.096 |
| 3 | 0.072 | 0.088 |
| 6 | 0.059 | 0.078 |

A deeper pool absorbs the same sell pressure with 25–30% less price impact.

---

## The Strategic Co-Founder — ODL Pivot

A single actor with $100K+ stablecoin activates the full ODL corridor and gives the protocol immediate credibility. The ROI for them: ~2.5% savings on cross-border payment fees on that capital (~$2,500/month in active use) + PXRM exposure with asymmetric upside.

Ideal profile: importer/exporter already moving that volume monthly in international payments.

---

## Genesis Round Invariants

```
PXRM Genesis ≤ 350K      →  Ecosystem bucket retains ≥ 650K for post-TGE
TVL Genesis ≥ $105K       →  ODL viable on day 1
Minimum vesting: 6m        →  Zero dump at TGE
Co-founders: 5–8           →  Distribution + viable pre-TGE coordination
```

---

## Healthy TGE Criteria

1. **Day 1 TVL ≥ $105K** — minimum for real ODL capacity (stablecoin vault needs $50K+)
2. **PXRM Genesis ≤ 8% of supply** — above this starts pressuring price post-vesting
3. **TGE float: ~200K PXRM circulating** (4% supply) — only Ecosystem TGE unlock (100K) + Treasury (100K). Initial market cap ~$41K vs TVL $295K → 7× ratio
4. **Mandatory vesting for all Genesis participants** — no TGE unlock, no cliff dump
5. **5–8 co-founders** — fewer is dangerous concentration; more than 8 makes pre-TGE coordination impractical

---

## PXRM Token — Reference Data

| Parameter | Value |
|-----------|-------|
| Total supply | 5,000,000 PXRM (fixed, never increases) |
| Standard | ICRC-1 + ICRC-2 |
| Decimals | 8 |
| Model | **Deflationary** — burned on swap + enterprise boost burn |
| Launch price | 1 PXRM = 0.1 native token (seeding ratio) |
| FDV at launch | ~$1.03M (at current native token price) |
| LatAm adoption target | $500M FDV |
| Institutional adoption target | $2.5B FDV |

**Supply distribution:**
| Group | PXRM | TGE Unlock | Vesting |
|-------|------|-----------|---------|
| Protocol Treasury | 2,000,000 | 5% | Governance-controlled |
| Team / Founders | 1,000,000 | 0% | 36m linear after 12m cliff |
| Ecosystem / Guilds | 1,000,000 | 10% | 36m emission |
| Staking Rewards | 500,000 | 0% | Per epoch / APR |
| DAO Governance | 500,000 | 0% | Community vote |

---

## Legal Framework

- Structured under Chilean Ley Fintech 21.521 (in-process)
- Capital received as vault deposit — not as equity or loan
- PXRM allocation is protocol-native reward, not a security offering under current structure
- Governance canister activates before production with real funds — founder transitions from sole controller to Proposal Promoter, with protocol upgrades requiring token-holder vote + 48h timelock

---

*Genesis Round Reference Document · 2026-08-15*
