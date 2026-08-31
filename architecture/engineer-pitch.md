# GREYVALLEY — ICP explicado al ingeniero/informático chileno
> Documento de referencia para conversaciones técnicas con IT professionals, ingenieros de sistemas, y equipos de infraestructura.
> Generado: 2026-08-08 | No es documentación oficial de DFINITY.

---

> ⚠️⚠️ **CORRECCIÓN REAL 2026-08-30 — Koywe NO es un partner activo.**
> Todo lo que este documento describe sobre Koywe (integración, KYC/AML
> delegado, "Fase 1 actual", acreditación PISP, etc.) es el **diseño de
> estrategia** para cuando exista un partner fiat así — hoy no existe
> ninguna relación real con Koywe, ni siquiera contacto comercial. El
> webhook técnico del lado GreyValley está construido y listo, pero no
> apunta a ningún partner confirmado todavía. Leer las secciones de abajo
> como plan de referencia, no como estado operativo actual.

## TABLA COMPARATIVA PRINCIPAL
### Node.js / PostgreSQL / AWS vs ICP — qué hace cada cosa y cómo se llama en ICP

| Stack tradicional | Qué hace | Equivalente ICP | Nombre técnico ICP |
|---|---|---|---|
| Node.js + Express | Servidor HTTP, procesa requests | **Canister** (módulo WebAssembly) | Update call (escribe estado) / Query call (solo lectura) |
| PostgreSQL / MySQL | Almacena datos de forma persistente | **Variables stable** dentro del canister | Orthogonal persistence (el estado sobrevive upgrades automáticamente) |
| Redis | Cache en memoria, lecturas rápidas | **Query calls** — no escriben estado, responden al instante | Query calls (no requieren consenso — solo un nodo responde) |
| AWS S3 | Almacenamiento de archivos/assets | **Asset canister** | Asset canister (hasta 64GB por subnet) |
| AWS API Gateway | Enruta requests HTTP al backend | **Boundary node + IC Agent** | Boundary nodes (los "puertos de entrada" de la red) |
| Nginx / Load balancer | Distribuye tráfico | **Boundary nodes** | Boundary nodes (gestionados por la red, no por GreyValley) |
| Auth0 / Firebase Auth | Autenticación de usuarios | **Internet Identity + Principal ID** | Principal (cada wallet/usuario tiene una identidad criptográfica única) |
| Chainlink / Pyth | Oracle externo de datos | **HTTPS Outcalls con consenso** | HTTPS Outcalls (los 28 nodos hacen la misma petición HTTP y votan por el resultado) |
| Wormhole / Axelar / LayerZero | Bridge entre blockchains | **Chain Fusion / tECDSA** | Chain Key Technology (no hay bridge — ICP firma nativamente en Ethereum) |
| Docker container | Entorno aislado de ejecución | **Canister** (WebAssembly sandbox) | WASM runtime (cada canister es un módulo WebAssembly aislado) |
| Pipeline CI/CD | Deploy del código | `dfx deploy <canister>` — **un comando** | dfx CLI |
| AWS Lambda | Función serverless | **Update call** a un canister | Update call (ejecutado en consenso por los 28 nodos) |
| Cron job / scheduler | Tareas automáticas periódicas | **Canister heartbeat / Timers** | `ic_cdk::timer` en Rust, `recurringTimer` en Motoko |
| SQS / RabbitMQ | Cola de mensajes async | **Inter-canister calls** | Inter-canister calls (un canister llama a otro canister directamente) |
| JWT / sesión | Gestión de sesión del usuario | **Delegated identity** | Internet Identity delegation (válido por tiempo y scope definido) |
| Variables de entorno (.env) | Configuración y secretos | ⚠️ Gap real: no hay secret manager nativo | Los secretos se guardan en estado del canister (solo el controller accede) |
| Monitoreo (Datadog, CloudWatch) | Observabilidad del sistema | **IC Dashboard + canister logs** | `dfx canister logs` + ic.rocks / icscan.io |
| DNS | Nombres de dominio | **Dominios propios** apuntando al boundary node | Custom domain (ICP lo resuelve, pero el DNS sigue siendo externo) |

---

## CONCEPTOS CLAVE — PREGUNTA POR PREGUNTA

### "Mismo proceso" — qué significa exactamente

En arquitectura tradicional, una app tiene **procesos separados que se comunican por red**:
- El servidor Node.js es un proceso.
- PostgreSQL es otro proceso, en otra máquina, que Node.js consulta por TCP.
- Redis es otro proceso.
- El API Gateway es otro proceso.
- Todos se "hablan" por HTTP o TCP — con latencia, con puntos de falla, con credenciales que manejar.

