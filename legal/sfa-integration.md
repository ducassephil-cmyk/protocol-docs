# Vaelix — Integración SFA-Sandbox (CMF Open Finance Chile)
> Documento técnico de referencia para la integración con sfasandbox.cl
> Generado: 2026-08-13 | Basado en documentación oficial sfasandbox.cl/docs.php
> Normativa: Ley Fintech 21.521 · NCG 502 · NCG 514 · FAPI 2.0

---

## 1. Contexto y Distinción Importante

**SFA-Sandbox NO es el sandbox del SII.** Son sistemas completamente distintos:

| Sistema | Para qué | Operador | Normativa |
|---------|----------|----------|-----------|
| **SFA-Sandbox** (`sfasandbox.cl`) | Open finance bancario: pagos TEF, lectura de cuentas, PISP/AISP | Equipo SFA-Sandbox (privado, alineado CMF) | NCG 502, NCG 514, FAPI 2.0 |
| **SII Sandbox** (`maullin2.sii.cl`) | Facturación electrónica (DTE), folios, notas de crédito | Servicio de Impuestos Internos | Resolución SII, formato XML |

Vaelix usa **SFA-Sandbox** para el flujo fiat on/off-ramp (CLP ↔ ckUSDC). El SII es un tema aparte de operación contable de Vaelix SpA.

---

## 2. Autenticación y Headers

Toda llamada a la API requiere dos headers obligatorios:

```http
x-sandbox-key: sfa_tu_llave_aqui
x-fapi-interaction-id: b1c67d3e-9081-42cb-a311-2df8a2f1c8e9
Content-Type: application/json
```

- **`x-sandbox-key`**: llave exclusiva por entorno, obtenida desde `sfasandbox.cl/console.php` tras login con LinkedIn o Dev Mode (`/auth.php?action=dev_login`).
- **`x-fapi-interaction-id`**: UUID único por request — sirve como traza forense por exigencia FAPI 2.0. Debe ser distinto en cada llamada.
- **NO usar** `Authorization: Bearer` como header de autenticación de la llave — eso aplica solo al Bearer JWT que se obtiene después del flujo de consentimiento OIDC/CIBA.

---

## 3. Endpoints Disponibles

Base URL: `https://sfasandbox.cl/api.php?action=<endpoint>`

| Endpoint | Método | Auth requerida | Descripción |
|----------|--------|---------------|-------------|
| `consents` | POST | x-sandbox-key | Crea intención de consentimiento FAPI 2.0 (redirect flow) |
| `token` | POST | x-sandbox-key | Canjea código temporal por Bearer JWT |
| `ciba_auth` | POST | x-sandbox-key | Inicia sesión CIBA backchannel (sin redirect) |
| `ciba_token` | POST | x-sandbox-key | Polling de estado CIBA |
| `payments_pisp` ⚠ | POST | **x-sandbox-key + Bearer JWT** (corregido 2026-08-24 — ver nota abajo) | Inicia TEF (PISP — pago desde cuenta bancaria) — corregido 2026-08-22, ver nota abajo |
| `payment_status` | GET | x-sandbox-key | Consulta estado de un pago por ID |
| `channels_status` | GET | x-sandbox-key | Disponibilidad operativa NCG 514 (reacciona al chaos engine) |
| `balances` | GET | x-sandbox-key + Bearer JWT | Saldo de cuenta del usuario (AISP) |
| `transactions` | GET | x-sandbox-key + Bearer JWT | Historial 5 años del usuario (AISP) |
| `branches` | GET | x-sandbox-key | Sucursales y ATMs geolocalizados |
| `products` | GET | ninguna | Catálogo de productos bancarios (open data) |

---

## 4. Flujo PISP — On-Ramp CLP → ckUSDC (Relevancia Directa Vaelix)

Este es el flujo principal para el on-ramp de Vaelix: el usuario paga desde su cuenta bancaria chilena y el protocolo mintea ckUSDC a su Principal.

