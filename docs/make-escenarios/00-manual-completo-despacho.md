# Escenario 01 — Despacho automático (Vapi → Make → Cotización)

**Última actualización:** 2026-06-12
**Trigger:** Webhook desde Vapi (llamada de voz con broker)
**Output:** JSON con el camión ganador, precio total y recomendación de oferta

---

## 🎯 Propósito (1 línea)

Recibir una solicitud de carga de un broker, elegir el mejor camión disponible (considerando HOS, deadhead, citas futuras, descanso), calcular el precio total y devolver la cotización a Vapi para que la diga al broker.

---

## 🧠 La pregunta que el escenario responde

```
¿Qué camión puede aceptar esta carga sin romper su agenda,
sin exceder HOS y llegando a tiempo, al mejor precio?
```

No buscamos "qué camión está libre". Buscamos **"qué camión es operacionalmente viable y rentable"**.

---

## 🗺️ Flujo completo (cajas y flechas)

```
[Webhook Vapi]
    ↓ recibe 12 variables de carga + datos de broker
[Set var: pickup_time_formateado]   (parseDate)
[Set var: hos_requerido_total]      (estimación rápida)
    ↓
[Airtable: Search Operadores]
    ↓ filtro: Activo OR En_Descanso, HOS suficiente, disponible a tiempo
[Iterator de candidatos]
    ↓ procesa cada operador uno por uno
[Set var: hora_disponible_camion]   ← respeta descanso DOT si aplica
[Google Maps: deadhead]              ← ubicacion → pickup_city
[Set var: deadhead_minutos_real]    (segundos / 60)
[Set var: deadhead_millas_real]     (metros / 1609.34)
[Set var: eta_llegada_pickup]
    ↓
[Google Maps: viaje]                 ← pickup → delivery
[Set var: viaje_minutos_real]
[Set var: viaje_millas_real]
    ↓
[Set var: buffer_carga_base]
[Set var: buffer_carga_pallets]
[Set var: buffer_carga_extras]
[Set var: buffer_carga]              ← suma de las 3
[Set var: buffer_descarga]           ← buffer_carga * 0.8
[Set var: buffer_carretera]          ← según duración del viaje
[Set var: tiempo_jornada_completa]   ← suma de TODO
    ↓
[Google Maps: regreso a casa]        ← delivery → base operacional
[Set var: minutos_regreso]
[Set var: hora_final_real]           ← cuándo termina realmente
    ↓
[Set var: puede_regresar]            ← cabe dentro del turno
[Set var: score_candidato]           ← puntuación para comparar
    ↓
[Array Aggregator]                   ← junta todos los candidatos viables
    ↓
[Set var: camion_ganador]            ← el de mayor score
    ↓
[Filter: validaciones finales]       ← eta <= pickup, puede_regresar, HOS OK
    ↓
[Set var: tarifa_por_milla_real]     ← del lookup en Airtable
[Set var: factor_peaje_real]
[Set var: costo_millas_cargadas]
[Set var: costo_deadhead]
[Set var: costo_peajes_real]         ← peaje × factor_peaje
[Set var: costo_servicios_especiales]
[Set var: subtotal_costo]
[Set var: precio_total]
[Set var: recomendacion_oferta]      ← ACEPTAR / CONTRAOFERTA / RECHAZAR
    ↓
[Webhook Response]                   ← JSON a Vapi
```

---

## 📦 Variables principales — diccionario completo

### Variables que llegan del webhook (Vapi)

