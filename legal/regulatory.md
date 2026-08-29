# VAELIX — Marco Regulatorio & Estrategia CMF
> Análisis legal-técnico bajo Ley 21.521 (Ley Fintech Chile, 2023)
> Creado: 2026-08-03 | Propósito: guía interna del founder para navegación regulatoria
>
> ⚠️ **BORRADOR — no es asesoría legal.** Este documento es un análisis técnico
> preliminar generado por IA para orientar la conversación con un abogado, no
> un sustituto de esa revisión. Ningún argumento acá (sandbox, tECDSA, Safe)
> debe presentarse a CMF ni usarse como base de decisión sin que un abogado
> especializado en Ley 21.521 lo revise primero — mismo criterio que
> `INSTRUCCIONES_FOUNDER.md` aplica a sus propios puntos legales.

---

## 1. Marco Regulatorio Chile — Lo que aplica a GreyValley

### Ley 21.521 (Ley Fintech, enero 2023)
La ley más relevante. Regula:
- **Plataformas de financiamiento colectivo** — no aplica directamente a GreyValley
- **Sistemas alternativos de transacción (SAT)** — posiblemente aplica al ODL y swap
- **Custodia de instrumentos financieros y activos virtuales** — **el riesgo principal**
- **Servicios de enrutamiento de órdenes** — posiblemente aplica al Corredor

### CMF (Comisión para el Mercado Financiero)
Regula la actividad. Tiene facultad de:
- Autorizar, suspender o cancelar operación de entidades reguladas
- Exigir capital mínimo, manuales de conducta, auditorías
- Multar hasta 15.000 UF por infracción grave

La Ley 21.521 crea un **sandbox regulatorio** (Art. 90+) que permite operar durante hasta 24 meses bajo supervisión CMF mientras se evalúa el modelo de negocio. **Esta es la vía de entrada recomendada para GreyValley.**

### UAF (Unidad de Análisis Financiero)
Regula AML/CFT. Aplica a entidades que manejan activos virtuales como PSAVs (Proveedores de Servicios de Activos Virtuales). Koywe absorbe gran parte de este riesgo al hacer el KYC/KYB del on-ramp.

### Banco Central de Chile
Regula sistemas de pago. El sCLP (peso chileno sintético) podría necesitar pronunciamiento del BC si escala. Los canisters de ckUSDC y ckBTC en principio son tokens foráneos, no emisión monetaria chilena.

---

## 2. La Pregunta de Custodia — ¿GreyValley Administra Fondos de Terceros?

### El triángulo de custodia del on-ramp CLP → ckUSDC

```
[Usuario] — paga CLP vía TEF/WebPay
    ↓
[Koywe] — recibe CLP, entrega USDC en EVM (REGULADO, KYC/AML absorbido)
    ↓
[EVM Custody Wallet / Smart Contract] ← AQUÍ ESTÁ LA PREGUNTA LEGAL
    ↓
[bridge_canister canister — ICP] — minta ckUSDC al user_principal
    ↓
[Usuario] — recibe ckUSDC en su wallet ICP
```

### El análisis honesto

| Punto de la cadena | ¿Quién tiene el activo? | ¿Riesgo de "custodia"? |
|--------------------|------------------------|------------------------|
| CLP en Koywe | Koywe (regulado, KYC) | ❌ No es GreyValley |
| USDC en EVM wallet | Contrato/wallet controlado por `bridge_canister` canister | ⚠️ **SÍ — este es el riesgo** |
| ckUSDC en ledger ICP | Usuario directamente (su Principal) | ✅ No es custodia de GreyValley |

**El riesgo concreto:** mientras el `bridge_canister` canister tenga un `controllerPrincipal` que sea el team GreyValley (una wallet que controla el equipo), existe control humano unilateral sobre el USDC. Un regulador puede trazar esa cadena: GreyValley desplegó el canister → el canister controla el EVM wallet → el EVM wallet tiene USDC de usuarios → GreyValley custodia USDC de terceros.

---

## 3. Mitigantes Técnicos — Arquitectura que Debilita el Argumento de Custodia

### Mitigante 1: tECDSA — sin clave privada humana

La clave privada del EVM wallet **no existe** en ningún servidor de GreyValley. Está distribuida como shares entre 28+ nodos independientes del subnet de ICP (operados por entidades distintas en distintas jurisdicciones). Para firmar una transacción, el protocolo ICP requiere el umbral (threshold) — ningún nodo individual (ni GreyValley) puede hacerlo solo.

**Argumento legal:** GreyValley no "tiene" la llave del EVM wallet en ningún sentido tradicional — es análogo a cómo un protocolo DeFi como Maker "tiene" el ETH colateral sin que ninguna empresa lo custodie.

**Limitación:** el argumento es nuevo en Chile y no tiene jurisprudencia. Un regulador conservador puede ignorarlo.

### Mitigante 2: EVM smart contract en lugar de EOA

En vez de un EVM wallet de tipo EOA (Externally Owned Account, controlado por tECDSA), desplegar un **contrato EVM** (custom escrow o Gnosis Safe) con reglas inmutables:

```solidity
// Regla del contrato EVM
function releaseUSDC(address recipient, uint256 amount, bytes calldata icpProof) external {
    // Solo callable desde la dirección tECDSA del bridge_canister canister
    require(msg.sender == KOYWE_BRIDGE_TECDSA_ADDRESS, "Unauthorized");
    // Solo libera si hay prueba de burn de ckUSDC en ICP
    require(verifyIcpBurn(icpProof), "Invalid ICP proof");
    usdc.transfer(recipient, amount);
}
```

**¿El Gnosis Safe "tiene custodia" del USDC?**
Sí, el USDC vive en la dirección del Safe. Pero la pregunta legal relevante no es quién tiene el activo sino **quién puede moverlo**:
- Safe con firmantes humanos (equipo GreyValley como signers): custodia humana → **MAL**
- Safe donde el **único signer autorizado es la dirección tECDSA del `bridge_canister`**: ningún humano puede liberar el USDC sin pasar por el algoritmo → **BIEN**

