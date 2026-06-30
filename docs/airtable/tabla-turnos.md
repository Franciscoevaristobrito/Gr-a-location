# Airtable — Tabla Turnos (disponibilidad multi-día)

**Última actualización:** 2026-06-28
**Tablas relacionadas:** `Operadores`, `Bookings and Operations`
**Tipo:** Tabla con Formulas, Lookups y Rollups

---

## 🎯 Propósito

Proyectar el horario de cada operador a **fechas concretas** (hoy, mañana, pasado)
para poder ofrecer disponibilidad futura. Sin esta tabla, solo sabes si un camión
está libre HOY. Con ella, puedes decir: *"hoy está lleno, pero te lo doy mañana
a las 6 AM"*.

```
Operadores  → define el PATRÓN (trabaja 5 AM - 4 PM, lunes a viernes)
Turnos      → instancias CONCRETAS por día (con disponibilidad real)
Bookings    → las citas que ocupan espacio en cada turno
```

---

## 🧱 PARTE 1 — Preparar la tabla `Operadores`

Antes de crear Turnos, agrega 2 campos nuevos a `Operadores` para los días libres.

### Campo: `dias_laborales`

```text
Field name:  dias_laborales
Field type:  Multiple select
Opciones (créalas en INGLÉS para que coincida con las fórmulas):
   Monday
   Tuesday
   Wednesday
   Thursday
   Friday
   Saturday
   Sunday
```

Para cada operador, marca SOLO los días que trabaja. Ejemplo Juan (no trabaja
fines de semana):
```
☑ Monday  ☑ Tuesday  ☑ Wednesday  ☑ Thursday  ☑ Friday
☐ Saturday  ☐ Sunday
```

### Campo: `dias_libres_extra`

```text
Field name:  dias_libres_extra
Field type:  Long text
Contenido:   fechas puntuales separadas por coma (vacaciones, feriados)
             Formato: YYYY-MM-DD
Ejemplo:     2026-07-04,2026-07-15,2026-07-16
```

> Estos son días libres que NO son recurrentes (un feriado, un permiso).
> Los días libres recurrentes (fines de semana) van en `dias_laborales`.

### Campos que ya deben existir en Operadores

La tabla Turnos los va a leer vía Lookup, así que verifica que existan:
```
Hora_Inicio_Turno   (DateTime o Duration — la hora de empezar, ej. 5:00 AM)
Hora_Final_Turno    (DateTime o Duration — la hora de terminar, ej. 4:00 PM)
Camion_Asignado     (Link a Inventario de Flota)
Estado_del_Conductor (Single select: Activo / En_Descanso / Inactivo)
```

---

## 🧱 PARTE 2 — Crear la tabla `Turnos`

Airtable → Add or import → Create empty table → nómbrala `Turnos`.

Crea los campos EN ESTE ORDEN (algunos dependen de otros).

### Campo 1 — `operador` (el enlace base)

```text
Field name:  operador
Field type:  Link to another record
Linked table: Operadores
☐ Allow linking to multiple records  (déjalo en SINGLE — un turno = un operador)
```

### Campo 2 — `fecha`

```text
Field name:  fecha
Field type:  Date
Date format: ISO (2026-06-29) o el que prefieras
Include time: NO  (solo el día, sin hora)
```

### Campo 3 — `operador_hora_inicio` (Lookup)

```text
Field name:  operador_hora_inicio
Field type:  Lookup
Linked record field: operador
Field to look up:    Hora_Inicio_Turno
```

### Campo 4 — `operador_hora_fin` (Lookup)

```text
Field name:  operador_hora_fin
Field type:  Lookup
Linked record field: operador
Field to look up:    Hora_Final_Turno
```

### Campo 5 — `operador_dias_laborales` (Lookup)

```text
Field name:  operador_dias_laborales
Field type:  Lookup
Linked record field: operador
Field to look up:    dias_laborales
```

### Campo 6 — `camion` (Lookup)

```text
Field name:  camion
Field type:  Lookup
Linked record field: operador
Field to look up:    Camion_Asignado
```

### Campo 7 — `dia_semana` (Formula)

Saca el nombre del día de la semana a partir de la fecha.

```text
Field name:  dia_semana
Field type:  Formula
```
```
DATETIME_FORMAT({fecha}, 'dddd')
```
Devuelve "Monday", "Tuesday", etc. (en inglés, coincide con dias_laborales).

### Campo 8 — `es_dia_laboral` (Formula)

Verifica si ese día de la semana está entre los días que trabaja el operador.

```text
Field name:  es_dia_laboral
Field type:  Formula
```
```
IF(
  FIND(
    {dia_semana},
    ARRAYJOIN({operador_dias_laborales}, ",")
  ) > 0,
  TRUE(),
  FALSE()
)
```
- Si `dia_semana` = "Saturday" y NO trabaja sábados → FALSE
- Si `dia_semana` = "Monday" y SÍ trabaja lunes → TRUE

### Campo 9 — `hora_inicio` (Formula)

Combina la fecha del turno con la hora de inicio del operador.