| Variable | Tipo | De dónde | Para qué |
|---|---|---|---|
| `tipo_camion` | enum | Vapi pregunta | Filtra flota: box_truck o van |
| `pickup_city` | string | Vapi pregunta | Google Maps: origen del viaje |
| `pickup_time` | datetime ISO | Vapi pregunta | Hora límite para llegar al pickup |
| `delivery_city` | string | Vapi pregunta | Google Maps: destino del viaje |
| `tipo_carga` | enum | Vapi pregunta | Compatibilidad con vehículo |
| `tipo_recogida` | enum | Vapi pregunta | live_load / wait_load — cambia buffer |
| `cantidad_pallets` | number | Vapi pregunta | Buffer de carga + capacidad |
| `metodo_carga` | enum | Vapi pregunta | Base del buffer de carga |
| `servicios_especiales` | string | Vapi pregunta | Costos extra y buffer extra |
| `hazmat` | boolean | Vapi pregunta | Buffer + filtro de conductor |
| `refrigerado` | boolean | Vapi pregunta | Buffer + filtro de camión |
| `tarifa_ofrecida` | number | Vapi pregunta | Comparar contra nuestro precio |
| `broker_nombre` | string | Vapi pregunta | Trazabilidad |
| `peso_total_libras` | number | Vapi pregunta | Capacidad del camión |

### Variables derivadas en Make (calculadas)

| Variable | Fórmula / origen | Para qué |
|---|---|---|
| `pickup_time_formateado` | `parseDate(pickup_time)` | Comparar como fecha real |
| `hos_requerido_total` | estimación inicial | Filtro previo de operadores |
| `ubicacion_operacional_real` | Parada_final OR Ubicacion_Actual OR Base | Origen del deadhead |
| `deadhead_minutos_real` | Google Maps segundos / 60 | Tiempo vacío hasta pickup |
| `deadhead_millas_real` | Google Maps metros / 1609.34 | Millas vacías hasta pickup |
| `viaje_minutos_real` | Google Maps segundos / 60 | Tiempo del viaje cargado |
| `viaje_millas_real` | Google Maps metros / 1609.34 | Millas del viaje cargado |
| `hora_disponible_camion` | Disponible_Desde_Calculado del operador | Cuándo arranca el camión |
| `eta_llegada_pickup` | hora_disponible_camion + deadhead_minutos | Si llega a tiempo |
| `buffer_carga_base` | según `metodo_carga` (20-90 min) | Tiempo base de carga |
| `buffer_carga_pallets` | `cantidad_pallets × min_por_pallet` | Extra por volumen |
| `buffer_carga_extras` | white_glove + inside_delivery + hazmat + reefer | Extras por servicios |
| `buffer_carga` | suma de las 3 anteriores | Total de carga |
| `buffer_descarga` | `round(buffer_carga × 0.8)` | Descarga = 80% de carga |
| `buffer_carretera` | escalado según `viaje_minutos` (30/60/90/120) | Margen por tráfico |
| `tiempo_jornada_completa` | suma de todos los tiempos | Cuánto dura el viaje completo |
| `hos_requerido_total` | `tiempo_jornada_completa + minutos_regreso` | HOS real necesario |
| `minutos_regreso` | Google Maps: delivery → base | Tiempo de regreso a casa |
| `hora_final_real` | pickup_time + jornada + regreso | Cuándo termina REAL |
| `puede_regresar` | `hora_final_real <= Hora_Final_Turno` | Si cabe en su turno |
| `score_candidato` | fórmula de scoring | Comparar candidatos |
| `camion_ganador` | candidato con mayor score | El elegido |
| `tarifa_por_milla_real` | `Tarifa_Milla` del lookup | Precio base |
| `factor_peaje_real` | `Factor_Peaje` del lookup | Multiplicador de peajes |
| `costo_millas_cargadas` | `viaje_millas_real × tarifa` | $ por viaje |
| `costo_deadhead` | `deadhead_millas_real × tarifa × 0.5` | $ por millas vacías |
| `costo_peajes_real` | `costo_peajes × factor_peaje_real` | $ peajes ajustado al camión |
| `costo_servicios_especiales` | suma de Rollup en Airtable | $ por extras |
| `subtotal_costo` | suma de los costos | $ antes de redondear |
| `precio_total` | `round(subtotal_costo; 2)` | $ final a cotizar |
| `recomendacion_oferta` | comparación con `tarifa_ofrecida` | ACEPTAR / CONTRAOFERTA |

