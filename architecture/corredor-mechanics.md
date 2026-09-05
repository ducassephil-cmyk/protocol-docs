# GREYVALLEY — Corredor: Mecánica, Comparativa y Modelo de Socios
> Versión: 2026-06-28 | Complementa TOKENOMICS.md §10 (Corredor) | Actualizado 2026-09-04

---

## 1. QUÉ ES EL CORREDOR Y POR QUÉ IMPORTA

El corredor de GreyValley es la mecánica que permite mover valor entre monedas/países en segundos usando un activo digital como puente, en vez de pre-fondear cuentas nostro en cada banco corresponsal — el mismo principio de "liquidez a demanda" que usa la industria de remesas, aplicado con activos chain-key en vez de un partner bancario intermediario.

En GreyValley, el corredor conecta **CLP chileno ↔ ckUSDC/ckEURC ↔ monedas de destino** usando ICP como capa de settlement (~2 segundos de finalidad). Dos corredores tienen custodia real hoy: **CLP↔USD** (ckUSDC) y **CLP↔EUR** (ckEURC, integrado 2026-09-04) — cada uno con pool independiente en `liquidity_pool`, precio real vía `oracle.getClpPerUsd()`/`getClpPerEur()` (mindicador.cl).

El Vault Exaltite (ckUSDC/ckUSDT/ckEURC) **ES el pool de liquidez del corredor**. No es una cuenta bancaria ni un custodio — es el inventario de ckUSDC/ckEURC disponible para ejecutar transacciones instantáneamente, uno por corredor (CLP↔USD y CLP↔EUR no comparten custodia).

---

## 2. CÓMO FUNCIONA UN PAGO POR EL CORREDOR EN GREYVALLEY

> ⚠️ **Corrección real (actualizado 2026-09-04, ver §6 más abajo para el
> detalle completo):** el diseño ORIGINAL de esta sección (un canister
> `koywe_bridge` con wallet EVM y minting propio) nunca se construyó y
> `src/koywe_bridge/main.mo` **no existe**. Pero eso NO significa que no
> haya código real — el camino que sí existe y está deployado en mainnet
> (`api_gateway` → `settlement` → `bridge` → `liquidity_pool`) es distinto
> y más simple, sin wallet EVM ni minting. Hoy está **apagado por feature
> flag** (`bridge=false`) y sin Koywe registrado todavía como institución
> — no por falta de código, sino porque el KYB comercial sigue pendiente
> (`INSTRUCCIONES_FOUNDER.md` §3). No usar la palabra "activo" para el
> corredor con Koywe hasta que el KYB esté aprobado y el flag prendido.
> sCLP nativo (Pegasus SpA + Fintoc, §7) sigue siendo la única variante
> bloqueada por CMF (no por código) — corre en sandbox.

### Flujo de remesa (Chile → exterior) — código real deployado, apagado hasta que Koywe apruebe KYB

```
1. Usuario deposita CLP en Koywe (custodia y KYB de Koywe — NO toca ningún
   canister de GreyValley; CLP no tiene representación on-chain sin sCLP)
2. El Vault Exaltite (pool ckUSDC financiado por depositantes) libera el ckUSDC
   equivalente on-chain instantáneo — no espera a que el CLP del paso 1 se
   "convierta"; patrón inventario-primero: el pool paga con inventario existente
3. ckUSDC viaja en ICP chain (~2 segundos)
4. En destino: Koywe redime el ckUSDC (minter oficial de ICP → USDC real, o
   sus propios rieles) y liquida a moneda local (USD, MXN, EUR, etc.)
5. Destinatario recibe fondos
6. El CLP del paso 1 (menos fees) es lo que eventualmente rebalancea el pool
   ckUSDC de GreyValley — no hay conversión CLP→ckUSDC atómica por transacción

Fee para GreyValley: 0.22% + rampa ~1% (total usuario: ~1.2%)
Fee SWIFT equivalente: ~2.5–3.5%
```

### El pool no se "vacía"

