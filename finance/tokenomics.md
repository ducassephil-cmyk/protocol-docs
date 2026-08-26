# Vaelix Protocol — Tokenomics & Features
> Versión: 2026-07-01 | APR Model V3 + Supply formal + vUSD Institutional Track + Guild tiers

---

## 1. Tokens del Ecosistema

### PXRM (Puranium) — Token Principal
| Parámetro | Valor |
|-----------|-------|
| Canister ledger | `q7nmw-diaaa-aaaah-quy4a-cai` (MAINNET) |
| Estándar | ICRC-1 + ICRC-2 |
| Decimales | 8 (1 PXRM = 100_000_000 e8s) |
| Nombre canister | Puranium / PXRM (símbolo y nombre correctos desde el init) |
| Modelo | **Deflacionario** — se quema en swap PXRM→ICP + enterprise boost burn |
| **Supply total** | **5.000.000 PXRM** (fijo, nunca aumenta) — real, verificado on-chain |
| Uso | Yield de vaults (boost), minipack bienvenida, swap, staking rewards, Guild rewards |

> Nota histórica: el ledger original (`5zqoe-hqaaa-aaaaj-qrupa-cai`) quedó comprometido
> (minting key perdida, ~20M supply fantasma) y fue reemplazado el 2026-08-09 por el ledger
> de arriba. Detalle completo en `VAELIX_MASTER_STATE.md` §29.

**Distribución supply — Vesting Schedule (2026-08-01):**

Tokens bloqueados — se liberan linealmente después del cliff. Supply total: 5.000.000 PXRM.

| Grupo | Tokens (PXRM) | Cliff | Vesting | TGE Unlock |
|---|---|---|---|---|
| Protocol Treasury | 2.000.000 | — | Governance-controlled | 5% |
| Equipo / Fundadores | 1.000.000 | 12 meses | 36 meses lineal | 0% |
| Ecosistema / Guilds | 1.000.000 | — | 36 meses (emission) | 10% |
| Staking Rewards | 500.000 | — | Por epoch / APR | 0% |
| DAO Governance | 500.000 | — | Voto de comunidad | 0% |

> Nota sobre Treasury: "Governance-controlled" es el mecanismo general — dentro de eso, la porción destinada al PXRM Base APR (+ Guild Multiplier para Track B) de vaults sigue el calendario de la Sunset Clause semestral (`VAELIX_APR_MODEL.md` §10: Año 1 boost 100%, Año 2 baja a 50% si el yield real supera 5% y el ODL supera $500K/mes, Año 3 puede llegar a 0%). El 5% TGE Unlock es aparte de ese calendario — liquidez inicial disponible desde el día 1. **Los 2.000.000 PXRM no son un gasto garantizado** — es un techo que se contrae si el protocolo genera suficiente fee real antes de tiempo; con un sunset clause activado temprano, gran parte de ese 40% del supply queda sin gastarse.
> Nota sobre Staking Rewards (500K PXRM): es un pool de supply separado del 35% de fee split que reciben los stakers en ICP/ckUSDC/ckBTC — este bucket se libera "por epoch/APR" como refuerzo adicional en PXRM, no reemplaza al fee split real.
>
> ⚠️ **Huérfano detectado (auditoría 2026-08-07):** este bucket no tiene
> ninguna implementación en código — `grep` en `staking/main.mo` no encuentra
> ninguna referencia a un pool separado de 500K PXRM ni a una fórmula de
> liberación. El único mecanismo de pago real en `staking/main.mo` es
> `distribute_fees()` (fees reales del bridge, ponderados por stake × lock
> multiplier) — que además está dormido hoy (ver nota en `VAELIX_APR_MODEL.md`
> §2, Capa 3). Antes de implementar este bucket falta decidir la fórmula de
> liberación (candidatas discutidas: proporcional al TVL general por epoch,
> vs. tasa fija de emisión proporcional al PXRM ya stakeado) — pendiente de
> definición del founder, no construir sin esa decisión.

**Precio de lanzamiento (v2, 2026-07-31):**
- Paridad fija: **1 PXRM = ICP / 10** — mecanismo real en `swapPXRMtoICP` (backend/main.mo) y espejado en el oráculo (`oracle.getPxrmPegRatio()`, ajustable sin redeploy vía `setPxrmPegRatio`, admin-only). No hay AMM todavía, así que este swap fijo es el único precio real.
- Ejemplo con ICP a precio real de hoy (~$2.07): 1 PXRM ≈ **$0.207** → FDV de lanzamiento ≈ **$1.03M** (5.000.000 PXRM × $0.207)
- El precio de PXRM se mueve 1:1 con el precio real de ICP (vía oráculo XRC) — no es un número fijo en USD, escala con el mercado.

