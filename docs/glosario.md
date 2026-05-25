# Glosario — AI Dispatch Brain

Diccionario único de variables y términos. Si dudas en cómo se llama algo, miras aquí.

> Regla: **una variable se nombra una sola vez**. Si la cambias, la cambias en todos lados.

---

## Variables del escenario de Make

| Nombre | Tipo | Origen | Significado |
|---|---|---|---|
| `pickup_city` | string | Vapi (webhook) | Ciudad de recogida solicitada |
| `delivery_city` | string | Vapi (webhook) | Ciudad de entrega solicitada |
| `pickup_time` | string | Vapi (webhook) | Hora de pickup tal como llega (texto) |
| `pickup_time_formateado` | Date | `parseDate(pickup_time)` | Hora de pickup convertida a fecha real |
| `truck_type` | string | Vapi (webhook) | Tipo de camión solicitado |
| `ubicacion_operacional_real` | string | Inventario de Flota / Bookings | Dónde está el camión operacionalmente |
| `deadhead_minutos` | número | Google Maps | Tiempo vacío hasta el pickup, en minutos |
| `deadhead_millas` | número | Google Maps | Millas vacías hasta el pickup |
| `viaje_minutos` | número | Google Maps | Tiempo del viaje cargado, en minutos |
| `viaje_millas` | número | Google Maps | Millas del viaje cargado |
| `buffer_operacional` | número | Tools (Set variable) | Margen según duración del viaje |
| `tiempo_total_operacional` | número | Tools | `deadhead + viaje + buffer` |
| `hora_disponible_camion` | Date | Tools | Cuándo queda libre el camión |
| `eta_llegada_pickup` | Date | Tools | `hora_disponible_camion + deadhead_minutos` |
| `hora_estimada_entrega` | Date | Tools | `pickup_time + viaje_minutos + buffer` |
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