En ICP, **un canister es un único módulo WebAssembly** que contiene:
- La lógica de negocio (el código)
- El almacenamiento persistente (las variables stable)
- La interfaz de entrada (los métodos públicos que equivalen a endpoints HTTP)

Todo esto ejecuta en **el mismo módulo**, sin red interna. Cuando `depositVault` llama a `calculateYield`, no hay un request HTTP — es una llamada de función dentro del mismo módulo. Esto es lo que significa "mismo proceso": no hay separación entre tu servidor, tu base de datos, y tu lógica.

**La diferencia práctica:** no hay latencia interna, no hay serialización/deserialización entre capas, no hay credenciales de base de datos que rotar.

---

### Por qué nadie puede apagarlo — ni Dfinity

Un canister corre en una **subnet**: un grupo de 28 computadoras físicas operadas por **node providers independientes** (empresas, universidades, data centers en distintos países). Para que el canister funcione, solo necesita que ≥19/28 nodos estén operativos.

**¿Puede Dfinity apagarlo?**

Dfinity no controla los 28 nodos directamente — son operadores independientes que firmaron un contrato con la red. Para "apagar" un canister específico que Dfinity no controla, necesitarían:
1. Convencer a ≥19 operadores de nodos de negarse a procesar ese canister, O
2. Pasar una propuesta por el NNS (el DAO de ICP con ~500K+ neuronas de miles de holders independientes) para intervenir ese canister.

Lo segundo es posible en teoría pero requiere mayoría del NNS — que incluye a competidores, inversionistas, y usuarios que no tienen incentivo en cerrar canisters arbitrariamente.

**El caso práctico para GreyValley:** si en algún momento los canisters de GreyValley no tienen un controller humano (sino el governance canister como controller), ni GreyValley SpA ni Dfinity pueden modificarlos unilateralmente. Eso es exactamente lo que hace el argumento CMF: "el protocolo no lo controla nadie, lo controlan las reglas del código."

**La honestidad:** hoy, los canisters de GreyValley SÍ tienen un controller humano (el founder). Eso es el punto crítico regulatorio pendiente de resolver — el objetivo es migrar el control al governance canister.

---

### APIs externas "nativas" — qué significa nativo

**Nativo** = el protocolo lo soporta sin instalar nada, sin depender de un tercero.

En Ethereum, si quieres que un smart contract obtenga el precio del USD/CLP, necesitas un oracle externo (Chainlink). El smart contract no puede hacer una petición HTTP por sí solo — Ethereum fue diseñado para ser determinista y aislado del mundo exterior.

En ICP, **HTTPS Outcalls** es una feature del protocolo: cualquier canister puede hacer una petición HTTPS a cualquier URL pública del mundo. Internamente:
1. Los 28 nodos de la subnet hacen **la misma petición HTTP independientemente**.
2. Cada nodo obtiene su respuesta.
3. La respuesta se "canonicaliza" (se eliminan partes que varían: timestamps, headers de servidor, etc.).
4. Si ≥19/28 nodos obtienen el mismo resultado canónico → la respuesta se acepta como válida.
5. El canister recibe el dato como si hubiera hecho un fetch normal.

**¿"Cualquier" API?** Cualquier endpoint HTTPS público. Koywe, Circle, precios de mercado, la API del SII, un ERP empresarial, un banco con API REST. Si tiene HTTPS, ICP puede llamarlo.

**Limitación real:** APIs que requieren autenticación (API key). La key se guarda en el estado del canister (no en .env), solo accesible por el controller. No es perfectamente secreto — si alguien gana acceso al controller, puede leerla. No hay un "secret manager" nativo como AWS Secrets Manager.

---

### Sin middlewares — cuáles y por qué

Los "middlewares" que ICP elimina:

**Bridges de blockchain (Wormhole, Axelar, LayerZero):**
Estos existen porque Ethereum no puede "hablar" con Solana directamente — necesitas un servicio externo que bloquea tokens en una cadena y los libera en otra. ICP usa tECDSA para firmar transacciones directamente en Ethereum sin intermediario. El ckUSDC no usa Wormhole — usa la propia criptografía de ICP.

**Oracles (Chainlink):**
Existen porque los smart contracts EVM no pueden hacer peticiones HTTP. Los HTTPS Outcalls eliminan esta necesidad: el canister consulta directamente la fuente de datos.

**API Gateways (Kong, AWS API Gateway):**
Existen para enrutar, autenticar, y rate-limitar peticiones al backend. En ICP, los boundary nodes hacen ese rol. La autenticación se hace por criptografía (Principal ID) dentro del canister — no necesitas configurar un gateway separado.