> **⚠ Corrección real 2026-08-22, corregida también en código 2026-08-24:**
> este documento (y `src/sfa_treasury/main.mo`) tenían el endpoint como
> `action=payments` — verificado contra el `openapi.json` REAL de
> sfasandbox.cl, el nombre correcto es **`action=payments_pisp`**. Con el
> nombre viejo la llamada real pegaba contra un endpoint inexistente. Ya
> corregido en ambos lados (doc y código) — el founder pidió empezar las
> pruebas reales de sandbox SFA. Además: ni el manual técnico ni el
> openapi.json publican valores de ejemplo
> (`creditor_account`/`creditor_rut`/`creditor_bank`) para el payload real —
> hay que sacarlos de la consola web ("Iniciación Pagos QR/PIX") o pedirlos
> a `equipo@sfasandbox.cl`.
>
> **⚠ Segunda corrección real, 2026-08-24 — el auth requerido estaba mal
> documentado.** Primera prueba real (endpoint ya corregido, credenciales
> reales cargadas) devolvió `401 {"error":"unauthorized_client"}` para
> `payments_pisp`, aunque la misma key funcionaba bien para
> `channels_status`. Se descartó el Chaos Engine (el founder lo desactivó,
> el error persistió idéntico). Verificado directo contra el
> `openapi.json` real: `payments_pisp` declara `security: bearerAuth`
> (JWT Bearer token) — `x-sandbox-key` sola no alcanza para esta acción
> específica, a diferencia de `channels_status`/`branches` donde sí
> alcanza. La tabla de arriba decía "x-sandbox-key" solo — corregido.
>
> **Flujo correcto real:** `consents` (crear intención de consentimiento)
> → firma del usuario (redirect o CIBA backchannel, §6) → `token` (canjea
> el código por el Bearer JWT) → recién ahí `payments_pisp` con
> `Authorization: Bearer <jwt>` + `x-sandbox-key`. El código actual
> (`sfa_treasury.registerPendingMint()`) todavía llama `payments_pisp`
> directo sin ese JWT — falta implementar el flujo de consentimiento
> completo antes de esa llamada. Queda como próximo paso real, no
> bloqueante para lo ya probado (`channels_status` funciona de punta a
> punta).

### Paso 1 — Iniciar pago TEF

```http
POST https://sfasandbox.cl/api.php?action=payments_pisp
x-sandbox-key: sfa_tu_llave
x-fapi-interaction-id: <uuid>
Content-Type: application/json

{
  "amount": 15000,
  "debtor_account": "12345678",
  "creditor_account": "987654321",
  "creditor_name": "Vaelix SpA",
  "creditor_rut": "76.XXX.XXX-X",
  "creditor_bank": "Banco Santander Chile"
}
```

**Respuesta (201 Created):**
```json
{
  "payment_req_id": "urn:sfa:payment:pay_req_a89bc...",
  "status": "AWAITING_AUTHORISATION",
  "consent_url": "https://sfasandbox.cl/auth.php?action=payment_auth&payment_req=...",
  "expires_in": 600
}
```

El usuario debe ser redirigido a `consent_url` para firmar con PIN bancario.

### Paso 2 — Polling de estado

```http
GET https://sfasandbox.cl/api.php?action=payment_status&id=urn:sfa:payment:pay_req_...
x-sandbox-key: sfa_tu_llave
x-fapi-interaction-id: <uuid>
```

**Respuesta cuando autorizado:**
```json
{
  "payment_req_id": "urn:sfa:payment:pay_req_...",
  "status": "AUTHORIZED",
  "transaction_code": "TEF-D8A9F1B2",
  "amount": 15000,
  "updated_at": "2026-08-13T10:00:00Z"
}
```

### Paso 3 — Minteo ckUSDC (solo cuando AUTHORIZED)

**Regla crítica:** el canister ICP solo debe mintear ckUSDC al Principal del usuario **después de confirmar `status: "AUTHORIZED"`**. Nunca mintear en base al `AWAITING_AUTHORISATION` inicial.

```
AWAITING_AUTHORISATION → polling cada 3s → AUTHORIZED → mint ckUSDC al Principal
                                         → REJECTED → no mint, devolver error al usuario
                                         → expirado (600s) → no mint, reiniciar flujo
```

---

## 5. Circuit Breaker — NCG 514 (Disponibilidad Operativa)