**FDV objetivo (adopción, sin cambios):**
- Adopción LatAm: $100/PXRM → $500M FDV
- Adopción bancaria: $500/PXRM → $2.5B FDV

> Corrección 2026-07-31: la versión anterior de esta sección decía "1 PXRM = 1 ICP (~$12) → $60M FDV" — paridad y precio de ICP desactualizados. La paridad real cambió a ICP/10 (antes 1:1) y el precio de ICP hoy es ~$2.07, no ~$12 — el FDV de lanzamiento real es ~60x menor que lo documentado antes.

---

## 1b. Genesis Round — Cofundadores Pre-TGE (2026-08-03)

El Genesis Round es el programa de captación de TVL antes del TGE. Los participantes son **cofundadores externos** — actores que depositan capital real (ckUSDC) en los vaults y a cambio reciben una asignación de PXRM con vesting. **No son el equipo Vaelix** (ese es el bucket Team/Founders).

### ¿De dónde sale el PXRM para el Genesis Round?

Del bucket **Ecosistema / Guilds (1.000.000 PXRM)** — esa es exactamente su función: early adopters y co-constructores del protocolo. El bucket Team/Founders (1M PXRM) es exclusivo del equipo interno (12 meses cliff) y no se toca para externos.

- **Reserva Genesis Round:** ~200–350K PXRM (20–35% del bucket Ecosistema)
- **Resta para Guilds / ecosystem post-TGE:** ~650–800K PXRM

### Tiers de cofundadores

| Tier | Cantidad ideal | TVL mínimo c/u | PXRM asignado c/u | Vesting | Beneficios adicionales |
|------|---------------|----------------|-------------------|---------|------------------------|
| 🔵 **Ángel** | 3–4 | $10K–$25K ckUSDC | 15K–25K PXRM | 6m post-TGE lineal | Early access, Corredor Guild priority |
| 🟣 **Semilla** | 2–3 | $25K–$75K ckUSDC | 40K–70K PXRM | 9m post-TGE lineal | Corredor Guild activo, co-branding |
| 🩷 **Estratégico** | 1–2 | $75K–$200K ckUSDC | 80K–150K PXRM | 12m post-TGE lineal | Advisory seat, Protocol Treasury voting, API dedicada |

> **Regla invariante:** ningún cofundador Genesis tiene unlock en TGE — todo el PXRM asignado tiene vesting post-TGE. Esto elimina el sell pressure en el día de lanzamiento.

### Escenarios de captación

| Escenario | Composición | TVL Genesis | PXRM asignado | % supply | TVL/FDV |
|-----------|------------|-------------|---------------|----------|---------|
| **Mínimo viable** | 2 Ángel + 1 Semilla | ~$70K | ~70K PXRM | 1.4% | 6.8% |
| **Lanzamiento creíble** ✅ | 3 Ángel + 2 Semilla + 1 Estratégico | ~$295K | ~310K PXRM | 6.2% | 28.5% |
| **Escala rápida** | 4 Ángel + 2 Semilla + 1 Estratégico | ~$320K | ~345K PXRM | 6.9% | ~31% |

**Escenario recomendado: Lanzamiento creíble.** Activa el ODL completo (Vault Exaltite ≥$50K ckUSDC), TVL/FDV = 28.5% (excelente para DeFi early stage), PXRM genesis ≤ 8% supply.

> "Escala rápida" está deliberadamente cerca del techo de las invariantes de abajo (7 cofundadores de 8 máximo, 345K PXRM de 350K máximo) — no hay margen para ir más agresivo sin romper el límite de concentración/coordinación. Si en la práctica aparece demanda real por más de 8 cofundadores o más de 350K PXRM, eso requiere revisar las invariantes mismas (decisión del founder), no forzar un cuarto escenario que las incumpla.

### Criterios de TGE sano

