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
- **Regalo de bienvenida**: cada tester recibe **40 PXRM** reales de arranque
  para tener con qué probar sin poner plata propia primero. Llega directo a
  tu wallet conectada — avisanos tu principal para que te lo mandemos.

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
| 3 | Cerrar CDP con colateral cerca del ratio mínimo, forzando zona de liquidación | Sin probar en esta ronda todavía. Confirmado 2026-09-04: PXRM como colateral **no está bloqueado** (mint de vUSD habilitado globalmente) — exige 200% de ratio mínimo, el más alto de los 4 tipos (ICP 150%, ckBTC 130%, ckETH 150%, PXRM 200%). Sin probar todavía con PXRM real específicamente. |
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