La arquitectura correcta: el Safe (o custom escrow) acepta transacciones **solo** desde la dirección que deriva el tECDSA del canister. Sin firmantes humanos. Esto convierte la custodia de "empresa controla fondos" a "protocolo algorítmico controla fondos".

### Mitigante 3: Wallet de tránsito — no de holding

El USDC **no debería permanecer** en el EVM wallet más de los segundos que tarda el webhook de Koywe → verificación tECDSA → mint de ckUSDC. Es un tránsito casi instantáneo, no un holding sostenido.

**Argumento legal:** difícilmente se puede caracterizar como "administración de fondos" un activo que pasa en tránsito automático durante segundos sin intervención humana.

### Mitigante 4: Governance con timelock (el más poderoso para el regulador)

Ver §5 abajo. Si ningún upgrade del canister puede ejecutarse sin aprobación de governance + espera de 48h, el team GreyValley no tiene control unilateral sobre los fondos. Este argumento es el más comprensible para un regulador no técnico.

---

## 4. Arquitectura EVM Recomendada — Decision Record

### Opción A: EOA controlada por tECDSA (estado actual del diseño)
- `bridge_canister` canister firma con tECDSA → dirección EOA en EVM
- ✅ Simple de implementar
- ⚠️ Difícil de explicar a regulador
- ❌ Si el canister se puede upgradear sin governance, hay control unilateral

### Opción B: Gnosis Safe (signer = dirección tECDSA del canister)
- El Safe es el contrato que "tiene" el USDC
- El único signer es la dirección tECDSA del canister (no humanos)
- ✅ Más auditable — el Safe es un estándar conocido
- ✅ Un regulador puede ver las reglas del Safe on-chain en Etherscan
- ✅ Multisig futuro posible (agregar CMF como co-signer de emergencia)
- ⚠️ Requiere deploy de Safe en EVM + wiring

### Opción C: Custom EVM Escrow Contract (inmutable)
- Contrato custom con reglas hardcodeadas: "solo libera USDC al user si hay burn verificado de ckUSDC en ICP"
- ✅ El argumento más fuerte: el contrato es inmutable, nadie puede cambiar las reglas
- ✅ Cero discrecionalidad humana sobre los fondos
- ❌ Más trabajo de auditoría del contrato EVM
- ❌ Sin flexibilidad para ajustes futuros sin re-deploy

### **Recomendación: Opción B (Gnosis Safe) en V1 → Opción C en V2**

En V1, el Gnosis Safe es la arquitectura más defensible legalmente porque:
1. Es un estándar auditado y conocido por reguladores internacionales
2. Las reglas de quién puede firmar son transparentes on-chain
3. Se puede presentar a CMF como "Multi-sig institucional donde GreyValley NO puede mover fondos sin el protocolo ICP"
4. Futuro: agregar a CMF o un trustee independiente como 2-of-3 co-signer → abre puerta al sandbox sin perder la arquitectura

---

## 5. Governance como Mitigante Regulatorio — Activar el Canister

### El argumento central

Si el equipo GreyValley puede upgradear el `bridge_canister` canister sin governance approval, tienen control unilateral de facto sobre el USDC en el EVM wallet. Un regulador sigue ese hilo directamente.

**La solución:** el `governance` canister (`src/governance/main.mo`, 71 líneas, **deployado en mainnet desde 2026-08-13** — `ynmo7-taaaa-aaaah-quzjq-cai`, status `Development`) debe **activarse** (a `#Live`/`#Restricted`) **antes de ir a producción con fondos reales** y debe tener control sobre los upgrades del `bridge_canister`. El deploy ya ocurrió; lo que sigue pendiente es la activación de status y la transferencia de controller descrita en los pasos de abajo.

### Modelo de governance recomendado (founder-led con DAO oversight)

```
Founder/Owner → crea Propuesta (texto + código de upgrade)
    ↓
PXRM Stakers → votan (weight proporcional a PXRM stakeado × tiempo)
    ↓
Si aprobada (quórum mínimo) → Timelock de 48h
    ↓
Después del timelock → el governance canister ejecuta el upgrade
```

**Por qué funciona para el regulador:**
- El founder propone pero no puede ejecutar unilateralmente
- El upgrade de `bridge_canister` (el canister que controla el EVM wallet) requiere aprobación de la comunidad
- El timelock de 48h da tiempo para que cualquier actor detecte propuestas maliciosas
- Esto rompe la cadena "GreyValley → control unilateral → fondos de usuarios"

### Governance como el founder lo quiere

El founder actúa como **Promotor de Propuestas** (Proposal Promoter), no como controlador ejecutivo:

| Acción | Quien la hace | Requiere governance |
|--------|--------------|---------------------|
| Cambiar fee split % | Founder (propone) | ✅ Sí — voto PXRM stakers |
| Upgrade de `bridge_canister` | Founder (propone) | ✅ Sí — voto + timelock 48h |
| Activar/desactivar feature flags | Founder (propone) | ✅ Sí |
| Respuesta a incidente activo | Founder (emergency) | ⚠️ Solo con admin key + timelock corto (6h) |
| Agregar Genesis Founder a whitelist | Founder (propone) | ✅ Sí |

**El modelo no es "DAO puro"** — el founder tiene peso de voto significativo (por su PXRM stakeado) y propone los temas. Pero no puede ejecutar cambios sobre fondos de usuarios sin que la comunidad lo valide. Esto es lo que el regulador necesita ver.

### Activación del governance canister — pasos

1. ~~Deploy `governance` canister en mainnet~~ ✅ hecho 2026-08-13
2. Transferir control del `bridge_canister` canister al `governance` canister (governance pasa a ser el controller) — pendiente
3. ~~Wiring en frontend: página `/governance` activa~~ ✅ hecho 2026-08-13, `GovernancePage.tsx` real
4. El founder hace la primera propuesta: "Activar bridge_canister en producción con Safe EVM" — pendiente
5. Los primeros Genesis Founders votan → aprobado → el canister se activa — pendiente

---

## 6. Estrategia Sandbox CMF — Hoja de Ruta

### ¿Qué es el sandbox de la Ley 21.521?