1. **TVL día 1 ≥ $105K** — mínimo para ODL capacity real (Vault Exaltite necesita $50K ckUSDC). Ver `VAELIX_ODL_MECHANICS.md` §5.
2. **PXRM Genesis ≤ 8% del supply** — por encima empieza a presionar precio post-vesting.
3. **Float TGE: ~200K PXRM circulantes** (4% supply) — solo TGE unlock Ecosistema (100K) + Treasury (100K). Market cap inicial ~$41K vs TVL $295K → ratio 7x.
4. **Vesting obligatorio para todos los Genesis** — sin TGE unlock, sin cliff dump.
5. **Número de cofundadores: 5–8** — menos es concentración riesgosa; más de 8 hace la coordinación pre-TGE impracticable.

### El cofundador Estratégico: el pivot del ODL

Un solo actor con $100K+ ckUSDC activa el Corredor ODL completo y da credibilidad inmediata al protocolo. El ROI para él: ahorro de ~2.5% en fees SWIFT sobre ese capital (~$2.500/mes en uso activo) + exposición a PXRM con upside. Perfil ideal: importador/exportador chileno que ya mueve esa cifra mensual en pagos internacionales.

### Invariantes del Genesis Round

```
PXRM Genesis ≤ 350K  →  Ecosistema bucket queda ≥ 650K para post-TGE
TVL Genesis ≥ $105K  →  ODL viable en día 1
Vesting mínimo: 6m   →  Cero dump en TGE
Número: 5–8          →  Distribución + coordinación viable
```

### Genesis Pool Sizing — Liquidez ICPSwap y protección de precio

El pool PXRM/ICP en ICPSwap es la infraestructura de precio del token. Su profundidad determina cuánta presión vendedora puede absorber el mercado antes de que el precio caiga.

**Origen de cada componente del pool:**

| Componente | De dónde viene | Capital requerido |
|---|---|---|
| PXRM del pool | Treasury (bucket Protocol Treasury) | $0 adicional — ya existe |
| ICP del pool | Founder (personal) + Genesis round (ICP de cofundadores) | Capital real |

**El PXRM del treasury nunca cuesta capital adicional.** El ICP sí — tiene que salir de algún bolsillo real. Por eso el modelo correcto es mixto: el founder siembra el mínimo viable, el genesis round lo profundiza.

**Modelo de pool recomendado:**

```
Fase 0 — Pre-genesis (founder solo):
  50.000 PXRM (treasury) + 2.500–5.000 ICP (founder personal)
  Valor pool: ~$20K–$40K | Ratio pool/emisión mensual: 8–16×

Fase 1 — Post-genesis (founder + cofundadores):
  100.000 PXRM (treasury) + 10.000 ICP (combinado)
  Valor pool: ~$80K | Ratio pool/emisión mensual: 33×  ← objetivo estable
```

**Por qué no poner $80K de ICP propio:**
El founder como LP único asume impermanent loss si PXRM y ICP divergen. Si PXRM cae 50% respecto a ICP, la posición LP vale ~30% menos que tener ICP puro. Ese capital es mejor como reserva operacional o como depósito en Vault Exaltium (gana yield y señaliza confianza al mercado).

**Narrativa para el genesis round:**
*"Tu aporte no solo da PXRM con upside — también funda la liquidez del mercado que protege el valor de ese PXRM."* Los cofundadores tienen incentivo directo en que el pool sea profundo porque eso estabiliza el precio de su propio vesting.

**Precio de lanzamiento del pool:**
1 PXRM = 0.1 ICP al sembrar. **No es un peg** — es el ratio inicial de siembra. Desde el momento de lanzar el pool, el AMM de ICPSwap determina el precio libre según oferta y demanda. El mecanismo `swapPXRMtoICP` interno del protocolo mantiene el peg 1/10 ICP solo para ese swap específico (con cap 2%/día) — no determina el precio de mercado general.

**Proyección de precio sin protección vs con pool profundo:**

| Mes | Pool $20K (solo founder) | Pool $80K (founder + genesis) |
|---|---|---|
| 0 | 0.100 ICP | 0.100 ICP |
| 1 | 0.086 ICP | 0.096 ICP |
| 3 | 0.072 ICP | 0.088 ICP |
| 6 | 0.059 ICP | 0.078 ICP |

*Supuesto: $75K TVL avg, 39% APR mixto, 60% de yield recipients venden. Sin catalizadores externos.*

