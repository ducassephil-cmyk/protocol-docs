# Engineer Pitch — Protocol for the Chilean IT Professional
> Technical reference for conversations with IT professionals, systems engineers, and infrastructure teams
> 2026-08-26

---

## Stack Comparison — Traditional vs. Protocol

| Traditional | What it does | Protocol equivalent | Technical name |
|---|---|---|---|
| Node.js + Express | HTTP server | **Canister** (WebAssembly module) | Update call / Query call |
| PostgreSQL | Persistent storage | **Stable variables** in canister | Orthogonal persistence |
| Redis | Fast read cache | **Query calls** | No-consensus read |
| AWS S3 | File storage | **Asset canister** | Up to 64GB/subnet |
| Auth0 | User authentication | **Internet Identity + Principal ID** | Cryptographic identity |
| Chainlink | External data oracle | **HTTPS Outcalls** | 28-node consensus HTTP |
| Wormhole/Axelar | Cross-chain bridge | **Chain Fusion / tECDSA** | No bridge needed |
| Docker | Isolated runtime | **Canister** (WASM sandbox) | WebAssembly |
| CI/CD pipeline | Code deployment | `dfx deploy` | One command |
| Cron job | Scheduled tasks | **Canister timers** | `recurringTimer` |
| JWT / session | Session management | **Delegated identity** | Time-bound delegation |

---

## SWIFT Problems — How the Protocol Solves Each One

| SWIFT Problem | What happens | Protocol solution |
|---|---|---|
| **Correspondent banking** | 2–4 intermediary banks, each charges $10–30 USD | ckUSDC goes wallet → wallet, no intermediaries |
| **Settlement time** | T+1 to T+5 business days | Finality in seconds, 24/7/365 |
| **FX spread** | 1–3% spread over interbank rate | 0.10% flat, no hidden spread |
| **Transit opacity** | Funds invisible while moving through correspondents | All on-chain: transaction hash immediately visible |
| **Rejection risk** | Correspondent bank can reject without explanation; funds return 3–7 days later minus fees | Smart contract accepts/rejects deterministically; no discretionary criteria |
| **Small amount economics** | $500 wire: $40 fee + 2% spread = 12% of amount | 0.10% works equally for $50 or $500,000 |

---

## Why No One Can Shut It Down

A canister runs on a **subnet**: 28 physical machines operated by **independent node providers** (companies, universities, data centers in different countries). The canister only needs ≥19/28 nodes to be operational.

To "shut down" a canister, you would need to convince ≥19 independent node operators to refuse to process it — or pass a proposal through the network DAO (500K+ neurons from thousands of independent holders).

**Byzantine fault tolerance:** with 28 nodes, the network tolerates up to 9 simultaneously malicious or failed nodes (f, where n = 3f+1). An attacker would need to control ≥10 physically distributed nodes operated by different companies in different countries simultaneously.

**Current state:** protocol canisters are controlled by the founder's controller key during bootstrap. The roadmap is to migrate control to the governance canister before handling real user funds at scale — at that point no single individual can modify the protocol unilaterally.

---

## HTTPS Outcalls — What "Native External APIs" Means

In EVM chains (Ethereum), a smart contract cannot make HTTP requests — the design prioritizes determinism over connectivity. External oracles (Chainlink) exist to bridge this gap with a third-party dependency.

Here, **HTTPS Outcalls** is a protocol-level feature, not an add-on:

1. All 28 subnet nodes make **the same HTTP request independently**
2. Each node gets its response
3. The response is canonicalized (non-deterministic parts removed: timestamps, server headers)
4. If ≥19/28 nodes get the same canonical result → accepted as valid
5. The canister receives the data as if it had done a normal fetch

**Scope:** any public HTTPS endpoint — Koywe, Circle, market price APIs, Chilean government APIs (SII, CMF), banking APIs with REST, enterprise ERPs.

**Design consideration for API keys:** stored in canister stable state, accessible only by the controller. No native secret manager equivalent to AWS Secrets Manager — this is a known design trade-off, not a gap.

---

## tECDSA — No Bridges to Ethereum

Cross-chain bridges exist because blockchains cannot "speak" directly to each other. This architecture eliminates them:

- **ckUSDC** is not bridged via Wormhole or Axelar
- The real USDC (issued by Circle) sits at a specific Ethereum address controlled by the network's cryptography
- The private key never exists complete anywhere — it is fragmented across 28 subnet nodes using Distributed Key Generation (DKG)
- To sign an Ethereum transaction, ≥19 nodes contribute their mathematical fragment; fragments combine to produce a valid ECDSA signature
- The resulting Ethereum address is real and usable — the signing key exists nowhere as a whole

**What "sending ckUSDC" actually is:** atomically changing the ledger record of which Principal holds that USDC representation. No intermediate state, no transit period, no correspondent risk.

**Credentials that disappear vs. traditional stack:**
- No `DATABASE_URL`
- No `REDIS_URL`
- No `AWS_ACCESS_KEY_ID`
- No `JWT_SECRET` — authentication is cryptographic, not a shared token

---

## Business Logic in the Canister

```motoko
public shared(msg) func depositVault(...) {
  requireLive();           // rule 1: system must be Live
  requireAuth(msg.caller); // rule 2: caller cannot be anonymous
  // ... deposit logic
}
```

These rules are in the WebAssembly code deployed on the subnet. All 28 nodes execute exactly that same code. To change the rules requires a **canister upgrade** — recorded on-chain, visible to anyone. No hidden admin mode. No silent redeploy.

---

## AI Agents on the Protocol

The protocol supports HTTPS Outcalls to AI APIs (Claude, OpenAI, etc.):

