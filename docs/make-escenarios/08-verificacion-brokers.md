# Guía — Verificación de brokers (anti-fraude)

**Última actualización:** 2026-07-19
**Propósito:** confirmar que un broker es real (MC number) antes de comprometer
un camión. Protege al dueño del fraude #1 de la industria (double brokering).

---

## 🧭 Las 3 decisiones de arquitectura (respuestas rápidas)

```
¿Nuevo escenario o el mismo que busca el camión?
   → EL MISMO. La búsqueda del broker es un paso más DENTRO del escenario
     de cotización (antes/junto con la búsqueda del camión). No dupliques.

¿Qué automatización necesito para "confirmar"?
   → El LOOKUP del broker va en el escenario de cotización (un Search).
   → Solo UNA automatización nueva y pequeña: avisar al dueño cuando llega
     un broker NUEVO, para que verifique en FMCSA antes de despachar.

¿Cómo valido el MC number?
   → Nivel 1 (MVP): comparar contra tu tabla de brokers conocidos + el
     dueño verifica los nuevos manualmente en FMCSA SAFER (gratis).
   → Nivel 3 (futuro): API de FMCSA para validar automático.
```

---

## 🔑 El principio

```
Broker CONOCIDO y verificado  → despacha rápido (ya confías en él)
Broker NUEVO                  → cotiza igual (no pierdas el lead)
                                PERO no rueda el camión hasta que el dueño
                                verifique el MC en FMCSA
```

Nunca rechazas al broker en la llamada — cotizas, pero el despacho de un
broker nuevo espera la luz verde del dueño.

---

## PARTE 1 — La tabla "Brokers" (el historial de confianza)

Crea una tabla nueva `Brokers`:

| Campo | Tipo | Para qué |
|---|---|---|
| `MC_Number` | Single line text | La llave — su licencia federal |
| `Company` | Single line text | Nombre de la empresa |
| `Broker_Contacto` | Single line text | Persona con quien tratas |
| `Email` | Email | Para factura/rate con |
| `Telefono` | Phone | Contacto |
| `Verificado` | Single select | Pendiente / Verificado / Rechazado |
| `Dias_Pago` | Number | Historial: ¿en cuántos días paga? |
| `Cargas_Totales` | Rollup/Number | Cuántas cargas ha hecho contigo |
| `Ultima_Carga` | Date | Última vez que trabajaron |
| `Notas` | Long text | "Paga bien", "cuidado con X", etc. |

### Link con Bookings
```
En Bookings (Reservas y Operaciones):
   Field: Broker (Link to another record → Brokers)
   → cada cita se enlaza a su broker
```

---

## PARTE 2 — Capturar el MC en la llamada (Vapi)

### Parámetros nuevos en el tool `calcular_ruta_y_precio`
```
broker_mc_number  (string)
broker_company    (string)
```

### En el prompt de reservas (la IA lo pregunta)
```
Antes de cotizar, get the broker's identity:
   "What's your MC number and company name?"
   → broker_mc_number, broker_company
If they refuse or don't have an MC → flag it, quote with caution,
   the load will need manual verification before dispatch.
```

---

## PARTE 3 — El lookup del broker (en el MISMO escenario de cotización)

Dentro del escenario que ya busca el camión, agrega ANTES de la búsqueda:

```
[Webhook Vapi]
   ↓
[Airtable Search: Brokers]                          ← NUEVO
   Formula: {MC_Number} = "{{broker_mc_number}}"
   Max: 1
   + Array Aggregator anti-rotura
   ↓
[Set var: broker_conocido]
   if( length(Array) > 0 ; true ; false )
   ↓
[Set var: broker_estado]
   if( broker_conocido = false ; "NUEVO" ;
   if( first(Array.Verificado) = "Verificado" ; "VERIFICADO" ;
   "PENDIENTE" ))
   ↓
[... sigue la búsqueda del camión y la cotización normal ...]
```

**Importante:** cotizas igual para los 3 casos — no pierdes el lead. La
diferencia es qué pasa al DESPACHAR (Parte 4).

### Al confirmar la cita, guarda el estado
```
En confirm_booking → al crear el Booking:
   Broker_Verificado = broker_estado (VERIFICADO / PENDIENTE / NUEVO)
   Broker (link) = el broker encontrado, o crea uno nuevo si es NUEVO
```

Si el broker es NUEVO, créalo en la tabla Brokers con `Verificado = Pendiente`
(así queda en la lista para que el dueño lo revise).

---

## PARTE 4 — El gate de despacho (la protección)