**Mecanismo de protección adicional — sunset clause activo:**
Si el precio cae a 0.080 ICP → reducir bps de Flexible 20% → menos PXRM emitido → menos presión vendedora. Este ajuste es manual (founder) hasta que governance esté activo. Documentar cada ajuste on-chain como propuesta de governance para el historial CMF.

---

### LUNX (Luminox) — Token Multichain Externo de Vaelix

**Diseño dual-token canónico:** PXRM es el token interno del protocolo (ICP, staking, epoch tiers). LUNX es la representación externa y multichain del valor de Vaelix — el token de mercado real.

| Parámetro | V1 Beta (hoy) | V2 (roadmap) |
|-----------|--------------|--------------|
| Tipo | Crédito local `lunxCredits` en canister | ERC-20 en Ethereum / SPL en Solana |
| Transferible | No (soulbound) | Sí — tradeable en Uniswap, CEXes |
| Acumulación | 1:1 con PXRM yield al hacer **harvest** | Igual + bridgeable a Ethereum |
| Storage | `Map<Principal, Nat>` en main.mo | Smart contract ERC-20 en Ethereum |
| Bridge | ❌ No activo | ✅ Via Chain Fusion (threshold ECDSA) |
| Precio | Interno (espejo de PXRM) | Descubrimiento de precio en mercado Ethereum |
| Supply | Flotante (espejo de PXRM bridgeado) | Sin cap — mint/burn según bridge |

**Por qué LUNX y no solo PXRM para todo:**
- PXRM stakeado en tiers T1-T5 **no debe salir del protocolo** — eso es lo que genera la retención de TVL
- LUNX permite al usuario tener liquidez y precio de mercado externo **sin romper el staking**
- Ticker "LUNX / Luminox" tiene más resonancia en mercados externos que "PXRM / Puranium"

**Uso V1 (hoy):** historial de participación, governance vote, boost de APR (+1% por cada 100K LUNX)  
**Uso V2:** bridge a Ethereum, liquidez Uniswap V3, listing CEX, colateral en protocolos EVM

**Ver arquitectura técnica completa:** `VAELIX_INTEGRATIONS.md` §11

### Otros tokens del ecosistema
| Token | Rol |
|-------|-----|
| **vUSD** | Stablecoin sintética sobrecolateralizada. Sale del CDP. **Live desde 2026-08-02** (`ENABLE_VUSD_MINT = true`, `vusd_ledger` deployado). |
| **sCLP** | Peso chileno sintético 1:1. Acuñado por Chanfusion al recibir CLP bancario. Requiere aprobación CMF. |

---

## 2. Vault APR — Modelo V3 con Epoch Tiers

### Anatomía del APR (3 capas siempre visibles en UI)

```
APR TOTAL = Base Real Yield + (PXRM Base APR × Guild Multiplier) + Epoch Tier Bonus
```

**Capa 1 — Base Real Yield:** viene del trabajo real del protocolo (NNS staking, ODL fees, AMM). Token de pago varía por vault.
**Capa 2 — PXRM Base APR (Treasury Incentive):** APR base en PXRM fondeado por el treasury (2M PXRM). No es un "boost sobre" otra base — ES el APR base de PXRM. Revisión semestral (Sunset Clause). Multiplicado por Guild Multiplier para Track B: ×1.3 Institucional, ×2.0 Apex. Fuente: mismo treasury bucket para PXRM Base APR y Guild Multiplier extra.
**Capa 3 — Epoch Tier Bonus (T1–T5):** recompensa por permanencia continua. Financiado por el Epoch Retention Pool (10% del fee split) — distinto del treasury. Ver §3.

### Vault Exaltium (ICP)
- **Capital al retirar:** ICP exacto depositado
- **Base Real Yield:** ~3.5%/año en ICP (via NNS neuron staking / WaterNeuron)
- **PXRM Base APR:** ~4.5%–10% USD en PXRM desde treasury (según lock period)
- **Guild Multiplier:** ×1.3 Institucional / ×2.0 Apex sobre el PXRM Base APR
- **+ Epoch Tier Bonus según §3**

### Vault Crypto (ckBTC, ckETH)
- **Capital al retirar:** ckBTC/ckETH exacto depositado
- **Base Real Yield:** ~0% en beta (AMM fees activables en V2)
- **PXRM Base APR:** acumulación PXRM — principal attractor de este vault
- **Guild Multiplier:** ×1.3 Institucional / ×2.0 Apex sobre el PXRM Base APR
- **+ Epoch Tier Bonus**
- ⚠️ USD APR mínimo a precios actuales (BTC>>PXRM en precio). Vault de acumulación PXRM, no de rendimiento USD.

