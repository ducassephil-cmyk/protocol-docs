# Protocol Architecture — Technical Overview
> Reference document for IT professionals and software engineers
> Version: 2026-08-26

---

## Stack Comparison — Traditional vs. Protocol Architecture

| Traditional Stack | What it does | Protocol Equivalent | Technical Name |
|---|---|---|---|
| Node.js + Express | HTTP server, processes requests | **Canister** (WebAssembly module) | Update call (writes state) / Query call (read-only) |
| PostgreSQL / MySQL | Persistent data storage | **Stable variables** inside the canister | Orthogonal persistence (state survives upgrades automatically) |
| Redis | In-memory cache, fast reads | **Query calls** — no state write, instant response | Query calls (no consensus required — single node responds) |
| AWS S3 | File/asset storage | **Asset canister** | Asset canister (up to 64GB per subnet) |
| AWS API Gateway | Routes HTTP to backend | **Boundary node + IC Agent** | Boundary nodes (network-managed entry points) |
| Auth0 / Firebase Auth | User authentication | **Internet Identity + Principal ID** | Principal (each wallet/user has a unique cryptographic identity) |
| Chainlink / Pyth | External data oracle | **HTTPS Outcalls with consensus** | HTTPS Outcalls (28 nodes make the same HTTP request and vote on the result) |
| Wormhole / Axelar | Cross-chain bridge | **Chain Fusion / tECDSA** | Chain Key Technology (no bridge — the protocol signs natively on Ethereum) |
| Docker container | Isolated execution environment | **Canister** (WebAssembly sandbox) | WASM runtime (each canister is an isolated WebAssembly module) |
| CI/CD pipeline | Code deployment | `dfx deploy <canister>` — **one command** | dfx CLI |
| AWS Lambda | Serverless function | **Update call** to a canister | Update call (executed in consensus by 28 nodes) |
| Cron job / scheduler | Scheduled background tasks | **Canister heartbeat / Timers** | `recurringTimer` in Motoko |
| SQS / RabbitMQ | Async message queue | **Inter-canister calls** | Inter-canister calls (one canister calls another directly) |
| JWT / session | Session management | **Delegated identity** | Internet Identity delegation (valid for defined time and scope) |
| Datadog / CloudWatch | Observability | **IC Dashboard + canister logs** | `dfx canister logs` + icscan.io |

---

## Key Concepts

### What "same process" means

In traditional architecture, an app has **separate processes communicating over the network**:
- Node.js server is one process
- PostgreSQL is another process, on another machine, queried over TCP
- Redis is another process
- All communicate via HTTP or TCP — with latency, failure points, and credentials to manage

In this architecture, **a canister is a single WebAssembly module** containing:
- Business logic (the code)
- Persistent storage (stable variables)
- The entry interface (public methods equivalent to HTTP endpoints)

All of this runs in **the same module**, with no internal network. When `depositVault` calls `calculateYield`, there is no HTTP request — it is a function call within the same module. No latency between layers, no serialization/deserialization, no database credentials to rotate.

---

### Why no one can shut it down

A canister runs on a **subnet**: a group of 28 physical machines operated by **independent node providers** (companies, universities, data centers in different countries). The canister only needs ≥19/28 nodes to be operational.

To "shut down" a canister, you would need to:
1. Convince ≥19 node operators to refuse to process that canister, OR
2. Pass a proposal through the NNS (the network DAO with 500K+ neurons from thousands of independent holders)

Neither is practically achievable by any single actor.

**Current state:** the protocol's canisters are controlled by the founder's controller key during the bootstrap phase. The roadmap is to migrate control to the governance canister — at which point no individual can modify the protocol unilaterally.

---

### HTTPS Outcalls — native external APIs

**Native** = the protocol supports it without installing anything or depending on a third party.

In EVM chains, a smart contract cannot make HTTP requests — they were designed to be deterministic and isolated from the outside world. External oracles (Chainlink) exist to bridge this gap.

In this architecture, **HTTPS Outcalls** is a protocol-level feature:

1. All 28 subnet nodes make **the same HTTP request independently**
2. Each node gets its response
3. The response is canonicalized (non-deterministic parts removed: timestamps, server headers)
4. If ≥19/28 nodes get the same canonical result → the response is accepted as valid
5. The canister receives the data as if it had done a normal fetch

**Scope:** any public HTTPS endpoint — payment processors, market price APIs, banking APIs, government APIs, enterprise ERPs. If it has HTTPS, the protocol can call it.

**Design note for API keys:** API keys are stored in the canister's stable state, accessible only by the controller. A dedicated secret manager is not available natively — this is a known design consideration.

