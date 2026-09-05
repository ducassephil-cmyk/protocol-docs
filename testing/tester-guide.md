# GreyValley — Guía real para testers de pre-lanzamiento

> Documento vivo para el grupo cerrado de prueba (círculo cercano de confianza,
> perfil técnico/"hackeo amistoso"). Objetivo: encontrar bugs reales con plata
> real, en montos chicos, antes de cualquier lanzamiento público. Esto NO es
> una demo — es producción real en mainnet ICP, con canisters reales y fondos
> reales.

---

## 1. Qué es esto, honestamente

GreyValley es un protocolo DeFi real en Internet Computer (vaults, staking,
AMM, CDP/vUSD, marketplace, corredor fiat). Está en una etapa de **pre-
auditoría interna** — el founder y el asistente de desarrollo vienen
encontrando y arreglando bugs reales de plata a un ritmo alto (varios por
semana en las últimas sesiones: escala de ratio del CDP, fondos huérfanos en
AMM, ledger de PXRM apuntando mal, timers cayendo en silencio por cycles
bajos). Eso no es una señal de alarma — es exactamente lo que se espera
encontrar en esta etapa. Pero significa que **todavía pueden aparecer bugs
nuevos de este calibre**, y por eso existe esta prueba cerrada antes de
cualquier apertura más amplia.

No hubo auditoría externa formal todavía. Todo lo verificado hasta hoy es
founder + asistente, con el método real: trazar cada función a su canister →
verificar el resultado on-chain → arreglar o decir la verdad → recién ahí
probar con plata real.

## 2. Qué necesitás para participar

- **Wallet real, con soporte confirmado por nivel:**
  - **Plug** — única wallet con soporte completo hoy. Usá esta si podés.
  - **Oisy** — custodia real de PXRM/ICP, podés tradear o solo mirar la app,
    pero **sin stake ni yield todavía**.
  - **Bitfinity** — solo lectura, no podés operar.
  - **NFID** — marcada "Próximamente", deshabilitada. Nunca conectó de
    verdad en producción pese a tener integración real construida —
    bug sin auditar, no lo intentes todavía.
- **Montos chicos, a propósito**: no metas nada que no puedas perder. Esto
  es un tope real del founder, no solo una sugerencia — la idea es que un
  bug real cueste centavos, no que arruine a nadie.
- Una wallet que puedas seguir de cerca vos mismo (revisar balances antes/
  después de cada acción, no confiar solo en lo que la UI te muestra).
- **Regalo de bienvenida**: cada tester recibe **50 PXRM** reales de arranque
  (~$12,6 al precio de hoy) para tener con qué probar sin poner plata propia
  primero. Llega directo a tu wallet conectada — avisanos tu principal para
  que te lo mandemos.
  - **Sugerencia (no obligatorio)**: probá el CDP/mint de vUSD con una
    parte — con ~15-20 PXRM de colateral podés mintear ~$2 de vUSD real
    (el CDP exige 200% de colateralización para PXRM, así que $2 de vUSD
    necesita ~$4 de colateral ≈ 15-20 PXRM al precio actual). El resto
    (~30-35 PXRM) quedá libre para stakearlo como prefieras — Flexible,
    o algún lock (90/180/365 días) si querés probar esa parte también.

## 3. Foco real de esta ronda — Swaps y Retiros

Priorizado por dónde ya encontramos bugs reales de plata este mes. Si tenés
tiempo para poco, empezá acá.

### Swaps

| # | Qué probar | Estado conocido |
|---|---|---|
| 1 | Swap grande en un pool con poca profundidad (varias cards del `/amm` dicen "Liquidez insuficiente") | **BUG REAL encontrado y corregido 2026-09-04**: el safeguard (~11% de impacto máximo) solo bloqueaba cuando el swap perjudicaba al usuario — un pool mal cotizado en la otra dirección (el trader recibe de MÁS a costa del pool) pasaba sin bloquearse. Confirmado en vivo contra `ckBTC_ckUSDC` (mal cotizada 8x, pagabas $0,09 y "recibías" $0,48). Ya corregido en `TradePage.tsx`/`PortalCard.tsx`/`AmmPoolStats.tsx` — probar de nuevo que efectivamente bloquea en AMBAS direcciones ahora. |
| 2 | Swap en pares con decimales distintos (ckETH = 18 decimales, ckUSDC/ckUSDT/ckEURC = 6, resto = 8) | Ya hubo bugs reales acá antes (fee mal calibrado, ckLINK 2026-09-03). Verificable solo probando de verdad — no hay forma de confirmarlo por revisión de código sola, hace falta ejecutar el swap real y comparar el monto recibido contra lo esperado. |
| 3 | Cap de 25 PXRM/tx en el swap fijo PXRM→ICP | **Confirmado**: 26 PXRM sí rechaza — pero el botón de la UI se desactiva antes de poder mandar ese monto, así que no se puede probar el límite exacto desde la interfaz normal. Si alguien quiere forzarlo (ej. vía consola/API directa al canister), avisen antes de intentarlo. |
| 4 | Pares bloqueados a propósito (`PXRM_ckUSDC`, `PXRM_ICP`) | Ya confirmado que rechazan. Si alguien encuentra una forma de saltarse el bloqueo, es un hallazgo crítico — reportarlo de inmediato. |
| 5 | Swaps chicos en pools recién integradas (`ckEURC_ckUSDC`, `ckLINK_ckUSDC`) | `ckEURC_ckUSDC` recién calibrada 2026-09-04 (~1% de diferencia contra el oráculo real). `ckLINK_ckUSDC` sigue casi vacía (reserveB=1 raw) — cualquier swap real ahí debería rechazar por el guard de impacto, no ejecutar a un precio roto. Reportar si ejecuta igual. |