El ckUSDC del vault no es consumido por cada transacción — se usa como **garantía de liquidez instantánea**. El flujo real:

- ckUSDC entra al pool cuando hay pagos en dirección inversa (entrada a Chile)
- ckUSDC sale cuando hay pagos hacia el exterior
- Si el flujo es **bidireccional**: pool se autorrepone, capital intacto permanentemente
- Si el flujo es **unidireccional**: pool puede desbalancearse → el protocolo rebalancea con fees acumulados (0.22% × volumen)

**El capital del depositante NUNCA desaparece.** Si el pool se desbalancea extremo, el bridge se pausa — pero el capital sigue accesible para retirar.

### Capacidad real vs TVL

```
$20K ckUSDC en vault → $20K de capacidad instantánea (en vuelo simultáneo)
Pero ICP liquida en ~2 segundos:
→ $20K pool puede procesar $500K-$1M+ en volumen mensual
   (si no hay más de $20K en vuelo en el mismo instante)

Riesgo real: flujo unidireccional extremo, no el tamaño del pool
```

### 2.1 Variantes de destino diseñadas (post-CMF / Chanfusion Satellite)

Migrado desde `GREYVALLEY_LIVING_SPEC.md` (borrado 2026-08-07). Estos diagramas describen la
ruta con sCLP vía Chanfusion Satellite — **fuera de alcance mientras sCLP siga bloqueado
por CMF**, no el flujo Koywe activo de la sección 2. Se conserva como diseño de referencia
para cuando ese corredor se habilite.

> **No confundir con el corredor CLP/EUR real (2026-09-04).** El diagrama
> "Chile → Europa" de abajo describe una ruta FUTURA vía sCLP (bloqueada
> por CMF, sin código de sCLP↔ckUSDC↔EUR construido). El corredor CLP/EUR
> que SÍ existe hoy (`liquidity_pool` par CLP_EUR, `bridge_canister`
> `depositEurcToCkEurc()`) es un camino distinto y más simple: ckEURC
> real (Circle EUR, ledger DFINITY) entra directo vía tECDSA desde una
> wallet EVM, sin pasar por sCLP ni por Chanfusion. Ambos caminos pueden
> coexistir a futuro (sCLP para el lado CLP custodiado, ckEURC para el
> lado EUR ya resuelto), pero hoy solo el segundo tiene custodia real.

**Chile → Europa (diseño futuro, sCLP — bloqueado por CMF)**
```
1. CLP → Fintoc Webhook → Chanfusion
2. Canister acuña sCLP 1:1
3. AMM: sCLP → ckUSDC (oráculo mindicador.cl)
4. Chain Fusion: USDC nativo en Ethereum
5. Partner europeo: USDC → EUR → banco
6. Burn sCLP on-chain — ciclo cerrado en <60s
```

**Chile → LATAM**
```
1. CLP → Khipu → Chanfusion → sCLP
2. AMM: sCLP → ckUSDC
3. Koywe API: USDC → COP/PEN/ARS/BRL
4. Depósito bancario en país destino
5. Burn sCLP
```

---

## 3. COMPARATIVA: GREYVALLEY vs XRP/RIPPLE vs SWIFT

### El modelo Ripple/XRP

Ripple usa XRP como activo puente entre market makers (Bitso, SBI Remit, etc.):

```
Empresa USA → USD → MM compra XRP → XRP viaja → MM México vende XRP → MXN → destinatario
```

Los market makers mantienen **inventario de XRP en ambos lados del corredor**. Ese inventario ES su liquidez del corredor. Ganan el spread + fees por transacción.

**El problema:** XRP es volátil. Si XRP cae 40% mientras el MM tiene el inventario, pierde en el principal. Solo entidades grandes con tolerancia a esa volatilidad se vuelven MMs serios.

### Tabla comparativa

