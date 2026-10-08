# GreyValley — Real pre-launch tester guide

> Living document for the closed testing group (trusted inner circle,
> technical/"friendly hacking" profile). Goal: find real bugs with real
> money, in small amounts, before any public launch. This is NOT a demo —
> it's real production on ICP mainnet, with real canisters and real funds.

---

## 1. What this is, honestly

GreyValley is a real DeFi protocol on the Internet Computer (vaults, staking,
AMM, CDP/vUSD, marketplace, fiat corridor). It's in a **pre-internal-audit**
stage — the founder and the development assistant have been finding and
fixing real money bugs at a high pace (several per week in recent sessions:
CDP ratio scaling, orphaned funds in the AMM, PXRM ledger pointing to the
wrong place, timers silently dying from low cycles). That's not a red flag —
it's exactly what's expected to find at this stage. But it means **new bugs
of this same caliber can still show up**, which is why this closed test
exists before any wider opening.

There has been no formal external audit yet. Everything verified so far is
founder + assistant, using the real method: trace every function to its
canister → verify the result on-chain → fix it or say the truth → only then
test with real money.

## 1.1 Quick summary — what's active, what's not

**Active, with real money today:**
- **Vaults** — Exaltium (ICP), Crypto (ckBTC/ckETH/ckLINK), Exaltite
  (ckUSDC/ckUSDT/ckEURC): deposit, harvest yield, withdraw (full or
  partial).
- **PXRM staking** (Puranium) — real lock of 0/90/180/365 days.
- **AMM** — real swaps between existing pairs, and add/remove liquidity
  yourself. Some pairs are still very shallow (see §3) — the impact
  guard (~11%, fixed this week in both directions) protects them.
- **CDP** — mint real vUSD against ICP, ckBTC, ckETH, or PXRM as
  collateral (minting enabled globally).

**Switched off or blocked today (on purpose, not a bug):**
- **Fiat corridor CLP↔USD / CLP↔EUR** — the real code already exists and
  is deployed, but the feature flag is still off and there's no partner
  (Koywe) with confirmed KYB yet. Nothing to test there for now.
- **Native sCLP** — blocked by regulatory approval (CMF), runs in
  sandbox, not with real money.
- **Governance** — deployed; its status is "Sandbox" (verified on-chain on
  2026-10-05, previously "Development") and it now has a proposal and voting
  engine, but no proposals and not activated: it does not yet govern anything real.
- **NFID** — no longer in the app (removed 2026-09-02, see §2). **Internet Identity** — added 2026-09-12; it can operate (see §2).

## 1.2 By what you already have — what to test with your token

If you don't know where to start, look at what's in your wallet:

- **You have ckUSDC / ckUSDT / ckEURC (stable savings)** → deposit them
  into **Vault Exaltite**. You earn PXRM Base APR + a share of the AMM
  fee (proportional to your share of the vault).
- **You have ckBTC** → you can **collateralize it in the CDP** (130%
  minimum ratio, the lowest of the 4 types) to mint real vUSD, or put it
  in **Vault Crypto** to accumulate PXRM.
- **You want to trade** (or "be a trader", whatever you want to call it)
  → go to `/trade` and try a real swap between any pair — it's a good
  moment to confirm the impact guard blocks evenly in both directions
  (just fixed).
- **You have ckETH** → same path as ckBTC (Vault Crypto or CDP, 150%
  ratio), but also the `ckETH_ckUSDC` pair in `/amm` is nearly empty
  today — a real opportunity to be among the first to provide it
  liquidity and see how a freshly funded pool behaves.
- **You have ICP** → **Vault Exaltium** (real WaterNeuron, ~3.5% base
  APR) or CDP (150% ratio) or the `ICP_ckUSDC` pair in `/amm`.
- **You have PXRM** (including your welcome gift, see §2) → **Puranium
  staking** with a real lock, or **CDP** (200% ratio, the highest of the
  4 — see the concrete amount suggestion in §2).

## 2. What you need to participate

- **A real wallet, with confirmed support by tier:**
  - **Plug** — wallet with full support (along with Internet Identity). Use this one if you can.
  - **Oisy** — real custody of PXRM/ICP, you can trade or just browse the app,
    but **no stake or yield yet**.
  - **Bitfinity** — read-only, you can't operate.
  - **Internet Identity** — added 2026-09-12; it provides a session that can call
    any protocol canister, so it can operate.
  - **NFID** — ⚠️ Correction (2026-10-07): no longer shown in the app. It was
    removed on 2026-09-02 because it never actually connected in production.
    (Previous text: "marked Coming soon, disabled".)
- **Small amounts, on purpose**: don't put in anything you can't afford to
  lose. This is a real cap set by the founder, not just a suggestion — the
  idea is that a real bug costs cents, not that it ruins anyone.