El Art. 90 y ss. de la Ley 21.521 permite a la CMF autorizar la operación temporal de modelos de negocio innovadores bajo supervisión, por hasta 24 meses, con restricciones de escala (máximos de clientes, volumen, etc.) mientras se determina si necesitan regulación permanente.

**Es la vía más inteligente para GreyValley:** operas legalmente, con cobertura regulatoria real, sin necesitar el proceso completo de autorización que puede tomar años.

### Prerrequisitos para aplicar al sandbox

| Requisito | Estado GreyValley | Acción |
|-----------|--------------|--------|
| Constitución como entidad legal en Chile | ❓ Verificar | Constituir SpA chilena si no existe |
| Descripción técnica del modelo de negocio | ✅ VAELIX_ODL_MECHANICS.md | Adaptar a formato CMF |
| Plan AML/CFT | ⚠️ Delegado a Koywe | Documentar el modelo de delegación |
| Capital mínimo operacional | ❓ Verificar | Determinar monto requerido para el sandbox |
| Manejo de quejas y SERNAC | ❌ No existe | Diseñar protocolo |
| Governance documentada | ⚠️ Canister deployado (2026-08-13), status `Development` | **Activar governance antes de aplicar** |
| Auditoría de smart contracts | ❌ No iniciada | Contratar auditor externo (Certora, OpenZeppelin) |

### Timeline estimado

```
Mes 1-2: Constituir SpA + governance activa + doc técnica CMF
Mes 3:   Reunión informal con CMF (recomendado antes de aplicar formalmente)
Mes 4:   Aplicación formal al sandbox
Mes 5-6: Evaluación CMF (2-3 meses típico)
Mes 7:   Inicio sandbox (24 meses de operación supervisada)
Mes 31+: Autorización permanente o cierre regulatorio
```

### Argumento central para CMF

> "GreyValley no custodia fondos de usuarios en el sentido tradicional. Los pesos chilenos permanecen siempre en Koywe (PSAV regulado). El USDC es tránsito algorítmico controlado por un protocolo distribuido en ICP con governance on-chain. Los usuarios reciben ckUSDC directamente en su wallet soberana. No hay discrecionalidad humana sobre los fondos en ningún punto del flujo."

---

## 7. Tabla de Riesgos Regulatorios y Estado

| Riesgo | Severidad | Mitigante técnico | Estado |
|--------|-----------|-------------------|--------|
| Governance unilateral del founder | 🔴 **CRÍTICO** | Activar `governance` canister + ceder control de `bridge_canister` | ⏸️ **BLOQUEANTE** — no operar con fondos reales sin esto |
| Custodia de USDC en EVM wallet | 🔴 Alto | tECDSA + Gnosis Safe + governance timelocks | ⚠️ EOA activa, Safe pendiente |
| Framing de comunicación incorrecto | 🔴 Alto | Guía de copy "Software Público" (ver §8) | ❌ No formalizado |
| Sin registro como PSAV ante CMF | 🟡 Medio | Sandbox CMF application | ❌ No iniciado |
| sCLP requiere autorización BC | 🔴 Alto | Feature flag `FEATURE_SCLP_BRIDGE=false` | ✅ Desactivado en V1 |
| AML/CFT sin proceso propio | 🟡 Medio | Delegado a Koywe (KYC/KYB) — documentar | ⚠️ No documentado formalmente |
| vUSD como "valor mobiliario" | 🟡 Medio | Colateral es del propio usuario (no pooling externo) | 🔍 Analizar |
| Auditoría de contratos | 🟡 Medio | Contratar auditor antes del sandbox | ❌ No iniciado |

---

## 8. Argumento "Software Público" — Guía de Comunicación CMF

### El principio

CMF puede ignorar un argumento técnico pero no puede ignorar un patrón de comunicación: si la plataforma se promueve como "un servicio financiero que te da rendimiento", el regulador lo trata como tal aunque técnicamente sea otra cosa. El framing de comunicación es tan importante como la arquitectura.

### Lo que GreyValley ES (y cómo decirlo)

> "GreyValley es una interfaz de código abierto que facilita la interacción con protocolos DeFi desplegados en Internet Computer Protocol. Los contratos inteligentes del protocolo son públicos, auditables y operan sin intervención de GreyValley una vez desplegados. Los servicios de conversión entre pesos chilenos y activos digitales son provistos exclusivamente por Koywe SpA, entidad regulada bajo la Ley 21.521."

### Copy PROHIBIDO (activa riesgo regulatorio)

| ❌ NO decir | ✅ Decir en cambio |
|-------------|------------------|
| "Deposita con GreyValley" | "Interactúa con el protocolo usando GreyValley" |
| "GreyValley te da un X% de rendimiento" | "El protocolo genera rendimiento de estas fuentes: [...]" |
| "Tus fondos con GreyValley" | "Tus activos en tu wallet, gestionados on-chain" |
| "Compra/vende a través de GreyValley" | "Koywe (tercero regulado) provee la conversión CLP↔USDC" |
| "GreyValley garantiza el valor del ckUSDC" | "ckUSDC está respaldado 1:1 por USDC en Ethereum, verificable por ICP/NNS" |
| "Tu dinero está seguro con nosotros" | "Los activos on-chain son custodiados por el protocolo ICP, no por GreyValley" |

### Los tres pilares del argumento "Software Público"

**Pilar 1 — Open source:** el código de los canisters de GreyValley es público en GitHub. Cualquier desarrollador puede auditarlo, forkear el protocolo o desplegar su propia instancia.

**Pilar 2 — Sin discrecionalidad:** una vez activado el governance canister (§5), ningún humano puede cambiar unilateralmente el comportamiento del protocolo con respecto a los fondos de usuarios. El upgrade requiere voto de la comunidad + timelock 48h.

**Pilar 3 — Tercero regulado para fiat:** GreyValley nunca toca pesos chilenos. El on/off-ramp CLP es exclusivo de Koywe, que opera bajo Ley 21.521. GreyValley es la interfaz; Koywe es el servicio financiero.

---

## 9. Argumento Técnico Central: ckUSDC 1:1 via ICP/NNS — Explicación Profunda

> Este es el argumento técnico más poderoso disponible para GreyValley ante CMF. Requiere explicarlo bien porque los reguladores no conocen ICP. Esta sección está escrita para ser adaptada a la presentación ante CMF con asesoría legal.