| Dimensión | SWIFT | XRP/Ripple | GreyValley |
|-----------|-------|-----------|--------|
| Activo puente | Nada (nostro directo) | XRP (volátil) | ckUSDC (estable $1) |
| Velocidad | 1-5 días hábiles | 3-5 segundos | ~2 segundos |
| Fee típico usuario | 2.5-3.5% | 0.3-0.5% | ~0.2-0.4% |
| Riesgo del MM | Bajo (fiat) | Alto (XRP volatilidad) | Muy bajo (ckUSDC estable) |
| Entrada de nuevos MMs | Acuerdo Ripple privado | Barrera alta | Depositar ckUSDC en vault |
| Gobernanza | Bancaria / bilateral | Ripple Inc. centraliza | On-chain, transparent |
| Corredor inicial | Global bancario | USD↔MXN, USD↔PHP, etc. | CLP↔USD (LATAM first) |
| Yield para el MM | Ninguno adicional | Solo spread/fees | APR vault + Volume Guild |

### Por qué ICP está mejor posicionado que XRP para un corredor global

XRP nació como activo especulativo y fue adaptado como puente. ICP nació como infraestructura de cómputo distribuido con:

1. **Finalidad en ~2 segundos** (vs 3-5s de XRP, sin reversiones)
2. **HTTP outcalls nativos** — el canister puede llamar APIs bancarias directamente (Koywe, Fintoc, SWIFT MX, etc.) sin middleware externo
3. **Ciclos de compute estables** — el costo de procesar una transacción del corredor no fluctúa con el precio de ICP
4. **ckAssets nativos** (ckBTC, ckUSDC, ckETH) — son representaciones 1:1 de activos reales dentro de ICP, sin bridging adicional
5. **Canisters = código que no puede apagarse** — el corredor no puede ser desactivado por un banco central o regulador

La analogía: XRP es una autopista privada construida sobre terreno prestado. ICP es una autopista pública que también sirve como capa de internet descentralizado — el corredor es solo uno de sus casos de uso.

---

## 4. EL MODELO DE SOCIOS EMPRESARIALES COMO MARKET MAKERS

El insight clave de este documento: **los socios empresariales de GreyValley no son "inversores" — son market makers del corredor CLP**.

### Qué hace un market maker en Ripple

- Bitso (México) deposita XRP en ambos lados del corredor USA↔MX
- Cuando alguien envía $1,000 USA→MX, Bitso ejecuta la conversión instantáneamente
- Bitso gana el spread (diferencia compra/venta de XRP) + comisión
- El tamaño de su inventario determina su capacidad de volumen

### Qué hace un socio empresarial en GreyValley

- Empresa deposita ckUSDC en Vault Crypto
- Ese ckUSDC es el inventario del corredor CLP↔USD
- Cuando alguien envía CLP a México/USA, el pool de ckUSDC ejecuta instantáneamente
- El socio gana: PXRM Base APR del vault (pagado en PXRM, no ckUSDC) + fees del AMM/corredor proporcional a su liquidez (variable, según volumen real) + Volume Guild si >$25K/mes en el corredor

### La doble ventaja del socio que también usa la app para sus propios pagos

Una empresa que deposita ckUSDC Y procesa sus propios pagos internacionales por la app obtiene
hasta 3 fuentes de ingreso — **ninguna es un monto fijo garantizado, todas menos una dependen
del volumen real que pase por el corredor**:

```
Ingreso 1 (semi-fijo, subsidiado por treasury):
  PXRM Base APR — 12-24% según lock, PAGADO EN PXRM, no en ckUSDC.
  Depende del treasury (bootstrap) hasta que el fee real lo sostenga solo.

Ingreso 2 (variable, depende de volumen real):
  Fees del AMM sobre el par ckUSDC pooleado (feeBps=33 hoy, ~0.33%/swap)
  — 0 si no hay volumen pasando por ese pool.

Ingreso 3 (variable, depende de volumen real del corredor):
  Share proporcional de tu ckUSDC/TVL sobre el 0.22% del corredor
  (Volume Guilds, 7% del fee pool) — SOLO si el corredor procesa
  >$25K/mes en volumen real. Hoy el corredor tiene volumen ~0 (ver
  GREYVALLEY_AUDIT_LIVE.md), este ingreso es ilustrativo de la
  MECÁNICA, no una proyección de lo que se cobra hoy.

Ahorro: tus propios pagos por el corredor salen al 0.22% en vez de
  2.5-3.5% SWIFT — este SÍ es real e inmediato, no depende de terceros.
```

