# Automation — Detector de conflictos de citas

**Última actualización:** 2026-06-12
**Tabla principal:** `Bookings and Operations`
**Tipo:** Airtable Automation con Run a script
**Estado:** Activa

---

## 🎯 Propósito

Detectar cuando un dispatcher (humano o IA) crea/edita una cita que choca con otra cita del mismo operador en horarios traslapados. Marca la cita en rojo con un mensaje explicativo.

```
NO bloquea la creación (Airtable no permite eso desde la UI).
SÍ avisa visualmente con color y mensaje, imposible de ignorar.
```

---

## 🧠 Por qué vive en Airtable y no en Make

Si la lógica de conflictos viviera en Make:

```
IA crea cita via Make → ✅ se detecta conflicto
Humano crea cita en celular → ❌ Make ni se entera
```

Al vivir en Airtable Automation:

```
IA crea cita via API → ✅ Automation dispara
Humano crea cita en celular → ✅ Misma Automation dispara
Humano edita cita existente → ✅ También dispara
```

**Regla:** la lógica que afecta a TODOS los caminos de entrada debe vivir lo más cerca de los datos posible.

---

## 🏗️ Setup completo

### 1. Campos nuevos en `Bookings and Operations`

| Campo | Tipo | Para qué |
|---|---|---|
| `Conflicto_Detectado` | Checkbox | Bandera roja/verde |
| `Mensaje_Conflicto` | Long text | Mensaje legible al dispatcher |

### 2. Color condicional en la vista

En la vista de Bookings:
```
Click "Color" → Add condition:

WHEN: Conflicto_Detectado = checked
COLOR: Red (intenso)
```

Resultado: la fila ENTERA se pinta de rojo cuando hay conflicto. Imposible no verlo.

### 3. La Automation

#### Trigger
```
When record matches conditions
Table: Bookings and Operations
Conditions (todas con AND):
   - Operadores asignado    is not empty
   - Hora_Inicio            is not empty
   - Hora_Fin_Estimada      is not empty
```

> **Importante:** "When record matches conditions" dispara tanto al **crear** como al **editar**. Si usas "When record is created" solo dispara al crear (no detecta cambios posteriores).

#### Action 1 — Run a script

**Input variables:**

| Name | Value (arrastrar del trigger) |
|---|---|
| `recordId` | Airtable Record ID |
| `operador` | Nombre del Operador (from Operadores asignado) |
| `horaInicio` | Hora_Inicio |
| `horaFin` | Hora_Fin_Estimada |

**Código JavaScript:**

```javascript
const inputConfig = input.config();
const recordId = inputConfig.recordId;
const operador = inputConfig.operador;
const horaInicio = new Date(inputConfig.horaInicio);
const horaFin = new Date(inputConfig.horaFin);

const table = base.getTable('Bookings and Operations');
const query = await table.selectRecordsAsync({
    fields: ['Operadores asignado', 'Hora_Inicio', 'Hora_Fin_Estimada', 'Company Name']
});

let conflictos = [];

for (const record of query.records) {
    if (record.id === recordId) continue;

    const otrosOperadores = record.getCellValue('Operadores asignado') || [];
    const mismoOperador = otrosOperadores.some(op => op.name === operador);
    if (!mismoOperador) continue;

    const otraInicio = record.getCellValue('Hora_Inicio');
    const otraFin = record.getCellValue('Hora_Fin_Estimada');
    if (!otraInicio || !otraFin) continue;

    const inicio = new Date(otraInicio);
    const fin = new Date(otraFin);

    const hayOverlap = inicio < horaFin && fin > horaInicio;
    if (hayOverlap) {
        conflictos.push(record.getCellValue('Company Name') || record.name);
    }
}

output.set('tieneConflicto', conflictos.length > 0);
output.set('mensajeConflicto', conflictos.length > 0
    ? '⚠️ Conflicto con: ' + conflictos.join(', ')
    : ''
);
```

#### Action 2 — Update record

```
Record ID: [chip: Trigger.Airtable Record ID]

Fields to update:
   Conflicto_Detectado     → [chip: Run script output → tieneConflicto]
   Mensaje_Conflicto       → [chip: Run script output → mensajeConflicto]
```

---