### El problema que resuelve

Un regulador tiene una pregunta obvia: *"Si el usuario deposita tokens que representan dólares, ¿quién garantiza ese valor? ¿Quién tiene los dólares reales?"*

La respuesta tradicional en DeFi — *"el emisor del wrapped token promete mantener la reserva"* — implica confianza en una empresa. Es exactamente lo que un regulador trata como riesgo de contraparte.

**La respuesta de GreyValley es diferente y verificable:** ckUSDC no lo emite ni lo respalda GreyValley. Lo respalda el **NNS de ICP** mediante un protocolo criptográfico distribuido sin intervención humana de ninguna empresa.

---

### 9.1 Qué es el NNS (Network Nervous System)

El NNS es el sistema de gobernanza on-chain del Internet Computer Protocol. Es el "gobierno algorítmico" de la red ICP — no una empresa, no una fundación con poderes unilaterales, sino un sistema de reglas ejecutado por el protocolo.

**Estructura del NNS:**

```
ICP holders → stakean ICP en "neurons" (mínimo 1 ICP, mín. 6 meses de lock)
    ↓
Neurons → votan propuestas de gobernanza
    Weight de voto = ICP stakeado × multiplicador por dissolve delay
    ↓
Propuestas aprobadas → se ejecutan automáticamente on-chain
    (sin intervención humana — el protocolo ejecuta el código de la propuesta)
```

**Escala actual:**
- ~500,000+ neurons activos en múltiples países y entidades
- DFINITY Foundation tiene una porción del voto, pero no mayoría absoluta — no puede actuar unilateralmente
- El NNS controla: upgrades del protocolo ICP, creación de subnets, despliegue y control de system canisters

**El punto clave:** El ckUSDC minter canister es un **system canister controlado por el NNS**. No por DFINITY Foundation. No por GreyValley. Por una DAO on-chain con 500K+ participantes. Ni GreyValley ni nadie puede modificar las reglas de cómo se emite o quema ckUSDC sin aprobación del NNS.

---

### 9.2 Qué es un "chain-key token" — diferencia con wrapped tokens

ckUSDC no es un "wrapped" token en el sentido tradicional. La diferencia es técnica y legalmente relevante:

| Característica | USDC Bridgeado tradicional (ej. Multichain/Synapse) | ckUSDC (ICP chain-key) |
|----------------|-----------------------------------------------------|------------------------|
| **Quién custodia el USDC en Ethereum** | Una empresa o multisig con firmantes humanos | Dirección EVM controlada por threshold signatures de 28+ nodos ICP independientes |
| **¿Puede una empresa mover el USDC?** | Sí — quien tiene las llaves, controla | No — se necesita >2/3 de los nodos del subnet para firmar |
| **Auditoría de reserva** | Attestation o auditoría de la empresa emisora | On-chain en Etherscan — verificable por cualquiera en tiempo real |
| **Riesgo de contraparte** | Quiebra o hack del emisor | Ninguna empresa — el protocolo ICP |
| **Quién controla las reglas de mint/burn** | El equipo del proyecto emisor | El NNS (DAO con 500K+ neurons) |
| **¿Puede GreyValley cambiar esto?** | N/A | **No** — GreyValley solo usa ckUSDC, no lo emite ni controla |

**Para CMF:** La distinción entre "wrapped token emitido por una empresa" y "chain-key token emitido por un protocolo descentralizado" es la diferencia entre riesgo de contraparte y riesgo de protocolo. CMF puede regular el primero; el segundo es análogo a como el riesgo de Ethereum como protocolo escapa a la regulación nacional.

---

### 9.3 Cómo funciona la custodia 1:1 — el mecanismo técnico

#### Paso 1: Generación de la dirección Ethereum de custodia

Cuando el NNS aprobó la creación de ckUSDC (Proposal NNS-XXXXX, verificable on-chain), generó una **dirección Ethereum** usando **Threshold ECDSA (tECDSA)**:

```
Threshold ECDSA — cómo funciona:

28 nodos del subnet "pzp6e" de ICP (nodos operados por entidades distintas en
distintas jurisdicciones) ejecutan un protocolo de generación de clave distribuida (DKG):

→ Cada nodo genera un "share" de la clave privada
→ Ningún nodo conoce la clave completa — solo su fragmento
→ Para firmar una tx Ethereum, ≥ 19 de 28 nodos deben colaborar
→ El resultado es una firma ECDSA válida sin que ningún nodo haya tenido la clave completa

Dirección Ethereum resultante: 0xA17a8883dA1abd57c690DF9Ebf58fD551d76042e
(USDC en esta dirección = backing de TODO el ckUSDC en circulación en ICP)
```

**Lo que esto implica legalmente:**
- DFINITY Foundation no puede mover ese USDC unilateralmente
- GreyValley no puede mover ese USDC — ni siquiera tiene acceso al sistema
- Un hack de cualquier empresa involucrada en ICP no compromete los fondos
- Comprometerlo requeriría comprometer >1/3 de los nodos del subnet simultáneamente — operados en múltiples países y proveedores de cloud

#### Paso 2: Verificación del depósito mediante HTTPS Outcalls desde el consenso

Cuando un usuario deposita USDC en la dirección de custodia, el ckUSDC minter canister (NNS-controlado) necesita verificar ese depósito ANTES de mintear ckUSDC. Aquí está la parte técnicamente más importante para el argumento CMF:

**¿Cómo verifica ICP que el USDC llegó a Ethereum?**

