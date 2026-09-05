# Protocol Documentation

Public technical and commercial documentation for the DeFi payment protocol built for Chilean and LatAm markets.

---

## What is this?

A DeFi protocol that connects Chilean fiat (CLP) to on-chain liquidity, enabling:
- **International payments** at 0.22% vs 2.5–3.5% SWIFT
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
| [ODL Agreement Template](partnerships/odl-agreement.md) | Conditional ODL Service Agreement — for prospective ODL corridor partners |

---

## Current Status

The protocol is **live on mainnet** with the following components operational:
- PXRM token ledger (ICRC-1/ICRC-2, 5M fixed supply)
- Protocol backend (vaults, yield calculation, staking)
- Frontend (React PWA, accessible from mobile browser)
- Oracle (price feeds every 300s)
- CDP (collateralized debt, vUSD mint live)
- Fee splitter, API gateway, staking canister

In development: Koywe on/off-ramp bridge, sCLP corridor (pending CMF approval), governance canister activation.

---

## Contact

Philippe Ducasse La Rivera — Founder  
ducasse.phil@gmail.com

For ODL partnership inquiries, see the [ODL Agreement Template](partnerships/odl-agreement.md).

---

*2026-08-28 · Fee del corredor corregido a 0.20% (era 0.5% en versiones previas) · Link roto a "APR Model" corregido — ese contenido vive en Tokenomics §2, nunca existió como archivo separado · Protocol documentation is updated as the codebase evolves.*