## 📐 La lógica del overlap (importante de entender)

Dos rangos de tiempo se traslapan si:

```
Cita A:  ████████████████
              ↑          ↑
          A.inicio    A.fin

Cita B:                ████████████
                       ↑          ↑
                    B.inicio    B.fin

Hay overlap si:
   A.inicio < B.fin    ✅ (A empieza antes que B termine)
   Y A.fin > B.inicio  ✅ (A termina después que B empiece)
```

En el script:
```javascript
const hayOverlap = inicio < horaFin && fin > horaInicio;
```

Esta fórmula es **estándar de la industria** para detección de overlaps. Sirve para citas, salas de reunión, reservas de cualquier cosa.

---

## ⚠️ Por qué NO se pudo usar la UI visual

La UI visual de Airtable Automations **no permite comparar dos campos de fecha entre sí**. Solo permite comparar contra una fecha fija ("one week from now", "exact date", etc.). Por eso necesitamos `Run a script` — JavaScript SÍ puede comparar fechas dinámicamente.

---

## 🧪 Cómo probarlo

### Escenario 1 — Sin conflicto

1. Crea una cita: Daniel cedillo, hoy 9am - 11am.
2. Esperado: `Conflicto_Detectado = false`, fila normal.

### Escenario 2 — Con conflicto

1. Sin borrar la anterior, crea otra cita: Daniel cedillo, hoy 10am - 12pm.
2. Esperado: `Conflicto_Detectado = true`, fila **roja**, mensaje:
   `⚠️ Conflicto con: [nombre de la primera cita]`

### Escenario 3 — Cambio que resuelve el conflicto

1. Cambia la segunda cita a hoy 12pm - 2pm.
2. Esperado: la fila vuelve a normal en ~5 segundos (la Automation se vuelve a disparar porque cambió `Hora_Inicio`).

---

## 🔔 Mejoras opcionales

### Notificación a Slack/Email

Agrega otra Action después del Update Record en la rama de conflicto:

```
Send Slack message
Channel: #dispatch-alerts
Message:
   "🚨 Conflicto detectado
    Cita: {{Trigger.Company Name}}
    Operador: {{Trigger.Operadores asignado}}
    Choca con: {{Script.mensajeConflicto}}"
```

### Bloqueo real con Interface Form

Si quieres que el sistema **literalmente no permita guardar** una cita conflictiva:

1. Crea una Interface tipo **Form**.
2. Dispatchers solo crean citas desde ahí.
3. Agrega un **Script element** que valide ANTES de guardar.

Más complejo pero es el único bloqueo verdadero en Airtable.

---

## 🚨 Errores comunes

### "Find records con date no acepta valor dinámico"

Por eso usamos Run a script. La UI visual no permite comparar `Hora_Inicio` contra el trigger.

### "El script tarda mucho"

Si tu tabla `Bookings` tiene miles de registros, el script puede tardar. Optimización: filtra primero por operador en `selectRecordsAsync`:

```javascript
const query = await table.selectRecordsAsync({
    fields: ['Operadores asignado', 'Hora_Inicio', 'Hora_Fin_Estimada', 'Company Name'],
    sorts: [{field: 'Hora_Inicio', direction: 'desc'}]
});
```

O mejor: limita a citas de los últimos 7 días + futuras (no necesitas revisar las del año pasado).

### "El campo Operadores asignado es array y no compara bien"

Por eso en el script usamos `.some(op => op.name === operador)`. El campo es un array de objetos `{id, name}`, no un string simple.

### "La Automation no se dispara al editar"

Probablemente tu trigger es "When a record is created" en vez de "When record matches conditions". Cámbialo.

---

## 🔗 Relación con el resto del sistema

```
Bookings tiene Conflicto_Detectado y Mensaje_Conflicto
   ↓
Cuando un dispatcher ve una fila roja
   ↓
Sabe que debe cambiar operador o ajustar hora
   ↓
Make/Vapi también puede consultar este campo antes de cotizar
   ("si la cita propuesta generaría conflicto, no la ofrezcas")
```

---

## 🔄 Cambios

- **2026-06-12** — versión inicial. Implementación con Run a script porque la UI visual no soporta comparación dinámica entre fechas.
