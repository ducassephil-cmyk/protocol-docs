# GREYVALLEY — Corredor: Mecánica, Comparativa y Modelo de Socios
> Versión: 2026-06-28 | Complementa TOKENOMICS.md §10 (Corredor) | Actualizado 2026-09-04

---

## 1. QUÉ ES EL CORREDOR Y POR QUÉ IMPORTA

El corredor de GreyValley es la mecánica que permite mover valor entre monedas/países en segundos usando un activo digital como puente, en vez de pre-fondear cuentas nostro en cada banco corresponsal — el mismo principio de "liquidez a demanda" que usa la industria de remesas, aplicado con activos chain-key en vez de un partner bancario intermediario.

En GreyValley, el corredor conecta **CLP chileno ↔ ckUSDC/ckEURC ↔ monedas de destino** usando ICP como capa de settlement (~2 segundos de finalidad). Dos corredores tienen custodia real hoy: **CLP↔USD** (ckUSDC) y **CLP↔EUR** (ckEURC, integrado 2026-09-04) — cada uno con pool independiente en `liquidity_pool`, precio real vía `oracle.getClpPerUsd()`/`getClpPerEur()` (mindicador.cl).

El Vault Exaltite (ckUSDC/ckUSDT/ckEURC) **ES el pool de liquidez del corredor**. No es una cuenta bancaria ni un custodio — es el inventario de ckUSDC/ckEURC disponible para ejecutar transacciones instantáneamente, uno por corredor (CLP↔USD y CLP↔EUR no comparten custodia).

---

## 2. CÓMO FUNCIONA UN PAGO POR EL CORREDOR EN GREYVALLEY

> ⚠️ **Corrección real (auditoría 2026-08-24, ver `INSTRUCCIONES_FOUNDER.md`
> §7.1):** el flujo de abajo describe el diseño de la variante "Koywe" —
> pero `src/koywe_bridge/main.mo` **no existe** (0 código, sin canister
> deployado, ver §6 más abajo). No usar la palabra "activo" para esta
> variante en ningún material de founder/marketing hasta que exista
> código real. La única variante con código real HOY es sCLP nativo
> (Pegasus SpA + Fintoc, §7) — corre en sandbox, bloqueada para dinero
> real por aprobación CMF pendiente.

### Flujo de remesa (Chile → exterior) — diseño, Koywe pendiente de construir

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

## 6. KOYWE BRIDGE — Arquitectura técnica del on-ramp CLP → ckUSDC

> **Sesión 2026-08-02** — documentación completa del modelo de integración con Koywe como partner del corredor.

### Pregunta clave resuelta: ¿Koywe necesita instalar código ICP?

**NO.** Koywe es puro web2. No instala nada, no integra ningún SDK de ICP. La integración técnica vive 100% en el lado de GreyValley:

```
Koywe:    REST API web2 (PAYIN / ONRAMP / OFFRAMP / PAYOUT)
GreyValley:   canister koywe_bridge en ICP que llama la API de Koywe
```

Koywe solo necesita:
1. Un **webhook URL** donde notificar cuando un pago confirma
2. Una **EVM address** (Ethereum/Polygon) donde enviar el USDC

Ambas las provee el `koywe_bridge` canister de GreyValley.

### Koywe API — endpoints relevantes

| Endpoint | Función |
|----------|---------|
| `POST /v3/deals` | Crear orden ONRAMP (CLP → USDC). Parámetros: amount, fromCurrency, toCurrency, network (POLYGON/ETH/BSC), destinationAddress |
| `GET /v3/deals/{id}` | Consultar estado de orden |
| `POST /v3/payouts` | OFFRAMP / PAYOUT — convertir USDC a CLP y enviar a cuenta bancaria |
| Webhook `POST [tu_url]/koywe-hook` | Koywe notifica cuando el deal confirma |

Contacto: `soporte@koywe.com` (BD y acuerdos técnicos)  
Chains soportadas por Koywe: **Ethereum, Polygon, BSC** — NO ICP directamente.

### El triángulo de custodia (Custody Triangle)