### Retiros

| # | Qué probar | Estado conocido |
|---|---|---|
| 1 | Retiro temprano de vault con yield ya devengado (retención) | **Confirmado por el founder** — el safeguard real (~11%) funcionó bien en la prueba. |
| 2 | Retirar un vault que tiene vUSD pooleado en el AMM (auto-pooling activo) | Sin confirmar todavía — ligado al mismo patrón que causó fondos huérfanos antes en `add_liquidity`. Sigue siendo zona de riesgo real, probarlo a fondo. |
| 3 | Abrir un CDP real con PXRM (sugerido: ~15-20 PXRM de tu regalo → ~$2 de vUSD, ver §2) y después cerrarlo con colateral cerca del ratio mínimo, forzando zona de liquidación | Confirmado 2026-09-04: PXRM como colateral **no está bloqueado** (mint de vUSD habilitado globalmente) — exige 200% de ratio mínimo, el más alto de los 4 tipos (ICP 150%, ckBTC 130%, ckETH 150%, PXRM 200%). Sin probar todavía con PXRM real específicamente — este es el primer intento sugerido. |
| 4 | Unstake de PXRM con lock activo (90/180/365 días) | **Confirmado**: no deja retirar antes de tiempo, probado a los 90 días. Nota real: posiciones de antes del fix del ledger de PXRM (bug real, corregido 2 veces esta sesión) tuvieron que sacrificarse — si tenés posiciones viejas, esperá algo raro y avisá. |
| 5 | Retirar varias posiciones de golpe (multi-click / un solo click para varias) | **Probado**: 9 posiciones retiradas con un click, 8 salieron bien. 1 posición de Exaltite AMM (~$0.10) no cerró al primer click, necesitó 2 clicks. **Bug real menor a investigar** — huele a condición de carrera o a que el estado de la UI no refleja el resultado real después del primer intento. |
| 6 | Retiro de Exaltite (ckUSDC/ckUSDT/ckEURC) — comparar lo que devuelve contra lo que depositaste | **No es un bug, pero sorprende**: Exaltite es un pool de shares compartido entre TODAS las posiciones históricas de esta ronda de testing — hoy el NAV real es chico comparado con la suma nominal de lo depositado alguna vez (confirmado 2026-09-04, un retiro de "$0,50" devolvió solo "$0,054" real, matemáticamente correcto dado el NAV/shares actuales). Nueva query pública `backend.getExaltiteShareDebug()` expone `totalShares`/`navUsdcEquiv` en vivo para verificar esto vos mismo antes de asumir que es un bug. |

## 4. Cómo reportar un hallazgo

Para cada cosa rara que veas, aunque parezca chica:

1. **Qué hiciste** — paso a paso, exacto (monto, wallet, botón).
2. **Qué esperabas que pasara.**
3. **Qué pasó de verdad** — screenshot si se puede.
4. **Balance antes/después** de tu wallet, si tocaste plata real.
5. **¿Se repite?** — probá la misma acción 2 veces antes de reportar, para
   distinguir un bug real de un glitch de red puntual.

No hace falta que sepas si es "grave" o no — reportá todo, se prioriza
después.

## 5. Qué NO es foco de esta ronda (a propósito)

Para no dispersar la prueba, estas áreas quedan fuera por ahora — son
conocidas, incompletas, y no aportan señal nueva:

- Corredor fiat CLP↔USD/CLP↔EUR — el código real ya existe y está
  deployado (`api_gateway → settlement → bridge → liquidity_pool`,
  confirmado 2026-09-04), pero sigue apagado por feature flag
  (`bridge=false`) y sin partner (Koywe) registrado — no hay nada que
  probar ahí todavía, no por falta de código sino por falta de KYB
  comercial.
- Rampa fiat nativa (Koywe/partners — sin KYB confirmado, honestamente
  marcado "pendiente" en toda la UI)
- Marketplace de servicios (funciona, pero no es el foco de esta ronda —
  se puede probar informalmente)
- NFID (deshabilitada, ver arriba)

---

*Última actualización: 2026-09-04. Este documento se actualiza a medida que
se confirman o descartan los hallazgos de arriba — no es un documento
final.*

---
---

# GreyValley — Real pre-launch tester guide (English)

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

## 2. What you need to participate

- **A real wallet, with confirmed support by tier:**
  - **Plug** — the only wallet with full support today. Use this one if you can.
  - **Oisy** — real custody of PXRM/ICP, you can trade or just browse the app,
    but **no stake or yield yet**.
  - **Bitfinity** — read-only, you can't operate.
  - **NFID** — marked "Coming soon", disabled. Never actually connected in
    production despite having real integration built — unaudited bug,
    don't try it yet.
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
- NFID (disabled, see above)

---

*Last updated: 2026-09-04. This document gets updated as the findings above
get confirmed or ruled out — it's not a final document.*