**Credenciales que desaparecen:**
- `DATABASE_URL=postgresql://user:pass@host:5432/db` → no existe en ICP
- `REDIS_URL=redis://...` → no existe
- `AWS_ACCESS_KEY_ID` → no existe
- `JWT_SECRET` → no existe (la autenticación es criptográfica, no por token compartido)

Lo que SÍ existe: el **controller principal** (el dfx identity del founder). Eso es LA credencial crítica de ICP — ver `INSTRUCCIONES_FOUNDER.md §13`.

---

### Todo en un canister — el ledger PXRM en el corazón de GreyValley

> Nota histórica: el ledger PXRM original quedó comprometido (minting key perdida, ~20M supply
> fantasma) y fue reemplazado el 2026-08-09 por el ID de abajo. Detalle en
> `VAELIX_MASTER_STATE.md` §29.

El **PXRM ledger canister** (canister ID: `q7nmw-diaaa-aaaah-quy4a-cai`) es un canister que implementa el estándar ICRC-1. Contiene:
- Todos los balances PXRM de todos los holders (en memoria stable)
- La lógica de transfer, approve, transfer_from
- El log de transacciones (ICRC-3)
- Las reglas de quién puede mintear (solo el minter designado)

No hay base de datos PostgreSQL detrás. No hay servidor Node.js manejando las transferencias. El ledger IS el banco. Es el mismo modelo que el PXRM ledger de ckUSDC (canister ID: `xevnm-gaaaa-aaaar-qafnq-cai`) — que gestiona TODOS los ckUSDC de todos los usuarios de ICP.

**El backend de GreyValley** (`main.mo`) es el canister principal que orquesta todo:
- Recibe depósitos → llama al ledger ICRC-2 para transferir tokens al canister
- Calcula yield → lógica puramente matemática in-canister
- Paga rendimiento → llama al ledger PXRM para transferir al usuario
- Consulta precios → HTTPS Outcall a la API de precios
- Verifica governance → inter-canister call al governance canister

Todo esto ocurre en canister-to-canister calls, sin ningún servidor externo coordinando.

---

### SWIFT — complicaciones específicas y cómo ICP las resuelve

| Problema SWIFT | Qué pasa exactamente | Solución ICP / GreyValley |
|---|---|---|
| **Correspondent banking** | Tu banco chileno no tiene cuenta directa en el banco del beneficiario. Usa 2–4 bancos intermediarios, cada uno cobra $10–30 USD. | ckUSDC va directo: wallet → wallet. Sin intermediarios. |
| **Tiempo de liquidación** | T+1 a T+5 días hábiles. Fin de semana → hasta el lunes. | Finalidad en segundos, 24/7/365. |
| **Spread cambiario** | El banco aplica un spread del 1–3% sobre el tipo de cambio interbancario. En $10.000 USD = $100–300 USD perdidos. | 0.10% flat sobre el monto, sin spread oculto. |
| **Opacidad en tránsito** | No sabes dónde está tu dinero mientras viaja por los corresponsales. | Todo on-chain: el hash de la transacción es inmediatamente visible. |
| **Riesgo de rechazo** | Un banco corresponsal puede rechazar la SWIFT por compliance sin explicación. Tu dinero vuelve 3–7 días después menos las comisiones. | Un smart contract acepta o rechaza determinísticamente. Si cumple las reglas del código, pasa. No hay criterio discrecional. |
| **Mínimos rentables** | Para $500 USD, una SWIFT de $40 USD + 2% spread = 12% del monto en costos. No tiene sentido para montos pequeños. | 0.10% funciona igual para $50 o $500.000. |
| **Bloqueo de fondos** | En algunos países (Argentina), el regulador puede congelar transferencias SWIFT. | No hay entidad con poder de congelar ckUSDC en tránsito (salvo que ICP entero sea atacado — improbable). |

**¿Por qué Koywe y no solo GreyValley?**

Koywe resuelve el problema del **primer y último kilómetro**: convertir CLP (que vive en el sistema bancario chileno) en ckUSDC (que vive en ICP). Esa conversión **todavía requiere el sistema bancario** — alguien tiene que recibir la transferencia CLP y emitir los ckUSDC.

GreyValley resuelve todo lo que viene **después**: el almacenamiento, el rendimiento, el envío internacional, la integración institucional. El SWIFT se evita en el tramo de mayor costo — el envío internacional — porque ese tramo ocurre 100% on-chain.

