# GreyValley en 3 minutos

> Borrador · 2026-10-05 · Describe lo que existe hoy, no promete rendimientos. No es asesoría legal ni financiera.

## Qué es
GreyValley es una **interfaz de código abierto** para protocolos desplegados en Internet Computer. Operas con **tu propia wallet**: tú firmas cada acción y los activos los controlan contratos públicos, no GreyValley. El eje es **ckUSDC**, un dólar digital.

## Qué puedes hacer (la app simplificada, `/app`)
| Función | Qué hace | Cómo funciona |
|---|---|---|
| **Convertir y Swap** | Cambiar tokens a ckUSDC y entre sí | Compara ICPSwap con los pools de GreyValley y descarta rutas con precios fuera de lo razonable |
| **Prestar** | Aportar ckUSDC a un pool de crédito | Otros usuarios piden prestado de ese pool; el interés que pagan se reparte entre quienes prestan |
| **Pedir** | Recibir ckUSDC sin vender tu cripto | Dejas ckBTC o ckETH como garantía. Las reglas (garantía mínima, liquidación, bono) se leen del contrato; al 2026-10-05: 200% para pedir, liquidación bajo 140% |
| **Pools y bóvedas** | Aportar un token a pools de intercambio | La retribución viene de comisiones de swap; hay pérdida impermanente |
| **Corredor** | Aportar inventario a un corredor de pagos CLP ↔ USD/EUR | En modo de prueba; depende de socios de rampa |
| **Enviar y retirar** | Mover activos a otra wallet o a Ethereum | ckETH y ckUSDC se pueden retirar a una wallet EVM |

Antes de cada firma hay un **resumen en lenguaje simple** (qué autorizas, qué riesgos existen y quién puede cambiar qué).

## Para quién
- **Usuarios:** convertir, ahorrar en dólares digitales o pedir prestado contra cripto.
- **Socios y empresas:** incrustar la app en su sitio con su nombre y logo y recibir una comisión por las operaciones de sus usuarios (ver `partnerships/market-maker-guide.md` y `architecture/corredor-mechanics.md`).

## Qué NO es
- No es un banco ni un depósito: **no hay seguro ni garantía** de recuperar lo aportado.
- No es una oferta de inversión: las retribuciones **no están garantizadas** y pueden ser bajas o nulas.
- No es un producto maduro: **etapa temprana**, pools propios con poca liquidez, sin auditoría externa del código y con el pool de crédito sin confirmación legal de la CMF.

## La app completa
Además existe una versión completa con bóvedas de rendimiento, staking, el token PXRM, vUSD y posiciones de colateral (CDP). Se explica en `finance/tokenomics.md`. La app simplificada no muestra PXRM ni vUSD; para verlos se conecta a la app completa con la misma wallet.

## Más información
`architecture/tech-overview.md` · `architecture/corredor-mechanics.md` · `legal/regulatory.md` · `finance/tokenomics.md`
