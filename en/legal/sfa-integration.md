# GreyValley — SFA-Sandbox Integration (CMF Open Finance Chile)
> Technical reference document for integration with sfasandbox.cl
> Generated: 2026-08-13 | Based on official sfasandbox.cl/docs.php documentation
> Regulation: Ley Fintech 21.521 · NCG 502 · NCG 514 · FAPI 2.0

---

## 1. Context and Important Distinction

**SFA-Sandbox is NOT the SII (Servicio de Impuestos Internos, Chile's tax authority) sandbox.** They are completely different systems:

| System | What it's for | Operator | Regulation |
|---------|----------|----------|-----------|
| **SFA-Sandbox** (`sfasandbox.cl`) | Banking open finance: TEF payments, account reading, PISP/AISP | SFA-Sandbox team (private, CMF-aligned) | NCG 502, NCG 514, FAPI 2.0 |
| **SII Sandbox** (`maullin2.sii.cl`) | Electronic invoicing (DTE), folios, credit notes | Servicio de Impuestos Internos (Chile's Internal Revenue Service) | SII Resolution, XML format |

GreyValley uses **SFA-Sandbox** for the fiat on/off-ramp flow (CLP ↔ ckUSDC). The SII is a separate matter for GreyValley SpA's accounting operations.

---

## 2. Authentication and Headers

Every API call requires two mandatory headers:

```http
x-sandbox-key: sfa_your_key_here
x-fapi-interaction-id: b1c67d3e-9081-42cb-a311-2df8a2f1c8e9
Content-Type: application/json
```

- **`x-sandbox-key`**: a key exclusive to the environment, obtained from `sfasandbox.cl/console.php` after logging in with LinkedIn or Dev Mode (`/auth.php?action=dev_login`).
- **`x-fapi-interaction-id`**: a unique UUID per request — serves as a forensic trace per FAPI 2.0 requirements. Must be different on each call.
- **Do NOT use** `Authorization: Bearer` as the header for authenticating the key — that applies only to the Bearer JWT obtained after the OIDC/CIBA consent flow.

---

## 3. Available Endpoints

Base URL: `https://sfasandbox.cl/api.php?action=<endpoint>`

| Endpoint | Method | Required auth | Description |
|----------|--------|---------------|-------------|
| `consents` | POST | x-sandbox-key | Creates a FAPI 2.0 consent intent (redirect flow) |
| `token` | POST | x-sandbox-key | Exchanges a temporary code for a Bearer JWT |
| `ciba_auth` | POST | x-sandbox-key | Starts a CIBA backchannel session (no redirect) |
| `ciba_token` | POST | x-sandbox-key | Polls CIBA status |
| `payments_pisp` ⚠ | POST | **x-sandbox-key + Bearer JWT** (corrected 2026-08-24 — see note below) | Initiates a TEF (PISP — payment from a bank account) — corrected 2026-08-22, see note below |
| `payment_status` | GET | x-sandbox-key | Queries a payment's status by ID |
| `channels_status` | GET | x-sandbox-key | NCG 514 operational availability (reacts to the chaos engine) |
| `balances` | GET | x-sandbox-key + Bearer JWT | User account balance (AISP) |
| `transactions` | GET | x-sandbox-key + Bearer JWT | User's 5-year history (AISP) |
| `branches` | GET | x-sandbox-key | Geolocated branches and ATMs |
| `products` | GET | none | Bank product catalog (open data) |

---

## 4. PISP Flow — CLP → ckUSDC On-Ramp (Direct Relevance for GreyValley)

This is the main flow for GreyValley's on-ramp: the user pays from their Chilean bank account and the protocol mints ckUSDC to their Principal.

> **⚠ Real correction 2026-08-22, also corrected in code on 2026-08-24:**
> this document (and `src/sfa_treasury/main.mo`) had the endpoint as
> `action=payments` — verified against sfasandbox.cl's REAL
> `openapi.json`, the correct name is **`action=payments_pisp`**. With
> the old name the real call was hitting a non-existent endpoint. Already
> fixed on both sides (doc and code) — the founder requested starting
> real SFA sandbox tests. Also: neither the technical manual nor the
> openapi.json publish example values
> (`creditor_account`/`creditor_rut`/`creditor_bank`) for the real
> payload — they have to be obtained from the web console ("Iniciación
> Pagos QR/PIX") or requested from `equipo@sfasandbox.cl`.
>
> **⚠ Second real correction, 2026-08-24 — the required auth was
> mis-documented.** The first real test (endpoint already fixed, real
> credentials loaded) returned `401 {"error":"unauthorized_client"}` for
> `payments_pisp`, even though the same key worked fine for
> `channels_status`. The Chaos Engine was ruled out (the founder disabled
> it, the error persisted identically). Verified directly against the
> real `openapi.json`: `payments_pisp` declares `security: bearerAuth`
> (JWT Bearer token) — `x-sandbox-key` alone is not enough for this
> specific action, unlike `channels_status`/`branches` where it is
> enough. The table above said "x-sandbox-key" only — corrected.
>
> **Correct real flow:** `consents` (create a consent intent)
> → user signature (redirect or CIBA backchannel, §6) → `token` (exchanges
> the code for the Bearer JWT) → only then `payments_pisp` with
> `Authorization: Bearer <jwt>` + `x-sandbox-key`. The current code
> (`sfa_treasury.registerPendingMint()`) still calls `payments_pisp`
> directly without that JWT — the full consent flow still needs to be
> implemented before that call. This remains a real next step, not
> blocking for what has already been tested (`channels_status` works
> end-to-end).

### Step 1 — Initiate a TEF payment

```http
POST https://sfasandbox.cl/api.php?action=payments_pisp
x-sandbox-key: sfa_your_key
x-fapi-interaction-id: <uuid>
Content-Type: application/json

{
  "amount": 15000,
  "debtor_account": "12345678",
  "creditor_account": "987654321",
  "creditor_name": "GreyValley SpA",
  "creditor_rut": "76.XXX.XXX-X",
  "creditor_bank": "Banco Santander Chile"
}
```

**Response (201 Created):**
```json
{
  "payment_req_id": "urn:sfa:payment:pay_req_a89bc...",
  "status": "AWAITING_AUTHORISATION",
  "consent_url": "https://sfasandbox.cl/auth.php?action=payment_auth&payment_req=...",
  "expires_in": 600
}
```

The user must be redirected to `consent_url` to sign with their bank PIN.

### Step 2 — Status polling

```http
GET https://sfasandbox.cl/api.php?action=payment_status&id=urn:sfa:payment:pay_req_...
x-sandbox-key: sfa_your_key
x-fapi-interaction-id: <uuid>
```

**Response when authorized:**
```json
{
  "payment_req_id": "urn:sfa:payment:pay_req_...",
  "status": "AUTHORIZED",
  "transaction_code": "TEF-D8A9F1B2",
  "amount": 15000,
  "updated_at": "2026-08-13T10:00:00Z"
}
```

### Step 3 — ckUSDC minting (only when AUTHORIZED)

**Critical rule:** the ICP canister should only mint ckUSDC to the user's Principal **after confirming `status: "AUTHORIZED"`**. Never mint based on the initial `AWAITING_AUTHORISATION`.

```
AWAITING_AUTHORISATION → polling every 3s → AUTHORIZED → mint ckUSDC to the Principal
                                         → REJECTED → no mint, return error to user
                                         → expired (600s) → no mint, restart the flow
```

---

## 5. Circuit Breaker — NCG 514 (Operational Availability)

Before starting any on-ramp, the canister must verify that the banking systems are operational:

```http
GET https://sfasandbox.cl/api.php?action=channels_status
x-sandbox-key: sfa_your_key
x-fapi-interaction-id: <uuid>
```

**Normal response:**
```json
{
  "status": "OPERATIONAL",
  "channels": {
    "apis": { "status": "OPERATIONAL", "availability_24h": 99.98 },
    "oauth": { "status": "OPERATIONAL", "availability_24h": 99.99 }
  }
}
```

**Canister logic:**
- If `status != "OPERATIONAL"` → reject the deposit with the message: "Chilean banking system temporarily unavailable. Try again in a few minutes."
- This endpoint reacts to the chaos engine in real time — if you activate 500 errors in the panel, this endpoint degrades.

---

## 6. CIBA Flow — Backchannel Authentication (No Redirect)

CIBA is the preferred flow for the GreyValley mobile app: the user approves from their banking app without leaving the GreyValley app.

> ⚠️ **Real correction 2026-09-29 (docs audit) — real blocker not previously documented here.**
> The CIBA login itself completes end-to-end against the sandbox (confirmed real), but the JWT
> Bearer it returns **always carries an empty `scope` array**, for both PISP and AISP flows —
> confirmed in code (`sfa_treasury/main.mo`, real comments about `payments_pisp` returning
> `403 invalid_consent_state with empty scopes in the JWT`). Without a valid scope in the JWT,
> `balances`/`transactions`/`payments_pisp` cannot be completed even though the CIBA login was
> successful. The founder is waiting for a response from the SFA-Sandbox team about this — it is
> not confirmed to be a bug on GreyValley's side yet, but it means **CIBA is not end-to-end
> operational today** for flows that depend on the JWT, even though the login itself works.

### Step 1 — Start a CIBA session

```http
POST https://sfasandbox.cl/api.php?action=ciba_auth
x-sandbox-key: sfa_your_key
x-fapi-interaction-id: <uuid>
Content-Type: application/json

{
  "login_hint": "12.345.678-9",
  "scope": "accounts",
  "binding_message": "SFA-8891"
}
```

**Response:**
```json
{
  "auth_req_id": "bcauth_7f2a8c3d9e01",
  "expires_in": 300,
  "interval": 5
}
```

### Step 2 — Polling

```http
POST https://sfasandbox.cl/api.php?action=ciba_token
Content-Type: application/json

{ "auth_req_id": "bcauth_7f2a8c3d9e01" }
```

**Possible responses:**
```json
{ "error": "authorization_pending" }   // user has not yet approved
{ "error": "expired_token" }           // timeout — restart the flow
{ "access_token": "eyJhbG...", "token_type": "Bearer", "expires_in": 3600 }  // approved
```

**Recommended timeout:** 5 minutes. If the user does not approve within that time, cancel and show a retry option.

---

## 7. AISP — Reading User Accounts

With the Bearer JWT obtained from the OIDC or CIBA flow, the user's bank accounts can be read. Relevant for the unified dashboard (bank balance + GreyValley balance on one screen).

### Balances

```http
GET https://sfasandbox.cl/api.php?action=balances
Authorization: Bearer eyJhbGciOiJQUzI1NiIs...
x-sandbox-key: sfa_your_key
x-fapi-interaction-id: <uuid>
```

**Response:**
```json
{
  "accountId": "ACC-992810",
  "currency": "CLP",
  "availableBalance": 8900340,
  "ledgerBalance": 8900340,
  "creditLimit": 1500000,
  "status": "ACTIVE"
}
```

### Transaction history (5 years — regulatory requirement)

```http
GET https://sfasandbox.cl/api.php?action=transactions
Authorization: Bearer eyJhbGciOiJQUzI1NiIs...
x-sandbox-key: sfa_your_key
x-fapi-interaction-id: <uuid>
```

---

## 8. Chaos Engine — Resilience Testing

The sandbox has a chaos engine, activatable from the panel, that simulates real banking core failures:

| Anomaly | Effect | How to test on GreyValley |
|----------|--------|----------------------|
| **Latency +3500ms** | All calls take an extra 3.5s | The canister's HTTPS Outcall must have a timeout ≥ 8s |
| **HTTP 500** | Endpoints return random errors | The canister must retry with exponential backoff (max 3 attempts) |
| **Infinite pending CIBA** | `authorization_pending` never resolves | 5-minute timeout in the canister's poller |
| **Rate limit 429** | Blocks excess requests | Implement a request queue, don't retry immediately |

**Recommended test protocol before production:**
1. Activate the chaos engine in "Latency +3.5s" mode
2. Run the full PISP flow and verify the canister waits correctly
3. Activate "HTTP 500" and verify the circuit breaker rejects the on-ramp cleanly
4. Verify that a deposit never remains permanently stuck in `AWAITING_AUTHORISATION`

---

## 9. Tunnel Proxy (Advanced)

For advanced teams: routing real traffic toward production banking APIs through the SFA chaos proxy.

```http
x-proxy-target: https://api.bancoreal.cl/v1/accounts
x-sandbox-key: sfa_my_key
```

This allows intercepting and sabotaging real traffic toward a bank. **Do not use in production** — for advanced integration testing only.

---

## 10. ICP Integration — HTTPS Outcalls from the Canister

In GreyValley's architecture, the ICP canister calls the SFA API directly with no intermediate Node.js backend. Each HTTPS Outcall is executed by the subnet's 28 nodes independently — ≥19/28 must agree on the response.

**Determinism problem:** SFA includes headers like `x-fapi-interaction-id` and timestamps that vary between nodes. The canister must use `transform` to strip these fields before consensus.

```motoko
// Transform: remove non-deterministic headers from the SFA response
public query func transformSFAResponse(raw : Http.TransformArgs) : async Http.HttpResponsePayload {
    {
        status  = raw.response.status;
        body    = raw.response.body;
        headers = [];  // remove all headers — only the body is deterministic
    }
};
```

**Request headers from the canister:**
```motoko
let headers = [
    { name = "x-sandbox-key";          value = sfaSandboxKey },
    { name = "x-fapi-interaction-id";  value = generateUUID() },
    { name = "Content-Type";           value = "application/json" },
];
```

**Note on `x-fapi-interaction-id`:** although it must be unique per request, in an HTTPS Outcall the 28 nodes send the same UUID (the canister generates it before the call). The SFA server accepts it just the same — the determinism problem is in the *response* headers, not the request headers.

---

## 11. GreyValley × SFA Integration Roadmap

| Phase | What gets integrated | Responsible | Status |
|------|---------------|-------------|--------|
| **Phase 1 (current)** | Koywe handles PISP — GreyValley just calls the Koywe API | Koywe (CMF-accredited) | ⏳ KYB pending |
| **Phase 2** | `channels_status` as a circuit breaker in the canister | GreyValley technical | ✅ Built (`sfa_treasury/main.mo:204-230`, audit correction 2026-08-24 — this doc said "not built" but the code already has it) |
| **Phase 2** | AISP: bank balance + GreyValley balance in a unified dashboard | GreyValley technical | ❌ Not built |
| **Phase 3** | GreyValley SpA registration as direct PISP (eliminates the Koywe fee) | Legal + CMF | ❌ Requires CMF registration |
| **Phase 3** | CIBA backchannel from the mobile app | GreyValley technical | ❌ Not built |

**Prerequisite for Phase 3:** GreyValley SpA must be a regulated entity (PSAV or PISP) and pass the SFA accreditation process with CMF. Requires: an active legal entity, implemented AML/KYC, a security audit, a guarantee deposit.

---

## 12. Immediate Action

| Priority | Action |
|-----------|--------|
| 🔴 **High** | Obtain an `x-sandbox-key` from `sfasandbox.cl/console.php` (LinkedIn login or Dev Mode) |
| 🔴 **High** | Test the full PISP flow (payments → polling → AUTHORIZED) with the chaos engine active |
| 🟡 **Medium** | Implement `channels_status` as a circuit breaker in the bridge canister |
| 🟡 **Medium** | Evaluate CIBA vs. OAuth2 redirect as the auth flow for the mobile on-ramp |
| 🟢 **Low** | Explore GreyValley SpA registration as AISP (less restrictive than PISP, first legal step) |

---

*GreyValley SFA Integration Reference | Updated: 2026-08-13 | Source: sfasandbox.cl/docs.php*