```
┌─────────────────────────────────────────────────────────────────┐
│                 TRIÁNGULO DE CUSTODIA GREYVALLEY                 │
│                                                                 │
│  [Koywe]              [EVM custody wallet]      [ICP Ledger]   │
│  Custodia CLP         Custodia USDC             ckUSDC         │
│  ≈500 CLP             ≈0.55 USDC               0.55 ckUSDC     │
│  (queda con Koywe)    (address del canister)   (en user_principal) │
│                                                                 │
│  Koywe HODL CLP   →   USDC llega a EVM    →   Bridge minta ck  │
│                       custody wallet           1:1 en ICP       │
└─────────────────────────────────────────────────────────────────┘

Invariante: ckUSDC en circulación = USDC locked en EVM custody wallet (1:1)
```

**Qué custodia qué:**
- **Koywe** retiene el CLP chileno — es su modelo de negocio (spread entre compra y venta de USDC)
- **EVM custody wallet** (una address Ethereum/Polygon controlada por `koywe_bridge` via tECDSA) retiene el USDC real
- **ICP `ckusdc_ledger`** registra el ckUSDC gemelo, acreditado al `user_principal` del usuario

### Arquitectura del canister koywe_bridge

```
src/koywe_bridge/main.mo          ← A CONSTRUIR (pendiente)

Responsabilidades:
  1. Exponer webhook endpoint via api_gateway (raw.ic0.app)
  2. Almacenar: orderId → icp_principal (stable storage)
  3. Verificar USDC en EVM via HTTPS Outcall → EVM RPC
  4. Llamar ckusdc_ledger.icrc1_mint(user_principal, amount)
  5. Para off-ramp: llamar Koywe PAYOUT API via HTTPS Outcall

Clave EVM:
  - La EVM custody wallet usa Threshold ECDSA (tECDSA)
  - La private key NUNCA existe en ningún servidor
  - Se fragmenta entre los nodos de la subred ICP
  - El canister pide firma al runtime → ICP firma → tx enviada a EVM RPC

Ciclos necesarios:
  - ~1B cycles por HTTPS Outcall (verificación EVM RPC)
  - ~500M cycles por HTTPS Outcall (Koywe API)
  - Cargar 1T cycles iniciales → ~100 on-ramps de runway cómodo
```

### Flujo completo on-ramp con tracking de Principal

```
Paso 1: usuario conecta wallet → user_principal = "abc12-xyz34-..."
Paso 2: frontend crea orden Koywe con metadata { icp_principal: "abc12-xyz34..." }
Paso 3: koywe_bridge.registerOrder(koywe_order_id, user_principal) → stable map
Paso 4: usuario paga via Khipu → CLP recibido por Koywe
Paso 5: Koywe envía USDC a EVM custody wallet del canister
Paso 6: Koywe hace POST webhook → api_gateway.raw.ic0.app/koywe-hook
Paso 7: koywe_bridge verifica tx via EVM RPC HTTPS Outcall
Paso 8: koywe_bridge recupera user_principal del stable map (por koywe_order_id)
Paso 9: koywe_bridge.icrc1_mint({ to: {owner: user_principal}, amount }) en ckusdc_ledger
Paso 10: wallet GreyValley actualiza balance (icrc1_balance_of consulta el ledger)
```

### Estado actual del canister

| Componente | Estado |
|------------|--------|
| `src/koywe_bridge/main.mo` | ❌ **A CONSTRUIR** — arquitectura diseñada, código no escrito |
| EVM custody wallet address | ❌ No calculada — necesita `dfx canister create koywe_bridge` primero |
| Configuración Koywe | ❌ Acuerdo comercial pendiente (KYB Track B) |
| Canister ID en mainnet | ❌ Sin deploy |

> Para la implementación técnica de HTTPS Outcalls, tECDSA y ICRC-1: ver `GREYVALLEY_ICP_TECH.md`

### 6b. Tres rutas distintas — no confundirlas (diseñado 2026-09-01)

El `koywe_bridge` sirve a **tres rutas reales, con mecánicas y riesgos
regulatorios distintos**. Documentado acá para que quede claro cuál se
está construyendo en cada momento — se conversaron las tres seguidas en
la misma sesión y era fácil mezclarlas.