Antes de iniciar cualquier on-ramp, el canister debe verificar que los sistemas bancarios estén operativos:

```http
GET https://sfasandbox.cl/api.php?action=channels_status
x-sandbox-key: sfa_tu_llave
x-fapi-interaction-id: <uuid>
```

**Respuesta normal:**
```json
{
  "status": "OPERATIONAL",
  "channels": {
    "apis": { "status": "OPERATIONAL", "availability_24h": 99.98 },
    "oauth": { "status": "OPERATIONAL", "availability_24h": 99.99 }
  }
}
```

**Lógica del canister:**
- Si `status != "OPERATIONAL"` → rechazar depósito con mensaje: "Sistema bancario chileno temporalmente no disponible. Intente en unos minutos."
- Este endpoint reacciona al chaos engine en tiempo real — si activas errores 500 en el panel, este endpoint se degrada.

---

## 6. Flujo CIBA — Autenticación Backchannel (Sin Redirect)

CIBA es el flujo preferido para la app móvil de Vaelix: el usuario aprueba desde su app bancaria sin salir de la app de Vaelix.

### Paso 1 — Iniciar sesión CIBA

```http
POST https://sfasandbox.cl/api.php?action=ciba_auth
x-sandbox-key: sfa_tu_llave
x-fapi-interaction-id: <uuid>
Content-Type: application/json

{
  "login_hint": "12.345.678-9",
  "scope": "accounts",
  "binding_message": "SFA-8891"
}
```

**Respuesta:**
```json
{
  "auth_req_id": "bcauth_7f2a8c3d9e01",
  "expires_in": 300,
  "interval": 5
}
```

### Paso 2 — Polling

```http
POST https://sfasandbox.cl/api.php?action=ciba_token
Content-Type: application/json

{ "auth_req_id": "bcauth_7f2a8c3d9e01" }
```

**Posibles respuestas:**
```json
{ "error": "authorization_pending" }   // usuario aún no ha aprobado
{ "error": "expired_token" }           // timeout — reiniciar flujo
{ "access_token": "eyJhbG...", "token_type": "Bearer", "expires_in": 3600 }  // aprobado
```

**Timeout recomendado:** 5 minutos. Si el usuario no aprueba en ese tiempo, cancelar y mostrar opción de reintentar.

---

## 7. AISP — Lectura de Cuentas del Usuario

Con el Bearer JWT obtenido del flujo OIDC o CIBA, se pueden leer cuentas bancarias del usuario. Relevante para el dashboard unificado (saldo banco + saldo Vaelix en una pantalla).

### Saldos

```http
GET https://sfasandbox.cl/api.php?action=balances
Authorization: Bearer eyJhbGciOiJQUzI1NiIs...
x-sandbox-key: sfa_tu_llave
x-fapi-interaction-id: <uuid>
```

**Respuesta:**
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

### Historial de transacciones (5 años — exigencia regulatoria)

```http
GET https://sfasandbox.cl/api.php?action=transactions
Authorization: Bearer eyJhbGciOiJQUzI1NiIs...
x-sandbox-key: sfa_tu_llave
x-fapi-interaction-id: <uuid>
```

---

## 8. Motor de Caos — Testing de Resiliencia

El sandbox tiene un chaos engine activable desde el panel que simula fallas reales de los cores bancarios:

| Anomalía | Efecto | Cómo probar en Vaelix |
|----------|--------|----------------------|
| **Latencia +3500ms** | Todas las llamadas tardan 3.5s extra | El HTTPS Outcall del canister debe tener timeout ≥ 8s |
| **HTTP 500** | Endpoints devuelven error aleatorio | El canister debe hacer retry con backoff exponencial (máx 3 intentos) |
| **CIBA pending infinito** | `authorization_pending` nunca resuelve | Timeout de 5 min en el poller del canister |
| **Rate limit 429** | Bloquea exceso de peticiones | Implementar cola de requests, no reintentar inmediato |

**Protocolo de prueba recomendado antes de producción:**
1. Activar chaos engine en modo "Latencia +3.5s"
2. Ejecutar flujo PISP completo y verificar que el canister espera correctamente
3. Activar "HTTP 500" y verificar que el circuit breaker rechaza el on-ramp limpiamente
4. Verificar que nunca queda un depósito en estado `AWAITING_AUTHORISATION` de forma permanente

