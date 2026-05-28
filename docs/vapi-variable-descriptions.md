# Vapi — Descripciones de variables (paste-ready)

Cada variable tiene su **Name**, **Type** y **Description** en inglés, lista para copiar y pegar en la configuración de Vapi.

Los brokers hablan inglés, así que las descripciones están en inglés. Las secciones de este doc están en español para tu referencia.

---

## 🟢 TIER 1 — Mínimas (Vapi NO termina la llamada sin estas)

---

### 1. tipo_camion

**Name:** `tipo_camion`
**Type:** `string`

**Description:**
```
Extract the truck type requested by the broker. Only valid values: "box_truck" or "van".

Examples:
- "I need a box truck" → "box_truck"
- "Do you have a cargo van available?" → "van"
- "Sprinter van" → "van"
- "26 footer" → "box_truck"
- "24 foot box" → "box_truck"
- "Small van" → "van"

If the broker says only "truck" without specifying size, ask:
"Do you need a Box Truck or a Cargo Van?"

Never assume. Only save "box_truck" or "van".
```

---

### 2. pickup_city

**Name:** `pickup_city`
**Type:** `string`

**Description:**
```
Extract the pickup city in the format "City, ST" (two-letter state code).

Examples:
- "Pick up in Detroit, Michigan" → "Detroit, MI"
- "From Chicago" → "Chicago, IL"
- "Pickup is Toledo Ohio" → "Toledo, OH"
- "Outside Atlanta" → "Atlanta, GA"

If the broker only says the city without the state, ask:
"What state is that in?"

Always use the 2-letter USPS state code. Never save the full state name.
```

---

### 3. pickup_time

**Name:** `pickup_time`
**Type:** `string`

**Description:**
```
Extract pickup date and time in ISO 8601 format: "YYYY-MM-DDTHH:mm:00".

Examples:
- "Tomorrow at 2pm" → "[tomorrow's date]T14:00:00"
- "Today at 10:30am" → "[today's date]T10:30:00"
- "Friday at 8am" → "[next Friday's date]T08:00:00"
- "May 15 at noon" → "2026-05-15T12:00:00"
- "ASAP" → use today's date and current hour + 2 hours

If no time is given, ask: "What time do you need the pickup?"
If no date is given, ask: "What day is the pickup?"

Always 24-hour format. Always include seconds as 00.
Never save ranges. Pick one exact time.
```

---

### 4. delivery_city

**Name:** `delivery_city`
**Type:** `string`

**Description:**
```
Extract the delivery city in the format "City, ST" (two-letter state code).

Examples:
- "Delivery in Chicago, Illinois" → "Chicago, IL"
- "Going to Houston" → "Houston, TX"
- "Drop off in Cleveland Ohio" → "Cleveland, OH"

If the broker only says the city without the state, ask:
"What state is that in?"

Always use the 2-letter USPS state code. Never save the full state name.
```

---

### 5. tipo_carga

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

### 6. tipo_recogida

**Name:** `tipo_recogida`
**Type:** `string`

**Description:**
```
Extract how the pickup will happen. Only valid values: "live_load" or "wait_load".

"live_load" = driver waits at the dock while cargo is being loaded.
"wait_load" = shipper has cargo ready and pre-loaded, driver just connects.

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

### 7. cantidad_pallets

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
If the broker says more, still save the number but flag will be raised later.

Never save a range like "8 to 10". Always pick one number, ask if needed.
```

---

### 8. broker_nombre

**Name:** `broker_nombre`
**Type:** `string`

**Description:**
```
Extract the broker's full name and/or company.

Examples:
- "This is John from ABC Logistics" → "John - ABC Logistics"
- "I'm Sarah Martinez, FastFreight" → "Sarah Martinez - FastFreight"
- "Total Quality Logistics, my name is Mike" → "Mike - Total Quality Logistics"
- "Just say it's from CH Robinson" → "CH Robinson"

If only the company is given, save the company.
If only the person is given, save the person.

Format: "Person Name - Company" when both are given.
```

---

### 9. tarifa_ofrecida

**Name:** `tarifa_ofrecida`
**Type:** `number`

**Description:**
```
Extract the total rate offered for this load, in USD. Number only, no symbols.

Examples:
- "$850" → 850
- "Eight fifty" → 850
- "1,200 dollars" → 1200
- "I can pay 950" → 950
- "Twelve hundred" → 1200
- "$2.50 per mile, 400 miles" → 1000

If the rate is given per mile, multiply by total miles if known.
If miles unknown, ask: "What's the total rate?"

Never include currency symbols, commas, or decimals.
Save the total flat rate, not per mile.
```