**Analogía:** Koywe es el puerto de entrada y salida. ICP es el sistema de transporte que opera entre puertos. El barco (ckUSDC) no usa rutas marítimas SWIFT — viaja instantáneamente por la red.

---

### "No viaja — representación inmediata — el banco no tiene tu dinero"

Cuando haces una transferencia bancaria tradicional:
1. Tu banco **débita** tu cuenta (el saldo baja en tu pantalla).
2. Envía un mensaje SWIFT al banco destino.
3. El dinero entra en el sistema de **corresponsales** — literalmente existe en la cuenta de tu banco en el banco corresponsal, que lo mueve a otro, que lo mueve a otro.
4. El banco destino **acredita** al beneficiario cuando recibe y procesa el SWIFT.
5. Durante 1–5 días, **el dinero no es de nadie** — está en el sistema bancario en tránsito.

Con ckUSDC:
- El USDC real (USD Coin emitido por Circle) **nunca se mueve** de la dirección Ethereum que controla ICP (`0xA17a8883dA1abd57c690DF9Ebf58fD551d76042e`). Ese USDC está quieto en Ethereum.
- Lo que cambia es el **registro en el ledger de ICP**: la línea que dice "este Principal tiene X ckUSDC" pasa a decir "ese Principal tiene X ckUSDC".
- Ese cambio de registro **es atómico** — ocurre en una sola transacción, en segundos. No hay estado intermedio donde el dinero "esté viajando".
- El banco nunca tuvo tu dinero: el USDC está custodiado por la criptografía de ICP (28 nodos con threshold ECDSA), no por una institución financiera que puede quebrar, ser hackeada, o congelarte la cuenta.

---

### Lógica de negocio "en el canister" — qué significa

**Lógica de negocio** = las reglas específicas de tu aplicación. En GreyValley:
- "Solo puedo depositar si el sistema está en estado Live"
- "El rendimiento se calcula como `bps/10000 × tiempo_transcurrido × precio_token`"
- "Solo el controller puede activar el sistema"
- "Si el usuario ya tiene posición abierta, sumar al balance existente"

En arquitectura tradicional, estas reglas viven en tu servidor Node.js. Si el servidor cae, las reglas no se aplican. Si hackean el servidor, pueden cambiar las reglas. Si el dueño quiere cambiarlas, edita el código y hace redeploy sin que nadie se entere.

**En el canister:**
- Las reglas están en el código WebAssembly deployado en la subnet.
- Todos los 28 nodos ejecutan exactamente ese mismo código.
- Para cambiar las reglas, hay que hacer un **upgrade del canister** — que queda registrado on-chain, visible para cualquiera.
- Las reglas aplican igual para cualquier llamada, de cualquier usuario, sin excepción — el código no tiene "modo administrador" oculto que saltee las reglas (salvo funciones admin explícitas en el código).

**Ejemplo real en GreyValley (`main.mo`):**
```motoko
public shared(msg) func depositVault(...) {
  requireLive();           // regla 1: el sistema debe estar Live
  requireAuth(msg.caller); // regla 2: el caller no puede ser anónimo
  // ... lógica de depósito
}
```
Estas dos líneas son lógica de negocio in-canister. Nadie puede llamar `depositVault` sin que el sistema esté Live y sin una identidad válida — sin importar cuánto lo intente.

---

### Todo por consenso — los 28 nodos y la criptografía

Cuando un usuario llama a `depositVault`:

1. El mensaje llega a cualquier nodo de la subnet.
2. Ese nodo lo propaga a todos los demás (gossip protocol).
3. Los 28 nodos acuerdan un **orden** para este mensaje (usando el protocolo de consenso BFT de ICP).
4. Todos los 28 nodos ejecutan **la misma instrucción WebAssembly** en el mismo orden.
5. Calculan el hash del nuevo estado del canister.
6. Si ≥19/28 tienen el mismo hash → la transacción es válida y el estado se actualiza.
7. El resultado vuelve al usuario firmado por el **threshold BLS signature** de la subnet.

**Por qué 28/19:** se llama tolerancia a fallas bizantinas (BFT). La red tolera que hasta f nodos sean maliciosos o caigan, donde n = 3f+1. Con 28 nodos: f = 9. Necesitas ≥19 correctos. Un atacante necesitaría controlar ≥10 nodos de la subnet simultáneamente — en nodos físicamente distribuidos en distintos países, operados por distintas empresas. El costo de ese ataque es astronómico.

