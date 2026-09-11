# Vapi — Variables de carga (solo lo esencial)

Variables enfocadas únicamente en **describir la carga**. Agrupadas por función operacional, no por "importancia genérica".

Todas las descripciones están en inglés porque los brokers hablan inglés. Listas para copiar y pegar en Vapi.

---

## 🎯 Filosofía de agrupación

```
ELEGIBILIDAD   → ¿podemos siquiera aceptar esta carga?
PLANIFICACIÓN  → ¿cuánto tiempo/buffer necesita?
CONDICIONAL    → solo si una pregunta anterior lo activa
```

Si falta una de **elegibilidad**, no podemos despachar — punto.
Si falta una de **planificación**, despachamos pero con buffer conservador.

---

## 🟢 ELEGIBILIDAD — definen si la carga es viable (7)

Estas decidente **qué vehículo o conductor califica**. Sin estas, no se puede asignar correctamente y se corre el riesgo de mandar un camión equivocado.

---

### 1. tipo_camion

**Name:** `tipo_camion`
**Type:** `string`

**Description:**
```
Extract the truck type requested by the broker. Only valid values: "box_truck" or "van".

Examples:
- "I need a box truck" → "box_truck"
- "Do you have a cargo van?" → "van"
- "Sprinter van" → "van"
- "26 footer" → "box_truck"
- "24 foot box" → "box_truck"
- "Small van" → "van"

If the broker says only "truck" without specifying size, ask:
"Do you need a Box Truck or a Cargo Van?"

Never assume. Only save "box_truck" or "van".
```

---

### 2. tipo_carga

**Name:** `tipo_carga`
**Type:** `string`

**Description:**
```
Extract the cargo type. Only valid values:
"palletized", "boxes", "loose", "fragile", "refrigerated_small"

Examples:
- "10 pallets" → "palletized"
- "We have skids" → "palletized"
- "Just boxes" → "boxes"
- "Cardboard boxes" → "boxes"
- "Floor loaded" → "loose"
- "Loose freight" → "loose"
- "Glass shipment" → "fragile"
- "Antiques" → "fragile"
- "Cold load, needs to stay at 38" → "refrigerated_small"
- "Refrigerated" → "refrigerated_small"

"Skids" always means "palletized".

If the broker says only "freight", ask:
"Is it palletized, boxes, or loose?"

Never invent a category. Only use the 5 valid values listed above.
```

---

### 3. cantidad_pallets

**Name:** `cantidad_pallets`
**Type:** `number`

**Description:**
```
Extract the number of pallets. Integer only.

Examples:
- "8 pallets" → 8
- "Six skids" → 6
- "A dozen pallets" → 12
- "Just one pallet" → 1
- "No pallets, just boxes" → 0
- "Floor loaded" → 0

If the broker says only "some pallets", ask:
"How many pallets exactly?"

If cargo is not palletized, save 0.

Max for Box Truck = 12. Max for Van = 6.
Save the number even if it exceeds the limit. Validation happens later.

Never save a range like "8 to 10". Always pick one number, ask if needed.
```

---

### 4. peso_total_libras

**Name:** `peso_total_libras`
**Type:** `number`

**Description:**
```
Extract total cargo weight in pounds. Number only.

Examples:
- "4500 pounds" → 4500
- "About 2 tons" → 4000
- "5000 lbs" → 5000
- "Three thousand pounds" → 3000
- "Under 1000 pounds" → 1000

Conversions:
- 1 ton = 2000 lb
- 1 kg = 2.2 lb (round)

If weight is in kg, convert before saving.
If the broker says only "light" or "I don't know", ask:
"Approximate weight in pounds?"

Limits:
- Van: 3000 lb max
- Box Truck: 10000 lb max

Save the number even if it exceeds the limit.
```

---

### 5. refrigerado

**Name:** `refrigerado`
**Type:** `boolean`