---

## 🟡 TIER 2 — Recomendadas (si la conversación lo permite)

---

### 10. pickup_address

**Name:** `pickup_address`
**Type:** `string`

**Description:**
```
Extract the full pickup address including street, city, state, and ZIP if mentioned.

Examples:
- "1234 Industrial Drive, Detroit MI 48201" → "1234 Industrial Dr, Detroit, MI 48201"
- "5678 Warehouse Avenue Chicago Illinois" → "5678 Warehouse Ave, Chicago, IL"
- "It's the Walmart DC in Houston" → "Walmart DC, Houston, TX"
- "Pick up at 1500 South Main Street" → "1500 S Main St"

If only the city is given, leave this empty (the pickup_city variable will handle it).

Use USPS abbreviations: Dr, Ave, St, Blvd, Rd, Ln, Pkwy, etc.
Use directional abbreviations: N, S, E, W.
```

---

### 11. delivery_address

**Name:** `delivery_address`
**Type:** `string`

**Description:**
```
Extract the full delivery address including street, city, state, and ZIP if mentioned.

Examples:
- "Drop at 9876 Commerce Blvd, Chicago IL 60601" → "9876 Commerce Blvd, Chicago, IL 60601"
- "FedEx terminal Memphis" → "FedEx Terminal, Memphis, TN"

If only the city is given, leave this empty.

Use USPS abbreviations: Dr, Ave, St, Blvd, Rd, etc.
```

---

### 12. delivery_time_requested

**Name:** `delivery_time_requested`
**Type:** `string`

**Description:**
```
Extract the delivery deadline only if the broker mentions one.
Format: ISO 8601 "YYYY-MM-DDTHH:mm:00".

Examples:
- "Needs to be there by 6pm same day" → same date as pickup, "T18:00:00"
- "Delivery tomorrow morning by 10am" → next day "T10:00:00"
- "Hot load, ASAP" → today + 4 hours from pickup
- "No specific deadline" → leave empty
- "Whenever you can get it there" → leave empty

If the broker mentions a delivery window like "between 8am and 12pm",
save the LATEST time: "T12:00:00".

If no deadline is given, leave empty. Do not invent one.
```

---

### 13. metodo_carga

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
- "Driver assist loading" → "driver_assist"
- "Hand loaded" → "manual"
- "No dock, no liftgate" → "manual"

If not mentioned, default to "manual" (safest assumption — slower buffer).
```

---

### 14. requiere_liftgate

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

### 15. numero_paradas

**Name:** `numero_paradas`
**Type:** `number`

**Description:**
```
Extract the total number of stops in the trip. Integer >= 1.

Count only delivery stops (or unique pickup-delivery pairs), not the trip origin.

Examples:
- "One pickup, one delivery" → 1
- "Direct" → 1
- "Straight run" → 1
- "Two pickups, one drop" → 2
- "Multi-stop, three deliveries" → 3
- "Round trip with 4 stops" → 4

Default to 1 if not mentioned.
```

---

### 16. cita_requerida_pickup

**Name:** `cita_requerida_pickup`
**Type:** `boolean`

**Description:**
```
Determine if the shipper requires an appointment for pickup.

Examples:
- "By appointment only" → true
- "Need to schedule a window" → true
- "Appointment required" → true
- "FCFS" → false (first come first served)
- "Walk in anytime" → false
- "Open dock until 5pm" → false

Default to false if not mentioned.
```

---

### 17. peso_total_libras

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

If weight is in kg, convert.
If the broker says only "light" or "I don't know", ask:
"Approximate weight in pounds?"

Limits:
- Van: 3000 lb max
- Box Truck: 10000 lb max

Save the number even if it exceeds the limit. The system will validate later.
```

---

### 18. broker_telefono

**Name:** `broker_telefono`
**Type:** `string`

**Description:**
```
Extract the broker's phone number in E.164 format: "+1XXXXXXXXXX" for US numbers.

Examples:
- "313 555 1234" → "+13135551234"
- "(419) 555-1234" → "+14195551234"
- "555-1234, area code 313" → "+13135551234"
- "Three one three, five five five, twelve thirty four" → "+13135551234"

Always include +1 country code for US numbers.
No spaces, dashes, or parentheses.

If the broker is calling FROM the number, no need to ask — use caller ID.
```

---

## 🔵 TIER 3 — Opcionales (solo si aplican)

---

### 19. servicios_especiales

**Name:** `servicios_especiales`
**Type:** `string` (comma-separated list, or array if Vapi supports it)