**Para tECDSA (firmar en Ethereum):**
- La clave privada nunca existe completa en ningún lugar.
- Se genera con DKG (Distributed Key Generation): cada nodo recibe un "fragmento" matemático.
- Para firmar una transacción ETH, ≥19 nodos contribuyen su fragmento.
- Los fragmentos se combinan matemáticamente para producir una firma ECDSA válida.
- La dirección Ethereum resultante es real y usable — pero la clave para moverla no existe en ningún servidor.

---

### El "un comando" — `dfx deploy`

```bash
dfx deploy backend --network ic
```

Eso es todo. Ese comando:
1. Compila el código Motoko a WebAssembly (WASM).
2. Empaqueta el WASM.
3. Envía el WASM al canister en mainnet.
4. La subnet distribuye el WASM a los 28 nodos.
5. El canister queda actualizado.

Comparar con deployar Node.js + PostgreSQL + Redis a AWS:
- Crear EC2, configurar security groups, instalar Node.js
- Crear RDS, configurar VPC, crear tablas con migrations
- Crear ElastiCache, configurar connection pooling
- Configurar ALB, ACM certificate, Route 53
- Escribir Dockerfile, subir a ECR
- Crear ECS task definitions, configurar auto-scaling
- Configurar CloudWatch, alertas, backups de RDS
- Total: horas o días de DevOps

En ICP: `dfx deploy`. Ciclos (fracciones de centavo). Listo.

---

### Limitantes de los agentes IA en ICP

ICP puede hacer HTTPS Outcalls a APIs de IA (Claude, OpenAI, Gemini) — eso funciona bien. Las limitaciones vienen del **runtime del canister**:

| Limitación | Por qué existe | Solución en GreyValley |
|---|---|---|
| No puedes correr un LLM dentro del canister | Un modelo de 7B parámetros requiere 14GB+ de RAM. El canister tiene máximo 4GB stable memory. | Los modelos corren en los servidores de Anthropic/OpenAI. El canister hace el call vía HTTPS Outcall. |
| Los LLMs no son deterministas | GPT o Claude pueden dar respuestas distintas a la misma pregunta. El consenso de 28 nodos requiere que todos lleguen al mismo resultado. | El canister hace el call desde UN nodo (no en consenso total) o acepta la variación con transform functions. |
| Límite de compute por mensaje | ~50 mil millones de instrucciones WebAssembly por update call. Suficiente para lógica financiera, no para inference de IA. | La inference corre en Anthropic/OpenAI, el canister solo orquesta y almacena resultados. |
| Latencia de HTTPS Outcall | 2–30 segundos por call a una API externa. | Para GreyValley: acceptable para análisis de portfolio, no para trading en tiempo real. |

**Estado actual en GreyValley:** los agentes IA (Haiku para datos, Opus para estrategia) se llaman desde el frontend directamente. La roadmap es mover la orquestación al backend canister para que los análisis sean on-chain y auditables.

---

### "Infraestructura completa" — qué queda FUERA de ICP

ICP cubre mucho, pero no todo:

| Lo que NO está en ICP | Quién lo maneja en GreyValley |
|---|---|
| DNS / dominio (greyvalley.xyz) | Registro de dominio externo (GoDaddy, Namecheap) + el boundary node de ICP como servidor |
| Fiat on/off ramp (CLP ↔ ckUSDC) | Koywe — empresa regulada con cuentas bancarias reales |
| KYC/AML | Koywe lo hace en su lado; GreyValley SpA es responsable en el suyo |
| Entidad legal (contratos, facturas, SII) | GreyValley SpA — fuera de ICP completamente |
| Email y notificaciones | HTTPS Outcalls a APIs de email (SendGrid, etc.) — no hay servidor de correo nativo |
| App stores (iOS/Android) para wallet nativa | Google Play / App Store — sus reglas son externas |
| Secretos de API keys para integraciones | El estado del canister (solo accesible por el controller) — no es un vault dedicado |

---

## POSICIONAMIENTO POR ZONA — SWIFT vs GREYVALLEY

| Zona | Cliente natural | Dolor principal | Lo que evita GreyValley | Producto |
|---|---|---|---|---|
| **Santiago** | Importadora/exportadora con pagos al exterior | SWIFT lento + spread 1–3% + corresponsales | El tramo internacional completo | ODL Bridge + Vault Exaltite |
| **Santiago** | Pyme con tesorería idle en CLP/USD | CLP sin rendimiento, inflación erosiona el capital | Nada que ver con SWIFT — es rendimiento en dólares digitales | Vault Exaltite (ckUSDC) |
| **Mendoza / ARG** | Empresa con operaciones cross-border Chile-Argentina | Restricciones cambiarias ARS, cepo, riesgo peso | ARS inestable → ckUSDC estable → CLP sin pasar por banca formal | ckUSDC como store of value + corredor CLP↔ARS |
| **Lima / Bogotá** | Empresa que paga proveedores en Chile o recibe remesas | Costo alto de transferencia internacional (SWIFT + FX) | Todo el tramo internacional | Corredor Guild Track B |
| **Santiago / Tech** | Startup que quiere pagos sin montar su propia infraestructura bancaria | Stripe no funciona en Chile para todos los casos; integraciones bancarias caras | La infraestructura de pagos internacionales | SDK GreyValley + vUSD Services |