### Vault Exaltite (ckUSDC, ckUSDT)
- **Capital al retirar:** ckUSDC exacto depositado
- **Base Real Yield:** 0%→2%/año variable en ckUSDC — **solo existe con volumen ODL**. En bootstrap = "--".
- **PXRM Base APR:** ~8%–17% USD en PXRM desde treasury (el attractor principal en bootstrap)
- **Guild Multiplier:** ×1.3 Institucional / ×2.0 Apex sobre el PXRM Base APR
- **+ Epoch Tier Bonus**
- El ckUSDC depositado actúa como inventario ODL bridge (ver VAELIX_ODL_MECHANICS.md)

---

## 3. Epoch Tiers — Sistema de Holding

No hay lock forzado. El capital siempre es retirable. El tier premia la permanencia.

| Tier | Nombre | Días desde depósito | Tipo de yield | Bonus APR |
|------|--------|---------------------|--------------|-----------|
| Gracia | — | 0–14d | sin tier | 0% |
| T1 | Navegante | 15–44d | streaming live ⟳ | +1% |
| T2 | Explorador | 45–89d | epoch 30d | +2% |
| T3 | Vórtice | 90–179d | epoch 30d | +4% |
| T4 | Singularidad | 180–464d | epoch 30d | +6% |
| T5 | El Heraldo | 465d+ | epoch 30d | +9% |

> V3 (2026-07-17): se agregó el período de gracia (0-14d) para eliminar free-riders, T4 se extendió ~100d y T5 pasó de 365d a 465d — el año completo ya no alcanza para T5. Canónico en `VAELIX_APR_MODEL.md` §4.1, implementado en `epoch_pool/main.mo`.

**Regla del 5%:**
- Retiro ≤5% del capital actual (incluyendo reinversiones) → tier se preserva
- Retiro >5% del capital actual → tier se resetea a T1
- El retiro NUNCA se bloquea — procede siempre, solo el tier cambia

**T1 Streaming:** El 1% de T1 corre en tiempo real, visible en el frontend. No hay que esperar un epoch para cobrarlo.

**T2-T5 Epoch:** Los bonus se acumulan y están disponibles para harvest, pero el mejor momento para hacerlo es en las ventanas de epoch (cada 30d) para maximizar el Epoch Pool (10% del fee pool distribuido a T2+ cada epoch).

**Harvest:** No resetea el tier ni el depositTimeNanos. Se puede hacer en cualquier momento.

---

## 4. Fee Split — Distribución de Fees del Protocolo (V1 — activo en `fee_splitter/main.mo`)

| Destino | % | Descripción |
|---------|---|-------------|
| PXRM Stakers | 35% | Fee pool en activos duros (ICP/ckUSDC/ckBTC) — no en PXRM |
| LP AMM providers | 25% | Proporcional a liquidez aportada |
| Treasury | 23% | Fondea PXRM Base APR + Guild Multiplier |
| Epoch Retention Pool | 10% | Epoch Tier Bonus T1–T5 — distribuido cada 30 días |
| Volume Guilds | 7% | >$25K ODL/mes en PXRM. Acumula si nadie activo. |
| ReFi Chile | 0% | Reservado — desactivado V1, activar V1.5 con voto T3+ |
| **Total** | **100%** | |

> **Propuesta no implementada (V1.5, requiere decisión + cambio de código):** reasignar 5 puntos
> del split (bajar Treasury a 18%) a un "vUSD Institutional Pool" dedicado para reforzar el
> track institucional de §5b. Es una idea sobre la mesa, no algo que el `fee_splitter`
> deployado haga hoy — el contrato real sigue en 35/25/23/10/7/0. Progresión completa V1.5/V2
> por volumen ODL: ver `VAELIX_APR_MODEL.md` §6.

---

## 5b. vUSD Institutional Track (NUEVO — 2026-07-01)

