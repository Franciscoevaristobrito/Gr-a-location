# Webhook Vapi → Make — Schema de variables

Contrato oficial entre Vapi (IA de voz) y Make. Define qué información debe extraer Vapi de la conversación con el broker y enviar a Make para ejecutar el despacho.

**Vehículos soportados:** Box Truck (16-26 ft) y Cargo Van (Sprinter, 14 ft).

---

## Estructura general

```json
{
  "core": { ... },           // datos básicos del viaje
  "carga": { ... },          // qué se transporta
  "operacional": { ... },    // cómo se carga/descarga
  "servicios_especiales": { ... },
  "broker": { ... }          // quién lo contrata
}
```

---

## 🟢 TIER 1 — Mínimo viable (Vapi DEBE preguntar siempre)

Sin estos datos no se puede despachar. Vapi no debe terminar la llamada sin obtenerlos.

| Variable | Tipo | Valores posibles | Descripción | Por qué importa |
|---|---|---|---|---|
| `tipo_camion` | enum | `"box_truck"` \| `"van"` | Vehículo solicitado por el broker | Filtra la flota: solo se buscan camiones de ese tipo |
| `pickup_city` | string | Ciudad + estado, ej. `"Detroit, MI"` | Origen | Calcula deadhead y ruta |
| `pickup_time` | datetime ISO | `"2026-05-15T14:00:00"` | Hora de recogida solicitada | Compara contra `eta_llegada_pickup` |
| `delivery_city` | string | Ciudad + estado | Destino | Calcula viaje principal |
| `tipo_carga` | enum | `"palletized"` \| `"boxes"` \| `"loose"` \| `"fragile"` \| `"refrigerated_small"` | Qué se transporta | Determina compatibilidad con vehículo y buffer de carga |
| `tipo_recogida` | enum | `"live_load"` \| `"wait_load"` | Modo de recogida | En Box/Van no hay drop & hook. Cambia el buffer |
| `cantidad_pallets` | number | 0-12 (Box Truck) / 0-6 (Van) | Pallets a transportar (0 si no es palletizado) | Buffer de carga + capacidad del vehículo |
| `broker_nombre` | string | Nombre de la persona o empresa | Quién pide la carga | Trazabilidad y contacto |
| `tarifa_ofrecida` | number (USD) | ej. `850` | Cuánto paga el broker | Cálculo de rentabilidad y score |

---

## 🟡 TIER 2 — Recomendado (Vapi pregunta si la conversación lo permite)

Mejoran mucho la precisión pero no bloquean el despacho.

| Variable | Tipo | Valores posibles | Descripción | Por qué importa |
|---|---|---|---|---|
| `pickup_address` | string | Dirección completa | Dirección exacta de recogida | Google Maps más preciso que solo ciudad |
| `delivery_address` | string | Dirección completa | Dirección exacta de entrega | Igual |
| `delivery_time_requested` | datetime ISO | `"2026-05-15T20:00:00"` | Hora deseada de entrega | Valida si llegamos a tiempo al delivery |
| `metodo_carga` | enum | `"forklift_dock"` \| `"liftgate"` \| `"manual"` \| `"driver_assist"` | Cómo se carga físicamente | Buffer de carga (forklift es rápido, manual es lento) |
| `requiere_liftgate` | boolean | `true` \| `false` | ¿El camión necesita liftgate? | Box Truck normalmente tiene, Van no. Filtra flota |
| `numero_paradas` | number | 1, 2, 3... | Cuántas paradas tiene el viaje | Multi-stop suma 30-45 min por parada |
| `cita_requerida_pickup` | boolean | `true` \| `false` | ¿El shipper trabaja con citas? | Si no hay cita, suma buffer de espera |
| `peso_total_libras` | number | ej. `4500` | Peso de la carga | Capacidad del vehículo (Box ~10,000 lb, Van ~3,000 lb) |
| `broker_telefono` | string | E.164, ej. `"+13135551234"` | Para confirmar o aclarar | Contacto operacional |

---

## 🔵 TIER 3 — Opcional (solo si aplica)

Servicios especiales que cambian la viabilidad o el precio.

