# Guía para socios: ser el "market maker" del corredor CLP

> Borrador · 2026-10-05 · Resumen comercial de `architecture/corredor-mechanics.md` §4, sin jerga técnica. No promete rendimientos y no es asesoría legal.

## La idea en una frase
En los corredores de pagos internacionales, quien pone **inventario** (dinero listo en ambos lados) permite que cada pago se ejecute al instante y capta una parte del valor de cada operación. En GreyValley, ese inventario es **ckUSDC**, un dólar digital estable.

## Cómo funciona
1. Una empresa aporta ckUSDC (o ckEURC, para el corredor en euros) al pool del corredor.
2. Cuando alguien envía pesos chilenos al exterior (o los recibe), el pool ejecuta el cambio al instante, a un precio tomado de un oráculo.
3. El aportante recibe una parte proporcional de las comisiones del volumen real que pasa por el corredor.

## Qué NO es
- **No es un interés fijo.** Los ingresos dependen del volumen real de pagos, que hoy es bajo o nulo. Cualquier cifra es ilustrativa de la mecánica, no una proyección.
- **No es un depósito asegurado.** No hay seguro ni garantía de terceros.
- **Las comisiones del corredor están bloqueadas** hasta que la gobernanza del protocolo esté activa, y el corredor opera **en modo de prueba**.

## Qué sí ofrece (según la mecánica descrita)
- **Inventario estable:** al estar en dólares digitales, no tiene riesgo de precio frente al dólar; sí existe el riesgo de tipo de cambio entre peso, dólar y euro.
- **Ahorro propio:** si la empresa además procesa sus pagos internacionales por el corredor, el costo de esos pagos es la comisión del corredor (hoy 0,66%, verificado en cadena) en vez de las comisiones tradicionales de transferencias internacionales.
- **Participación variable:** parte de las comisiones y, si el volumen mensual supera el umbral definido, un reparto adicional entre los socios con más liquidez.

## Riesgos principales
- Volumen bajo: los ingresos pueden ser nulos.
- Dependencia de socios de rampa y de que la regulación lo permita.
- Riesgo de contrato inteligente y de oráculo; sin auditoría externa todavía.
- Los fondos están en contratos controlados hoy por un único principal (ver `legal/regulatory.md` §11.3).

## Qué necesitas
Una wallet de Internet Computer, ckUSDC o ckEURC, y leer el acuerdo en lenguaje simple antes de firmar. Para participar como operador con integración propia, el modelo requiere un acuerdo aparte (ver `partnerships/odl-agreement.md`, plantilla pendiente de la entidad firmante).