> ⚠️ **Nota de riesgo regulatorio (2026-08-07):** "No requiere CMF" abajo es una
> afirmación de producto, no una conclusión legal — este track (depósito colectivo,
> yield % explícito, dirigido a "inversores/empresas/bancos") tiene características
> de valor mobiliario que todavía no se analizaron formalmente. Distinto del vUSD
> minteado vía CDP para pagar en el Marketplace (individual, sin pooling), que sí
> está bien encuadrado como medio de pago. Ver `VAELIX_REGULATORY_PLANB.md` §6 antes
> de escalar este track o de presentarlo a inversores institucionales reales.

Track de rendimiento estable para inversores clásicos, empresas y bancos. **No requiere CMF** *(afirmación pendiente de validar — ver nota arriba)*.

| Parámetro | Valor |
|-----------|-------|
| APR | 4–6% anual estable en USD — **objetivo de diseño, no real hoy** (ver fuente base) |
| Token de yield | vUSD (stablecoin sintética 1:1 USD) |
| Fuente base | ⚠️ **0% real hoy** — el diseño original apuntaba a "2% ODL fees (Vault Exaltite base real)", pero Exaltite tiene 0% de yield base real (el hook externo se sacó 2026-08-07, nunca se reemplazó) y el corredor ODL no tiene volumen real todavía |
| Fuente suplementaria | Boost adicional en PXRM vía Guild Multiplier (Treasury, §2) — existe como clasificación en `staking-vault` desde 2026-08-13, pero sin wiring a ningún payout real todavía |
| Colateral | ckUSDC del usuario (CDP interno, ratio 110%) |
| Al salir | vUSD quemado, ckUSDC devuelto íntegro |
| Activación | ✅ Mint live desde 2026-08-02 (`ENABLE_VUSD_MINT = true`) — pero vUSD no tiene ningún uso real más allá de cerrar el propio CDP: no es enviable, no es swapeable, el Marketplace es solo UI sin canister (ver `VAELIX_MASTER_STATE.md` §32) |

**Acceso por Guild tier — corregido 2026-08-14, permissionless en los tres:**

| Guild | Requisito | Boost vUSD | Cap |
|-------|-----------|-----------|-----|
| Explorador | Solo wallet ICP, $5K combinados | +1.5% APR objetivo — sin fuente real hoy | $50K |
| Institucional | Solo wallet ICP, $15K combinados | +2.5% APR objetivo — sin fuente real hoy | ilimitado |
| Apex | Solo wallet ICP, $50K combinados | +3.5% APR objetivo — sin fuente real hoy | ilimitado |

---

## 5c. Guild Program (NUEVO — 2026-07-01, corregido 2026-08-14)

Tres tiers de Track A (Volume Guild). **Permissionless en los tres — nunca requiere KYC
ni KYB.** El tier se calcula solo, en vivo, desde tu posición real en los vaults
(`useGuildTier.ts`, mostrado en la Wallet) — sin registro, sin formulario. (Track B,
el programa de operadores API, es el que sí requiere KYB en todos sus tiers — ver
`docs/CORREDOR_GUILD_OPERATOR_GUIDE.md`.)

### Guild Explorador (individual)
- Wallet ICP + depósito combinado ≥ $5.000 USD (mín. 50% en ckUSDC/Exaltite)
- Sin empresa, sin KYC

### Guild Institucional
- Wallet ICP + depósito combinado ≥ $15.000 USD
- 7% fee pool en PXRM (VolumeGuilds bucket) — todavía sin destino ni distribución real

### Guild Apex
- Wallet ICP + depósito combinado ≥ $50.000 USD
- Mismos beneficios que Institucional, sin requisitos adicionales de identidad

### Tabla de fees del protocolo
| Operación | Fee |
|-----------|-----|
| ODL bridge | 0.5% de cada transacción |
| AMM swap volátil | 0.3% |
| AMM swap estable | 0.1% |
| Ramp fiat CLP (sobre fee Koywe ~1%) | +0.2% |
| CDP préstamo | Sin fee de interés — no existe ese mecanismo en `cdp/main.mo` |
| Liquidación CDP | 13% del colateral (10% liquidador / 3% treasury / 87% al dueño) — real, confirmado en código |
| Swap PXRM→ICP | 0.3% (cap 25 PXRM) |

---

## 5. Staking PXRM

### Modelo híbrido: Lock multiplier + Epoch Tier bonus

A diferencia de los vaults (que usan solo Epoch Tiers sin lock), el staking PXRM mantiene **lock forzado** porque:
- PXRM stakers participan en gobernanza → el lock crea alineación real
- El lock reduce supply circulante de PXRM → estabiliza el token
- Análogo a NNS neurons: dissolve delay = compromiso