---

### No bridges — tECDSA and Chain Fusion

Cross-chain bridges exist because blockchains cannot "speak" to each other directly. This architecture eliminates them:

- **ckUSDC** is not bridged via Wormhole or Axelar. The real USDC (issued by Circle) sits at a specific Ethereum address controlled by the network's cryptography (`0xA17a8883dA1abd57c690DF9Ebf55d51d76042e`). The network holds the private key across 28 nodes using threshold ECDSA — the key never exists complete in any single location.
- Sending ckUSDC is not "transferring" USDC from one chain to another. It is atomically changing the ledger record of which Principal holds that USDC representation. No intermediate state, no transit period, no correspondent risk.

**Credentials that disappear compared to traditional stack:**
- No `DATABASE_URL`
- No `REDIS_URL`
- No `AWS_ACCESS_KEY_ID`
- No `JWT_SECRET` — authentication is cryptographic, not a shared token

---

### Consensus — how 28 nodes process a transaction

When a user calls `depositVault`:

1. Message arrives at any subnet node
2. That node propagates it to all others (gossip protocol)
3. 28 nodes agree on an **order** for this message (BFT consensus protocol)
4. All 28 nodes execute **the same WebAssembly instruction** in the same order
5. They compute the hash of the new canister state
6. If ≥19/28 have the same hash → transaction is valid and state is updated
7. Result returns to the user signed by the subnet's **threshold BLS signature**

**Byzantine fault tolerance:** the network tolerates up to f malicious/failed nodes where n = 3f+1. With 28 nodes: f = 9. An attacker would need to control ≥10 physically distributed nodes operated by different companies in different countries simultaneously.

**For tECDSA (signing on Ethereum):**
- The private key never exists complete anywhere
- Generated with DKG (Distributed Key Generation): each node holds a mathematical fragment
- To sign an ETH transaction, ≥19 nodes contribute their fragment
- Fragments combine mathematically to produce a valid ECDSA signature
- The resulting Ethereum address is real and usable — but the signing key exists nowhere as a whole

---

### Deployment — one command

```bash
dfx deploy backend --network ic
```

That command:
1. Compiles Motoko code to WebAssembly (WASM)
2. Packages the WASM
3. Sends it to the canister on mainnet
4. The subnet distributes the WASM to 28 nodes
5. Canister is updated

Compare to deploying Node.js + PostgreSQL + Redis to AWS: EC2 configuration, RDS setup, VPC, migrations, ALB, certificates, auto-scaling, CloudWatch — hours or days of DevOps. Here: one command, fractions of a cent in cycles.

---

### Business logic in the canister

**Business logic** = the specific rules of the application:
- "Deposits are only accepted when the system is in Live state"
- "Yield is calculated as `bps/10000 × elapsed_time × token_price`"
- "Only the controller can change the system state"

In traditional architecture, these rules live on the Node.js server. If the server is compromised, rules can be changed. If the owner wants to change them silently, they can.

**In the canister:**
- Rules are in the WebAssembly code deployed on the subnet
- All 28 nodes execute exactly that same code
- To change rules requires a **canister upgrade** — recorded on-chain, visible to anyone
- Rules apply equally to every call from every user without exception

Example from the protocol's backend (`main.mo`):
```motoko
public shared(msg) func depositVault(...) {
  requireLive();           // rule 1: system must be Live
  requireAuth(msg.caller); // rule 2: caller cannot be anonymous
  // ... deposit logic
}
```
No call can bypass these rules regardless of who makes it.

---

## What Stays Outside the Protocol

The protocol covers infrastructure, not the full stack:

| What is NOT on-chain | Who handles it |
|---|---|
| DNS / domain | External domain registrar + boundary node as server |
| Fiat on/off ramp (CLP ↔ ckUSDC) | Koywe — regulated company with real bank accounts |
| KYC/AML | Koywe handles its side; the protocol entity handles its own |
| Legal entity (contracts, invoices, tax) | Vaelix SpA — entirely outside the protocol |
| Email / notifications | HTTPS Outcalls to email APIs (no native mail server) |
| Mobile app distribution (iOS/Android) | Google Play / App Store — external rules apply |

---

## SWIFT vs. Protocol ODL — Problem/Solution

