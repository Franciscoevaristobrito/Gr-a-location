# Airtable — HOS automático (descuento sin intervención)

**Última actualización:** 2026-06-12
**Tablas afectadas:** `Operadores`, `Bookings and Operations`
**Tipo:** Formulas + Rollups (sin Automations)

---

## 🎯 Propósito

Cuando se crea o edita una cita en `Bookings`, los minutos consumidos deben restarse del HOS del operador **automáticamente**, sin que el dispatcher tenga que tocar la tabla de Operadores.

Antes:
```
Dispatcher crea cita → debe ir a Operadores → restar minutos a mano  ❌
```

Después:
```
Dispatcher crea cita → Airtable recalcula HOS solo                   ✅
```

Esto funciona para:
- Citas creadas via IA (Make)
- Citas creadas por dispatcher humano en celular/laptop
- Citas editadas (cambia hora → HOS recalcula)
- Citas eliminadas (libera HOS)

---

## 🧠 Por qué Formulas + Rollups en vez de Automation

| | Formulas / Rollups | Automation |
|---|---|---|
| Velocidad | Instantáneo | 1-5 segundos |
| Funciona offline | ✅ Sí | ❌ Requiere sync |
| Costo de runs | Cero | Cuenta cada disparo |
| Sincronización | Imposible perderla | Puede fallar |
| Mantenimiento | Una fórmula | Pasos + condiciones |

Para **derivar datos**, Formulas/Rollups siempre ganan. Las Automations son para **side effects** (notificaciones, llamar APIs, sincronizar con otras tablas).

---

## 🏗️ Setup completo — 4 piezas

### Pieza 1 — Link bidireccional Bookings ↔ Operadores

**En Bookings:**
```
Field name: Operadores asignado
Type: Link to another record
Linked table: Operadores
☑ Allow linking to multiple records (opcional)
```

**En Operadores:** Airtable crea automáticamente el campo inverso. Lo renombras:
```
Field name: Citas-asignados
Type: Link to another record (auto, viene del link en Bookings)
```

> ⚠️ **NO crear** el campo `Citas-asignados` manualmente. Si lo haces, son 2 links separados que NO se sincronizan. Debe ser el reverse automático del link en Bookings.

### Pieza 2 — `Minutos_Consumidos` en Bookings (Formula)

Calcula automáticamente los minutos de cada cita:

```
Field name: Minutos_Consumidos
Field type: Formula
Formula:
   IF(
     AND(Hora_Inicio, Hora_Fin_Estimada),
     DATETIME_DIFF(Hora_Fin_Estimada, Hora_Inicio, 'minutes'),
     0
   )
```

Lógica:
- Si tiene ambas horas → calcular diferencia en minutos.
- Si falta alguna → 0 (no cuenta como tiempo consumido).

Ejemplo: cita de 8:00 AM a 11:00 AM → `Minutos_Consumidos = 180`.

### Pieza 3 — `Minutos_Consumidos_Hoy` en Operadores (Rollup)

Suma automáticamente los minutos de TODAS las citas del operador para HOY:

```
Field name: Minutos_Consumidos_Hoy
Field type: Rollup
Link field:      Citas-asignados
Field to rollup: Minutos_Consumidos
Aggregation:     SUM(values)
```

**Filtro del Rollup (importante):**

Click en "Add conditions" y agrega:
```
Only include linked records where:
   IS_SAME({Hora_Inicio}, TODAY(), 'day') = TRUE
```

Esto asegura que solo suma las citas de HOY, no todo el histórico.

### Pieza 4 — `Minutos_Disponibles_HOS` en Operadores (Formula)

El cálculo final que Make consume al filtrar candidatos:

```
Field name: Minutos_Disponibles_HOS
Field type: Formula
Formula:
   MAX(0, 660 - {Minutos_Consumidos_Hoy})
```

- **660 minutos** = 11 horas × 60 min (límite DOT de manejo por día).
- **MAX(0, ...)** evita números negativos cuando un operador se pasa.

---

## 🎬 Cómo se ve en la práctica

```
Dispatcher abre Bookings → crea cita
   ↓ Operador: Daniel cedillo, 9am-1pm (4 horas)
   ↓ Click Save
                                    ⏱ instantáneo
                                    
Airtable calcula automáticamente:
   1. Bookings.Minutos_Consumidos = 240   (Formula)
   2. Operadores.Citas-asignados de Daniel se actualiza (link bidireccional)
   3. Operadores.Minutos_Consumidos_Hoy de Daniel suma +240 (Rollup)
   4. Operadores.Minutos_Disponibles_HOS de Daniel = 660 - 240 = 420 (Formula)
                                    
Make/Vapi consultan la próxima vez → ven HOS actualizado ✅
```

Cero clicks extra. Cero código en Make. Cero Automations.

---

## 🧪 Cómo probarlo

### Test 1 — HOS arranca completo