**Description:**
```
Determine if cargo requires temperature control.
This is an ELIGIBILITY question: only trucks with reefer equipment qualify.

Examples:
- "Cold load" → true
- "Refrigerated" → true
- "Frozen" → true
- "Needs to stay at 38 degrees" → true
- "Temperature controlled" → true
- "Reefer load" → true
- "Dry van OK" → false
- "Ambient" → false
- "Room temperature" → false

Default to false if not mentioned.

If true, the system will only match trucks equipped with refrigeration.
```

---

### 6. hazmat

**Name:** `hazmat`
**Type:** `boolean`

**Description:**
```
Determine if the cargo is hazardous material.
This is an ELIGIBILITY question: only drivers with hazmat certification qualify.

Examples:
- "Hazmat load" → true
- "Class 3 flammable" → true
- "Corrosives" → true
- "Lithium batteries" → true
- "Just regular freight" → false
- "Non-hazardous" → false
- "Dry goods" → false

Default to false if not mentioned.

If true, the system will only match drivers with valid hazmat endorsement.
```

---

### 7. servicios_especiales

**Name:** `servicios_especiales`
**Type:** `string`

**Description:**
```
Extract any special services requested.
Some services restrict which trucks or drivers can take the load.

Valid values: "white_glove", "inside_delivery", "signature_required", "tarping"

Examples:
- "White glove service" → "white_glove"
- "Needs to be brought inside" → "inside_delivery"
- "Signature required at delivery" → "signature_required"
- "White glove and inside delivery" → "white_glove,inside_delivery"
- "Standard service" → ""
- Not mentioned → ""

Save as comma-separated string with no spaces between values.
If none mentioned, save empty string "".

Never invent service names. Only use the 4 valid values.

Note: white_glove requires trained drivers; inside_delivery may require
specific vehicle access. The system filters based on this.
```

---

## 🟡 PLANIFICACIÓN — afinan el buffer y la ruta (3)

Estas no descartan camiones, pero **ajustan el tiempo** que se le asigna a la operación. Si faltan, se usa el peor caso (más conservador).

---

### 8. tipo_recogida

**Name:** `tipo_recogida`
**Type:** `string`

**Description:**
```
Extract how the pickup will happen. Only valid values: "live_load" or "wait_load".

"live_load" = driver waits while cargo is being loaded.
"wait_load" = cargo is pre-loaded and ready, driver just picks up.

Examples:
- "Live load" → "live_load"
- "Driver waits to be loaded" → "live_load"
- "It will be ready when he arrives" → "wait_load"
- "Pre-loaded" → "wait_load"
- "Ready to go" → "wait_load"

If not mentioned, default to "live_load" (most common for Box Truck and Van).

Note: drop and hook does NOT apply to Box Truck or Van.
Never save "drop_hook".
```

---

### 9. metodo_carga

**Name:** `metodo_carga`
**Type:** `string`

**Description:**
```
Extract how the cargo will be loaded at pickup.
Valid values: "forklift_dock", "liftgate", "manual", "driver_assist".

Examples:
- "Loading dock with forklift" → "forklift_dock"
- "They have a dock" → "forklift_dock"
- "Standard dock loading" → "forklift_dock"
- "Needs liftgate" → "liftgate"
- "No dock, but liftgate works" → "liftgate"
- "Driver helps unload" → "driver_assist"
- "Driver assist" → "driver_assist"
- "Hand loaded" → "manual"
- "No dock, no liftgate" → "manual"

If not mentioned, default to "manual" (safest assumption — longest buffer).
```

---

### 10. requiere_liftgate

**Name:** `requiere_liftgate`
**Type:** `boolean`

**Description:**
```
Determine if the truck needs a liftgate. true or false only.

Liftgate = hydraulic platform at the back of the truck that lifts cargo
from ground level to truck bed.

Examples:
- "Need a liftgate" → true
- "No dock available" → true
- "Ground level loading" → true
- "Residential delivery" → true
- "They have a dock" → false
- "Forklift will load" → false
- Not mentioned → false

Box Trucks usually have a liftgate. Cargo Vans usually do NOT.
```