**Description:**
```
Extract any special services requested.
Valid values: "white_glove", "inside_delivery", "signature_required", "appointment_only", "tarping"

Examples:
- "White glove service" → "white_glove"
- "Needs to be brought inside" → "inside_delivery"
- "Signature required at delivery" → "signature_required"
- "White glove and inside" → "white_glove,inside_delivery"
- "Standard service" → ""
- Not mentioned → ""

Save as comma-separated string with no spaces.
If none mentioned, save empty string "".
Never invent service names. Only use the 5 valid values.
```

---

### 20. hazmat

**Name:** `hazmat`
**Type:** `boolean`

**Description:**
```
Determine if the cargo is hazardous material.

Examples:
- "Hazmat load" → true
- "Class 3 flammable" → true
- "Corrosives" → true
- "Just regular freight" → false
- "Non-hazardous" → false
- "Dry goods" → false

Default to false if not mentioned.
```

---

### 21. hazmat_clase

**Name:** `hazmat_clase`
**Type:** `string`

**Description:**
```
DOT hazmat class number. Only fill if hazmat = true.
Valid values: "1" through "9".

Examples:
- "Class 3 flammable liquids" → "3"
- "Class 8 corrosives" → "8"
- "Class 9 misc" → "9"

If hazmat = false, leave empty.
If hazmat = true but the class is not specified, ask:
"What's the hazmat class?"

Save only the single digit as a string.
```

---

### 22. refrigerado

**Name:** `refrigerado`
**Type:** `boolean`

**Description:**
```
Determine if cargo requires temperature control.

Examples:
- "Cold load" → true
- "Refrigerated" → true
- "Frozen" → true
- "Needs to stay at 38 degrees" → true
- "Temperature controlled" → true
- "Dry van OK" → false
- "Ambient" → false

Default to false if not mentioned.
```

---

### 23. temperatura_requerida_f

**Name:** `temperatura_requerida_f`
**Type:** `number`

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

If refrigerado = true but no temperature given, ask:
"What's the target temperature in Fahrenheit?"

If refrigerado = false, leave empty.
```

---

### 24. referencia_carga

**Name:** `referencia_carga`
**Type:** `string`

**Description:**
```
Extract the load reference number (PO, BOL, load number).

Examples:
- "PO 78421" → "PO-78421"
- "Load number 12345" → "LOAD-12345"
- "BOL ABC-9876" → "BOL-ABC-9876"
- "Reference is L-555-2026" → "L-555-2026"

Preserve the original format.
If the broker says only a number without prefix, ask:
"Is that a PO, BOL, or load number?"

If not given, leave empty.
```

---

### 25. broker_email

**Name:** `broker_email`
**Type:** `string`

**Description:**
```
Extract the broker's email address.

Examples:
- "john at abc logistics dot com" → "john@abclogistics.com"
- "sarah.martinez@fastfreight.com" → "sarah.martinez@fastfreight.com"
- "Send to dispatch at totalquality dot com" → "dispatch@totalquality.com"

Validate basic format: must contain @ and a dot after.
If not given, leave empty.
```

---

### 26. comentarios

**Name:** `comentarios`
**Type:** `string`

**Description:**
```
Free text field for important details the broker mentions that don't fit other variables.

Examples:
- "The receiver is closed on weekends"
- "Tight dock, only one truck at a time"
- "Driver needs hard hat and safety vest"
- "Repeat customer, monthly load"
- "Gate code is 1234"

Save only operationally relevant information.
Maximum 500 characters.

If nothing notable is mentioned, leave empty.
Never invent comments.
```

---

## 📋 Checklist para configurar Vapi

```text
[ ] Crear las 9 variables del Tier 1
[ ] Configurar Vapi para que NO termine la llamada si falta alguna del Tier 1
[ ] Crear las 9 variables del Tier 2
[ ] Crear las 8 variables del Tier 3
[ ] Configurar el webhook que envía todo a Make
[ ] Probar con una llamada real
[ ] Guardar el JSON resultante en docs/bundles-ejemplo/
```

---

## ⚠️ Regla universal para todas las descripciones

Todas las descripciones siguen el mismo patrón:

```text
1. Una línea explicando QUÉ extraer y en qué formato.
2. "Examples:" — entre 4 y 7 ejemplos de "lo que dice el broker" → "lo que guardar".
3. (Opcional) Instrucción de qué preguntar si falta información.
4. (Opcional) Regla de "Never do X" para evitar alucinaciones.
```

Si después agregas variables nuevas, mantén el mismo patrón para consistencia.