| Variable | Tipo | Valores posibles | Descripción | Por qué importa |
|---|---|---|---|---|
| `servicios_especiales` | array | `["white_glove", "inside_delivery", "signature_required", "appointment_only"]` | Servicios extra solicitados | Cada uno suma tiempo al buffer |
| `hazmat` | boolean | `true` \| `false` | ¿Carga peligrosa? | Requiere certificación del conductor + paperwork |
| `hazmat_clase` | string | `"3"`, `"8"`, `"9"`, etc. | Clase DOT si hazmat = true | No todos los conductores califican |
| `refrigerado` | boolean | `true` \| `false` | ¿Requiere temperatura controlada? | Filtra solo vehículos con reefer |
| `temperatura_requerida_f` | number | ej. `38` | °F objetivo si refrigerado | Validación pre-trip |
| `referencia_carga` | string | PO #, BOL #, etc. | Identificador del broker | Trazabilidad |
| `broker_email` | string | email válido | Para confirmación escrita | Documentación del trato |
| `comentarios` | string | texto libre | Cualquier nota del broker | Información cualitativa |

---

## 📦 Ejemplo de payload completo (Box Truck)

```json
{
  "core": {
    "tipo_camion": "box_truck",
    "pickup_city": "Detroit, MI",
    "pickup_address": "1234 Industrial Dr, Detroit, MI 48201",
    "pickup_time": "2026-05-15T14:00:00",
    "delivery_city": "Chicago, IL",
    "delivery_address": "5678 Warehouse Ave, Chicago, IL 60601",
    "delivery_time_requested": "2026-05-15T20:00:00"
  },
  "carga": {
    "tipo_carga": "palletized",
    "cantidad_pallets": 8,
    "peso_total_libras": 4500
  },
  "operacional": {
    "tipo_recogida": "live_load",
    "tipo_entrega": "live_unload",
    "metodo_carga": "forklift_dock",
    "requiere_liftgate": false,
    "numero_paradas": 1,
    "cita_requerida_pickup": true,
    "cita_requerida_delivery": false
  },
  "servicios_especiales": {
    "lista": [],
    "hazmat": false,
    "refrigerado": false
  },
  "broker": {
    "broker_nombre": "ABC Logistics",
    "broker_telefono": "+13135551234",
    "tarifa_ofrecida": 850,
    "referencia_carga": "PO-78421"
  }
}
```

---

## 📦 Ejemplo de payload completo (Van — expedited)

```json
{
  "core": {
    "tipo_camion": "van",
    "pickup_city": "Toledo, OH",
    "pickup_time": "2026-05-15T10:00:00",
    "delivery_city": "Cleveland, OH",
    "delivery_time_requested": "2026-05-15T14:00:00"
  },
  "carga": {
    "tipo_carga": "boxes",
    "cantidad_pallets": 0,
    "peso_total_libras": 800
  },
  "operacional": {
    "tipo_recogida": "live_load",
    "tipo_entrega": "live_unload",
    "metodo_carga": "manual",
    "requiere_liftgate": false,
    "numero_paradas": 1,
    "cita_requerida_pickup": false
  },
  "servicios_especiales": {
    "lista": ["signature_required"],
    "hazmat": false,
    "refrigerado": false
  },
  "broker": {
    "broker_nombre": "FastFreight Inc",
    "broker_telefono": "+14195551234",
    "tarifa_ofrecida": 350
  }
}
```

---

## 🚛 Compatibilidad vehículo ↔ carga

Esto define **qué cargas puede tomar cada vehículo**. Make lo usa para filtrar antes de cualquier cálculo.

| Atributo | Box Truck | Van |
|---|---|---|
| Capacidad pallets | hasta 12 | hasta 6 |
| Capacidad peso | ~10,000 lb | ~3,000 lb |
| Tiene liftgate típicamente | ✅ sí (la mayoría) | ❌ no |
| Acepta `tipo_carga: palletized` | ✅ | ✅ (hasta 6) |
| Acepta `tipo_carga: boxes` | ✅ | ✅ |
| Acepta `tipo_carga: loose` | ✅ | ✅ |
| Acepta `tipo_carga: fragile` | ✅ | ✅ (ideal) |
| Acepta `tipo_carga: refrigerated_small` | ✅ (si tiene reefer) | ✅ (si tiene reefer) |
| Hazmat | depende del conductor | depende del conductor |