---

## CHILE + PERÚ + COLOMBIA — EL CASO MILA Y POR QUÉ NO FUNCIONÓ

**MILA (Mercado Integrado Latinoamericano):** en 2011, Chile (Bolsa de Santiago), Perú (BVL) y Colombia (BVC) integraron sus mercados de valores. Luego se sumó México. La promesa: un inversor en Lima podía comprar acciones de Falabella sin pasar por intermediarios internacionales.

**El problema de no-correlación que se mencionó:**

La tesis era que diversificar entre LatAm reduciría volatilidad (correlación baja entre mercados). En la práctica:
- En crisis globales (COVID 2020, macro selloff 2022), **todos los mercados LatAm caen juntos** porque el capital extranjero sale de "mercados emergentes" como categoría. La no-correlación desaparece exactamente cuando más la necesitas.
- El inversor que compró acciones peruanas desde Chile asumió **doble riesgo cambiario**: CLP → PEN al comprar, PEN → CLP al vender. Si PEN cayó mientras las acciones subían, el rendimiento desapareció en el FX.
- **Liquidez asimétrica:** el spread bid/ask en acciones peruanas visto desde Chile era mayor que en el mercado local. Los market makers locales no cubrían el volumen cross-border bien.

**Precios "acá y allá" no correlacionados:** la misma acción en dos mercados debería tener el mismo precio (arbitraje). Pero con 2 días de liquidación, tipos de cambio fluctuantes, y liquidez asimétrica, el mismo activo podía diferir 2–5% entre mercados — sin que nadie pudiera arbitrar eficientemente porque el ciclo de liquidación era más lento que la diferencia de precio.

**Cómo GreyValley resuelve esto (roadmap):**

Si acciones tokenizadas (RWAs) liquidaran en ckUSDC:
- Un inversor en Lima compra acciones de Falabella tokenizadas → paga en ckUSDC.
- La liquidación es en segundos, en la misma unidad de valor (ckUSDC).
- No hay FX entre PEN y CLP — ambos liquidaron en dólares digitales.
- El spread cross-market se arbitraría instantáneamente porque el ciclo de liquidación es ~2 segundos, no 2 días.

Este es el caso de uso RWA/tokenización que ICP habilita — no disponible en GreyValley V1, pero es parte de la visión de "adopción institucional" que menciona el CLAUDE.md.

---

## TESORERÍA IDLE — RECOMENDACIONES PARA LA EMPRESA CHILENA

### Con qué dinero entra

La ruta actual (sin ODL activo):
1. Empresa chilena transfiere CLP a Koywe → recibe ckUSDC en su wallet ICP.
2. Deposita ckUSDC en Vault Exaltite → empieza a acumular PXRM como rendimiento.
3. Cuando quiere salir: retira ckUSDC → Koywe lo convierte de vuelta a CLP.

**Recomendación práctica:** empezar con USD ya disponibles (si la empresa exporta o tiene dólares) — evita el primer tramo CLP→ckUSDC que tiene su propio spread Koywe.

### Implicación de la declaración de impuestos

La declaración de renta en Chile es en **abril** por el año anterior (enero–diciembre).

| Escenario | Cuándo tributa | Recomendación |
|---|---|---|
| Empresa deposita ckUSDC en diciembre 2026 y retira en enero 2027 | El rendimiento (PXRM) se declararía en la renta de 2027 (pagadera abril 2028), si el SII considera el hecho gravable al momento de recibir los tokens. | Si quieres diferir el impuesto, el holding corto dentro del mismo año fiscal puede ser más limpio. |
| Empresa deposita en enero 2026 y retira en diciembre 2026 | Todo el rendimiento cae en el ejercicio 2026 → declaración abril 2027. | Máxima exposición fiscal en un solo ejercicio. |
| Empresa retira en noviembre 2026 antes de año fiscal | Cierra el ciclo dentro del ejercicio 2026. Si convierte PXRM a CLP antes del 31 dic → ganancias de capital realizadas en 2026. | Ganancia de capital realizada = declarar en abril 2027. |

