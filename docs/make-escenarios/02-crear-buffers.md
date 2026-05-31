# Guía — Crear los buffers en Make

Cómo construir `buffer_carga`, `buffer_descarga` y `buffer_carretera` usando las 12 variables de carga.

**Principio:** una fórmula gigante es imposible de debuggear. Partimos cada buffer en cajitas pequeñas (Set Variable) que se suman al final.

---

## 🧱 Estructura general

```
[Webhook con las 12 variables]
    ↓
[Set var: buffer_carga_base]      ← depende de metodo_carga
    ↓
[Set var: buffer_carga_pallets]   ← depende de cantidad_pallets
    ↓
[Set var: buffer_carga_extras]    ← depende de servicios, hazmat, refrigerado
    ↓
[Set var: buffer_carga]           ← suma las 3 anteriores
    ↓
[Set var: buffer_descarga]        ← buffer_carga * 0.8
    ↓
[Set var: buffer_carretera]       ← depende de viaje_minutos (ya existente)
    ↓
[resto del escenario...]
```

Cada `[Set var]` es un módulo **Tools → Set variable**.

---

## CAJA 1 — buffer_carga_base

Tiempo base de carga según el método. Sin contar pallets ni extras.

```text
Variable name:  buffer_carga_base
Variable value:
```

```
if(metodo_carga = "forklift_dock"; 20;
if(metodo_carga = "liftgate"; 40;
if(metodo_carga = "manual"; 60;
if(metodo_carga = "driver_assist"; 90;
45))))
```

Lógica:
- Dock con forklift → 20 min (lo más rápido)
- Liftgate → 40 min
- Manual → 60 min
- Driver assist → 90 min
- Cualquier otra cosa (default) → 45 min

---

## CAJA 2 — buffer_carga_pallets

Minutos extra según cuántos pallets, y qué tan rápido se cargan según el método.

```text
Variable name:  buffer_carga_pallets
Variable value:
```

```
if(metodo_carga = "forklift_dock"; cantidad_pallets * 2;
if(metodo_carga = "liftgate"; cantidad_pallets * 3;
cantidad_pallets * 5))
```

Lógica:
- Con forklift → 2 min por pallet (rápido)
- Con liftgate → 3 min por pallet
- Manual / driver_assist → 5 min por pallet (lento)

Ejemplo: 8 pallets con forklift = 8 × 2 = 16 min.

---

## CAJA 3 — buffer_carga_extras

Minutos extra por servicios especiales, hazmat y refrigeración.

```text
Variable name:  buffer_carga_extras
Variable value:
```

```
(if(contains(servicios_especiales; "white_glove"); 30; 0))
+
(if(contains(servicios_especiales; "inside_delivery"); 20; 0))
+
(if(hazmat = true; 30; 0))
+
(if(refrigerado = true; 15; 0))
```

Lógica:
- White glove → +30 min
- Inside delivery → +20 min
- Hazmat → +30 min (paperwork extra)
- Refrigerado → +15 min (check de temperatura)

> ⚠️ Si `hazmat` o `refrigerado` te llegan como TEXTO ("true"/"false") en
> vez de boolean, cambia `hazmat = true` por `hazmat = "true"`.
> Revisa cómo llega en tu bundle de prueba.

---

## CAJA 4 — buffer_carga (la suma)

Junta las 3 cajas anteriores.

```text
Variable name:  buffer_carga
Variable value:
```

```
buffer_carga_base + buffer_carga_pallets + buffer_carga_extras
```

(Aquí seleccionas las 3 variables del panel, no las escribas a mano.)

Ejemplo completo:
- base (forklift) = 20
- pallets (8 × 2) = 16
- extras (ninguno) = 0
- **buffer_carga = 36 min**

---

## CAJA 5 — buffer_descarga

La descarga suele ser ~80% del tiempo de carga (menos paperwork, sale más rápido).

```text
Variable name:  buffer_descarga
Variable value:
```