```text
Field name:  hora_inicio
Field type:  Formula
```
```
DATETIME_PARSE(
  DATETIME_FORMAT({fecha}, 'YYYY-MM-DD') & ' ' &
  DATETIME_FORMAT({operador_hora_inicio}, 'HH:mm'),
  'YYYY-MM-DD HH:mm'
)
```

### Campo 10 — `hora_fin` (Formula)

```text
Field name:  hora_fin
Field type:  Formula
```
```
DATETIME_PARSE(
  DATETIME_FORMAT({fecha}, 'YYYY-MM-DD') & ' ' &
  DATETIME_FORMAT({operador_hora_fin}, 'HH:mm'),
  'YYYY-MM-DD HH:mm'
)
```

### Campo 11 — `minutos_turno_total` (Formula)

Duración total del turno en minutos.

```text
Field name:  minutos_turno_total
Field type:  Formula
```
```
DATETIME_DIFF({hora_fin}, {hora_inicio}, 'minutes')
```
Ejemplo: 5 AM a 4 PM = 660 minutos.

### Campo 12 — `citas_del_turno` (Link)

```text
Field name:  citas_del_turno
Field type:  Link to another record
Linked table: Bookings and Operations
☑ Allow linking to multiple records  (un turno puede tener varias citas)
```

### Campo 13 — `minutos_consumidos` (Rollup)

Suma los minutos de las citas enlazadas a este turno.

```text
Field name:  minutos_consumidos
Field type:  Rollup
Linked record field: citas_del_turno
Field to roll up:    Minutos_Consumidos
Aggregation:         SUM(values)
```

### Campo 14 — `minutos_disponibles` (Formula)

Lo que queda libre en el turno.

```text
Field name:  minutos_disponibles
Field type:  Formula
```
```
{minutos_turno_total} - {minutos_consumidos}
```

### Campo 15 — `estado` (Formula)

El campo que Make consulta para ofrecer disponibilidad.

```text
Field name:  estado
Field type:  Formula
```
```
IF(
  {es_dia_laboral} = FALSE(), "Día_Libre",
  IF(
    IS_BEFORE({hora_fin}, NOW()), "Pasado",
    IF(
      {minutos_disponibles} <= 30, "Lleno",
      "Disponible"
    )
  )
)
```

Orden de prioridad:
1. ¿Día libre? → "Día_Libre" (no se ofrece)
2. ¿Ya pasó? → "Pasado"
3. ¿Quedan ≤30 min? → "Lleno"
4. Si no → "Disponible" ✅

---

## 🔗 PARTE 3 — Conectar Bookings con Turnos

Para que el Rollup `minutos_consumidos` funcione, cada cita debe enlazarse a su turno.

### En la tabla `Bookings and Operations`

Al crear el link en el Campo 12, Airtable crea automáticamente el campo inverso
en Bookings. Renómbralo:

```text
Field name:  turno
Field type:  Link to another record (auto, viene de Turnos.citas_del_turno)
```

### Cómo se llena (en el Make scenario de confirmación)

```
[Create Booking]
      ↓
[Airtable Search en Turnos]
   Formula:
     AND(
       {operador} = "{{operador_asignado}}",
       IS_SAME({fecha}, {{fecha_del_pickup}}, 'day')
     )
   Maximum: 1
      ↓
[Update Booking]
   turno = {{turno encontrado}}
```

Al enlazar la cita al turno, el Rollup recalcula solo y `minutos_disponibles` baja.

---

## 🔄 PARTE 4 — Generar turnos automáticamente (Make scenario)

No crees turnos a mano. Un escenario programado los genera cada noche.

```
[Scheduler: cada noche 11 PM]
      ↓
[Airtable Search: Operadores]
   Formula: {Estado_del_Conductor} = "Activo"
      ↓
[Iterator por operador]
      ↓
[Repeater: 1 a 2]   ← genera mañana (+1) y pasado (+2)
      ↓
[Set var: fecha_objetivo = addDays(now; repeater.value)]
[Set var: dia_semana = formatDate(fecha_objetivo; "dddd")]
[Set var: fecha_texto = formatDate(fecha_objetivo; "YYYY-MM-DD")]
      ↓
[Filter: ¿debe trabajar ese día?]
   Condición 1: contains(operador.dias_laborales; dia_semana) = true
   Condición 2 (AND): NOT contains(operador.dias_libres_extra; fecha_texto)
      ↓ solo pasa si SÍ trabaja
[Airtable Search: ¿ya existe turno?]
   Formula:
     AND(
       {operador} = "{{operador.id}}",
       IS_SAME({fecha}, "{{fecha_texto}}", 'day')
     )
   Maximum: 1
      ↓
[Router]
   ├─ Si NO existe → [Airtable Create Record en Turnos]
   │     operador = operador actual
   │     fecha = fecha_objetivo
   └─ Si SÍ existe → no hacer nada (evita duplicados)
```

### Las 2 validaciones del Filter (días libres)