| SWIFT Problem | What happens exactly | Protocol solution |
|---|---|---|
| **Correspondent banking** | Your bank doesn't have a direct account at the beneficiary's bank. Uses 2–4 intermediary banks, each charges $10–30 USD. | ckUSDC goes directly: wallet → wallet. No intermediaries. |
| **Settlement time** | T+1 to T+5 business days. Weekends → until Monday. | Finality in seconds, 24/7/365. |
| **FX spread** | Banks apply 1–3% spread over interbank rate. On $10K USD = $100–300 USD lost. | 0.10% flat on the amount, no hidden spread. |
| **Transit opacity** | No visibility into where funds are while moving through correspondents. | All on-chain: the transaction hash is immediately visible. |
| **Rejection risk** | A correspondent bank can reject a SWIFT transfer for compliance reasons without explanation. Funds return 3–7 days later minus fees. | A smart contract accepts or rejects deterministically. If it meets the code rules, it passes. No discretionary criteria. |
| **Minimum thresholds** | For $500 USD, a $40 SWIFT fee + 2% spread = 12% of the amount in costs. | 0.10% works equally for $50 or $500,000. |

---

## Data Privacy — Ley 21.719 (Chile, 2027)

Chile's data protection law (modeled on GDPR) requires two capabilities:
- **Right of access / portability:** user can request all their data
- **Right to erasure:** user can request complete deletion, verifiably

**Why traditional systems struggle:** a typical company has user data dispersed across the main DB, application logs, S3 backups, cache, email marketing systems, CRM, analytics platforms, payment processors, data warehouses, and read replicas. Finding everything takes weeks; verifying complete deletion is nearly impossible.

**Protocol design advantage:** all user state is associated with their Principal ID. The canister is the single source of state — no dispersal across 11 systems.

```motoko
// Find all data for a user (one function):
query_user_data(principal: Principal) : async UserDataExport {
    {
        vaultPositions = Map.get(vaultPositions, principal);
        stakePositions = Map.get(stakePositions, principal);
        harvestHistory = Map.get(harvestHistory, principal);
    }
}

// Delete all data for a user (one transaction):
update delete_user_data(principal: Principal) {
    requireController();
    Map.delete(vaultPositions, principal);
    Map.delete(stakePositions, principal);
    Map.delete(harvestHistory, principal);
}
```

**Design boundary:** the transaction log (ICRC-3) is append-only by cryptographic nature — a recorded transaction cannot be deleted without breaking the integrity chain. The solution is by design: the log contains only the Principal (a cryptographic pseudonym), never the real-world identity (name, RUT). If the Principal ↔ real identity mapping is deleted from mutable state, the log remains as anonymous data.

**Commercial argument:** what Chilean companies are building for 2027 compliance — data subject request systems — this architecture provides by design from day one. No separate data catalog to maintain, no coordinated deletion across 11 systems, no data governance consulting needed.

---

## AI Agents on the Protocol

The protocol supports HTTPS Outcalls to AI APIs (Claude, OpenAI, etc.). Runtime constraints:

| Constraint | Why it exists | How it's handled |
|---|---|---|
| Cannot run an LLM inside a canister | A 7B parameter model requires 14GB+ RAM; canisters have max 4GB stable memory | Models run on Anthropic/OpenAI servers; canister calls via HTTPS Outcall |
| LLMs are non-deterministic | GPT/Claude can give different responses to the same question; 28-node consensus requires identical results | Canister makes the call from one node or accepts variation with transform functions |
| Compute limit per message | ~50 billion WebAssembly instructions per update call — enough for financial logic, not AI inference | Inference runs externally; canister orchestrates and stores results |
| HTTPS Outcall latency | 2–30 seconds per external API call | Acceptable for portfolio analysis; not for real-time trading |

---

## LatAm Market Positioning

| Zone | Natural customer | Main pain | What the protocol avoids | Product |
|---|---|---|---|---|
| Santiago | Importer/exporter with international payments | SWIFT slow + 1–3% spread + correspondents | The full international leg | ODL Bridge + Stablecoin Vault |
| Santiago | SME with idle treasury in CLP/USD | CLP yields nothing, inflation erodes capital | — (different pain: yield on digital dollars) | Stablecoin Vault |
| Mendoza / ARG | Company with Chile-Argentina cross-border operations | ARS restrictions, exchange controls, peso risk | Unstable ARS → stable ckUSDC → CLP without formal banking | ckUSDC as store of value + corridor |
| Lima / Bogotá | Company paying Chilean suppliers or receiving remittances | High international transfer cost (SWIFT + FX) | Full international leg | Operator API (Track B) |
| Santiago / Tech | Startup wanting payments without building banking infrastructure | Stripe doesn't cover all Chilean cases; banking integrations expensive | International payment infrastructure | Protocol SDK |

---

*Technical Overview · Protocol Architecture · 2026-08-26*