El APR de staking proviene del **35% del fee pool del protocolo** (no del treasury). Pagado en activos duros: ICP, ckUSDC, ckBTC proporcional al volumen de cada fee.

### Lock multiplier (base)

| Lock | Multiplicador | Ejemplo con APR base 8% |
|------|--------------|------------------------|
| Flexible | ×1.0 | 8% |
| 3 meses | ×1.3 | 10.4% |
| 6 meses | ×1.6 | 12.8% |
| 12 meses | ×2.0 | 16% |

### Epoch Tier bonus (encima del lock multiplier)

El tiempo continuo en staking (sin retirar >5%) agrega un bonus sobre el APR ya multiplicado:

| Tier | Nombre | Días en staking | Bonus |
|------|--------|-----------------|-------|
| T1 | Navegante | 0–30d | +0% |
| T2 | Explorador | 30–90d | +5% |
| T3 | Vórtice | 90–180d | +10% |
| T4 | Singularidad | 180–365d | +15% |
| T5 | El Heraldo | 365d+ | +20% |

**Regla 5%:** Retirar >5% de la posición total (capital + rewards acumulados) resetea el Epoch Tier a T1. El lock period sigue corriendo independientemente — el lock tiene su propia penalización de salida anticipada.

### Ejemplo combinado (Lock 6 meses · T3 Vórtice)

```
APR base fee pool:    8%
× Lock 6 meses:       ×1.6  → 12.8%
+ T3 Epoch bonus:     +10%  → 22.8% efectivo
Pagado en:            ICP + ckUSDC + ckBTC (proporcional a fees)
LUNX acreditado:      1:1 con rewards al hacer claimStakeRewards
```

### APR según volumen de protocolo

El APR base (8% en el ejemplo) es referencial — depende del volumen real:

| ODL volumen mensual | Fee pool mensual (35%) | APR PXRM staking est. |
|--------------------|----------------------|----------------------|
| $100K | ~$175 | ~2–4% |
| $1M | ~$1,750 | ~8–12% |
| $10M | ~$17,500 | ~20–30% |

Los multiplicadores de lock y epoch aplican sobre el APR real, no sobre el estimado.

---

## 6. Minipack (Programa de Bienvenida)

- **Monto:** 10 PXRM (`1_000_000_000` e8s) al primer depósito de cualquier vault
- **Condición:** Solo al primer depósito histórico de cada wallet
- **Ejecución:** Best-effort — si falla, encola con max 10 retries
- **Timer:** Procesa hasta 5 pendientes por tick (cada 300s)
- **Regla crítica:** NUNCA bloquear el depósito por el minipack

---

## 7. Swap PXRM → ICP

| Parámetro | Valor |
|-----------|-------|
| Cap | 25 PXRM por transacción |
| Fee del protocolo | 0.3% |
| Tasa | 1 PXRM = ICP / 10 (corrección 2026-07-31 — ver §1) |
| Mecanismo PXRM | Burn via `icrc2_transfer_from` con memo `"burn"` |
| Mecanismo ICP | Sale del treasury |
| Rollback | Si ICP transfer falla → PXRM devuelto al usuario |

---

## 8. CDP (Collateralized Debt Position)

Ratios de colateralización requeridos:
| Activo | Ratio mínimo | Liquidation fee |
|--------|-------------|----------------|
| ckBTC | 130% | 13% |
| ckETH | 150% | 13% |
| ICP | 175% | 13% |
| ckUSDC | 110% | 13% |

**Interés (stability fee), deployado 2026-08-16:** 0.3% mensual sobre el vUSD acuñado,
lineal (no compuesto, mismo criterio "simple interest" que el resto del protocolo —
PXRM Base APR, yield de vault.mo). Se computa on-the-fly desde `createdAt` (no hay
timer, no hay campo nuevo en el CDP — `accruedInterestE2s()` en `cdp/main.mo`) y se
cobra al `close_cdp()`: el owner debe quemar `vusdMinted + interés acumulado`, no solo
el principal. El interés real va a `fee_splitter` (mint del mismo monto quemado de más
→ burn+mint supply-neutral, sin inflación neta de vUSD) como fuente `cdp_interest`,
categoría `#VaultBacked` — misma ruta que `cdp_liquidation`. Query `getAccruedInterest(owner)`
deja que el frontend muestre el monto exacto antes de cerrar (`CdpPage.tsx`). La única
otra penalidad real sigue siendo la liquidación (13%, one-time, solo si el ratio cae
bajo el mínimo) — el interés no se cobra ahí, se acumula pero no se exige explícito
(el borrower ya pierde el 13% del colateral).