**Punto gris SII (del `VAELIX_SII_FISCAL.md §4`):** el criterio sobre cuándo tributa el rendimiento en PXRM aún no está consolidado. Posición conservadora: el PXRM no es ingreso hasta que se convierte a CLP. Consultar contador con experiencia en crypto.

**Consejo operacional antes de la consulta al contador:** mantener registro exacto de:
- Fecha de depósito + monto en USD
- Fecha de cada harvest/retiro + monto de PXRM recibido
- Precio de PXRM en CLP al momento de cada conversión

ICP tiene logs on-chain (ICRC-3) que generan ese registro automáticamente — es la "auditoría gratuita" que viene con la blockchain.

---

## ANDROID — PREVISUALIZACIÓN DE APP EN CELULAR

Para GreyValley en su estado actual (web app en ICP):
- La app ya es accesible desde **Android Chrome** apuntando al URL del frontend canister.
- No hay APK — es una Progressive Web App (PWA). El usuario puede agregarla al home screen desde Chrome.

Para la **Wallet móvil nativa (Fase 2 del roadmap en `WALLET.md`):**

| Método | Qué es | Cuándo usar |
|---|---|---|
| **Google Play Internal Testing** | Subes el APK a Play Console y lo distribuyes a hasta 100 testers sin publicar | Cuando hay una build nativa lista para probar con usuarios reales |
| **Firebase App Distribution** | Plataforma de Google para distribuir APKs de prueba fuera del Play Store | Testing más amplio antes de ir a producción |
| **Sideload directo (APK)** | El usuario instala el APK directamente descargándolo (sin Play Store) | Solo para el equipo de desarrollo — no para usuarios |
| **Expo Go** | App de React Native que permite previsualizar sin compilar un APK | Si la wallet se hace en React Native — el más rápido para iterar |

**Recomendación para GreyValley:** hasta que haya wallet nativa, el "Android preview" es simplemente abrir el URL del canister en Android Chrome. Funciona — y el diseño mobile está especificado en `WALLET.md §Wallet Móvil`.

---

## LEY 21.719 — DATOS PERSONALES 2027 Y POR QUÉ ICP LO RESUELVE POR DISEÑO

### El problema que están construyendo (y que ya está resuelto en ICP)

**Ley N° 21.719 de Protección de Datos Personales** — promulgada diciembre 2024, vigencia plena 2027. Modelada sobre GDPR europeo. Los dos derechos que obligan a tener infraestructura técnica nueva:

- **Derecho de acceso / portabilidad:** el usuario puede pedir "entrégueme todos mis datos" y la empresa tiene plazo legal para responderte con todo.
- **Derecho al olvido / borrado:** el usuario puede pedir "bórrese todo lo mío" y la empresa debe poder hacerlo de forma verificable.

**Por qué los sistemas actuales no pueden responder rápido:**

Una empresa típica chilena tiene los datos de un usuario dispersos en:

```
DB principal (PostgreSQL)      → nombre, RUT, email, historial
Logs de aplicación             → IPs, acciones, errores con datos del usuario
Backups (S3, cintas)           → snapshot completo de la DB en alguna fecha
Caché (Redis)                  → sesiones activas, preferencias
Email marketing (HubSpot)      → historial de correos enviados
CRM (Salesforce)               → interacciones comerciales
Analytics (GA, Mixpanel)       → comportamiento en el sitio
Sistema de pagos (Transbank)   → historial de transacciones
Data warehouse (Redshift)      → reportes históricos
Réplicas de lectura de la DB   → copias sincronizadas
Archivos adjuntos (S3)         → documentos subidos por el usuario
```

Cuando el usuario pide "bórrese" — nadie tiene un mapa de dónde está todo. Encontrarlo toma semanas de trabajo manual. Borrarlo completamente, sin dejar trazas en backups históricos o logs, es casi imposible de garantizar.

**Lo que muchas empresas estaban construyendo para cumplir:** un **data catalog** que mantiene un índice de "para este usuario, sus datos existen en estas N ubicaciones". Cuando llega la solicitud, el sistema recorre el índice y coordina la extracción o borrado. El problema: ese catálogo hay que construirlo, mantenerlo actualizado a medida que la arquitectura cambia, y los backups históricos siempre se escapan.

---

### Cómo ICP resuelve esto por diseño — y dónde tiene el mismo límite

En ICP, **todo el estado de un usuario está asociado a su Principal ID**. No hay datos dispersos en 11 servicios — el canister es la única fuente de estado.