---

## 🧱 Conceptos clave (por qué se hacen las cosas así)

### ¿Por qué buffers separados por segmento?

Un solo buffer general mezcla problemas distintos: tráfico, espera en dock, paperwork. Cada uno tiene causas diferentes y se afina por separado. Detalle en [`decisiones/0002-modelo-de-buffers.md`](../decisiones/0002-modelo-de-buffers.md).

```
buffer_carga       → tiempo en pickup (dock, dock workers, paperwork)
buffer_carretera   → tiempo en ruta (tráfico, clima, fuel stops)
buffer_descarga    → tiempo en delivery (~80% del de carga)
```

### ¿Por qué trabajar en minutos?

Google Maps devuelve segundos, los humanos hablan en horas, pero las matemáticas necesitan una unidad consistente. Minutos es el "mediano" que funciona para los 3. Detalle en [`decisiones/0001-minutos-en-vez-de-horas.md`](../decisiones/0001-minutos-en-vez-de-horas.md).

```
14209s → 237 minutos → "casi 4 horas" (para hablar)
```

### ¿Por qué `parseDate()` el pickup_time?

Vapi manda `"2026-05-15T14:00:00"` como **string**. Si lo comparas como string, `"2026-05-15"` > `"2026-05-09"` da resultados raros porque compara caracter por caracter. `parseDate()` lo convierte a un objeto Date real y las comparaciones funcionan bien.

### ¿Por qué dividir metros y segundos de Google Maps?

Google Routes API devuelve:
- Distancia en **metros** (no millas)
- Duración en **segundos** (no minutos)

Hay que convertir SIEMPRE al salir:
```
millas    = metros / 1609.34
minutos   = segundos / 60
```

Si no lo haces, `costo_millas_cargadas` da $1,160,460 en vez de $720.

### ¿Por qué `Factor_Peaje`?

Google Maps estima peajes para **carro normal**. Camiones grandes pagan más (a veces el doble). El factor ajusta el estimado al tamaño real del camión:

```
costo_peajes_real = peaje_google × factor_peaje
                    ↓
              Box truck → 1.2x
              Semi      → 2.0x
```

### ¿Por qué buscar Activo Y En_Descanso al mismo tiempo?

Si filtras solo "Activo", pierdes a conductores que terminan su descanso a las 6pm — perfectos para un pickup a las 7pm. El campo `Disponible_Desde_Calculado` en Operadores resuelve esto:

```
Activo            → disponible NOW
En_Descanso       → disponible (Hora_Final_Turno + 10h DOT reset)
Inactivo/vacaciones → disponible en 365 días (= nunca)
```

### ¿Por qué el score en vez de solo "el más cercano"?

Un conductor cercano pero con poco HOS o que va a romper su próxima cita es peor que uno un poco más lejos pero descansado. El score combina varios factores:

```
score = 1000
      - deadhead_minutos          (mientras menos vacío, mejor)
      + Minutos_Disponibles_HOS / 10   (más margen HOS, mejor)
      + bonus si Activo (vs En_Descanso)
```

### ¿Por qué validar `puede_regresar`?

Aceptar un viaje sin pensar en el regreso lleva a:
- Conductor varado lejos de casa
- HOS rota antes de poder regresar
- Camión "perdido" hasta el día siguiente

`puede_regresar = (pickup + jornada + regreso) <= Hora_Final_Turno` evita esto.

### ¿Por qué un Array Aggregator después del Iterator?

El Iterator procesa **uno a uno** los candidatos. Sin el Aggregator, solo verías el último. El Aggregator junta TODOS en un array para poder elegir el de mayor score con `max()`.

### ¿Por qué dividir el costo de deadhead a la mitad?

