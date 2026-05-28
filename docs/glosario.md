# Glosario — AI Dispatch Brain

Diccionario único de variables y términos. Si dudas en cómo se llama algo, miras aquí.

> Regla: **una variable se nombra una sola vez**. Si la cambias, la cambias en todos lados.

---

## Variables del webhook Vapi → Make

Esquema completo en [`webhook-vapi-schema.md`](./webhook-vapi-schema.md). Resumen:

### Core (Tier 1 — siempre presentes)

| Nombre | Tipo | Valores | Significado |
|---|---|---|---|
| `tipo_camion` | enum | `box_truck` \| `van` | Vehículo solicitado |
| `pickup_city` | string | - | Ciudad de recogida |
| `pickup_time` | datetime ISO | - | Hora de recogida solicitada |
| `delivery_city` | string | - | Ciudad de entrega |
| `tipo_carga` | enum | `palletized` \| `boxes` \| `loose` \| `fragile` \| `refrigerated_small` | Qué se transporta |
| `tipo_recogida` | enum | `live_load` \| `wait_load` | Modo de carga |
| `cantidad_pallets` | number | 0-12 (Box) / 0-6 (Van) | Pallets a transportar |
| `broker_nombre` | string | - | Quién pide la carga |
| `tarifa_ofrecida` | number USD | - | Cuánto paga |

### Operacionales (Tier 2)

| Nombre | Tipo | Significado |
|---|---|---|
| `metodo_carga` | enum: `forklift_dock` \| `liftgate` \| `manual` \| `driver_assist` | Cómo se carga físicamente |
| `requiere_liftgate` | boolean | ¿Necesita liftgate? |
| `numero_paradas` | number | 1 = directo, >1 = multi-stop |
| `cita_requerida_pickup` | boolean | ¿Trabaja con citas? |
| `peso_total_libras` | number | Peso de la carga |
| `delivery_time_requested` | datetime ISO | Hora deseada de entrega |

### Servicios especiales (Tier 3)

| Nombre | Tipo | Significado |
|---|---|---|
| `servicios_especiales` | array | `white_glove`, `inside_delivery`, `signature_required`, etc. |
| `hazmat` | boolean | Carga peligrosa |
| `refrigerado` | boolean | Temperatura controlada |
| `temperatura_requerida_f` | number | °F objetivo si refrigerado |

---

## Variables calculadas dentro del escenario de Make

| Nombre | Tipo | Origen | Significado |
|---|---|---|---|
| `pickup_time_formateado` | Date | `parseDate(pickup_time)` | Hora de pickup como Date real |
| `ubicacion_operacional_real` | string | Inventario de Flota / Bookings | Dónde está el camión operacionalmente |
| `deadhead_minutos` | número | Google Maps | Tiempo vacío hasta el pickup |
| `deadhead_millas` | número | Google Maps | Millas vacías hasta el pickup |
| `viaje_minutos` | número | Google Maps | Tiempo del viaje cargado |
| `viaje_millas` | número | Google Maps | Millas del viaje cargado |
| `buffer_carretera` | número | Tools | Margen por tráfico/clima según duración |
| `buffer_carga` | número | Tools | Margen por carga en pickup (basado en método + pallets + extras) |
| `buffer_descarga` | número | Tools | Margen por descarga en delivery |
| `tiempo_jornada_completa` | número | Tools | `deadhead + buffer_carga + viaje + buffer_carretera + buffer_descarga` |
| `hora_disponible_camion` | Date | Tools | Cuándo queda libre el camión |
| `eta_llegada_pickup` | Date | Tools | `hora_disponible_camion + deadhead_minutos` |
| `hora_estimada_entrega` | Date | Tools | `pickup_time + viaje_minutos + buffer_descarga` |
| `score` | número | Tools | Puntuación del candidato |

---

## Campos clave en Airtable

### Bookings and Operations
- `Vehicle_assigned` — link a Inventario de Flota
- `Operador_Asignado` — link a Operadores
- `Parada_final` — dónde termina operacionalmente el camión
- `Hora_Inicio`, `Hora_Fin_Estimada`
- `Estado_Temporal_Cita` — Pasada / Actual / Futura
- `Trip-Status`

### Operadores
- `Estado_del_Conductor` — Activo / Inactivo / etc.
- `Camion_Asignado` — link a Inventario de Flota
- `Minutos_Disponibles_HOS`
- `Hora_Inicio_Turno`, `Hora_Final_Turno`

### Inventario de Flota
- `truck_type`
- `estado`
- `Operador_Asignado` — link a Operadores
- `Ubicacion_Operacional_Actual`
- `Disponible_Desde`
- `Base_Operacional`

---

## Términos del dominio

| Término | Definición |
|---|---|
| **HOS** | Hours of Service. Horas legales de manejo de un conductor (DOT). |
| **Deadhead** | Tiempo o millas que recorre el camión vacío hasta el pickup. |
| **Buffer operacional** | Margen extra agregado al tiempo total para absorber retrasos. |
| **Parada final** | Punto real donde termina el camión después de una cita (puede ser delivery, base, hotel, truck stop). |
| **Ubicación operacional real** | Dónde está el camión disponible ahora mismo, según las reglas definidas. |
| **Cita actual** | La que está en curso (`Hora_Inicio <= NOW <= Hora_Fin_Estimada`). |
| **Cita futura** | La que comienza después de NOW. |
| **Score** | Puntuación de un candidato (camión) para decidir el mejor. |

---

## Convenciones

- **Unidades:** minutos para cálculos. Horas solo para mostrar a humanos.
- **Fechas:** siempre tipo Date dentro de Make, nunca string en comparaciones.
- **Nombres:** snake_case en variables de Make. PascalCase o `Espacios` en Airtable según ya esté establecido.