> Nota: el snippet de abajo es **ilustrativo del patrón de diseño**, no código que ya exista
> en `backend/main.mo` — `query_user_data`/`delete_user_data` no están implementadas hoy.

```motoko
// Encontrar todos los datos de un usuario:
query_user_data(principal: Principal) : async UserDataExport {
    {
        vaultPositions = Map.get(vaultPositions, principal);
        stakePositions = Map.get(stakePositions, principal);
        kycData        = Map.get(kycData, principal);
        harvestHistory = Map.get(harvestHistory, principal);
    }
}

// Borrar todos los datos de un usuario:
update delete_user_data(principal: Principal) {
    requireController();
    Map.delete(vaultPositions, principal);
    Map.delete(stakePositions, principal);
    Map.delete(kycData, principal);
    Map.delete(harvestHistory, principal);
}
```

**Una sola función. Una sola transacción. Sin recorrer 11 sistemas.** El "data catalog" es el propio diseño del canister — todos los datos del usuario están keyed por el mismo Principal.

---

### El límite que comparte con cualquier blockchain — y cómo diseñar para sortearlo

El **log de transacciones (ICRC-3)** es append-only por naturaleza criptográfica. Una transacción registrada no se puede borrar sin romper la integridad del historial. Esto es el mismo dilema de GDPR en Ethereum.

La solución es de **diseño previo**, no de borrado posterior:

| Dato | Dónde vive | ¿Borrable bajo Ley 21.719? | Diseño correcto |
|---|---|---|---|
| PII real (nombre, RUT, email) | Estado mutable del canister (stable var) | ✅ Sí — `delete_user_data()` lo elimina | Guardar PII solo en state mutable, nunca en logs |
| Posiciones vault/stake | Estado mutable del canister | ✅ Sí | Keyed por Principal — borrable en una llamada |
| Historial transacciones (ICRC-3) | Log append-only | ❌ No borrable | Solo contiene el Principal (pseudónimo criptográfico), NO el RUT — el log queda, la identidad real no |
| KYC data (Koywe side) | Sistemas de Koywe | ✅ Koywe tiene obligación propia bajo la ley | Koywe gestiona su cumplimiento; GreyValley gestiona el suyo |

**La clave de diseño:** si el Principal ID (una dirección criptográfica, no un nombre ni RUT) es el único identificador en los logs, el log puede quedar intacto sin violar la ley. El derecho al olvido aplica al **PII vinculado a la persona real**, no al registro de que "alguien hizo una transacción". Si la vinculación Principal ↔ RUT real se borra del state mutable, el log queda como datos anónimos.

---

### Por qué esto es un argumento comercial para GreyValley

Lo que esas empresas chilenas están construyendo para 2027 — el sistema de data subject requests — GreyValley lo tiene por diseño desde el día 1. Eso es argumento comercial real para el segmento corporativo y institucional:

> *"Si tu empresa necesita cumplir la Ley 21.719 y manejar datos financieros de usuarios, una arquitectura ICP te da el data subject request nativo. No necesitas construir un catálogo de datos separado, no necesitas coordinar borrados en 11 sistemas, no necesitas contratar una consultoría de data governance. El diseño del canister ya tiene esa respuesta: `query_user_data(principal)` y `delete_user_data(principal)`. Eso es infraestructura de compliance incluida en la arquitectura, no agregada después."*

**Posicionamiento específico por segmento:**

| Segmento | Por qué les importa la Ley 21.719 | Argumento GreyValley |
|---|---|---|
| Fintech / startup | Tienen datos de usuarios y están construyendo cumplimiento desde cero | ICP como arquitectura compliance-by-design desde el inicio — más barato que retrofitar |
| Pyme con sistema propio | Tienen sistemas legacy que no saben dónde guardan todo | Migrar la capa financiera a GreyValley/ICP resuelve ese vector de datos |
| Empresa con clientes institucionales | Sus clientes corporativos van a exigir evidencia de cumplimiento | Audit trail on-chain + borrado verificable en una transacción |
| Empresa exportadora | Datos de contrapartes internacionales pueden estar sujetos a GDPR europeo también | Mismo mecanismo cumple GDPR y Ley 21.719 simultáneamente |

---

*VAELIX_ICP_PITCH_INGENIERO.md | Creado: 2026-08-08*
*Cross-reference: `VAELIX_ICP_TECH.md` (profundidad técnica NNS/tECDSA), `VAELIX_REGULATORY.md §9` (argumento CMF), `VAELIX_SII_FISCAL.md` (tesorería e impuestos), `WALLET.md` (roadmap mobile), `VAELIX_REGULATORY.md §1` (UAF/CMF)*