### 8.1 UX del Leverage Loop — decisión tomada

Los botones x2/x3/x4 de CdpPage son **shortcuts que prellenan el Leverage Simulator, no
ejecutan directamente**. El simulador siempre muestra, antes de cualquier confirmación:
exposición total, precio de liquidación, ratio actual, y probabilidad de liquidación a 30
días. La confirmación es explícita — nunca un solo clic desde los botones de shortcut.

### 8.2 Safe Haven — salida rápida en caída de mercado

Flujo de 3 clics para salir de exposición cripto hacia estable, disponible desde
cualquier posición de vault o CDP:
1. Salir del vault (ej. Exaltium/ICP) → recibe el ck-token líquido subyacente.
2. Swap del ck-token → vUSD (congela el valor en dólares).
3. Portfolio en modo "Capital Preservation" → rebalancea el resto hacia vUSD.

---

## 9. Oracle de Precios

| Fuente | Datos | Frecuencia |
|--------|-------|-----------|
| CoinGecko HTTP | ICP, BTC, ETH en USD | Cada 300s |
| mindicador.cl HTTP | CLP/USD | Cada 300s |
| Fallback | Caché anterior | Si falla el request |

**Reglas:**
- CLP: **nunca dividir por 100** — viene como entero (ej: 950 = 950 CLP por 1 USD)
- Si no hay caché: mostrar `"--"` en UI — NUNCA inventar datos
- Transform function obligatoria en todas las HTTP outcalls

---

## 10. ODL Bridge

El Vault Exaltite (ckUSDC) actúa como inventario de liquidez del bridge:
- $1 depositado en Vault Exaltite = $1 de capacidad ODL instantánea
- ICP liquida en ~2 segundos → el mismo pool puede procesar mucho más volumen mensual
- Capital del depositante siempre intacto — el pool no se "consume"
- Ver **VAELIX_ODL_MECHANICS.md** para detalle completo

---

## 11. Governance

```
Development → Sandbox → Restricted → Live
                              ↕
                           Paused
```

| Estado | Depósitos | Harvest | Withdraw | Staking |
|--------|-----------|---------|---------|---------|
| Development | ❌ | ✅ | ✅ | ❌ |
| Sandbox | ❌ | ✅ | ✅ | ❌ |
| Restricted | ✅ (whitelist) | ✅ | ✅ | ✅ |
| Live | ✅ | ✅ | ✅ | ✅ |
| Paused | ❌ | ✅ | ✅ | ❌ |

Solo `controllerPrincipal` puede cambiar el estado.

---

## 12. Feature Flags

| Flag | Estado (2026-08-03) | Activa |
|------|---------------------|--------|
| `ENABLE_SCLP_BRIDGE` | false | Cross-chain bridge sCLP |
| `ENABLE_CRYPTO_BRIDGE` | false | Bridge ETH/BTC nativo |
| `ENABLE_VUSD_MINT` | ✅ true | vUSD stablecoin (CDP) |
| `ENABLE_SCLP_MINT` | false | Acuñación sCLP |
| `ENABLE_ODL_PENALTIES` | false | Penalidades corredor ODL |
| `ENABLE_API_GATEWAY` | false | API access (Tier Corporativo/Apex) |

---

## 13. Roadmap Tokenomics

### V1.5 — Governance LUNX + ReFi
- LUNX acumulado da derecho a voto en parámetros del protocolo
- Activar ReFi (0% → X%) via votación T3+
- Boost de APR: 100,000 LUNX → +1% APR en cualquier vault

### V2.0 — RFRY Index Token
- Token que representa el portfolio de 40+ thesis tokens
- Rebalanceo automático cada 30 días
- Yield distribuido proporcionalmente a holders

### V2.1 — Vault Bundles (Slice ETF)
- 4 bóvedas temáticas (Brain / Interchain / Hardware / Consumer)
- Usuario deposita ckUSDC → exposure ponderada al slice
- Yield en PXRM

---

*Tokenomics v2.0 | Vaelix Protocol | 2026-06-28*