```
El proceso de verificación (ejecutado por el NNS minter canister):

1. El ckUSDC minter canister inicia una HTTPS Outcall hacia proveedores de
   RPC de Ethereum (Cloudflare Ethereum Gateway, Alchemy, etc.)

2. TODOS LOS NODOS del subnet (28 nodos independientes) ejecutan
   INDEPENDIENTEMENTE la misma consulta HTTP al RPC de Ethereum:
   
   GET eth_getBalance(0xA17a8883..., "latest")
   o
   GET eth_getLogs(filter: {address: USDC_contract, event: Transfer, to: 0xA17a...})

3. Cada nodo recibe su respuesta del RPC y la comparte con el grupo

4. El runtime de ICP aplica la función de canonicalización "transform":
   elimina timestamps, request IDs y cualquier campo variable de la respuesta
   dejando solo el contenido estable (el balance de USDC)

5. Los 28 nodos comparan sus respuestas canonicalizadas:
   → Si ≥ 19 nodos (>2/3) reportan el mismo balance: ese balance es verdad
   → Si hay discrepancia: el nodo divergente queda en minoría, es ignorado

6. Solo cuando el consenso de >2/3 de los nodos confirma el depósito,
   el canister minta el ckUSDC al principal del usuario
```

**Por qué esto NO es un oracle centralizado:**

| Oracle centralizado (Chainlink, etc.) | HTTPS Outcalls de ICP |
|--------------------------------------|-----------------------|
| Una empresa oracle consulta Ethereum y firma el resultado | Los 28 nodos del subnet consultan Ethereum en paralelo e independientemente |
| Si el oracle falla o miente → el sistema falla | Un nodo malicioso no puede afectar el resultado — necesita corromper >1/3 del subnet |
| Hay un único punto de falla | No hay punto único de falla |
| El resultado viene firmado por la empresa oracle | El resultado viene del consenso del protocolo (BLS threshold signatures del subnet) |
| Requiere confiar en la empresa oracle | Requiere confiar en que >2/3 de los 28 nodos no estén colludidos |

**Argumento para CMF:** La verificación de que existe el USDC real en Ethereum no la hace GreyValley ni ninguna empresa. La hace el consenso distribuido de 28 nodos independientes de ICP — matemáticamente, es la misma garantía que el consenso de la blockchain de Ethereum mismo para sus transacciones.

#### Paso 3: El mint de ckUSDC

Una vez verificado el depósito por consenso, el NNS minter canister ejecuta:

```motoko
// Código del NNS ckUSDC minter canister (público, auditado por DFINITY)
// GreyValley no controla ni puede modificar este código

ckusdc_ledger.icrc1_mint({
  to     = { owner = user_principal };    // usuario que depositó USDC
  amount = deposited_usdc_in_e6s;        // 1:1 con el USDC en Ethereum (6 decimales)
  memo   = ?ethereum_tx_hash;            // prueba on-chain del depósito
})
```

El ledger canister de ckUSDC (`xevnm-gaaaa-aaaar-qafnq-cai`) registra el saldo. Este ledger es **completamente público** — cualquier persona puede consultar:
- El total supply de ckUSDC en ICP
- Que ese total supply coincide con el USDC en `0xA17a8883...` en Etherscan
- El historial completo de cada mint y burn

#### Paso 4: Redención (burn ckUSDC → USDC en Ethereum)

```
Usuario → llama ckUSDC minter canister: burn(amount, destino_evm)
    ↓
Minter canister → reduce supply en el ledger ckUSDC
    ↓
Minter canister → construye tx Ethereum: transferir amount USDC desde 0xA17a... a destino_evm
    ↓
IC.sign_with_ecdsa() → los 28 nodos del subnet colaboran para generar la firma
    ↓
Tx firmada → transmitida a Ethereum via HTTPS Outcall
    ↓
Usuario recibe USDC en su wallet Ethereum
```

**Invariante verificable en todo momento:**
```
Total Supply ckUSDC en ICP = USDC en dirección 0xA17a8883... en Ethereum
```

---

### 9.4 Por qué GreyValley queda fuera de esta cadena de custodia

GreyValley usa ckUSDC como token nativo para los vaults. Pero:

1. **No emite ckUSDC** — el minter canister es del NNS, GreyValley no tiene acceso ni control
2. **No puede modificar las reglas** — el minter canister requiere propuesta NNS aprobada para cualquier cambio
3. **No puede acceder al USDC subyacente** — la dirección Ethereum está bajo threshold signatures de 28 nodos independientes
4. **Si GreyValley desaparece mañana**, los usuarios siguen pudiendo:
   - Quemar ckUSDC via el minter canister del NNS directamente
   - Recuperar USDC en Ethereum
   - Retirar sus fondos de los vaults on-chain (los canisters de vault siguen funcionando)

---

### 9.5 El argumento completo para CMF (versión no técnica, para el abogado)

> "Los usuarios de GreyValley que depositan ckUSDC están interactuando con el protocolo Internet Computer (ICP) —  no con GreyValley como custodio. Cada unidad de ckUSDC representa un USDC real depositado en una dirección de la red Ethereum cuya clave privada no existe completa en ningún servidor: está distribuida matemáticamente entre 28 nodos independientes de ICP operados por distintas entidades en distintas jurisdicciones. Las reglas de emisión y redención de ckUSDC están codificadas en un canister controlado por el NNS de ICP — una DAO con más de 500.000 participantes de votación — que ni GreyValley ni DFINITY Foundation pueden modificar unilateralmente.
>
> La verificación de que los USDC reales existen en Ethereum la realiza el consenso de esos mismos 28 nodos independientes mediante consultas HTTP paralelas al blockchain de Ethereum — sin ningún oracle centralizado de por medio. El total supply de ckUSDC en ICP siempre iguala el USDC en la dirección de custodia Ethereum, y esta igualdad es verificable on-chain por cualquier persona, en tiempo real, sin confiar en GreyValley ni en ninguna empresa.
>
> GreyValley no custodia ckUSDC en ningún sentido jurídico o técnico. GreyValley es la interfaz; el NNS de ICP es el custodio algorítmico. La distinción es análoga a la diferencia entre quien usa internet y quien opera la infraestructura TCP/IP."

---

### 9.6 Dónde verificar — links para CMF

| Qué verificar | Dónde |
|---------------|-------|
| USDC en dirección Ethereum de custodia ICP | Etherscan: `0xA17a8883dA1abd57c690DF9Ebf58fD551d76042e` |
| Total supply de ckUSDC en ICP | IC Dashboard → Token Ledgers → `xevnm-gaaaa-aaaar-qafnq-cai` → circulating supply |
| NNS propuesta que creó ckUSDC | NNS dApp (`nns.ic0.app`) → Governance → Proposals |
| Código del ckUSDC minter canister | DFINITY GitHub: `ic/rs/ethereum/cketh/minter` (misma arquitectura para ckUSDC) |
| Nodos del subnet que controla el threshold | IC Dashboard → Subnets → Subnet `pzp6e` |
| Equivalencia supply/reserva en tiempo real | IC Dashboard compara automáticamente |