**Ningún número de arriba es "deposita $100, recibe $X/año garantizado".** El único ingreso
con un piso semi-predecible es el PXRM Base APR (pagado en PXRM, subsidiado por treasury
mientras el fee real todavía no lo sostiene) — todo lo demás (Ingreso 2, Ingreso 3) es
estrictamente proporcional al volumen real que pase por el AMM/corredor, que hoy es bajo
o nulo. El pitch correcto no es "te pagamos X% fijo" — es "sé el Bitso de Chile: tu
inventario en ckUSDC (estable, sin riesgo de precio) capta valor de CADA transacción real
que pasa, y crece con el volumen, no con una promesa de rendimiento fijo."

---

## 5. BOOTSTRAP CAPITAL — CUÁNTO SE NECESITA

### Mínimo viable (solo el fundador)

| Vault | Mínimo demo | Lanzamiento creíble |
|-------|------------|---------------------|
| Exaltium (ICP) | $15K | $40K |
| Crypto (ckBTC/ckETH) | $5K | $15K |
| **Exaltite (ckUSDC/ckEURC)** ← crítico | **$20K** | **$50K** |
| Total TVL personal | $40K | $105K |

> Naming corregido 2026-08-03: Puranium es el canister de staking PXRM, separado de estos 3 vaults — no es "el vault ICP". Ver `GREYVALLEY_APR_MODEL.md` §12 para el mapping completo.

**Treasury para PXRM Base APR (6 meses, TVL mínimo):**
- Costo bruto: ~$2,000 en valor PXRM
- NNS staking del ICP genera ~$500-700/mes (reduce el costo real)
- Costo neto efectivo: ~$1,000-1,500 para 6 meses

**El ckUSDC/ckEURC es el cuello de botella:** sin ellos no hay capacidad del corredor y los APRs del Vault Exaltite se quedan en `"--"`.

### Con socios empresariales (escalable)

| Perfil socio | Pagos propios/mes | TVL sugerido en vault | Ahorro vs SWIFT | Guild |
|-------------|------------------|-----------------------|----------------|-------|
| Pequeño (empresa ~$15K pagos) | $15K | $10-25K | ~$375/mes | No |
| Mediano (empresa ~$30K pagos) | $30K | $25-75K | ~$750/mes | No |
| Grande (empresa ~$75K pagos) | $75K | $75K-300K | ~$1,875/mes | ✓ Sí |

**Estructura de captación sugerida fase 1:**
- 3 socios Ángel ($15K TVL c/u) → $45K TVL externo
- 1 socio Semilla ($40K TVL) → $40K TVL
- 1 socio Estratégico ($100K TVL) → $100K TVL + Guild activo
- Total externo: ~$185K TVL → protocolo puede self-sustain el PXRM Base APR

### Revenue projection (del APR Model V3)

| Corredor Mensual | Fees del Corredor | NNS Yield | Swap+CDP | Total/mes |
|-------------|---------|-----------|---------|----------|
| $50K | $250 | $600 | $250 | ~$1,100 |
| $200K | $1,000 | $1,200 | $500 | ~$2,700 |
| $500K | $2,500 | $2,000 | $1,200 | ~$5,700 |
| $1M | $5,000 | $3,000 | $2,500 | ~$10,500 |
| $2M+ | $10,000 | $4,000 | $5,000 | ~$19,000 |

---

## 6. KOYWE — Arquitectura técnica real de la integración