```
1. ¿Trabaja ese día de la semana?
   contains(dias_laborales; "Saturday") → si NO, no crear turno

2. ¿Tiene día libre puntual esa fecha?
   contains(dias_libres_extra; "2026-07-04") → si SÍ, no crear turno
```

Así nunca generas turnos en fines de semana (si no trabaja) ni en vacaciones.

---

## 🔍 PARTE 5 — Filtros / Vistas recomendadas

### Vista "Disponibles ahora"

```text
Filtro: estado = "Disponible"
Sort:   hora_inicio ascending
Para:   ver de un vistazo qué turnos se pueden ofrecer
```

### Vista "Por operador"

```text
Group by: operador
Sort:     fecha ascending
Para:     revisar la agenda de cada conductor
```

### Vista "Días libres"

```text
Filtro: estado = "Día_Libre"
Para:   verificar que los descansos se respetan
```

---

## 🔌 PARTE 6 — Cómo Make ofrece disponibilidad futura

En el escenario de cotización, si el pickup no cabe HOY:

```
[Airtable Search en Turnos]
   Formula:
     AND(
       {camion} = "...",   (o filtra por truck_type del camión)
       IS_AFTER({hora_inicio}, NOW()),
       {estado} = "Disponible",
       {minutos_disponibles} >= {{tiempo_jornada_completa}}
     )
   Sort:    hora_inicio ascending
   Maximum: 1
      ↓
[El primer resultado = la próxima disponibilidad]
      ↓
[Devolver a Vapi: fecha + hora legible]
```

Como el filtro pide `estado = "Disponible"`, automáticamente salta los días
"Día_Libre", "Pasado" y "Lleno". El primer turno que aparece es el que se ofrece.

---

## 🧪 PARTE 7 — Cómo probar

Crea 3 filas de prueba para Juan (que NO trabaja fines de semana):

| fecha | dia_semana | es_dia_laboral | estado esperado |
|---|---|---|---|
| Viernes 28 | Friday | TRUE | Disponible |
| Sábado 29 | Saturday | FALSE | Día_Libre |
| Lunes 1 | Monday | TRUE | Disponible |

Si los 3 dan el estado correcto, la lógica de días libres funciona.

Luego agrega una cita de prueba enlazada al turno del viernes y verifica que
`minutos_consumidos` sube y `minutos_disponibles` baja.

---

## ⚠️ Detalles importantes

### Idioma de los días

`DATETIME_FORMAT({fecha}, 'dddd')` devuelve los días en **inglés**. Por eso
`dias_laborales` debe usar opciones en inglés (Monday, Tuesday...) para que el
`FIND` coincida.

Si prefieres español en el Multiple Select, cambia `dia_semana` por:
```
SWITCH(
  WEEKDAY({fecha}),
  0, "Domingo", 1, "Lunes", 2, "Martes",
  3, "Miércoles", 4, "Jueves", 5, "Viernes", 6, "Sábado"
)
```
Y pon los días en español en `dias_laborales`. Lo crítico: que coincidan EXACTO.

### HOS se resetea entre turnos

Cada turno tiene sus propios `minutos_turno_total` (660 frescos). Cuando ofreces
mañana, el operador llega descansado — no arrastra el HOS gastado de hoy.

### No duplicar turnos

El Make scenario revisa "¿ya existe turno?" antes de crear. Si corre 2 veces el
mismo día, no genera duplicados.

---

## ✅ Checklist de creación

```text
OPERADORES (agregar):
[ ] Campo dias_laborales (Multiple select, días en inglés)
[ ] Campo dias_libres_extra (Long text)
[ ] Verificar Hora_Inicio_Turno, Hora_Final_Turno, Camion_Asignado existen

TURNOS (crear tabla):
[ ] 1. operador (Link a Operadores)
[ ] 2. fecha (Date sin hora)
[ ] 3. operador_hora_inicio (Lookup)
[ ] 4. operador_hora_fin (Lookup)
[ ] 5. operador_dias_laborales (Lookup)
[ ] 6. camion (Lookup)
[ ] 7. dia_semana (Formula)
[ ] 8. es_dia_laboral (Formula)
[ ] 9. hora_inicio (Formula)
[ ] 10. hora_fin (Formula)
[ ] 11. minutos_turno_total (Formula)
[ ] 12. citas_del_turno (Link a Bookings)
[ ] 13. minutos_consumidos (Rollup)
[ ] 14. minutos_disponibles (Formula)
[ ] 15. estado (Formula)

CONEXIÓN:
[ ] Renombrar el campo inverso en Bookings a "turno"

PRUEBA:
[ ] 3 filas de prueba (viernes/sábado/lunes) → estados correctos
[ ] 1 cita enlazada → minutos_disponibles baja

AUTOMATIZACIÓN:
[ ] Make scenario que genera turnos de los próximos 2 días cada noche
[ ] Filter que salta días libres (semana + puntuales)
```

---

## 🔄 Cambios

- **2026-06-28** — versión inicial. Tabla Turnos con soporte de días libres
  recurrentes (dias_laborales) y puntuales (dias_libres_extra) para ofrecer
  disponibilidad futura sin generar turnos en días de descanso.