---

## 10. Governance — Acción Inmediata Requerida

El riesgo más urgente (🔴 CRÍTICO en §7) es que el founder puede modificar canisters unilateralmente hoy. Sin governance activo, los argumentos de §8 y §9 son válidos para el ckUSDC del NNS pero **no para los canisters propios de GreyValley** (vaults, bridge_canister, fee_splitter).

**Pasos concretos — ver `INSTRUCCIONES_FOUNDER.md §13.1 item F` para comandos dfx exactos.**

Timeline mínimo antes de ir a producción con fondos de terceros:
1. ~~Deploy governance canister en mainnet~~ ✅ hecho 2026-08-13
2. Transferir controller de `bridge_canister` al governance canister — pendiente
3. Primera propuesta: "Activar bridge_canister en producción" (votada por Genesis Founders) — pendiente
4. ~~Activar la página `/governance` en el frontend~~ ✅ hecho 2026-08-13, `GovernancePage.tsx` real

---

---

## 11. Modelo de Amenazas de Seguridad — Red Team / Blue Team

> Este análisis fue generado como ejercicio de "piensa como un atacante" sobre el código real de GreyValley (2026-08-07). Documentado en VAELIX_REGULATORY.md porque la seguridad técnica del protocolo es parte del argumento regulatorio: CMF evaluará si los fondos de usuarios están protegidos no solo por arquitectura institucional sino por el código mismo.

### 11.1 Mapa de vectores por severidad

| # | Vector | Severidad | ¿Defendido en código? | Acción |
|---|--------|-----------|----------------------|--------|
| 1 | Compromiso de identidad dfx del founder | 🔴 Crítico | ❌ Solo por key management externo | Activar governance canister |
| 2 | Replay de paymentId tras upgrade de canister | 🔴 Crítico | ❌ **BUG CONFIRMADO** en `bridge/main.mo` | Fix inmediato (1 línea) |
| 3 | Governance takeover con PXRM | 🟡 Alto | ⚠️ Governance.mo no tiene votación real aún | Implementar módulo de votación antes de ceder control |
| 4 | Inflación artificial de TVL | 🟡 Alto | ⚠️ `maxDepositE8s = null` — sin cap activo | Activar cap + withdrawal cooldown |
| 5 | Arbitraje del swap fijo PXRM/ICP | 🟡 Medio | ⚠️ Cap 2%/24h implementado — insuficiente si PXRM muy barata | Freeze swap si precio < paridad |
| 6 | Reentrancy en `harvestVault` | 🟢 — | ✅ Protegido — `harvestedYieldPXRM` actualizado antes del `await` | Ninguna |
| 7 | Frontend injection / supply chain JS | 🟢 — | ✅ Frontend en canister ICP con certified assets | Ninguna |
| 8 | Rate limiting en bridge | 🟢 — | ✅ 20 calls/60s implementado | Ninguna |
| 9 | Llamadas anónimas | 🟢 — | ✅ Rechazadas en todos los métodos shared | Ninguna |

---

### 11.2 Bug crítico — replay attack en `bridge/main.mo`

**Descripción:** El mapa `processedPayments` que previene replay de `paymentId` es `var` (volátil), no `stable var`. Se borra en cada upgrade del canister.

```motoko
// CÓDIGO ACTUAL — VULNERABLE:
var processedPayments : Map.Map<Text, Bool> = Map.empty();

// FIX REQUERIDO:
stable var processedPayments : Map.Map<Text, Bool> = Map.empty();
// + añadir a preupgrade/postupgrade
```

**Escenario de ataque:**
1. Atacante completa un pago legítimo → obtiene un `paymentId` procesado
2. Espera a que el founder haga cualquier upgrade del canister (borra `processedPayments`)
3. Replantea la misma tx → el canister la acepta como nueva → doble pago

**Impacto:** recibir fondos dos veces por un único pago real. Con ODL activo y volumen significativo, este es el vector de extracción más concreto disponible sin comprometer la identidad del founder.

**Mitigante hasta que se parchee:** no hacer upgrades de `bridge_canister` mientras haya pagos recientes con menos de 48h de antigüedad.

---

### 11.3 Gap crítico — Governance código ≠ Governance spec

**Lo que dice la spec** (`VAELIX_REGULATORY.md §5`, `INSTRUCCIONES_FOUNDER.md §13.5`):
> "Propuesta on-chain + quórum de PXRM stakers + timelock 48h"

**Lo que existe en `src/governance/main.mo` hoy:**
```motoko
// governance/main.mo — solo esto:
public shared ({ caller }) func set_status(new_status : SystemStatus) : async () {
  assert(isOwner(caller));   // ← un solo owner, igual que el founder
  status := new_status;
};
// No hay: votación de tokens, quórum, timelock, propuestas, delegación
```

**Consecuencia:** transferir el control de `bridge_canister` al governance canister actual NO descentraliza nada — simplemente mueve el control de un single-owner principal a otro single-owner canister. Para que el argumento CMF ("comunidad PXRM controla el protocolo") sea verdadero, el governance canister necesita:

- [ ] Registro de PXRM stakers como votantes
- [ ] Creación de propuestas con texto visible
- [ ] Período de votación configurable (mínimo 48h)
- [ ] Quórum mínimo (ej. 10,000 PXRM stakeados)
- [ ] Ejecución automática on-chain al alcanzar quórum

**Prioridad:** implementar antes de presentar el argumento de governance a CMF o a inversionistas como mitigante.

---

### 11.4 Vector 1 — Identidad del founder y key management

**El ataque más probable no es técnico — es operacional:**

```
Atacante obtiene la private key del founder (dfx identity)
    ↓
dfx canister --network ic install backend --mode upgrade <wasm_malicioso>
    ↓
El wasm malicioso tiene una función drain(to: Principal) sin guard
    ↓
Todos los fondos en vault salen en una transacción
```