> **Actualizado 2026-09-04** — el diseño original de esta sección (sesión
> 2026-08-02) describía un canister `koywe_bridge` con wallet EVM propia
> y minting de ckUSDC vía tECDSA. **Ese diseño nunca se construyó y ya no
> es el camino real** — `src/koywe_bridge/main.mo` no existe, 0 líneas de
> código, sin canister deployado (confirmado en el filesystem del repo).
> La integración real que SÍ está deployada y funcionando es más simple:
> Koywe (o cualquier institución registrada) llama un endpoint HTTP
> genérico ya construido — `api_gateway` → `settlement` → `bridge` →
> `liquidity_pool` — sin ningún canister dedicado a Koywe, sin wallet EVM,
> sin minting nuevo. Reescrito para reflejar esto.

### Pregunta clave: ¿Koywe necesita instalar código ICP?

**NO.** Koywe es puro web2 (REST API). No instala nada, no integra ningún
SDK de ICP. Toda la integración vive del lado de GreyValley, en
canisters **ya deployados en mainnet**:

```
Koywe:      REST API web2 propia (PAYIN / ONRAMP / OFFRAMP / PAYOUT)
GreyValley: api_gateway (webhook + registro de institución)
              → settlement (tracking del pago)
              → bridge (execute_bridge — motor real: precio de oráculo + cola FIFO)
              → liquidity_pool (swap() contra el inventario ckUSDC/ckEURC ya pooleado)
```

Los cuatro canisters de arriba **ya están deployados y en producción**
(ver `GREYVALLEY_MASTER_STATE.md` §1) — no falta construir nada técnico
para que Koywe empiece a mandar pagos, solo falta:
1. Que Koywe apruebe el KYB (documentos §3 de `INSTRUCCIONES_FOUNDER.md`).
2. Registrar a Koywe como `Institution` en `api_gateway` (`registerInstitution()`,
   admin-only) — API key, corredores autorizados (ej. `CLP_USD`, `CLP_EUR`),
   límite diario en CLP.
3. Prender el feature flag `bridge` (hoy `false` — ver `getFeatureFlags()`).

### Koywe API — endpoints relevantes (lado Koywe)

| Endpoint | Función |
|----------|---------|
| `POST /v3/deals` | Crear orden ONRAMP (CLP → USD). Parámetros: amount, fromCurrency, toCurrency, network, destinationAddress |
| `GET /v3/deals/{id}` | Consultar estado de orden |
| `POST /v3/payouts` | OFFRAMP / PAYOUT — convertir a CLP y enviar a cuenta bancaria destino |

