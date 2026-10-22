# GreyValley en 3 minutos

> Borrador · 2026-10-05, actualizado 2026-10-13 y 2026-10-22 · Describe lo que existe hoy, no promete rendimientos. No es asesoría legal ni financiera.

## Qué es
GreyValley es una **interfaz de código abierto** para protocolos desplegados en Internet Computer. Operas con **tu propia wallet**: tú firmas cada acción y los activos los controlan contratos públicos, no GreyValley. El eje es **ckUSDC**, un dólar digital.

## Qué puedes hacer (la app simplificada, `/app`)
| Función | Qué hace | Cómo funciona |
|---|---|---|
| **Convertir y Swap** | Cambiar tokens a ckUSDC y entre sí | Compara ICPSwap con los pools de GreyValley y descarta rutas con precios fuera de lo razonable. ICP va por ICPSwap; ckEURC y ckUSDT, por los pools de GreyValley; el resto, por la mejor ruta. Cuando la ruta pasa por ICPSwap, GreyValley cobra una **comisión de 0,22 %** del monto con una transferencia aparte, que se ve antes de firmar. Solo se cobra si es mayor que 3 veces el fee de red del token. Si llegaste por el enlace de un socio, la mitad es para el socio (desde 2026-10-16) |
| **Límite y Automático** | Órdenes de compra o venta a un precio (Límite) y compras periódicas (Automático) | Los fondos de la orden quedan en el contrato de órdenes hasta que se ejecuta o la cancelas. El contrato revisa las órdenes cada 5 minutos |
| **Prestar** | Aportar ckUSDC a un pool de crédito | Otros usuarios piden prestado de ese pool; el interés que pagan se reparte entre quienes prestan. Solo puedes retirar lo que no está prestado en ese momento; el resto se libera cuando los prestatarios devuelven. Cada retiro paga el fee de red de ckUSDC (0,01). La tasa sube con el uso del pool: entre 0,33 % y 3,30 % mensual para quien pide, al 2026-10-22. El 15 % del interés va primero a una reserva que cubre pérdidas, hasta el 5 % del pool, y después a la tesorería de GreyValley; el 85 % es para quienes prestan. La app muestra la salud agregada de todos los préstamos |
| **Pedir** | Recibir ckUSDC sin vender tu cripto | Dejas ckBTC o ckETH como garantía. Las reglas (garantía mínima, liquidación, bono) se leen del contrato. Al 2026-10-22: 200 % para pedir, liquidación bajo 140 %, bono de liquidación del 10 % y hasta la mitad de la deuda por liquidación. Mínimo 1 ckUSDC por préstamo, y un tope total de 10.000 ckUSDC de deuda por tipo de garantía. Un keeper liquida las posiciones bajo el mínimo y vende la garantía por la mejor ruta (ICPSwap o los pools de GreyValley) |
| **Documentos cifrados** (prototipo) | Guardar notas y archivos cifrados en la cadena y compartirlos con quien elijas | Cifrado con vetKeys de Internet Computer: solo tú y quienes autorices pueden abrirlos; el título y la referencia son públicos. Hoy con la llave de prueba de la red |
| **Pools y bóvedas** | Aportar un token a pools de intercambio | La retribución viene de comisiones de swap; hay pérdida impermanente. En la app están en tres pestañas: Vaults Swaps, ICP Staking Vault y Corredor Vaults |
| **Corredor** | Aportar inventario a un corredor de pagos CLP ↔ USD/EUR | En modo de prueba; depende de socios de rampa |
| **Enviar y retirar** | Mover activos a otra wallet o a Ethereum | ckETH y ckUSDC se pueden retirar a una wallet EVM |

Antes de cada firma hay un **resumen en lenguaje simple** (qué autorizas, qué riesgos existen y quién puede cambiar qué).

**Precios.** Vienen del oráculo de GreyValley: el XRC de Internet Computer para ICP, BTC y ETH; CoinGecko para LINK; mindicador.cl, con un respaldo, para CLP y EUR. Cada contrato que usa un precio (Prestar y Pedir, los pools de GreyValley, el CDP) revisa la antigüedad del precio de ese activo y no opera con uno de más de 25 minutos: primero le pide al oráculo uno nuevo. En esta etapa de pruebas, el oráculo refresca solo cada algunas horas y a pedido de las operaciones, para gastar menos (desde 2026-10-19).

## Para quién
- **Usuarios:** convertir, ahorrar en dólares digitales o pedir prestado contra cripto.
- **Socios y empresas — Distrito GV** (`/distrito`): cada empresa tiene su **Ribera** dentro de la app (`/app/ribera`), donde incrusta la app en su sitio con su nombre y logo y recibe una comisión por las operaciones de sus usuarios. Cada socio registra en cadena los sitios donde puede incrustarla (hasta 5); en cualquier otro sitio la app no se muestra, para evitar que un tercero la use para engañar a usuarios (ver `partnerships/market-maker-guide.md` y `architecture/corredor-mechanics.md`).

## Qué NO es
- No es un banco ni un depósito: **no hay seguro ni garantía** de recuperar lo aportado.
- No es una oferta de inversión: las retribuciones **no están garantizadas** y pueden ser bajas o nulas.
- No es un producto maduro: **etapa temprana**, pools propios con poca liquidez, sin auditoría externa del código y con el pool de crédito sin confirmación legal de la CMF.

## La app completa
Además existe una versión completa con bóvedas de rendimiento, staking, el token PXRM, vUSD y posiciones de colateral (CDP). Se explica en `finance/tokenomics.md`. La app simplificada no muestra PXRM ni vUSD; para verlos se conecta a la app completa con la misma wallet. Hoy no se puede mintear vUSD nuevo (`vusdMint = false`, verificado 2026-10-22).

## Más información
`architecture/tech-overview.md` · `architecture/corredor-mechanics.md` · `legal/regulatory.md` · `finance/tokenomics.md`