---

## 🔵 CONDICIONALES — solo si una anterior las activa (2)

Estas variables **solo tienen sentido si otra ya marcó algo**. Vapi no debe preguntarlas si no aplican.

---

### 11. hazmat_clase

**Name:** `hazmat_clase`
**Type:** `string`

**Trigger:** Only ask if `hazmat = true`.

**Description:**
```
DOT hazmat class number. Only fill if hazmat = true.
Valid values: "1" through "9".

Examples:
- "Class 3 flammable liquids" → "3"
- "Class 8 corrosives" → "8"
- "Class 9 misc" → "9"

If hazmat = false, leave empty. Do not ask.
If hazmat = true but the class is not specified, ask:
"What's the hazmat class number, 1 through 9?"

Save only the single digit as a string.
```

---

### 12. temperatura_requerida_f

**Name:** `temperatura_requerida_f`
**Type:** `number`

**Trigger:** Only ask if `refrigerado = true`.

**Description:**
```
Target temperature in Fahrenheit. Only fill if refrigerado = true.

Examples:
- "38 degrees" → 38
- "Frozen, zero degrees" → 0
- "Below 40" → 40
- "Keep it cold, around 35" → 35

Conversion:
- Celsius to Fahrenheit: (C × 9/5) + 32. Round to nearest integer.

If refrigerado = false, leave empty. Do not ask.
If refrigerado = true but no temperature is given, ask:
"What's the target temperature in Fahrenheit?"
```

---

## 📋 Resumen visual

```text
ELEGIBILIDAD (7) — sin estas no se puede asignar vehículo
  ├─ tipo_camion          box_truck | van
  ├─ tipo_carga           palletized | boxes | loose | fragile | refrigerated_small
  ├─ cantidad_pallets     0-12
  ├─ peso_total_libras    número
  ├─ refrigerado          true | false       ← filtra flota con reefer
  ├─ hazmat               true | false       ← filtra conductores certificados
  └─ servicios_especiales white_glove | inside_delivery | signature_required | tarping

PLANIFICACIÓN (3) — ajustan el buffer
  ├─ tipo_recogida        live_load | wait_load
  ├─ metodo_carga         forklift_dock | liftgate | manual | driver_assist
  └─ requiere_liftgate    true | false

CONDICIONALES (2) — solo si la anterior lo activa
  ├─ hazmat_clase            1-9         (solo si hazmat = true)
  └─ temperatura_requerida_f número °F   (solo si refrigerado = true)
```

---

## ⚠️ Reglas universales para Vapi

1. **Nunca inventar valores.** Si no encaja en la lista válida, preguntar.
2. **Defaults conservadores.** Si no se menciona, asumir el peor caso (ej. `metodo_carga = "manual"`).
3. **Conversiones automáticas.** Toneladas → libras, kg → libras, °C → °F.
4. **Una variable = una respuesta.** Nunca rangos ("8 a 10"), nunca múltiples opciones, nunca "depende".
5. **Condicionales NO se preguntan si el trigger es false.** No pregunte `hazmat_clase` si `hazmat = false`.

---

## 🔁 Orden recomendado de preguntas

Vapi debe seguir este orden para minimizar preguntas innecesarias:

```text
1. tipo_camion              ¿Box Truck o Van?
2. tipo_carga               ¿Qué tipo de carga?
3. cantidad_pallets         (si tipo_carga ≠ "loose")
4. peso_total_libras        ¿Peso aproximado?
5. refrigerado              ¿Necesita refrigeración?
6.   └─ temperatura_requerida_f  (solo si refrigerado = true)
7. hazmat                   ¿Es hazmat?
8.   └─ hazmat_clase             (solo si hazmat = true)
9. servicios_especiales     ¿Algún servicio especial?
10. tipo_recogida           ¿Live load o pre-cargado?
11. metodo_carga            ¿Con dock, liftgate o manual?
12. requiere_liftgate       (deducir de metodo_carga, no preguntar dos veces)
```