---

## 9. Proxy Túnel (Avanzado)

Para equipos avanzados: enrutar tráfico real hacia APIs bancarias de producción a través del proxy de caos de SFA.

```http
x-proxy-target: https://api.bancoreal.cl/v1/accounts
x-sandbox-key: sfa_mi_llave
```

Esto permite interceptar y sabotear tráfico real hacia un banco. **No usar en producción** — solo para pruebas de integración avanzadas.

---

## 10. Integración ICP — HTTPS Outcalls desde Canister

En la arquitectura Vaelix, el canister ICP llama directamente a la API SFA sin ningún backend Node.js intermedio. Cada HTTPS Outcall la ejecutan los 28 nodos del subnet independientemente — ≥19/28 deben acordar la respuesta.

**Problema de determinismo:** SFA incluye headers como `x-fapi-interaction-id` y timestamps que varían entre nodos. El canister debe usar `transform` para eliminar estos campos antes del consenso.

```motoko
// Transform: eliminar headers no-determinísticos de la respuesta SFA
public query func transformSFAResponse(raw : Http.TransformArgs) : async Http.HttpResponsePayload {
    {
        status  = raw.response.status;
        body    = raw.response.body;
        headers = [];  // eliminar todos los headers — solo el body es determinístico
    }
};
```

**Headers de request desde el canister:**
```motoko
let headers = [
    { name = "x-sandbox-key";          value = sfaSandboxKey },
    { name = "x-fapi-interaction-id";  value = generateUUID() },
    { name = "Content-Type";           value = "application/json" },
];
```

**Nota sobre `x-fapi-interaction-id`:** aunque debe ser único por request, en un HTTPS Outcall los 28 nodos envían el mismo UUID (el canister lo genera antes de la llamada). El servidor SFA lo acepta igual — el problema de determinismo es en los headers de *respuesta*, no de request.

---

## 11. Roadmap de Integración Vaelix × SFA

| Fase | Qué se integra | Responsable | Estado |
|------|---------------|-------------|--------|
| **Fase 1 (actual)** | Koywe maneja PISP — Vaelix solo llama API Koywe | Koywe (acreditado CMF) | ⏳ KYB pendiente |
| **Fase 2** | `channels_status` como circuit breaker en el canister | Vaelix técnico | ✅ Construido (`sfa_treasury/main.mo:204-230`, corrección auditoría 2026-08-24 — este doc decía "no construido" pero el código ya lo tiene) |
| **Fase 2** | AISP: saldo bancario + saldo Vaelix en dashboard unificado | Vaelix técnico | ❌ No construido |
| **Fase 3** | Registro Vaelix SpA como PISP directo (elimina fee Koywe) | Legal + CMF | ❌ Requiere registro CMF |
| **Fase 3** | CIBA backchannel desde la app móvil | Vaelix técnico | ❌ No construido |

**Prerequisito para Fase 3:** Vaelix SpA debe ser entidad regulada (PSAV o PISP) y superar el proceso de acreditación SFA con CMF. Requiere: entidad legal activa, AML/KYC implementado, auditoría de seguridad, depósito de garantía.

---

## 12. Acción Inmediata

| Prioridad | Acción |
|-----------|--------|
| 🔴 **Alta** | Obtener `x-sandbox-key` desde `sfasandbox.cl/console.php` (login LinkedIn o Dev Mode) |
| 🔴 **Alta** | Probar flujo PISP completo (payments → polling → AUTHORIZED) con chaos engine activo |
| 🟡 **Media** | Implementar `channels_status` como circuit breaker en el canister bridge |
| 🟡 **Media** | Evaluar CIBA vs redirect OAuth2 como flujo de auth para el on-ramp móvil |
| 🟢 **Baja** | Explorar registro Vaelix SpA como AISP (menos restrictivo que PISP, primer paso legal) |

---

*Vaelix SFA Integration Reference | Actualizado: 2026-08-13 | Fuente: sfasandbox.cl/docs.php*