**Regla de filtro inicial en Make:**

```
SI tipo_camion = "van" Y cantidad_pallets > 6        → rechazar
SI tipo_camion = "van" Y peso_total_libras > 3000    → rechazar
SI tipo_camion = "van" Y requiere_liftgate = true    → rechazar
SI tipo_camion = "box_truck" Y cantidad_pallets > 12 → rechazar
SI tipo_camion = "box_truck" Y peso_total_libras > 10000 → rechazar
```

---

## ⏱️ Fórmula del buffer de carga

Lista para pegar en Make como Set variable `buffer_carga`:

```text
base_metodo + (pallets * minutos_por_pallet) + extras
```

**Base por método de carga:**

```text
if( metodo_carga = "forklift_dock";   20;
if( metodo_carga = "liftgate";        40;
if( metodo_carga = "manual";          60;
if( metodo_carga = "driver_assist";   90;
                                       45 ))))
```

**Minutos por pallet:**

```text
if( metodo_carga = "forklift_dock"; cantidad_pallets * 2;
if( metodo_carga = "liftgate";      cantidad_pallets * 3;
                                     cantidad_pallets * 5 ))
```

**Extras (sumar si aplican):**

```text
+ if( contains(servicios_especiales; "white_glove");     30; 0)
+ if( contains(servicios_especiales; "inside_delivery"); 20; 0)
+ if( contains(servicios_especiales; "appointment_only") AND cita_requerida_pickup = false; 30; 0)
+ if( hazmat = true; 30; 0)
+ if( refrigerado = true; 15; 0)
```

**Fórmula completa (resumida):**

```text
buffer_carga = base_metodo + minutos_por_pallet + extras_servicios
```

---

## ⏱️ Fórmula del buffer de descarga

Idéntica estructura pero ajustada — la descarga suele ser un poco más rápida:

```text
buffer_descarga = base_metodo + (pallets * minutos_por_pallet) + extras
```

Mismas reglas, pero la descarga toma típicamente **70-80%** del tiempo de carga. Si quieres simplificar:

```text
buffer_descarga = buffer_carga * 0.8
```

---

## 🛡️ Defaults — qué hacer si Vapi no recibe un campo

| Campo faltante | Default |
|---|---|
| `metodo_carga` | `"manual"` (asumir el peor caso) |
| `cantidad_pallets` | `0` |
| `peso_total_libras` | calcular: `cantidad_pallets * 1500` |
| `numero_paradas` | `1` |
| `cita_requerida_pickup` | `false` |
| `requiere_liftgate` | `true` si `tipo_carga = palletized`, si no `false` |
| `tipo_recogida` | `"live_load"` |
| `servicios_especiales.lista` | `[]` |
| `hazmat` | `false` |
| `refrigerado` | `false` |

---

## 🗣️ Preguntas que Vapi debe hacer (script sugerido)

Para que Vapi capture el Tier 1 completo:

```text
1. "¿Qué tipo de vehículo necesitas, Box Truck o Cargo Van?"
2. "¿De dónde sale la carga? Ciudad y estado."
3. "¿A dónde va? Ciudad y estado."
4. "¿A qué hora necesitas la recogida?"
5. "¿Qué tipo de carga es: pallets, cajas, carga suelta o frágil?"
6. "¿Cuántos pallets aproximadamente?" (si aplica)
7. "¿Es live load o el shipper espera al camión cargado?"
8. "¿Cuál es tu nombre y empresa?"
9. "¿Cuál es la tarifa que ofreces?"
```

Para Tier 2 (si la conversación fluye):

```text
10. "¿Hay cita programada en el pickup?"
11. "¿Necesitas liftgate?"
12. "¿Es una sola parada o multi-stop?"
13. "¿A qué hora necesitas la entrega?"
```

---

## Cambios

- **2026-05-23** — versión inicial. Diseñado para Box Truck y Cargo Van. No incluye semi-trucks.