Contacto: `business@koywe.com` (BD y acuerdos técnicos) | [koywe.com/partners](https://koywe.com/partners)

### Endpoint real del lado GreyValley (ya construido)

```
POST api_gateway.raw.ic0.app/v1/payments

Headers: Authorization con la API key de la institución (hash SHA256
         comparado server-side — la key nunca se guarda en texto plano)
Body:    { institutionId, sourceAmount, sourceCurrency, destCurrency,
           destAccount, destInstitution }

Flujo interno real (api_gateway/main.mo, línea ~250 en adelante):
  1. Valida feature flag `bridge` (backend.getFeatureFlags()) — si está
     apagado, responde 503 "Corredor deshabilitado".
  2. Valida API key contra el hash guardado de la institución (401 si falla).
  3. Valida que la institución esté autorizada para ESE corredor
     específico (source_dest, ej. "CLP_USD") — 403 si no.
  4. Valida límite diario en CLP (429 si se excede; reset automático por
     cambio de día calendario).
  5. Registra el pago en `settlement.initiate_payment()` (tracking real,
     no un placeholder).
  6. Llama `bridge.execute_bridge()` — este SÍ es el motor real: usa el
     precio del oráculo (mindicador.cl) + el mismo modelo de inventario y
     cola FIFO (`#Queued`) que ya corre en `liquidity_pool.swap()` para
     usuarios retail. Responde #Success, #Queued, #InsufficientLiquidity,
     #UnsupportedCorridor, #SystemNotLive o #Error — nunca inventa un
     resultado.
```

**No hay minting de ckUSDC/ckEURC nuevo en este flujo.** El corredor usa
el inventario que YA existe — ckUSDC/ckEURC pooleado por los depositantes
de Vault Exaltite (ver §2 de este documento, "modelo de inventario"). Koywe
custodia el CLP y entrega la moneda destino por su propio lado (su
off-ramp, fuera de ICP) — GreyValley nunca toca ese tramo ni el CLP.

### Estado real (verificado en el filesystem y el código, 2026-09-04)

| Componente | Estado |
|------------|--------|
| `api_gateway` (webhook + registro institución) | ✅ **Deployado en mainnet**, `ua27v-4yaaa-aaaah-quyfa-cai` |
| `settlement` (tracking de pagos) | ✅ **Deployado en mainnet**, `yknil-6yaaa-aaaah-quzja-cai` |
| `bridge` (motor real del corredor) | ✅ **Deployado en mainnet**, `us4im-qiaaa-aaaah-quyga-cai` — feature flag `bridge=false` hoy |
| `liquidity_pool` (swap contra inventario real) | ✅ **Deployado en mainnet**, `uv5oy-5qaaa-aaaah-quygq-cai` |
| `src/koywe_bridge/main.mo` (diseño original de 2026-08-02) | ❌ **No existe — nunca se construyó, ya no es el plan** |
| Koywe registrado como `Institution` real | ❌ Pendiente — depende del KYB (§3 `INSTRUCCIONES_FOUNDER.md`) |
| KYB Koywe (documentos, sandbox, producción) | ⏳ Pendiente |

> Para la implementación técnica de HTTPS Outcalls, tECDSA y ICRC-1 (usadas
> en OTROS canisters del protocolo, no en este flujo): ver `GREYVALLEY_ICP_TECH.md`

### 6b. Dos rutas distintas — no confundirlas

Se diseñaron dos rutas posibles para partners tipo Koywe. Solo una tiene
código real hoy.

**Ruta A — On/off-ramp individual con mint/burn directo al usuario**
(diseño de 2026-08-02, sección §6 original de este documento):
requeriría una wallet EVM custodiada por tECDSA que reciba USDC real y
mintee ckUSDC 1:1 al usuario recién tras verificar la llegada por HTTPS
Outcall. **No tiene código — es un diseño sin construir**, distinto del
que sí corre hoy.

**Ruta B — Remesa vía corredor con liquidez ya pooleada (la real, la que
describe §6 de arriba)**: no mintea nada nuevo — usa el ckUSDC/ckEURC que
ya está pooleado por los depositantes de Exaltite como el lado "destino"
de la transacción, mismo modelo de inventario + oráculo que
`liquidity_pool.swap()`. El partner (Koywe) recibe el CLP del cliente por
su lado y entrega la moneda destino directamente — no hay una reposición
de vuelta a un "wallet EVM de GreyValley" porque esa wallet no existe en
este diseño; el ciclo se cierra simplemente con el CLP que Koywe se
queda como margen de su propio negocio.

**Qué pasa si el pool se queda sin inventario**: ya existe cola FIFO real
en `liquidity_pool.swap()` (`#queued`, `drainQueue()`) para esto — nunca
ejecuta a peor precio ni rechaza, espera. **Gap real identificado
2026-09-01, sin código todavía**: hoy el pool se drena literal hasta
`foreignReserve = 0` antes de encolar — no hay ningún piso de reserva
(a diferencia de los vaults, que sí tienen `checkBufferFloor` al 5% de
NAV). Pendiente de diseño: agregar un piso configurable (10-15% del
pool) que empiece a encolar ANTES de llegar a cero real, para que el
staker que puso el primer ckUSDC nunca vea el pool completamente vacío.

**Qué pasa si el partner se demora en reponer (paso 5-6)**: el pool
puede agotarse antes de que llegue la reposición. Ya existe cola FIFO
real en `liquidity_pool.swap()` (`#queued`, `drainQueue()`) para esto —
nunca ejecuta a peor precio ni rechaza, espera. **Gap real identificado
2026-09-01, sin código todavía**: hoy el pool se drena literal hasta
`foreignReserve = 0` antes de encolar — no hay ningún piso de reserva
(a diferencia de los vaults, que sí tienen `checkBufferFloor` al 5% de
NAV). Pendiente de diseño: agregar un piso configurable (10-15% del
pool) que empiece a encolar ANTES de llegar a cero real, para que el
staker que puso el primer ckUSDC nunca vea el pool completamente vacío.

**Fondeo propio como mitigante adicional** (no reemplaza el piso de
arriba, lo complementa): el mínimo real ya documentado en
`TOKENOMICS.md` (Vault Exaltite ≥$50K ckUSDC para capacidad del corredor día 1)
hace que agotarse sea raro en la práctica — pero es plata quieta, no
una garantía de código.

**Variante futura de la Ruta B (v3, con aprobación CMF)**: en vez de que
el partner maneje el CLP off-chain (como Koywe hoy), el lado CLP podría
venir de **sCLP → ckUSDC/ckEURC**, con GreyValley custodiando el CLP real
(vía Pegasus SpA + Fintoc, ver §7 más abajo) en lugar de depender de un
partner externo para esa pierna. Esto es estrictamente posterior a la
aprobación CMF del sandbox regulatorio — hoy la Ruta B corre 100% con
partner externo (Koywe), sin que GreyValley toque CLP en ningún punto.

**Por qué la Ruta B es más liviana regulatoriamente que sCLP solo**:
GreyValley nunca custodia CLP en esta ruta — coincide exacto con el
"Pilar 3" de la defensa regulatoria actual (`GREYVALLEY_REGULATORY.md`):
"GreyValley nunca toca pesos chilenos". La variante v3/CMF de arriba sí
cruzaría esa línea — por eso queda marcada explícitamente como futura y
condicionada a aprobación, no como parte del diseño actual.

**sCLP standalone** (§7 más abajo) — bloqueado por CMF. Custodia CLP
propia sin depender de ningún partner — es la pieza que una variante
futura de la Ruta B usaría el día que exista, pero es un desarrollo
aparte, ya documentado en §7.

---

---

## 7. CUSTODIA MULTI-FIAT — sCLP / ckBRL / ckARS / ckMXN (2026-08-15)

Pregunta del founder: para escalar el corredor más allá de CLP, ¿se busca partners
nuevos por país, o se arma custodia propia (GreyValley/founder) en cada fiat? Esta
sección registra el análisis y la recomendación.

### Qué ya existe (sCLP, el único corredor con código real)

`src/sclp_ledger/main.mo` + `src/sclp_treasury/main.mo` implementan el patrón
completo: depósito CLP en la cuenta bancaria de **Pegasus SpA** (custodia propia,
empresa chilena del founder) → detectado vía **Fintoc** (Open Banking API chilena,
polling HTTPS outcall cada 60s) → mint 1:1 de sCLP → reconciliación cada 10 min
contra el saldo bancario real (circuit breaker: `reconciliationOk`). Requiere
aprobación CMF (Ley 21.521) antes de manejar dinero real; el código puede existir
y probarse en sandbox sin esa aprobación (ver §6 de este documento sobre el
patrón Koywe/CMF, y `GREYVALLEY_CORREDOR_GREYVALLEY_SANDBOX.md` si se documenta aparte).

Esto es viable porque Pegasus SpA **ya es una entidad chilena real** — Fintoc solo
opera sobre bancos chilenos y mexicanos, así que la pieza que falta para CLP es
regulatoria (CMF), no bancaria ni de código.

**Hueco real encontrado 2026-08-24 — off-ramp (retiro) sin payout real:**
`sclp_treasury.requestRedeem()` quema el sCLP del usuario y registra la
solicitud, pero nunca dispara el TEF de vuelta al banco — el código solo
tenía el mint (on-ramp) completo. Investigado el endpoint real de Fintoc
(`POST /v2/transfers`) para completarlo: requiere **firma JWS por
request** (JSON Web Signature), un mecanismo distinto y más complejo que
el `Authorization: apiKey` que ya usa el polling de movimientos/balance.
Necesita que el founder genere las claves de firma en su dashboard de
Fintoc antes de poder implementar esto — ver `INSTRUCCIONES_FOUNDER.md`
§7.2 para el detalle completo y el siguiente paso.

### Qué haría falta para BRL / ARS / MXN

| Fiat | Cobertura Fintoc | Qué requiere custodia propia | Viabilidad |
|------|------------------|-------------------------------|------------|
| **ckMXN** | Sí (Fintoc cubre México) | Cuenta bancaria real en México a nombre de una entidad — la del founder o de un partner mexicano | Técnicamente el camino más corto de los tres: mismo patrón que sCLP, mismo proveedor (Fintoc), pero exige abrir banco/entidad en México — no es solo desplegar un canister |
| **ckBRL** | No — Fintoc no cubre Brasil | Proveedor de Open Banking brasileño (ej. Belvo, Pluggy — ambos con soporte Pix) + cuenta bancaria en Brasil + registro ante BACEN si se opera como institución de pago | Custodia propia implica compliance regulatorio brasileño completo — alto costo para operar en solitario |
| **ckARS** | No hay equivalente viable | Cualquier custodia de USD/ARS choca con los controles de cambio del BCRA (cepo cambiario) | El más difícil de los tres — no recomendado como próximo paso mientras persista el cepo |

### Recomendación

**Custodia 100% propia (founder/Pegasus SpA) solo es realista para CLP** — ya está
armada y es la extensión natural de una empresa que el founder ya controla. Para
BRL y MXN, el camino rápido no es reconstruir el mismo triángulo de custodia en
3 países más (banco propio + licencia + compliance local en cada uno), sino
**buscar partners locales ya licenciados** — EMIs o PSPs con cuenta bancaria y
autorización regulatoria propia en su país — que jueguen el mismo rol que Pegasus
SpA juega para CLP. El canister-side (`sclp_ledger`/`sclp_treasury`) es
reutilizable como plantilla por fiat (mismo patrón mint/burn/reconcile), cambiando
solo el proveedor de Open Banking y el custodio bancario detrás.

ARS queda fuera del roadmap cercano hasta que cambien las restricciones
cambiarias — no es un problema de arquitectura, es un problema regulatorio externo
que ninguna integración técnica resuelve.

---

## 8. PREGUNTAS TÉCNICAS PENDIENTES (V1)

1. **NNS Neuron staking en nombre del usuario:** ¿Puede el canister de GreyValley stakear el ICP depositado en NNS en nombre del usuario? ¿O el canister es el "dueño" del neuron y distribuye el yield manualmente? → Investigar custodia.

2. **T1 streaming implementation:** El yield de T1 corre segundo a segundo. En Motoko, esto implica o bien un timer muy frecuente o bien cálculo lazy al momento del harvest. ¿Cuál es más eficiente en ciclos de ICP?

3. **Rebalanceo unidireccional automático:** ¿Puede el protocolo ejecutar un swap automático ckUSDC→ICP→ckUSDC para rebalancear sin intervención manual, usando los fees acumulados?

4. **ckUSDC en ICP vs real USDC:** El ckUSDC en ICP es una representación 1:1 del USDC en Ethereum via Chainfusion. ¿Hay delay o riesgo de depegging en condiciones extremas de mercado?

---

*Documento: corredor-mechanics.md (antes GREYVALLEY_ODL_MECHANICS.md) | 2026-06-28 · Actualizado: 2026-09-04 (renombrado, referencias "ODL" retiradas, fee real 0.22% tras la subida +10% de 2026-08-28, corredor CLP/EUR agregado como real)*  
*Relacionado: GREYVALLEY_APR_MODEL.md, TOKENOMICS.md §10, GREYVALLEY_ICP_TECH.md (mecánica Principal/HTTPS Outcalls/tECDSA)*