**Por qué el código no puede defenderse solo de esto:** el modelo de seguridad de ICP delega la protección de upgrades al controller. Si el controller es comprometido, el protocolo no puede distinguir al founder legítimo del atacante.

**Mitigantes operacionales (no en código, en comportamiento):**

| Medida | Estado | Prioridad |
|--------|--------|-----------|
| Hardware wallet (Ledger) para la identidad dfx de producción | ❌ No confirmado | 🔴 INMEDIATA |
| Identidad dfx de producción ≠ identidad dfx de desarrollo | ❌ Verificar | 🔴 INMEDIATA |
| 2FA en todos los servicios relacionados (GitHub, npm, CI/CD) | ❌ Verificar | Alta |
| Governance canister como segundo controller de bridge_canister | ❌ No deployado | Alta |
| Auditoría de dependencias npm del build pipeline | ❌ No realizado | Media |

**El hardware wallet es el mitigante más impactante y más fácil de implementar.** Un Ledger con la identidad de producción significa que cualquier `dfx canister install` requiere aprobación física en el dispositivo.

---

### 11.5 Lo que está bien cubierto — no necesita acción

**Reentrancy en `harvestVault`:** el diseño usa un running total (`total_yield - harvestedYieldPXRM`) en vez de resetear un timestamp. `harvestedYieldPXRM` se escribe en `positions` ANTES del `await icrc1_transfer`. Un segundo harvest concurrente calcula yield=0 y retorna inmediatamente. Este es el patrón correcto para Motoko.

**Frontend tampering:** el frontend de GreyValley es un canister ICP — no se puede reemplazar sin un upgrade firmado por el controller. Plug Wallet muestra el Principal real de destino antes de cada transacción, permitiendo que el usuario verifique. El modelo de certified assets de ICP garantiza integridad del frontend servido.

**Rate limiting y spam:** `bridge/main.mo` tiene `RATE_LIMIT_MAX_CALLS = 20` por 60s por caller, y `processedPayments` previene replay en condiciones normales (el bug §11.2 es solo en upgrades). `backend/main.mo` rechaza anónimos en todos los métodos shared.

---

### 11.6 Implicaciones regulatorias del threat model

Para CMF, el threat model tiene valor en dos direcciones:

**Argumento positivo:** el protocolo fue diseñado con mitigantes explícitos contra los vectores más comunes (reentrancy, spam, replay). Existe documentación interna de análisis de seguridad, lo cual es señal de madurez operacional.

**Riesgo de argumentar lo contrario:** si CMF pide "¿cómo protegen los fondos de usuarios?" y la respuesta honesta es "hay un bug de replay en el bridge no parcheado y el governance no tiene votación real", ese es exactamente el tipo de exposición que activa revisión de operación. Por eso el patch de §11.2 y la implementación de governance real (§11.3) son prerequisitos antes de cualquier contacto formal con CMF.

---

---

## §12 SFA-Sandbox — Infraestructura Open Finance Chile (Ley Fintech 21.521)

### 12.1 Qué es SFA-Sandbox

**SFA-Sandbox** (`sfasandbox.cl`) es el entorno de testing de open finance chileno, operado bajo **Ley Fintech 21.521** y las Normas de Carácter General (NCG) **502** y **514** de la CMF. Implementa el estándar internacional **FAPI 2.0** (Financial-grade API), el mismo que se usa en UK Open Banking y Brasil PIX.

Es el equivalente técnico de la "sandbox regulatoria" que los operadores de servicios de pago deben superar para obtener acreditación CMF. Cualquier empresa que quiera ser **PISP** (Payment Initiation Service Provider) o **AISP** (Account Information Service Provider) en Chile necesita operar contra estas APIs.

**Relevancia directa para GreyValley:** Koywe ya está acreditado como PISP bajo este ecosistema — lo que significa que cuando GreyValley usa Koywe para el on-ramp/off-ramp fiat, GreyValley hereda el cumplimiento FAPI 2.0 *sin esfuerzo propio*. En Fase 3, GreyValley SpA podría registrarse directamente como PISP+AISP, eliminando la comisión de Koywe en el lado fiat.

---

### 12.2 APIs disponibles y relevancia para GreyValley

| API | Endpoint | Qué hace | Relevancia GreyValley |
|-----|----------|----------|-------------------|
| **Consentimiento OAuth2** | `POST /api.php?action=consents` | Redirect flow para obtener consentimiento usuario | Base del flujo; Koywe lo maneja en Fase 1 |
| **Token OAuth2** | `POST /api.php?action=token` | Intercambio de código por JWT Bearer | Necesario para AISP y PISP |
| **CIBA (backchannel)** | `POST /api.php?action=ciba_auth` | Inicia autenticación sin redirect (push to móvil) | Alta relevancia: flujo sin abrir browser → ideal para app móvil GreyValley |
| **CIBA token poll** | `POST /api.php?action=ciba_token` | Polling hasta que usuario aprueba en su banco | Necesario si GreyValley implementa CIBA directamente |
| **PISP — iniciar pago** | `POST /api.php?action=payments` | Debita cuenta bancaria → TEF a cuenta destino | El corazón del on-ramp: convierte CLP bancario en flujo GreyValley |
| **PISP — estado pago** | `GET /api.php?action=payment_status&id=X` | Polling del estado de la transferencia | Confirmación on-chain solo cuando pago PISP confirmado |
| **NCG 514 estado** | `GET /api.php?action=channels_status` | Disponibilidad operacional de canales bancarios | GreyValley puede detectar que el sistema bancario está caído antes de iniciar on-ramp |
| **AISP — balances** | `GET /api.php?action=balances` | Saldo de cuentas del usuario (requiere JWT Bearer) | Dashboard de portfolio completo: saldo banco + saldo GreyValley en una pantalla |
| **AISP — transacciones** | `GET /api.php?action=transactions` | Historial 5 años del usuario (requiere JWT Bearer) | Análisis de flujos: detectar si usuario es importador recurrente |
| **Catálogo productos** | `GET /api.php?action=products` | Productos bancarios públicos, sin auth | Comparador de spreads / tasas para pitch comercial |
| **Sucursales/ATMs** | `GET /api.php?action=branches` | Geolocalización de puntos físicos | Bajo impacto para GreyValley |

