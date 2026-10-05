# Protocol Documentation (English)

Public technical and commercial documentation for the DeFi payment protocol built for Chilean and LatAm markets.

*This is a full English translation of the repo's original documentation. The original working versions (mixed Spanish/English) are available one level up from [the repo root](../README.md).*

---

## What is this?

A DeFi protocol that connects Chilean fiat (CLP) to on-chain liquidity, enabling:
- **International payments** at 0.66% vs 2.5–3.5% SWIFT
- **Yield on stablecoins** via the corredor liquidity pool
- **Cross-border corredor** Chile ↔ USD ↔ EUR ↔ LatAm destinations
- **Institutional-grade compliance** under Ley Fintech 21.521 + Ley 21.719

The core stack: WebAssembly canisters on a distributed blockchain network with ~2 second finality, native Ethereum signing (no bridges), and native HTTPS Outcalls (no oracles).

---

## Documentation Index

### Architecture
| Document | What's inside |
|----------|--------------|
| [Technical Overview](architecture/tech-overview.md) | Full stack comparison, consensus mechanics, tECDSA, HTTPS Outcalls, data privacy (Ley 21.719) |
| [Corredor Mechanics](architecture/corredor-mechanics.md) | How the payment corridor works, SWIFT comparison, Koywe bridge architecture, partner model |
| [Engineer Pitch](architecture/engineer-pitch.md) | Current code state, deployment instructions, LatAm market positioning |

### Product
| Document | What's inside |
|----------|--------------|
| [GreyValley in 3 minutes](product/overview.md) | What the simplified app does, who it is for and what it is not |

### Finance
| Document | What's inside |
|----------|--------------|
| [Tokenomics](finance/tokenomics.md) | PXRM supply, distribution, vesting, APR model (§2), epoch tiers, fee split |
| [Genesis Round](finance/genesis-round.md) | Co-founder program, tiers, captation scenarios, pool sizing |

### Legal
| Document | What's inside |
|----------|--------------|
| [Regulatory Framework](legal/regulatory.md) | CMF sandbox (Art. 90 Ley 21.521), SFA-Sandbox integration, compliance roadmap |
| [SFA Integration](legal/sfa-integration.md) | Complete open finance API reference (PISP/AISP/FAPI 2.0) |

### Partnerships
| Document | What's inside |
|----------|--------------|
| [Market-maker guide](partnerships/market-maker-guide.md) | Commercial summary of the corridor partner model, no technical jargon |
| [ODL Agreement Template](partnerships/odl-agreement.md) | Conditional ODL Service Agreement — for prospective ODL corridor partners |

### Testing
| Document | What's inside |
|----------|--------------|
| [Tester Guide](testing/tester-guide.md) | Real, honest pre-launch testing guide — what's confirmed working, what's a known real bug, how to report findings |

---

## Current Status

The protocol is **live on mainnet** with the following components operational:
- PXRM token ledger (ICRC-1/ICRC-2, ~5M supply — deflationary model; see Tokenomics for the 2026-08-30 correction: a small welcome-fund mint means supply is no longer strictly fixed)
- Protocol backend (vaults, yield calculation, staking)
- Frontend (React PWA, accessible from mobile browser)
- Oracle (price feeds every 300s)
- CDP (collateralized debt, vUSD mint live)
- Fee splitter, API gateway, staking canister

In development: Koywe on/off-ramp bridge, sCLP corridor (pending CMF approval), governance canister activation.

---

## Contact

Philippe Ducasse La Rivera — Founder
Contact via GitHub issues on this repository, or through official GreyValley Protocol channels.

For ODL partnership inquiries, see the [ODL Agreement Template](partnerships/odl-agreement.md).

---

*2026-09-18 · Real corridor fee: 0.66% (66 bps, verified on-chain via getFeeBps on the corridor canister; previously 0.22%, 0.20%, and 0.5% in earlier versions) · Broken link to "APR Model" fixed — that content lives in Tokenomics §2, it never existed as a separate file · Protocol documentation is updated as the codebase evolves.*