Crea un operador nuevo sin citas:
```
Esperado:
   Minutos_Consumidos_Hoy   = 0
   Minutos_Disponibles_HOS  = 660
```

### Test 2 — Crear cita resta minutos

Crea una cita: ese operador, hoy 8am-12pm (4 horas):
```
Esperado:
   Bookings.Minutos_Consumidos       = 240
   Operadores.Minutos_Consumidos_Hoy = 240
   Operadores.Minutos_Disponibles_HOS = 420
```

### Test 3 — Editar la cita actualiza HOS

Cambia la hora de fin a 1pm (ahora son 5 horas):
```
Esperado:
   Bookings.Minutos_Consumidos       = 300
   Operadores.Minutos_Consumidos_Hoy = 300
   Operadores.Minutos_Disponibles_HOS = 360
```

### Test 4 — Borrar la cita libera HOS

Elimina la cita:
```
Esperado:
   Operadores.Minutos_Consumidos_Hoy = 0
   Operadores.Minutos_Disponibles_HOS = 660
```

Si los 4 tests pasan → el sistema está funcionando perfecto.

---

## 🔧 Campo opcional — `Disponible_Desde_Calculado`

Para que el filtro de Make también considere conductores en descanso:

```
Field name: Disponible_Desde_Calculado
Field type: Formula
Output type: Date with time
Formula:
   IF(
     Estado_del_Conductor = "Activo",
     NOW(),
     IF(
       AND(
         Estado_del_Conductor = "En_Descanso",
         Hora_Final_Turno
       ),
       DATEADD(Hora_Final_Turno, 10, 'hours'),
       DATEADD(NOW(), 365, 'days')
     )
   )
```

Lógica:
- **Activo** → disponible NOW
- **En_Descanso + tiene Hora_Final_Turno** → disponible 10h después del fin del turno (DOT reset)
- **Cualquier otro caso** → disponible en 365 días (= nunca lo elige el filtro)

Esto permite que Make encuentre conductores que terminarán su descanso justo a tiempo para el pickup.

---

## 🚨 Errores comunes

### "Minutos_Consumidos_Hoy suma citas antiguas"

Causa: el Rollup no tiene filtro de fecha.
Fix: agregar `IS_SAME({Hora_Inicio}, TODAY(), 'day') = TRUE` en las condiciones del Rollup.

### "Minutos_Disponibles_HOS da negativo"

Causa: la fórmula no tiene `MAX(0, ...)`.
Fix: envolver con `MAX(0, 660 - {Minutos_Consumidos_Hoy})`.

### "Citas-asignados está vacío aunque hay citas"

Causa: hay 2 link fields separados (uno en cada tabla) en vez de 1 bidireccional.
Fix: borrar el campo manual y dejar solo el reverse automático del link en Bookings.

### "El Rollup no actualiza al cambiar horas"

Esto es muy raro pero pasa cuando el campo `Minutos_Consumidos` en Bookings está congelado.
Fix: borrar el campo y recrearlo como Formula (no como Number manual).

### "Make sigue viendo el HOS viejo"

Causa: Make tiene un caché de los datos de Airtable de la última corrida.
Fix: el Search Records de Make siempre lee datos frescos. Verifica que no estés usando un módulo "Get Record" con un ID guardado.

---

## 🎁 Beneficios adicionales (gratis con esta arquitectura)

Una vez que tienes esto montado, **automáticamente ganas**:

1. **Análisis histórico:** puedes hacer una vista en Bookings agrupada por operador para ver consumo de HOS por semana/mes.

2. **Cumplimiento DOT:** un campo formula extra te puede alertar si un operador se pasa:
   ```
   IF(Minutos_Consumidos_Hoy > 660, "⚠️ VIOLACIÓN HOS", "✅ OK")
   ```

3. **Reportes:** los datos están limpios y agregados, listos para exportar a Excel o conectar con Looker/Tableau.

4. **Multi-dispositivo:** funciona igual en la app de celular que en la web. Mismas fórmulas, mismos resultados.

---

## 🔗 Cómo Make consume estos campos

En el escenario de despacho:

```
Airtable Search Records → Operadores
Filtro:
   AND(
     OR({Estado_del_Conductor} = "Activo", {Estado_del_Conductor} = "En_Descanso"),
     IS_BEFORE({Disponible_Desde_Calculado}, DATETIME_PARSE("{{pickup_time_formateado}}")),
     {Minutos_Disponibles_HOS} >= {{hos_requerido_total}}
   )
```

Make solo **LEE** estos campos. Nunca los **escribe**. Esa es la separación de responsabilidades:

- Airtable calcula HOS (Formulas + Rollups).
- Make decide qué operador usar (basado en lo que lee).

---

## 🔄 Cambios

- **2026-06-12** — versión inicial. Reemplaza el flujo manual donde había que actualizar el HOS desde 2 sitios (Bookings + Operadores).
