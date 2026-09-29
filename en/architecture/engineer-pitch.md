# GREYVALLEY — ICP explained to the Chilean engineer/IT professional
> Reference document for technical conversations with IT professionals, systems engineers, and infrastructure teams.
> Generated: 2026-08-08 | Not official DFINITY documentation.

---

## MAIN COMPARISON TABLE
### Node.js / PostgreSQL / AWS vs ICP — what each thing does and its ICP equivalent

| Traditional stack | What it does | ICP equivalent | ICP technical name |
|---|---|---|---|
| Node.js + Express | HTTP server, processes requests | **Canister** (WebAssembly module) | Update call (writes state) / Query call (read-only) |
| PostgreSQL / MySQL | Stores data persistently | **Stable variables** inside the canister | Orthogonal persistence (state survives upgrades automatically) |
| Redis | In-memory cache, fast reads | **Query calls** — don't write state, respond instantly | Query calls (no consensus required — only one node responds) |
| AWS S3 | File/asset storage | **Asset canister** | Asset canister (up to 64GB per subnet) |
| AWS API Gateway | Routes HTTP requests to the backend | **Boundary node + IC Agent** | Boundary nodes (the network's "entry ports") |
| Nginx / Load balancer | Distributes traffic | **Boundary nodes** | Boundary nodes (managed by the network, not by GreyValley) |
| Auth0 / Firebase Auth | User authentication | **Internet Identity + Principal ID** | Principal (every wallet/user has a unique cryptographic identity) |
| Chainlink / Pyth | External data oracle | **HTTPS Outcalls with consensus** | HTTPS Outcalls (the 28 nodes make the same HTTP request and vote on the result) |
| Wormhole / Axelar / LayerZero | Bridge between blockchains | **Chain Fusion / tECDSA** | Chain Key Technology (no bridge — ICP signs natively on Ethereum) |
| Docker container | Isolated execution environment | **Canister** (WebAssembly sandbox) | WASM runtime (each canister is an isolated WebAssembly module) |
| CI/CD pipeline | Code deployment | `dfx deploy <canister>` — **one command** | dfx CLI |
| AWS Lambda | Serverless function | **Update call** to a canister | Update call (executed by consensus of the 28 nodes) |
| Cron job / scheduler | Periodic automated tasks | **Canister heartbeat / Timers** | `ic_cdk::timer` in Rust, `recurringTimer` in Motoko |
| SQS / RabbitMQ | Async message queue | **Inter-canister calls** | Inter-canister calls (one canister calls another canister directly) |
| JWT / session | User session management | **Delegated identity** | Internet Identity delegation (valid for a defined time and scope) |
| Environment variables (.env) | Configuration and secrets | ⚠️ Real gap: no native secret manager | Secrets are stored in canister state (only the controller has access) |
| Monitoring (Datadog, CloudWatch) | System observability | **IC Dashboard + canister logs** | `dfx canister logs` + ic.rocks / icscan.io |
| DNS | Domain names | **Own domains** pointing to the boundary node | Custom domain (ICP resolves it, but DNS itself is still external) |

---

## KEY CONCEPTS — QUESTION BY QUESTION

### "Same process" — what it actually means

In a traditional architecture, an app has **separate processes that talk over the network**:
- The Node.js server is one process.
- PostgreSQL is another process, on another machine, that Node.js queries over TCP.
- Redis is another process.
- The API Gateway is another process.
- They all "talk" to each other over HTTP or TCP — with latency, failure points, and credentials to manage.

On ICP, **a canister is a single WebAssembly module** that contains:
- The business logic (the code)
- Persistent storage (the stable variables)
- The entry interface (the public methods, equivalent to HTTP endpoints)

All of this runs in **the same module**, with no internal network. When `depositVault` calls `calculateYield`, there is no HTTP request — it's a function call within the same module. This is what "same process" means: there is no separation between your server, your database, and your logic.

**The practical difference:** no internal latency, no serialization/deserialization between layers, no database credentials to rotate.

---

### Why no one can shut it down — not even Dfinity

A canister runs on a **subnet**: a group of 28 physical computers operated by **independent node providers** (companies, universities, data centers in different countries). For the canister to keep working, it only needs ≥19/28 nodes to be operational.

**Can Dfinity shut it down?**

Dfinity doesn't control the 28 nodes directly — they are independent operators who signed a contract with the network. To "shut down" a specific canister that Dfinity doesn't control, they would need to:
1. Convince ≥19 node operators to refuse to process that canister, OR
2. Pass a proposal through the NNS (ICP's DAO, with ~500K+ neurons from thousands of independent holders) to intervene on that canister.

The second is theoretically possible but requires an NNS majority — which includes competitors, investors, and users who have no incentive to arbitrarily shut down canisters.

**The practical case for GreyValley:** if at some point GreyValley's canisters no longer have a human controller (but instead the governance canister as controller), neither GreyValley SpA nor Dfinity can modify them unilaterally. That is exactly the CMF argument: "no one controls the protocol, the code's rules do."

**Being honest:** today, GreyValley's canisters DO have a human controller (the founder). That is the critical pending regulatory point — the goal is to migrate control to the governance canister.

---

### "Native" external APIs — what native means

**Native** = the protocol supports it with nothing installed, with no dependence on a third party.

On Ethereum, if you want a smart contract to get the USD/CLP price, you need an external oracle (Chainlink). The smart contract can't make an HTTP request on its own — Ethereum was designed to be deterministic and isolated from the outside world.

On ICP, **HTTPS Outcalls** is a protocol feature: any canister can make an HTTPS request to any public URL in the world. Internally:
1. The subnet's 28 nodes make **the same HTTP request independently**.
2. Each node gets its response.
3. The response is "canonicalized" (parts that vary are removed: timestamps, server headers, etc.).
4. If ≥19/28 nodes get the same canonical result → the response is accepted as valid.
5. The canister receives the data as if it had made a normal fetch.

**"Any" API?** Any public HTTPS endpoint. Koywe, Circle, market prices, the SII's (Chilean tax authority) API, a corporate ERP, a bank with a REST API. If it has HTTPS, ICP can call it.

**Real limitation:** APIs that require authentication (API key). The key is stored in canister state (not in .env), accessible only by the controller. It's not perfectly secret — if someone gains access to the controller, they can read it. There's no native "secret manager" like AWS Secrets Manager.

---

### No middlewares — which ones, and why

The "middlewares" that ICP eliminates:

**Blockchain bridges (Wormhole, Axelar, LayerZero):**
These exist because Ethereum can't "talk" to Solana directly — you need an external service that locks tokens on one chain and releases them on another. ICP uses tECDSA to sign transactions directly on Ethereum with no intermediary. ckUSDC doesn't use Wormhole — it uses ICP's own cryptography.

**Oracles (Chainlink):**
These exist because EVM smart contracts can't make HTTP requests. HTTPS Outcalls remove this need: the canister queries the data source directly.

**API Gateways (Kong, AWS API Gateway):**
These exist to route, authenticate, and rate-limit requests to the backend. On ICP, boundary nodes play that role. Authentication is done cryptographically (Principal ID) inside the canister — no need to configure a separate gateway.

**Credentials that disappear:**
- `DATABASE_URL=postgresql://user:pass@host:5432/db` → doesn't exist on ICP
- `REDIS_URL=redis://...` → doesn't exist
- `AWS_ACCESS_KEY_ID` → doesn't exist
- `JWT_SECRET` → doesn't exist (authentication is cryptographic, not via a shared token)

What DOES exist: the **principal controller** (the founder's dfx identity). That is THE critical ICP credential — see `INSTRUCCIONES_FOUNDER.md §13`.

---

### Everything in one canister — the PXRM ledger at the heart of GreyValley

> Historical note: the original PXRM ledger was compromised (lost minting key, ~20M
> phantom supply) and was replaced on 2026-08-09 by the ID below. Detail in
> `GREYVALLEY_MASTER_STATE.md` §29.

The **PXRM ledger canister** (canister ID: `q7nmw-diaaa-aaaah-quy4a-cai`) is a canister that implements the ICRC-1 standard. It contains:
- All PXRM balances of all holders (in stable memory)
- The transfer, approve, transfer_from logic
- The transaction log (ICRC-3)
- The rules for who can mint (only the designated minter)

There is no PostgreSQL database behind it. There is no Node.js server handling transfers. The ledger IS the bank. It's the same model as the ckUSDC PXRM ledger (canister ID: `xevnm-gaaaa-aaaar-qafnq-cai`) — which manages ALL the ckUSDC of ALL ICP users.

**GreyValley's backend** (`main.mo`) is the main canister that orchestrates everything:
- Receives deposits → calls the ICRC-2 ledger to transfer tokens to the canister
- Calculates yield → purely mathematical in-canister logic
- Pays out returns → calls the PXRM ledger to transfer to the user
- Queries prices → HTTPS Outcall to the price API
- Verifies governance → inter-canister call to the governance canister

All of this happens through canister-to-canister calls, with no external server coordinating anything.

---

### SWIFT — specific complications and how ICP solves them

| SWIFT problem | What exactly happens | ICP / GreyValley solution |
|---|---|---|
| **Correspondent banking** | Your Chilean bank doesn't have a direct account at the beneficiary's bank. It uses 2–4 intermediary banks, each charging $10–30 USD. | ckUSDC goes direct: wallet → wallet. No intermediaries. |
| **Settlement time** | T+1 to T+5 business days. Weekend → until Monday. | Finality in seconds, 24/7/365. |
| **FX spread** | The bank applies a 1–3% spread over the interbank exchange rate. On $10,000 USD = $100–300 USD lost. | 0.66% flat on the amount, no hidden spread (FIX 2026-09-14: it said 0.10%, the fee has since gone up twice). |
| **Opacity in transit** | You don't know where your money is while it travels through correspondents. | Everything on-chain: the transaction hash is immediately visible. |
| **Rejection risk** | A correspondent bank can reject the SWIFT for compliance reasons with no explanation. Your money comes back 3–7 days later minus fees. | A smart contract deterministically accepts or rejects. If it meets the code's rules, it goes through. No discretionary judgment. |
| **Profitable minimums** | For $500 USD, a $40 USD SWIFT + 2% spread = 12% of the amount in costs. Doesn't make sense for small amounts. | 0.66% works the same for $50 or $500,000. |
| **Fund freezes** | In some countries (Argentina), the regulator can freeze SWIFT transfers. | No entity has the power to freeze ckUSDC in transit (unless the whole of ICP is attacked — unlikely). |

**Why Koywe and not just GreyValley?**

Koywe solves the **first- and last-mile** problem: converting CLP (which lives in the Chilean banking system) into ckUSDC (which lives on ICP). That conversion **still requires the banking system** — someone has to receive the CLP transfer and issue the ckUSDC.

GreyValley solves everything that comes **after**: storage, yield, international transfer, institutional integration. SWIFT is avoided at the most expensive leg — international transfer — because that leg happens 100% on-chain.

**Analogy:** Koywe is the port of entry and exit. ICP is the transport system that operates between ports. The ship (ckUSDC) doesn't use SWIFT shipping lanes — it travels instantly across the network.

---

### "It doesn't travel — immediate representation — the bank doesn't have your money"

When you make a traditional bank transfer:
1. Your bank **debits** your account (the balance drops on your screen).
2. It sends a SWIFT message to the destination bank.
3. The money enters the **correspondent** system — it literally exists in your bank's account at the correspondent bank, which moves it to another, which moves it to another.
4. The destination bank **credits** the beneficiary when it receives and processes the SWIFT.
5. For 1–5 days, **the money belongs to no one** — it's in the banking system, in transit.

With ckUSDC:
- The real USDC (USD Coin issued by Circle) **never moves** from the Ethereum address controlled by ICP (`0xA17a8883dA1abd57c690DF9Ebf58fD551d76042e`). That USDC sits still on Ethereum.
- What changes is the **record in ICP's ledger**: the line that says "this Principal has X ckUSDC" changes to say "that Principal has X ckUSDC."
- That record change **is atomic** — it happens in a single transaction, in seconds. There is no intermediate state where the money is "traveling."
- The bank never had your money: the USDC is custodied by ICP's cryptography (28 nodes with threshold ECDSA), not by a financial institution that can go bankrupt, be hacked, or freeze your account.

---

### Business logic "in the canister" — what it means

**Business logic** = the specific rules of your application. In GreyValley:
- "I can only deposit if the system is in Live state"
- "Yield is calculated as `bps/10000 × elapsed_time × token_price`"
- "Only the controller can activate the system"
- "If the user already has an open position, add to the existing balance"

In a traditional architecture, these rules live on your Node.js server. If the server goes down, the rules don't apply. If the server gets hacked, the rules can be changed. If the owner wants to change them, they edit the code and redeploy without anyone knowing.

**In the canister:**
- The rules live in the WebAssembly code deployed on the subnet.
- All 28 nodes execute exactly that same code.
- To change the rules, you need to do a **canister upgrade** — which is recorded on-chain, visible to anyone.
- The rules apply the same way to any call, from any user, with no exceptions — the code has no hidden "admin mode" that skips the rules (except explicit admin functions in the code).

**Real example in GreyValley (`main.mo`):**
```motoko
public shared(msg) func depositVault(...) {
  requireLive();           // rule 1: the system must be Live
  requireAuth(msg.caller); // rule 2: the caller cannot be anonymous
  // ... deposit logic
}
```
These two lines are business logic in-canister. Nobody can call `depositVault` unless the system is Live and there's a valid identity — no matter how hard they try.

---

### Everything by consensus — the 28 nodes and cryptography

When a user calls `depositVault`:

1. The message arrives at any node in the subnet.
2. That node propagates it to all the others (gossip protocol).
3. The 28 nodes agree on an **order** for this message (using ICP's BFT consensus protocol).
4. All 28 nodes execute **the same WebAssembly instruction** in the same order.
5. They compute the hash of the canister's new state.
6. If ≥19/28 have the same hash → the transaction is valid and the state updates.
7. The result goes back to the user, signed by the subnet's **threshold BLS signature**.

**Why 28/19:** this is called Byzantine fault tolerance (BFT). The network tolerates up to f nodes being malicious or down, where n = 3f+1. With 28 nodes: f = 9. You need ≥19 correct ones. An attacker would need to control ≥10 nodes of the subnet simultaneously — nodes physically distributed across different countries, operated by different companies. The cost of that attack is astronomical.

**For tECDSA (signing on Ethereum):**
- The private key never fully exists anywhere.
- It's generated with DKG (Distributed Key Generation): each node receives a mathematical "share."
- To sign an ETH transaction, ≥19 nodes contribute their share.
- The shares are combined mathematically to produce a valid ECDSA signature.
- The resulting Ethereum address is real and usable — but the key to move it doesn't exist on any server.

---

### The "one command" — `dfx deploy`

```bash
dfx deploy backend --network ic
```

That's it. That command:
1. Compiles the Motoko code to WebAssembly (WASM).
2. Packages the WASM.
3. Sends the WASM to the canister on mainnet.
4. The subnet distributes the WASM to the 28 nodes.
5. The canister is updated.

Compare that with deploying Node.js + PostgreSQL + Redis to AWS:
- Create an EC2 instance, configure security groups, install Node.js
- Create RDS, configure VPC, create tables with migrations
- Create ElastiCache, configure connection pooling
- Configure ALB, ACM certificate, Route 53
- Write a Dockerfile, push to ECR
- Create ECS task definitions, configure auto-scaling
- Configure CloudWatch, alerts, RDS backups
- Total: hours or days of DevOps

On ICP: `dfx deploy`. Cycles (fractions of a cent). Done.

---

### AI agent limitations on ICP

ICP can make HTTPS Outcalls to AI APIs (Claude, OpenAI, Gemini) — that works fine. The limitations come from the **canister runtime**:

| Limitation | Why it exists | GreyValley's solution |
|---|---|---|
| You can't run an LLM inside the canister | A 7B-parameter model needs 14GB+ of RAM. A canister has a maximum of 4GB of stable memory. | The models run on Anthropic's/OpenAI's servers. The canister makes the call via HTTPS Outcall. |
| LLMs aren't deterministic | GPT or Claude can give different answers to the same question. 28-node consensus requires everyone to reach the same result. | The canister makes the call from ONE node (not full consensus) or accepts the variance with transform functions. |
| Compute limit per message | ~50 billion WebAssembly instructions per update call. Enough for financial logic, not for AI inference. | Inference runs on Anthropic/OpenAI, the canister just orchestrates and stores results. |
| HTTPS Outcall latency | 2–30 seconds per call to an external API. | For GreyValley: acceptable for portfolio analysis, not for real-time trading. |

**Current state at GreyValley:** the AI agents (Haiku for data, Opus for strategy) are called directly from the frontend. The roadmap is to move orchestration to the backend canister so analyses are on-chain and auditable.

---

### "Complete infrastructure" — what stays OUTSIDE ICP

ICP covers a lot, but not everything:

| What is NOT on ICP | Who handles it at GreyValley |
|---|---|
| DNS / domain (greyvalley.xyz) | External domain registrar (GoDaddy, Namecheap) + ICP's boundary node as the server |
| Fiat on/off ramp (CLP ↔ ckUSDC) | Koywe — a regulated company with real bank accounts |
| KYC/AML | Koywe handles it on its side; GreyValley SpA is responsible on its own |
| Legal entity (contracts, invoices, SII) | GreyValley SpA — completely outside ICP |
| Email and notifications | HTTPS Outcalls to email APIs (SendGrid, etc.) — there's no native mail server |
| App stores (iOS/Android) for the native wallet | Google Play / App Store — their rules are external |
| API key secrets for integrations | Canister state (accessible only by the controller) — not a dedicated vault |

---

## POSITIONING BY REGION — SWIFT vs GREYVALLEY

| Region | Natural client | Main pain point | What GreyValley avoids | Product |
|---|---|---|---|---|
| **Santiago** | Import/export company with payments abroad | Slow SWIFT + 1–3% spread + correspondents | The entire international leg | ODL Bridge + Exaltite Vault |
| **Santiago** | SME with idle CLP/USD treasury | CLP with no yield, inflation erodes capital | Nothing to do with SWIFT — it's yield in digital dollars | Exaltite Vault (ckUSDC) |
| **Mendoza / ARG** | Company with cross-border Chile-Argentina operations | ARS exchange restrictions, *cepo*, peso risk | Unstable ARS → stable ckUSDC → CLP without going through formal banking | ckUSDC as store of value + CLP↔ARS corridor |
| **Lima / Bogotá** | Company that pays suppliers in Chile or receives remittances | High cost of international transfer (SWIFT + FX) | The entire international leg | Corridor Guild Track B |
| **Santiago / Tech** | Startup that wants payments without building its own banking infrastructure | Stripe doesn't work in Chile for all cases; bank integrations are expensive | International payment infrastructure | GreyValley SDK + vUSD Services |

---

## CHILE + PERU + COLOMBIA — THE MILA CASE AND WHY IT DIDN'T WORK

**MILA (Latin American Integrated Market):** in 2011, Chile (Santiago Stock Exchange), Peru (BVL), and Colombia (BVC) integrated their stock markets. Mexico joined later. The promise: an investor in Lima could buy Falabella shares without going through international intermediaries.

**The non-correlation problem that was raised:**

The thesis was that diversifying across LatAm would reduce volatility (low correlation between markets). In practice:
- During global crises (COVID 2020, macro selloff 2022), **all LatAm markets fall together** because foreign capital exits "emerging markets" as a category. Non-correlation disappears exactly when you need it most.
- An investor who bought Peruvian shares from Chile took on **double currency risk**: CLP → PEN when buying, PEN → CLP when selling. If PEN fell while the shares rose, the return vanished in FX.
- **Asymmetric liquidity:** the bid/ask spread on Peruvian shares seen from Chile was wider than in the local market. Local market makers didn't cover the cross-border volume well.

**"Here and there" prices not correlated:** the same share on two markets should have the same price (arbitrage). But with 2-day settlement, fluctuating exchange rates, and asymmetric liquidity, the same asset could differ by 2–5% between markets — with no one able to arbitrage efficiently because the settlement cycle was slower than the price difference.

**How GreyValley solves this (roadmap):**

If tokenized shares (RWAs) settled in ckUSDC:
- An investor in Lima buys tokenized Falabella shares → pays in ckUSDC.
- Settlement happens in seconds, in the same unit of value (ckUSDC).
- There's no FX between PEN and CLP — both settled in digital dollars.
- The cross-market spread would be arbitraged instantly because the settlement cycle is ~2 seconds, not 2 days.

This is the RWA/tokenization use case that ICP enables — not available in GreyValley V1, but part of the "institutional adoption" vision mentioned in CLAUDE.md.

---

## IDLE TREASURY — RECOMMENDATIONS FOR THE CHILEAN COMPANY

### What money comes in with

The current route (without an active ODL corridor):
1. The Chilean company transfers CLP to Koywe → receives ckUSDC in its ICP wallet.
2. It deposits ckUSDC into the Exaltite Vault → starts accumulating PXRM as yield.
3. When it wants to exit: withdraws ckUSDC → Koywe converts it back to CLP.

**Practical recommendation:** start with USD already on hand (if the company exports or already holds dollars) — this avoids the first CLP→ckUSDC leg, which carries its own Koywe spread.

### Tax filing implications

Income tax filing in Chile happens in **April** for the previous year (January–December).

| Scenario | When it's taxed | Recommendation |
|---|---|---|
| Company deposits ckUSDC in December 2026 and withdraws in January 2027 | The yield (PXRM) would be declared in the 2027 tax return (payable April 2028), if the SII considers the taxable event to occur when the tokens are received. | If you want to defer the tax, holding within the same fiscal year can be cleaner. |
| Company deposits in January 2026 and withdraws in December 2026 | All the yield falls in fiscal year 2026 → return due April 2027. | Maximum fiscal exposure in a single fiscal year. |
| Company withdraws in November 2026 before the fiscal year ends | Closes the cycle within fiscal year 2026. If it converts PXRM to CLP before Dec 31 → capital gains realized in 2026. | Realized capital gain = declare in April 2027. |

**Gray area with the SII (from `GREYVALLEY_SII_FISCAL.md §4`):** the criteria on when PXRM yield is taxed is not yet settled. Conservative position: PXRM is not income until it is converted to CLP. Consult an accountant with crypto experience.

**Operational advice before consulting the accountant:** keep an exact record of:
- Deposit date + amount in USD
- Date of each harvest/withdrawal + amount of PXRM received
- PXRM price in CLP at the moment of each conversion

ICP has on-chain logs (ICRC-3) that generate that record automatically — it's the "free audit trail" that comes with the blockchain.

---

## ANDROID — APP PREVIEW ON A PHONE

For GreyValley in its current state (web app on ICP):
- The app is already accessible from **Android Chrome** by pointing to the frontend canister's URL.
- There's no APK — it's a Progressive Web App (PWA). The user can add it to their home screen from Chrome.

For the **native mobile Wallet (Phase 2 of the roadmap in `WALLET.md`):**

| Method | What it is | When to use it |
|---|---|---|
| **Google Play Internal Testing** | You upload the APK to Play Console and distribute it to up to 100 testers without publishing | When there's a native build ready to test with real users |
| **Firebase App Distribution** | Google's platform for distributing test APKs outside the Play Store | Broader testing before going to production |
| **Direct sideload (APK)** | The user installs the APK directly by downloading it (no Play Store) | Only for the dev team — not for users |
| **Expo Go** | A React Native app that lets you preview without compiling an APK | If the wallet is built in React Native — the fastest way to iterate |

**Recommendation for GreyValley:** until there's a native wallet, "Android preview" is simply opening the canister's URL in Android Chrome. It works — and the mobile design is specified in `WALLET.md §Mobile Wallet`.

---

## LEY 21.719 — PERSONAL DATA 2027 AND WHY ICP SOLVES IT BY DESIGN

### The problem being built for (and already solved on ICP)

**Ley N° 21.719, Personal Data Protection Law** — enacted December 2024, full effect 2027. Modeled on the European GDPR. The two rights that require new technical infrastructure:

- **Right of access / portability:** the user can ask "give me all my data" and the company has a legal deadline to respond with everything.
- **Right to erasure / deletion:** the user can ask "delete everything of mine" and the company must be able to do so verifiably.

**Why current systems can't respond quickly:**

A typical Chilean company has a user's data scattered across:

```
Main DB (PostgreSQL)            → name, RUT, email, history
Application logs                → IPs, actions, errors with user data
Backups (S3, tapes)             → complete DB snapshot at some date
Cache (Redis)                   → active sessions, preferences
Email marketing (HubSpot)       → history of emails sent
CRM (Salesforce)                → sales interactions
Analytics (GA, Mixpanel)        → on-site behavior
Payment system (Transbank)      → transaction history
Data warehouse (Redshift)       → historical reports
DB read replicas                → synced copies
Attachments (S3)                → documents uploaded by the user
```

When the user asks to be "deleted" — nobody has a map of where everything is. Finding it takes weeks of manual work. Deleting it completely, with no trace left in historical backups or logs, is nearly impossible to guarantee.

**What many companies were building to comply:** a **data catalog** that keeps an index of "for this user, their data exists in these N locations." When a request comes in, the system walks the index and coordinates the extraction or deletion. The problem: that catalog has to be built, kept up to date as the architecture changes, and historical backups always slip through.

---

### How ICP solves this by design — and where it has the same limit

On ICP, **all of a user's state is tied to their Principal ID**. There's no data scattered across 11 services — the canister is the single source of state.

> Note: the snippet below is **illustrative of the design pattern**, not code that
> already exists in `backend/main.mo` — `query_user_data`/`delete_user_data` are not
> implemented today.

```motoko
// Find all of a user's data:
query_user_data(principal: Principal) : async UserDataExport {
    {
        vaultPositions = Map.get(vaultPositions, principal);
        stakePositions = Map.get(stakePositions, principal);
        kycData        = Map.get(kycData, principal);
        harvestHistory = Map.get(harvestHistory, principal);
    }
}

// Delete all of a user's data:
update delete_user_data(principal: Principal) {
    requireController();
    Map.delete(vaultPositions, principal);
    Map.delete(stakePositions, principal);
    Map.delete(kycData, principal);
    Map.delete(harvestHistory, principal);
}
```

**One single function. One single transaction. No walking through 11 systems.** The "data catalog" is the canister's own design — all of a user's data is keyed by the same Principal.

---

### The limit it shares with any blockchain — and how to design around it

The **transaction log (ICRC-3)** is append-only by cryptographic nature. A recorded transaction cannot be deleted without breaking the integrity of the history. This is the same GDPR dilemma faced on Ethereum.

The solution is a matter of **upfront design**, not after-the-fact deletion:

| Data | Where it lives | Deletable under Ley 21.719? | Correct design |
|---|---|---|---|
| Real PII (name, RUT, email) | Mutable canister state (stable var) | ✅ Yes — `delete_user_data()` removes it | Store PII only in mutable state, never in logs |
| Vault/stake positions | Mutable canister state | ✅ Yes | Keyed by Principal — deletable in one call |
| Transaction history (ICRC-3) | Append-only log | ❌ Not deletable | Contains only the Principal (a cryptographic pseudonym), NOT the RUT — the log stays, the real identity doesn't |
| KYC data (Koywe side) | Koywe's systems | ✅ Koywe has its own obligation under the law | Koywe manages its own compliance; GreyValley manages its own |

**The design key:** if the Principal ID (a cryptographic address, not a name or RUT) is the only identifier in the logs, the log can remain intact without violating the law. The right to erasure applies to **PII linked to the real person**, not to the record that "someone made a transaction." If the Principal ↔ real-RUT link is deleted from mutable state, the log remains as anonymous data.

---

### Why this is a business argument for GreyValley

What those Chilean companies are building for 2027 — the data subject request system — GreyValley has by design from day 1. That is a real business argument for the corporate and institutional segment:

> *"If your company needs to comply with Ley 21.719 and handle users' financial data, an ICP architecture gives you native data subject requests. You don't need to build a separate data catalog, you don't need to coordinate deletions across 11 systems, you don't need to hire a data governance consultancy. The canister's design already has that answer built in: `query_user_data(principal)` and `delete_user_data(principal)`. That's compliance infrastructure included in the architecture, not bolted on afterward."*

**Specific positioning by segment:**

| Segment | Why Ley 21.719 matters to them | GreyValley's argument |
|---|---|---|
| Fintech / startup | They have user data and are building compliance from scratch | ICP as compliance-by-design architecture from the start — cheaper than retrofitting |
| SME with its own system | They have legacy systems and don't know where everything is stored | Migrating the financial layer to GreyValley/ICP resolves that data vector |
| Company with institutional clients | Their corporate clients will demand evidence of compliance | On-chain audit trail + verifiable deletion in one transaction |
| Exporting company | Data on international counterparties may also be subject to European GDPR | The same mechanism satisfies both GDPR and Ley 21.719 at once |

---

*GREYVALLEY_ICP_PITCH_INGENIERO.md | Created: 2026-08-08*
*Cross-reference: `GREYVALLEY_ICP_TECH.md` (technical depth on NNS/tECDSA), `GREYVALLEY_REGULATORY.md §9` (CMF argument), `GREYVALLEY_SII_FISCAL.md` (treasury and taxes), `WALLET.md` (mobile roadmap), `GREYVALLEY_REGULATORY.md §1` (UAF/CMF)*