**Ruta A — On/off-ramp individual** (ya especificada arriba, §6, sin
cambios — esta sección solo confirma el orden de operaciones, que ya
estaba bien en el diseño original):

```
On-ramp:  usuario paga CLP a Koywe → Koywe manda USDC real a la wallet
          EVM → koywe_bridge VERIFICA la llegada (HTTPS Outcall a RPC) →
          RECIÉN AHÍ mintea ckUSDC 1:1 al usuario (Paso 7 antes que
          Paso 9, nunca al revés)

Off-ramp: usuario quema su ckUSDC → koywe_bridge confirma que el USDC
          salió real de la wallet EVM (o que el partner confirmó el
          depósito CLP en el banco destino) → RECIÉN AHÍ se refleja el
          quemado como definitivo
```

Es un **mint/burn directo al usuario** — no toca el pool del corredor
para nada. Nunca se adelanta la contabilidad a la confirmación real
(mismo criterio de esta sesión aplicado a otros bugs reales: CDP ratio,
staleness del oráculo — jamás mover el ledger antes de confirmar el
estado real del otro lado).

**Ruta B — Remesa internacional vía corredor (liquidez pooleada de
stakers), en el fondo un "Bank to Bank" vía Chain Fusion**:

A diferencia de la Ruta A, acá NO se mintea ckUSDC nuevo — se usa
ckUSDC que YA existe, pooleado por stakers en Exaltite, y el partner
repone la liquidez usada un rato después.

```
1. Staker deposita ckUSDC en Exaltite → se autopoolea al pool del
   corredor (liquidity-pool, CLP_USD). Ya 1:1 real vía el minter
   oficial de ckUSDC (DFINITY) — nada que respaldar de más acá.

2. Cliente de remesa paga CLP en Chile, pide que llegue USD a un banco
   destino en el extranjero.

3. El corredor usa el ckUSDC ya pooleado del staker como el lado "USD"
   de esta transacción — mismo modelo de inventario + oráculo que ya
   corre en liquidity_pool.swap() (sin AMM, sin slippage). El pool baja.

4. El partner (Koywe u otro) recibe el CLP del cliente por su lado,
   fuera de ICP — y es quien entrega el USD real al banco destino
   (su off-ramp, no el de GreyValley).

5. Un par de minutos después, el partner devuelve el equivalente en
   USDC real a la wallet EVM de GreyValley.

6. koywe_bridge mintea ckUSDC fresco con ese USDC recién llegado y lo
   devuelve al pool del corredor → rebalancea el pool a como estaba
   antes del paso 3. El staker recupera su respaldo real.
```

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

**Variante futura de la Ruta B (v3, con aprobación CMF)**: en vez de
que el partner maneje el CLP off-chain (como Koywe hoy), la reposición
del paso 4-6 podría venir de **sCLP → ckUSDC → USDC** — GreyValley
custodiando el CLP real (vía Pegasus SpA + Fintoc, ver §7 más abajo) en
lugar de depender de un partner externo para esa pierna. Esto es
estrictamente posterior a la aprobación CMF del sandbox regulatorio —
hoy la Ruta B corre 100% con partner externo (Koywe), sin que GreyValley
toque CLP en ningún punto.

**Por qué la Ruta B es más liviana regulatoriamente que sCLP solo**:
GreyValley nunca custodia CLP en esta ruta — coincide exacto con el
"Pilar 3" de la defensa regulatoria actual (`GREYVALLEY_REGULATORY.md`):
"GreyValley nunca toca pesos chilenos". La variante v3/CMF de arriba sí
cruzaría esa línea — por eso queda marcada explícitamente como futura y
condicionada a aprobación, no como parte del diseño actual.

**sCLP standalone** (§7 más abajo) — bloqueado por CMF, fuera de
alcance de esta sesión, no se toca. Custodia CLP propia sin depender de
ningún partner — es la pieza que la variante v3 de la Ruta B usaría el
día que exista, pero es un desarrollo aparte, ya documentado en §7.

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