El deadhead son millas VACÍAS — no transportas carga, no generas valor, pero gastas combustible. Cobrar a tarifa completa = inflar el precio. No cobrar nada = perder dinero. La convención de la industria es 50% de la tarifa cargada.

### ¿Por qué un Rollup para `Minutos_Consumidos_Hoy`?

Si lo llenaras a mano cada vez que se crea una cita, tarde o temprano lo olvidas. El Rollup lee automáticamente las citas enlazadas al operador (via el link bidireccional `Citas-asignados`) y suma sus minutos. No requiere trabajo manual ni Make.

### ¿Por qué el filtro de conflictos en Airtable y no en Make?

Si lo pones en Make, el dispatcher humano puede crear conflictos desde su celular y Make no se entera. La Automation de Airtable corre **siempre** — para la IA y para humanos. La lógica vive con los datos.

---

## 🔌 Airtable — tablas usadas

### `Operadores`
- `Estado_del_Conductor` → Activo / En_Descanso / Inactivo
- `Camion_Asignado` → link a Inventario de Flota
- `Minutos_Disponibles_HOS` → Formula: `660 - Minutos_Consumidos_Hoy`
- `Minutos_Consumidos_Hoy` → Rollup de citas del día
- `Hora_Final_Turno` → cuándo termina su jornada
- `Disponible_Desde_Calculado` → Formula con DOT reset
- `Citas-asignados` → link bidireccional con Bookings

### `Inventario de Flota`
- `Nombre_de_Unidad`
- `truck_type` → box_truck / van
- `estado` → Disponible / Mantenimiento
- `Ubicacion_Operacional_Actual`
- `Base_Operacional`
- `Tarifa_Milla` → $2.50 / $3.50 según tamaño
- `Factor_Peaje` → 1.2 / 2.0

### `Bookings and Operations`
- `Operadores asignado` → link a Operadores (bidireccional)
- `Vehicle_assigned` → link a Inventario de Flota
- `Hora_Inicio`, `Hora_Fin_Estimada`
- `Minutos_Consumidos` → Formula: `DATETIME_DIFF(fin, inicio, 'minutes')`
- `Parada_final`
- `Trip-Status`
- `Estado_Temporal_Cita` → Pasada / Actual / Futura
- `Conflicto_Detectado` → Checkbox (lo llena la Automation)
- `Mensaje_Conflicto` → texto del conflicto detectado

### `Servicios_Especiales` (recomendada)
- `nombre_servicio` → white_glove, inside_delivery, etc.
- `costo_servicio` → $ por servicio
- `activo` → si está vigente

---

## 🤖 Airtable Automations relacionadas

### Auto 1 — Detector de conflictos de horario

```
Trigger: Bookings, when record matches conditions
         (Operadores asignado, Hora_Inicio, Hora_Fin_Estimada no vacíos)

Action: Run a script
        - Lee todas las citas del mismo operador
        - Compara overlaps de tiempo
        - Marca Conflicto_Detectado y Mensaje_Conflicto

Side effect: color condicional rojo en la vista
```

### Auto 2 — Actualizar ubicación al terminar viaje (sugerida)

```
Trigger: Trip-Status cambia a "Delivered"
Action: Update Inventario_de_Flota
        - Ubicacion_Operacional_Actual = Parada_final
        - Disponible_Desde = Hora_Fin_Estimada
```

---

## 🚨 Errores comunes y cómo se resolvieron

### "score_candidato me da el mismo número para todos"
Causa: el score no incluía nada que dependiera del operador específico.
Fix: agregar `Minutos_Disponibles_HOS / 10` para que cada operador tenga su propio bonus.

### "buffer_carga_pallets me da 0"
Causa: `cantidad_pallets` llegaba como texto.
Fix: `parseNumber(cantidad_pallets) * 2`.

### "Array Aggregator no me agrega nada"
Causa: el Source Module estaba mal configurado.
Fix: en el Aggregator, "Source Module" debe ser el Iterator que tiene los candidatos.