| Constraint | Why | How handled |
|---|---|---|
| Cannot run LLM inside canister | 7B model needs 14GB+ RAM; canisters have max 4GB stable memory | Models run on Anthropic/OpenAI servers; canister calls via HTTPS Outcall |
| LLMs are non-deterministic | Different responses to same prompt breaks 28-node consensus | Call from one node or accept variation with transform functions |
| Compute limit per message | ~50B WASM instructions per update call | Inference runs externally; canister orchestrates and stores results |
| HTTPS Outcall latency | 2–30 seconds per external API call | Acceptable for portfolio analysis; not for real-time trading |

---

## Data Privacy — Ley 21.719 (Chile, vigencia 2027)

Chile's new data protection law (GDPR-modeled) requires:
- **Right of access / portability:** user can request all their data
- **Right to erasure:** user can request verifiable complete deletion

**Why traditional systems struggle:** user data is dispersed across the main DB, application logs, S3 backups, cache, CRM, analytics, payment processors, data warehouses, and read replicas. Finding and verifying complete deletion across all of these is practically impossible.

**Protocol design advantage:** all user state is keyed by Principal ID. The canister is the single source of truth.

```motoko
// Find all data for a user — one function:
query_user_data(principal: Principal) : async UserDataExport {
    {
        vaultPositions = Map.get(vaultPositions, principal);
        stakePositions = Map.get(stakePositions, principal);
        harvestHistory = Map.get(harvestHistory, principal);
    }
}

// Delete all data for a user — one transaction:
update delete_user_data(principal: Principal) {
    requireController();
    Map.delete(vaultPositions, principal);
    Map.delete(stakePositions, principal);
    Map.delete(harvestHistory, principal);
}
```

**Blockchain append-only boundary:** the transaction log (ICRC-3) is cryptographically append-only. The design solution: logs contain only the Principal (cryptographic pseudonym), never PII (name, RUT). Deleting the Principal ↔ real identity mapping from mutable state leaves the log as anonymous data — compliant by design.

**Commercial argument for Chilean enterprises:** what companies are building for 2027 compliance — data subject request systems, deletion coordination across 11 systems — this architecture provides from day one. No data catalog to maintain. No consulting engagement needed.

---

## Current Code State (2026-08-26)

### Live on mainnet

| Component | Canister | Status |
|-----------|---------|--------|
| PXRM token ledger | `q7nmw-diaaa-aaaah-quy4a-cai` | ✅ Live — ICRC-1/ICRC-2 |
| Protocol backend | `backend` canister | ✅ Live — vaults, yield, staking |
| Frontend | Asset canister | ✅ Live — React PWA |
| Oracle | Price oracle canister | ✅ Live — CoinGecko + mindicador.cl |
| CDP | CDP canister | ✅ Live — ckBTC/ckETH/ckUSDC/ICP collateral |
| vUSD ledger | ICRC-1 canister | ✅ Live — CDP mint active |
| API gateway | `api_gateway` canister | ✅ Live — combined portfolio endpoint |
| Staking | `staking` canister | ✅ Live — lock multipliers |
| Fee splitter | `fee_splitter` canister | ✅ Live — 35/25/23/10/7 split |

### In development

| Component | Status |
|-----------|--------|
| `koywe_bridge` canister | Architecture designed, implementation in progress |
| `sCLP` corridor (Fintoc) | Code complete, pending CMF regulatory approval |
| WaterNeuron integration | Designed, deployment pending |
| Governance canister | Designed, to activate before production with real funds |
| Epoch pool wiring | T1–T5 accrual in code, on-chain payout wiring in progress |
| LUNX bridge to Ethereum | Roadmap V2 |

### Deployment

```bash
# Compile and deploy any canister to mainnet:
dfx deploy backend --network ic

# Check canister logs:
dfx canister logs backend --network ic

# Verify canister state:
dfx canister status backend --network ic
```

No Docker, no EC2, no RDS, no ALB. One command compiles Motoko to WebAssembly and distributes to 28 nodes.

---

## LatAm Market — Where the Protocol Fits

| Zone | Customer | Pain | Protocol product |
|---|---|---|---|
| Santiago | Importer/exporter with international payments | SWIFT slow + 1–3% spread | ODL Bridge + Stablecoin Vault |
| Santiago | SME with idle CLP/USD treasury | No yield, inflation erodes capital | Stablecoin Vault (ckUSDC APR) |
| Mendoza / ARG | Company with Chile-Argentina cross-border ops | ARS exchange controls, peso risk | ckUSDC as store of value + corridor |
| Lima / Bogotá | Company paying Chilean suppliers | High international transfer cost | Operator API (Track B Guild) |
| Santiago / Tech | Startup wanting payment infrastructure without banking setup | Stripe doesn't cover all Chilean cases | Protocol SDK + vUSD Services |

---

## MILA Case Study — Why On-Chain Settlement Changes the Math

MILA (Mercado Integrado Latinoamericano) integrated Chile, Peru, and Colombia's stock exchanges in 2011. The thesis: diversification across LatAm reduces volatility through low correlation.

**What failed:**
- In global crises, all LatAm markets fall together — the correlation disappears exactly when you need it most
- Cross-border investors absorbed double FX risk: CLP→PEN at buy, PEN→CLP at sell
- The same stock could differ 2–5% in price between markets because the 2-day settlement cycle was slower than the price difference — arbitrage was impossible

**How on-chain settlement changes the math:** if tokenized assets (RWAs) settled in ckUSDC, cross-market price differences would be arbitraged in ~2 seconds instead of 2 days. Both buyer and seller settle in the same unit of value with no FX exposure in transit. The correlation problem becomes a non-issue when settlement is atomic.

This is the RWA/tokenization case that this architecture enables — not in V1, but the design assumption behind the institutional adoption roadmap.

---

*Engineer Pitch · Protocol Architecture · 2026-08-26*