Aquí está la clave anti-fraude. El operador NO recibe la carga hasta que
el broker esté verificado:

```
La automatización que avisa al operador (Trip-Status = Confirmada)
   → agrégale una condición:
   AND Broker_Verificado = "VERIFICADO"

Traducción:
   Broker verificado → el operador recibe la carga, rueda el camión ✅
   Broker pendiente  → NO se despacha hasta que el dueño verifique ⛔
```

---

## PARTE 5 — La automatización NUEVA: avisar al dueño de brokers nuevos

Esta es la única automatización nueva que necesitas:

```
[Airtable Automation]
Trigger: When record matches conditions (Bookings)
   Broker_Verificado = "PENDIENTE" (o "NUEVO")
   AND Trip-Status = "Confirmada"
   ↓
Action: Send email al DUEÑO
   Asunto: ⚠️ Broker nuevo por verificar — {{Company}}
   Cuerpo:
      "Un broker NUEVO reservó una carga:
       Empresa: {{broker_company}}
       MC: {{broker_mc_number}}
       Carga: {{ID-travel}} — {{pickup_city}} → {{delivery_city}}

       Verifícalo antes de despachar:
       1. Entra a safer.fmcsa.dot.gov
       2. Busca el MC {{broker_mc_number}}
       3. Confirma: activo, nombre coincide, tiene bond $75k
       4. En el panel, marca el broker como Verificado"
```

---

## PARTE 6 — El panel de verificación (Interface del dueño)

```
Nueva página: "🔐 Brokers por Verificar"
Source: Brokers
Filter: Verificado = "Pendiente"
Editable: Verificado (para que el dueño lo marque)
Muestra: MC_Number, Company, Email, Telefono, Notas
```

El dueño:
```
1. Ve el broker pendiente + su MC
2. Abre FMCSA SAFER en otra pestaña → verifica
3. Marca Verificado = "Verificado" (o "Rechazado")
   ↓
   Si Verificado → la carga se despacha (el operador recibe el aviso)
   Si Rechazado → la carga NO se mueve, el dueño llama a cancelar
```

---

## 🚨 Señales de fraude (para las notas del dueño)

```
🚩 Rate demasiado bueno para ser verdad
🚩 Presión para mover sin papeleo
🚩 MC nuevo que no cuadra con el nombre de la empresa
🚩 Piden cambiar info de pago a último momento (identity theft)
🚩 Email genérico (@gmail) en vez de dominio de empresa
🚩 No quieren firmar el rate con
🚩 El teléfono no coincide con el registrado en FMCSA
```

---

## 💰 Recordatorio del modelo de dinero (brokers)

```
NO se cobra por adelantado. El ciclo:
   Rate con firmado → despacha → entrega + POD → factura al broker
   → NET 30-45 días (o factoring para cobrar hoy)
   → el bond de $75k del broker protege el pago
```

---

## ✅ Checklist de construcción (en orden)

```
PARTE 1 — Tabla Brokers:
[ ] Crear tabla con MC_Number, Company, Verificado, etc.
[ ] Link Bookings → Brokers

PARTE 2 — Vapi:
[ ] Parámetros broker_mc_number, broker_company en el tool
[ ] Prompt: la IA pide el MC + company

PARTE 3 — Escenario de cotización (el MISMO):
[ ] Search Brokers por MC + anti-rotura
[ ] Variables broker_conocido, broker_estado
[ ] Guardar Broker_Verificado al confirmar + crear broker nuevo

PARTE 4 — Gate:
[ ] Automatización del operador: + condición Broker_Verificado = Verificado

PARTE 5 — Automatización nueva:
[ ] Avisar al dueño cuando llega broker Pendiente

PARTE 6 — Panel:
[ ] "🔐 Brokers por Verificar" para que el dueño marque verificado
```

---

## 🎯 Orden recomendado

```
1. Tabla Brokers + link (Parte 1)
2. Capturar MC en Vapi (Parte 2)
3. Lookup en el escenario de cotización (Parte 3)
4. Panel de verificación (Parte 6) — el dueño ya puede verificar
5. Gate + automatización de aviso (Partes 4-5)
6. RECIÉN AHÍ: la automatización del formulario al operador
   (que ahora respeta el gate del broker verificado)
```

---

## Cambios

- 2026-07-19 — versión inicial. Capa de verificación de brokers: tabla de
  historial, lookup por MC en el escenario de cotización, gate de despacho,
  aviso al dueño y panel de verificación manual contra FMCSA.