- A wallet you can track closely yourself (check balances before/after every
  action, don't rely only on what the UI shows you).
- **Welcome gift**: every tester gets **50 real PXRM** to start (~$12.6 at
  today's price) so you have something to test with before putting in your
  own money. It lands directly in your connected wallet — tell us your
  principal so we can send it.
  - **Suggestion (not mandatory)**: try the CDP/vUSD mint with part of it —
    with ~15-20 PXRM of collateral you can mint ~$2 of real vUSD (the CDP
    requires 200% collateralization for PXRM, so $2 of vUSD needs ~$4 of
    collateral ≈ 15-20 PXRM at the current price). The rest (~30-35 PXRM)
    is free to stake however you prefer — Flexible, or a lock (90/180/365
    days) if you want to test that part too.

## 3. Real focus of this round — Swaps and Withdrawals

Prioritized by where we've already found real money bugs this month. If you
have little time, start here.

### Swaps

| # | What to test | Known state |
|---|---|---|
| 1 | A large swap in a shallow pool (several `/amm` cards say "Insufficient liquidity") | **REAL BUG found and fixed 2026-09-04**: the safeguard (~11% max impact) only blocked when the swap hurt the user — a mispriced pool in the OTHER direction (the trader receives MORE at the pool's expense) went through unblocked. Confirmed live against `ckBTC_ckUSDC` (mispriced 8x, you'd pay $0.09 and "receive" $0.48). Already fixed in `TradePage.tsx`/`PortalCard.tsx`/`AmmPoolStats.tsx` — test again that it actually blocks in BOTH directions now. |
| 2 | Swaps in pairs with different decimals (ckETH = 18 decimals, ckUSDC/ckUSDT/ckEURC = 6, rest = 8) | There were real bugs here before (miscalibrated fee, ckLINK 2026-09-03). Only verifiable by actually testing — no way to confirm from code review alone, you need to execute the real swap and compare the amount received against what's expected. |
| 3 | The 25 PXRM/tx cap on the fixed PXRM→ICP swap | **Confirmed**: 26 PXRM does get rejected — but the UI button disables itself before you can send that amount, so the exact limit can't be tested from the normal interface. If someone wants to force it (e.g. via console/direct API to the canister), ask before trying. |
| 4 | Pairs blocked on purpose (`PXRM_ckUSDC`, `PXRM_ICP`) | Already confirmed they reject. If anyone finds a way around the block, that's a critical finding — report it immediately. |
| 5 | Small swaps in newly integrated pools (`ckEURC_ckUSDC`, `ckLINK_ckUSDC`) | `ckEURC_ckUSDC` was just calibrated 2026-09-04 (~1% off the real oracle price). `ckLINK_ckUSDC` is still nearly empty (reserveB=1 raw) — any real swap there should reject via the impact guard, not execute at a broken price. Report if it executes anyway. |

### Withdrawals

| # | What to test | Known state |
|---|---|---|
| 1 | Early withdrawal from a vault with yield already accrued (penalty) | **Confirmed by the founder** — the real safeguard (~11%) worked well in testing. |
| 2 | Withdrawing a vault that has vUSD pooled in the AMM (active auto-pooling) | Not confirmed yet — tied to the same pattern that caused orphaned funds before in `add_liquidity`. Still a real risk area, test it thoroughly. |
| 3 | Open a real CDP with PXRM (suggested: ~15-20 PXRM from your gift → ~$2 of vUSD, see §2) and later close it with collateral near the minimum ratio, forcing a liquidation zone | Confirmed 2026-09-04: PXRM as collateral is **not blocked** (vUSD minting is enabled globally) — it requires a 200% minimum ratio, the highest of the 4 types (ICP 150%, ckBTC 130%, ckETH 150%, PXRM 200%). Not tested yet with real PXRM specifically — this is the first suggested attempt. |
| 4 | Unstaking PXRM with an active lock (90/180/365 days) | **Confirmed**: it doesn't let you withdraw early, tested at 90 days. Real note: positions from before the PXRM ledger fix (a real bug, fixed twice this session) had to be sacrificed — if you have old positions, expect something odd and let us know. |
| 5 | Withdrawing several positions at once (multi-click / one click for several) | **Tested**: 9 positions withdrawn with one click, 8 went fine. 1 Exaltite AMM position (~$0.10) didn't close on the first click, needed 2 clicks. **Minor real bug to investigate** — smells like a race condition or the UI state not reflecting the real result after the first attempt. |
| 6 | Withdrawing from Exaltite (ckUSDC/ckUSDT/ckEURC) — compare what it returns against what you deposited | **Not a bug, but surprising**: Exaltite is a shares pool shared across ALL historical positions from this testing round — today the real NAV is small compared to the nominal sum of everything ever deposited (confirmed 2026-09-04, a "$0.50" withdrawal returned only "$0.054" real, mathematically correct given the current NAV/shares). New public query `backend.getExaltiteShareDebug()` exposes `totalShares`/`navUsdcEquiv` live so you can verify this yourself before assuming it's a bug. |

## 4. How to report a finding

For anything odd you see, even if it seems small:

1. **What you did** — step by step, exact (amount, wallet, button).
2. **What you expected to happen.**
3. **What actually happened** — screenshot if possible.
4. **Wallet balance before/after**, if you touched real money.
5. **Does it repeat?** — try the same action twice before reporting, to
   tell a real bug apart from a one-off network glitch.

You don't need to know if it's "serious" or not — report everything,
priority gets sorted out afterward.

## 5. What is NOT the focus of this round (on purpose)

To keep the test focused, these areas are out of scope for now — they're
known, incomplete, and don't add new signal:

- Fiat corridor CLP↔USD/CLP↔EUR — the real code already exists and is
  deployed (`api_gateway → settlement → bridge → liquidity_pool`, confirmed
  2026-09-04), but it's still switched off via feature flag (`bridge=false`)
  and has no partner (Koywe) registered — there's nothing to test there yet,
  not for lack of code but for lack of commercial KYB.
- Native fiat ramp (Koywe/partners — no confirmed KYB, honestly marked
  "pending" throughout the UI)
- Services marketplace (it works, but it's not this round's focus — can be
  tested informally)
- NFID (no longer in the app, see above) · Internet Identity (available)

---

*Last updated: 2026-09-04. This document gets updated as the findings above
get confirmed or ruled out — it's not a final document.*