### "Eta_llegada_pickup calcula con el tiempo total"
Causa: se sumaba `tiempo_total_operacional` en vez de solo `deadhead_minutos`.
Fix: `eta = hora_disponible_camion + deadhead_minutos` (NO incluye carga ni viaje).

### "Camion_ganador muestra varias operaciones"
Causa: el Set Variable estaba ANTES del Array Aggregator.
Fix: moverlo DESPUÉS del Aggregator para que vea el array completo.

### "costo_millas_cargadas = $1,160,460"
Causa: Google Maps devuelve metros, no millas.
Fix: convertir con `parseNumber(viaje_millas) / 1609.34`.

### "RuntimeError: Unknown field names 2026-06-09"
Causa: en el filtro de Airtable, la fecha del chip se inyecta sin envoltorio.
Fix: envolver con `DATETIME_PARSE("{{chip}}")`.

### "El campo Citas-asignados no se llena solo"
Causa: hay 2 link fields independientes (uno en cada tabla) en vez de 1 bidireccional.
Fix: borrar el manual y dejar solo el que Airtable creó automáticamente como reverse del link en Bookings.

---

## 🎯 JSON de respuesta a Vapi (Webhook Response)

```json
{
  "precio_a_cotizar": 940,
  "precio_minimo_aceptable": 880,
  "estado": "ACEPTAR",
  "camion_unidad": "BOX-001",
  "operador_nombre": "Juan García",
  "eta_pickup_legible": "1:45 PM",
  "millas_viaje": 285,
  "costo_peajes_total": 55,
  "puntos_defender": [
    "Peajes de Indiana suman $55",
    "8 pallets con forklift_dock"
  ]
}
```

Vapi lo usa para responder al broker:
> *"Tengo un Box Truck con Juan García, llega al pickup a la 1:45 PM. Mi tarifa para este viaje es $940."*

Si el broker objeta:
> *"Son $940 porque tienes peajes de $55 en Indiana. Puedo bajarlo a $890."*

---

## 📐 Principio arquitectónico

```
Airtable      guarda la verdad operacional (HOS, citas, ubicaciones).
Make          decide el camión y calcula precios.
Google Maps   calcula rutas, distancias y peajes.
Vapi          conversa con el broker.

Airtable lleva la lógica que afecta a TODOS (HOS, conflictos).
Make lleva la lógica que solo aplica al despacho via IA.
```

### Regla central

```
No preguntar "¿está libre el camión?"
Preguntar  "¿puede ejecutar esta carga sin romper restricciones operacionales?"
```

---

## 🔄 Cambios importantes

- **2026-06-12** — versión inicial. Documenta el escenario completo de despacho desde Vapi hasta la respuesta con precio total. Incluye:
  - Filtro de Activo + En_Descanso con `Disponible_Desde_Calculado`
  - Sistema de buffers separados (carga, carretera, descarga)
  - Score multifactor para elegir ganador
  - Cálculo de precios con Tarifa_Milla y Factor_Peaje
  - Validación de regreso a casa
  - Automation de detección de conflictos en Airtable

---

## ✅ Checklist de mantenimiento

Cuando algo se rompa, revisa en este orden:

```text
[ ] El webhook recibe las 12 variables de carga (revisar bundle de Vapi)
[ ] parseDate funciona en pickup_time
[ ] Airtable Search devuelve operadores (revisar fórmula de filtro)
[ ] Google Maps responde (revisar quota y API key)
[ ] Las conversiones metros→millas y segundos→minutos están aplicadas
[ ] El Iterator está conectado correctamente al Array Aggregator
[ ] El score se diferencia entre candidatos
[ ] Las tarifas en Airtable (Tarifa_Milla, Factor_Peaje) están actualizadas
[ ] Webhook Response devuelve JSON válido a Vapi
```
