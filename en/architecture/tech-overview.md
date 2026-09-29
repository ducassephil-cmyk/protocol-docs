# GreyValley — Technical Overview

> FIX 2026-09-14 (docs audit): this file was a one-line stub ("See
> engineer-pitch.md") while `README.md` promised real content here — full
> stack comparison, consensus mechanics, tECDSA, HTTPS Outcalls, Ley
> 21.719. Written for real below. `engineer-pitch.md` keeps its own scope
> (current code state, deployment, LatAm positioning) — this file is the
> conceptual/technical reference, `engineer-pitch.md` is the pitch/status
> doc.

## Stack comparison — why Internet Computer, not a typical chain

| | Typical L1 (Ethereum-style) | Internet Computer |
|---|---|---|
| Where code runs | EVM, gas-metered bytecode | WASM, canisters — full general-purpose compute |
| Storage | Expensive, external indexers usually needed | Native, a canister holds its own state directly |
| Serving a web frontend | Needs a centralized server/CDN | A canister can serve HTTP directly — no server in the loop |
| Talking to other chains | Needs a bridge (custodial or multisig) | Native signing via chain-key cryptography — no bridge contract |
| Calling external APIs | Not possible on-chain | HTTPS outcalls, with subnet consensus on the response |

GreyValley runs entirely as canisters on ICP — the frontend, the vaults,
the AMM, the CDP, the corridor logic. No traditional server anywhere in
the request path.

## Consensus mechanics (short version)

ICP is organized into **subnets** — independent groups of nodes, each
running a full replica of the canisters assigned to it. Every canister
call that changes state gets executed independently by every node in the
subnet; the subnet only finalizes a result once enough nodes agree on the
identical outcome. This is what makes chain-key signing and HTTPS
outcalls safe: no single node can forge a signature or a fetched value —
the whole subnet has to compute the same thing.

## tECDSA — threshold ECDSA signing

Chain-key cryptography lets a canister hold a **Bitcoin or Ethereum
address it directly controls**, without ever holding a full private key
anywhere. The signing key is split into shares distributed across the
subnet's nodes (threshold cryptography); a valid signature only comes out
when enough of those shares cooperate, and no single node — not even a
majority minus one — ever reconstructs the full key. This is the real
mechanism behind ck-tokens (ckBTC, ckETH, ckUSDC, etc.): the minting
canister genuinely controls funds on the source chain, with no custodial
bridge and no multisig committee.

## HTTPS Outcalls

A canister can make a real HTTPS request to any external API (a price
oracle, a bank's open-finance API, etc.) directly from its own code — no
off-chain relayer required. The catch, and the safety property: every
node in the subnet independently makes the same HTTPS call, and the
result is only accepted if enough nodes get byte-identical responses.
This is why outcalls work well for deterministic, cacheable endpoints
(a price feed at a given timestamp) and poorly for endpoints that return
different data per request.

## Ley 21.719 (Chilean data protection, effective 2026) — why this
architecture helps by design

Ley 21.719 tightens requirements around where personal data lives, who
can access it, and how breaches get handled. A canister-based backend has
two structural properties that help here, without needing extra
compliance tooling bolted on:

- **No central database to breach** — application state lives inside the
  canisters themselves, replicated across a subnet, not in a single
  server's SQL database that a single intrusion can dump wholesale.
- **Auditable state transitions** — every state change is a canister
  call, versioned and inspectable; there's no silent out-of-band write to
  a users table.

This does **not** mean GreyValley is automatically compliant — legal
review of Ley 21.719 obligations (consent, data-subject rights, breach
notification) is still a real, separate requirement, same caveat as the
rest of `legal/regulatory.md`. It means the underlying architecture makes
some of the harder infrastructure-security problems structurally
smaller, not that compliance is automatic.

---

*See `engineer-pitch.md` for current code/deployment state and LatAm
market positioning, and `corredor-mechanics.md` for how the payment
corridor itself works.*