**Headers requeridos en toda llamada:**
```
x-sandbox-key: <api-key-de-sfasandbox>
x-fapi-interaction-id: <UUID-único-por-request>
```

---

### 12.3 Motor de Caos (Chaos Engine)

SFA-Sandbox tiene un motor de caos activable que simula condiciones reales de fallo bancario:

| Modo caos | Efecto | Cómo afecta a GreyValley |
|-----------|--------|----------------------|
| **Latencia +3500ms** | Todo endpoint tarda 3.5s extra | El canister HTTPS Outcall timeout debe cubrir esto |
| **Colapso HTTP 500** | Endpoints devuelven error 500 aleatorio | GreyValley necesita retry con backoff exponencial |
| **CIBA pending infinito** | Auth nunca completa | Timeout de espera usuario configurable (ej: 5 min) |

**Uso recomendado:** antes de conectar Koywe en staging, correr los flujos de on-ramp contra el chaos engine. Si GreyValley sobrevive el modo caos sin dejar fondos en estado inconsistente, el sistema es robusto para producción.

También es útil para probar el **circuit breaker** del canister: si el banco está caído según NCG 514, el canister debe rechazar el depósito con mensaje claro antes de intentar el PISP.

---

### 12.4 ICP HTTPS Outcalls — Integración directa desde canister

**Arquitectura estándar (con backend server):**
```
App móvil → Backend Node.js → SFA API → banco
```

**Arquitectura ICP (sin backend):**
```
App móvil → Canister ICP → HTTPS Outcall → SFA API → banco
```

El canister llama directamente a `sfasandbox.cl/api.php` (o en producción, la API SFA real) mediante HTTPS Outcalls. Los 28 nodos del subnet hacen la llamada independientemente, canonicalizan la respuesta, y ≥19/28 deben acordar el resultado antes de que el canister lo procese.

**Implicación:** no existe un servidor backend Node.js que pueda ser comprometido, apagado, o que cause un punto de falla entre el canister y el sistema bancario. La lógica PISP vive *en el canister*.

**Limitación a considerar:** las respuestas HTTPS Outcalls deben ser determinísticas entre los 28 nodos. Si SFA devuelve `x-fapi-interaction-id` diferente en cada request, el canister debe usar `transform` para eliminar esos campos antes del consenso.

```motoko
// Ejemplo: transform para eliminar headers no-determinísticos
public query func transformSFAResponse(raw : Types.TransformArgs) : async Http.HttpResponsePayload {
    {
        status = raw.response.status;
        body = raw.response.body;
        headers = []; // eliminar headers con timestamps/IDs únicos
    }
};
```

---

### 12.5 Roadmap de integración SFA en GreyValley

| Fase | Actor | Qué se integra | Cómo |
|------|-------|---------------|------|
| **Fase 1 (actual)** | Koywe | PISP + OAuth2 + CIBA | Koywe ya acreditado; GreyValley solo llama API Koywe |
| **Fase 2** | GreyValley SpA | NCG 514 monitoring | Canister consulta `channels_status` antes de iniciar on-ramp |
| **Fase 2** | GreyValley SpA | AISP (lectura) | App muestra saldo bancario + saldo GreyValley en pantalla unificada |
| **Fase 3** | GreyValley SpA | PISP directo | Registro CMF como PISP → canister inicia pagos TEF sin Koywe |
| **Fase 3** | GreyValley SpA | CIBA backchannel | Flujo sin redirect → usuario aprueba desde su banco-app → canister confirma |

**Prerequisito para Fase 3:** GreyValley SpA debe ser entidad regulada (PSAV o similar) y superar el proceso de acreditación SFA con CMF. Requiere: entidad legal activa, AML/KYC implementado, auditoría de seguridad, depósito de garantía.

---

### 12.6 Relación con §6 — Argumento ante CMF

La existencia de SFA-Sandbox y el cumplimiento FAPI 2.0 de Koywe **fortalece el argumento regulatorio de GreyValley** ante CMF:

1. **No somos un sistema opaco:** todo el flujo fiat entra/sale por APIs bancarias reguladas (NCG 502/514), no por canales informales.
2. **Koywe como buffer regulado:** GreyValley no necesita acreditación propia en Fase 1/2 porque el PISP está del lado de Koywe (entidad regulada CMF).
3. **Arquitectura auditable:** los HTTPS Outcalls desde el canister a las APIs SFA son registrables en logs de consenso del subnet — cualquier auditor puede verificar qué llamadas se hicieron y cuándo.
4. **Roadmap de cumplimiento progresivo:** en lugar de "vamos a violar la ley y pedir perdón después", GreyValley presenta un camino Fase 1→3 donde en cada etapa existe un actor regulado responsable del fiat.

**Riesgo a gestionar:** si CMF cambia NCG 502/514 o agrega requisitos FAPI 2.1, GreyValley hereda ese cambio a través de Koywe en Fase 1. El canister solo llama a la API de Koywe — Koywe absorbe los cambios regulatorios. Esto es una ventaja arquitectónica, no una deuda técnica.

---

### 12.7 Acción inmediata

| Prioridad | Acción | Responsable |
|-----------|--------|-------------|
| **Alta** | Obtener `x-sandbox-key` de sfasandbox.cl y probar flujo PISP completo en staging contra chaos engine | Founder / CTO |
| **Alta** | Documentar en INSTRUCCIONES_FOUNDER.md el flujo de confirmación on-chain solo después de `payment_status = SETTLED` | Founder |
| **Media** | Evaluar si el canister bridge debe llamar `channels_status` como circuit breaker antes de aceptar depósitos | Arquitectura |
| **Media** | Analizar si CIBA backchannel es mejor UX que redirect OAuth2 para el flujo mobile de on-ramp | Producto |
| **Baja** | Iniciar exploración legal para registro GreyValley SpA como AISP (menos restrictivo que PISP) | Legal |

---

*GreyValley Regulatory Reference | Actualizado: 2026-08-09 | Próxima revisión: antes de aplicación sandbox CMF*