```
round(buffer_carga * 0.8)
```

Ejemplo: 36 × 0.8 = 28.8 → redondeado = 29 min.

---

## CAJA 6 — buffer_carretera (el que ya tenías)

Este NO usa las variables de carga, usa la duración del viaje.
Si ya lo tienes con el nombre `buffer_operacional`, renómbralo a
`buffer_carretera` para evitar confusión.

```text
Variable name:  buffer_carretera
Variable value:
```

```
if(viaje_minutos <= 180; 30;
if(viaje_minutos <= 360; 60;
if(viaje_minutos <= 600; 90;
120)))
```

Lógica:
- Viaje ≤ 3h → 30 min
- Viaje ≤ 6h → 60 min
- Viaje ≤ 10h → 90 min
- Más largo → 120 min

---

## 🧮 Cómo se usan los 3 buffers juntos

Una vez tengas los 3, el tiempo total operacional queda:

```
tiempo_jornada_completa =
    deadhead_minutos
  + buffer_carga
  + viaje_minutos
  + buffer_carretera
  + buffer_descarga
```

Ejemplo real:
```
deadhead        = 80 min
buffer_carga    = 36 min
viaje           = 237 min
buffer_carretera= 60 min
buffer_descarga = 29 min
─────────────────────────
TOTAL           = 442 min  (≈ 7 horas 22 min)
```

Ese número se compara contra `Minutos_Disponibles_HOS` del conductor.

---

## ⚠️ Errores comunes

### "La fórmula da error de sintaxis"

- Revisa que cada `if(` tenga su `)` de cierre. Cuenta paréntesis.
- En Make el separador es **punto y coma `;`**, no coma.
- NO uses el botón verde visual para los operadores. Escribe `=`, `<=`,
  `+` a mano dentro de la fórmula (el botón verde causa errores, ya lo
  viste antes).

### "cantidad_pallets * 2 me da 0"

- `cantidad_pallets` está llegando como texto, no número.
- Solución: envuélvelo en `parseNumber()`:
  ```
  parseNumber(cantidad_pallets) * 2
  ```

### "contains() no detecta el servicio"

- Revisa que `servicios_especiales` tenga el valor exacto.
- `contains("white_glove,inside_delivery"; "white_glove")` → true ✅
- Cuidado con mayúsculas: contains es sensible a mayúsculas.

### "buffer_descarga me da decimales"

- Usa `round()` alrededor de toda la operación:
  ```
  round(buffer_carga * 0.8)
  ```

---

## ✅ Checklist

```text
[ ] Caja 1: buffer_carga_base creada y probada
[ ] Caja 2: buffer_carga_pallets creada y probada
[ ] Caja 3: buffer_carga_extras creada y probada
[ ] Caja 4: buffer_carga (suma) creada y probada
[ ] Caja 5: buffer_descarga creada y probada
[ ] Caja 6: buffer_carretera renombrada/creada
[ ] Probado con bundle box-truck-normal → buffer_carga = 36
[ ] Probado con bundle edge-case (reefer+hazmat) → buffer_carga mayor
```

---

## 🧪 Valores esperados con los bundles de prueba

### carga-box-truck-normal.json
```
metodo_carga = forklift_dock, pallets = 8, sin extras
base=20, pallets=16, extras=0
buffer_carga = 36
buffer_descarga = 29
```

### carga-van-expedited.json
```
metodo_carga = manual, pallets = 0, signature_required (no suma)
base=60, pallets=0, extras=0
buffer_carga = 60
buffer_descarga = 48
```

### carga-edge-case-reefer-hazmat.json
```
metodo_carga = liftgate, pallets = 6, white_glove+inside, hazmat, refrigerado
base=40, pallets=18, extras=30+20+30+15=95
buffer_carga = 153
buffer_descarga = 122
```

Usa estos números para verificar que tus fórmulas están bien. Si te dan
distinto, hay un error en alguna caja.
