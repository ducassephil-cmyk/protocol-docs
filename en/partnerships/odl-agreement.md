# ODL Service Intent Agreement
### Conditional ODL Service Agreement (COSA) — v1.0

---

## PARTIES

**The Protocol:**
GreyValley SpA (a Chilean simplified stock corporation, "SpA," in the process of being formed), RUT (Chilean taxpayer ID / *Rol Único Tributario*) pending, represented by Philippe Ducasse La Rivera, RUT XX.XXX.XXX-X, email [official contact pending confirmation].

> ⚠️ **Real correction 2026-09-29 (docs audit) — "in the process of being formed" is not confirmed as accurate today.** On 2026-08-24 the founder confirmed that the real Chilean entity behind the protocol (Pegasus SpA) **is already incorporated**, as a *financiera* (a regulated Chilean financial-services company) — it is not a new SpA pending formation. It remains an open, unresolved question (as of 2026-08-30) whether it makes sense to create a separate, purely tech-purpose SpA to sign agreements of this kind without exposing Pegasus SpA (the *financiera*) to the custody risk of an ODL partner. **Do not sign this agreement under the name/RUT of any entity until the founder's lawyer confirms which real entity signs** — this PARTIES block is a placeholder pending that decision, not verified data.

**The Client:**
[Company name], RUT [XX.XXX.XXX-X], represented by [Name], RUT [XX.XXX.XXX-X], email [email].

---

## BACKGROUND

1. GreyValley is a decentralized financial protocol designed to facilitate high-frequency international payments through an on-demand liquidity (ODL) corridor, operated on blockchain infrastructure with CLP ↔ USD fiat ramps regulated under Ley Fintech 21.521 (Chile's Fintech Law).

2. The Client conducts regular foreign-trade operations that create a need to transfer funds between Chile and abroad, currently subject to bank fees of between 1.5% and 2.5% per transaction.

3. Both parties are interested in establishing the conditions under which the Client will use GreyValley's ODL corridor once it becomes operational.

---

## PURPOSE

This agreement establishes the Client's **conditional binding intent** to use GreyValley's ODL corridor for its international payment operations, subject to satisfaction of the conditions precedent described in Clause 3.

This agreement **does not involve any transfer of capital, deposit, or payment** by the Client at this stage.

---

## CONDITIONS PRECEDENT

The agreement activates automatically once **the following three conditions** are met:

| # | Condition | Verification |
|---|-----------|-------------|
| 1 | Genesis Round closed with TVL (Total Value Locked) ≥ USD $105,000 in the protocol | Public on-chain dashboard |
| 2 | First ODL test transaction completed successfully (CLP → ckUSDC → CLP) | On-chain record shared with the Client |
| 3 | Written notice from GreyValley to the Client declaring the corridor operational | Email to [Client's email] with ≤ 7 business days' advance notice |

If any of the three conditions is not met within **12 months** of the signing of this agreement, this agreement becomes void with no penalty to either party.

---

## CLIENT COMMITMENTS (once activated)

Once the conditions precedent are met, the Client commits to:

1. **Minimum monthly volume:** route a minimum of **CLP $[_______]** (approximate equivalent USD $[_______]) per month through GreyValley's ODL corridor during the first **[6 / 12] months** of active operation.

2. **Partial exclusivity:** direct at least **[30% / 50%]** of its monthly international payment operations to the GreyValley corridor during the indicated period.

3. **KYB onboarding:** complete the KYB (Know Your Business) verification process required by the fiat-ramp partner (Koywe) within 10 business days of the activation notice.

4. **Operational feedback:** participate in at least one monthly technical feedback session during the first 3 months, to help co-improve the corridor.

---

## GREYVALLEY COMMITMENTS (once activated)

Once the conditions precedent are met, GreyValley commits to:

1. **Preferential launch fee:** apply a fee of **0.15%** per corridor transaction (instead of the standard fee — **corrected 2026-09-29: the real retail fee today is 0.66%, not 0.22%**; the founder raised `FEE_BPS` from 22→66 via `setFeeBps()` on 2026-09-17, verified on-chain with `getFeeBps()` — this document still had the previous reference value) during the Client's first **[6 / 12] months** of active operation.

2. **Guaranteed liquidity slot:** reserve sufficient ODL capacity for the Client's committed volume, with priority over users onboarded later.

3. **Dedicated onboarding:** assign a direct point of contact for the Client's technical integration and KYB process.

4. **Status transparency:** notify the Client monthly of the Genesis Round's progress until its close, with updated TVL figures.

5. **Settlement window:** guarantee settlement of funds within ≤ [2 hours / 24 hours] of the payment instruction, during Chilean banking hours (Monday to Friday, 9:00 AM–5:00 PM).

---

## FEE COMPARISON — REFERENCE

| Item | Traditional banking (estimated) | GreyValley Corridor |
|----------|----------------------------|-----------------|
| Fee per transaction | 1.5% – 2.5% | **0.15%** preferential vs. **0.66%** standard today (FIX 2026-09-14: previously said "0.3%," which did not match the 0.15% committed above; FIX 2026-09-29: the standard reference fee was 0.22%, outdated — the real retail fee rose to 0.66% on 2026-09-17) |
| Settlement time | 1–3 business days | ≤ 2 hours |
| Traceability | Partial (SWIFT) | 100% on-chain |
| Estimated monthly savings* | — | **CLP $[______]** |

*Calculated on the volume committed in the preceding clause.

---

## CONFIDENTIALITY

Both parties agree to keep the specific terms of this agreement confidential from unrelated third parties. GreyValley may publicly state that it "has signed intent agreements with ODL clients" without revealing the Client's identity or specific amounts, except with the Client's express written authorization.

---

## NATURE OF THE AGREEMENT

- This agreement **does not constitute a promise of a financial services contract** under Ley 21.521 until the conditions precedent are met.
- This agreement **does not involve custody of funds** by GreyValley at this stage.
- This agreement **is not a debt instrument or a security**.
- Any dispute arising from this agreement is subject to the jurisdiction of the ordinary courts of Santiago, Chile.

---

## TERM

This agreement takes effect on the date of signing and has a maximum term of **12 months**. If the conditions precedent are not met within that period, the agreement expires with no effect and no penalty.

---

## SIGNATURES

Signed in Santiago, Chile, on the [___] day of [_______], 2026.

&nbsp;

**For GreyValley SpA (in the process of being formed)**

&nbsp;

_______________________________
Philippe Ducasse La Rivera
Founder · GreyValley Protocol
[official contact pending confirmation]

&nbsp;

**For [Company Name]**

&nbsp;

_______________________________
[Representative name]
[Title]
[Email]

---

*COSA v1.0 · GreyValley Protocol · 2026-08-15*
*This document is an intent agreement — not an active financial services contract.*
