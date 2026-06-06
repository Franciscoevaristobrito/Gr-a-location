# Guía — Crear los precios (Tarifa × Millas + Servicios)

Cómo construir `tarifa_por_milla_real`, `costo_millas_cargadas`, `costo_deadhead`, `costo_servicios_especiales` y `precio_total` usando los datos de Google Maps, Airtable y servicios especiales.

**Principio:** igual que los buffers, dividimos cada precio en cajas pequeñas (Set Variable) que se suman al final.

---

## 🧱 Estructura general

```
[Ganador seleccionado (después Array Aggregator)]
    ↓
[Set var: tarifa_por_milla_real]        ← lookup en Airtable por tipo_camion
    ↓
[Set var: costo_millas_cargadas]        ← viaje_millas × tarifa_por_milla_real
    ↓
[Set var: costo_deadhead]               ← deadhead_millas × tarifa_por_milla_real × 0.5
    ↓
[Set var: costo_servicios_especiales]   ← suma de extras (white_glove, inside_delivery, etc.)
    ↓
[Set var: costo_peajes_real]            ← ya viene de Google Maps, solo limpiamos si es necesario
    ↓
[Set var: subtotal_costo]               ← suma todos los costos
    ↓
[Set var: precio_total]                 ← subtotal redondeado
    ↓
[Set var: recomendacion_oferta]         ← comparar contra tarifa_ofrecida del broker
    ↓
[resto del escenario...]
```

Cada `[Set var]` es un módulo **Tools → Set variable**.

---

## CAJA 1 — tarifa_por_milla_real

**Lookup en Airtable** para obtener la tarifa base según el tipo de camión.

```text
Variable name:  tarifa_por_milla_real
Variable value:
```

### Opción A — Usando un lookupValues desde Airtable

En Make, crea un módulo **Airtable → Search Records** ANTES de este Set Variable que busque en tu tabla de Tarifas (o en Inventario de Flota si ahí guardas el `tarifa_por_milla` por `truck_type`).

Luego en el Set Variable:

```
[Airtable: buscar tarifa_por_milla donde truck_type = tipo_camion]
    ↓
[Set var: tarifa_por_milla_real = tarifa_por_milla del resultado]
```

Ejemplo de fórmula si usas la búsqueda de Airtable:
```text
tarifa_por_milla_real = resultado_airtable[0].tarifa_por_milla
```

### Opción B — Si usas una tabla separada "Tarifas"

Si creaste una tabla `Tarifas` con campos:
- `truck_type` (box_truck, van)
- `tarifa_por_milla` (número, USD)

**En Make:**

1. Agrega un módulo **Airtable → Search records** que busque en la tabla Tarifas.
2. **Filtro:** `truck_type = camion_ganador.truck_type` (o `tipo_camion`)
3. En el Set Variable siguiente, extrae el valor:

```text
Variable name:  tarifa_por_milla_real
Variable value: (si el lookup devolvió un array con un resultado)
                 parseNumber(resultado_airtable[0].tarifa_por_milla)
```

**⚠️ Importante:** asegúrate que el módulo Airtable Search ejecute ANTES del Set Variable. Si lo haces después, no tendrá el dato.

### Fallback si no encuenta en Airtable

Si la búsqueda falla o devuelve vacío, usa una fórmula if para un default:

```text
if( isempty(resultado_airtable) ; 
    if( tipo_camion = "box_truck" ; 2.50 ;  /* default box truck */
        1.50 )  /* default van */
    ;
    parseNumber(resultado_airtable[0].tarifa_por_milla)
)
```

---

## CAJA 2 — costo_millas_cargadas

Minutos multiplicado por la tarifa por milla.

```text
Variable name:  costo_millas_cargadas
Variable value:
```

```
parseNumber(viaje_millas) * tarifa_por_milla_real
```

Ejemplo: 237 millas × $2.50/milla = $592.50

---

## CAJA 3 — costo_deadhead

Millas vacías (deadhead) pero a 50% de la tarifa (menos rentable, pero hay que cobrarlo).

```text
Variable name:  costo_deadhead
Variable value:
```

```
parseNumber(deadhead_millas) * tarifa_por_milla_real * 0.5
```

Ejemplo: 45 millas × $2.50/milla × 0.5 = $56.25

---

## CAJA 4 — costo_servicios_especiales

Minutos/costos extra por servicios especiales. **No todos aplican a todas las cargas.**

```text
Variable name:  costo_servicios_especiales
Variable value:
```

### Opción A — Si cada servicio tiene un costo fijo

```
sum(
  if(contains(servicios_especiales; "white_glove"); 75; 0);
  if(contains(servicios_especiales; "inside_delivery"); 50; 0);
  if(contains(servicios_especiales; "signature_required"); 0; 0);
  if(hazmat = true; 100; 0);
  if(refrigerado = true; 50; 0)
)
```

Lógica:
- White glove → +$75 (servicio premium)
- Inside delivery → +$50 (descarga adentro)
- Signature required → $0 (sin costo extra, ya está en el base)
- Hazmat → +$100 (paperwork, certificación)
- Refrigerado → +$50 (monitoreo de temp)

### Opción B — Lookup en tabla de Servicios (más flexible)

Si creaste una tabla `Servicios_Especiales` con:
- `nombre_servicio` (white_glove, inside_delivery, etc.)
- `costo_servicio` (USD)

Entonces:

1. **Airtable → Search records** en tabla Servicios_Especiales.
2. **Filtro múltiple** (buscar cada servicio):
   ```
   OR(
     nombre_servicio = "white_glove" AND contains(servicios_especiales; "white_glove"),
     nombre_servicio = "inside_delivery" AND contains(servicios_especiales; "inside_delivery"),
     ...
   )
   ```
3. En Set Variable, suma todos los resultados:
   ```
   sum( array de resultados[*].costo_servicio )
   ```

---

## CAJA 5 — costo_peajes (ya viene de Google Maps, solo validar)

Google Maps Routes API devuelve peajes así:

```text
Variable name:  costo_peajes_real
Variable value:
```

Si Google Maps devolvió `costo_peajes` como número:
```
parseNumber(costo_peajes)
```

Si devolvió string con símbolo (ej: "$15.75"):
```
parseFloat(replace(costo_peajes; "$"; ""))
```

Si devolvió vacío o null:
```
if(isempty(costo_peajes); 0; parseNumber(costo_peajes))
```

---

## CAJA 6 — subtotal_costo

Suma todo: millas cargadas + deadhead + servicios + peajes.

```text
Variable name:  subtotal_costo
Variable value:
```

```
sum(
  costo_millas_cargadas;
  costo_deadhead;
  costo_servicios_especiales;
  costo_peajes_real
)
```

Ejemplo:
```
592.50 (millas cargadas)
+ 56.25 (deadhead)
+ 75.00 (white_glove)
+ 12.50 (peajes)
─────────────────────
= 736.25 USD
```

---

## CAJA 7 — precio_total

El precio final que vas a ofrecerle al broker. Redondeado a 2 decimales.

```text
Variable name:  precio_total
Variable value:
```

```
round(subtotal_costo; 2)
```

Si prefieres redondeado a número entero:
```
round(subtotal_costo)
```

---

## CAJA 8 — recomendacion_oferta (opcional pero útil)

Compara el precio que NOSOTROS calculamos vs. el que el BROKER ofrece.

```text
Variable name:  recomendacion_oferta
Variable value:
```

```
if( parseNumber(tarifa_ofrecida) >= precio_total * 1.1 ;
    "ACEPTAR" ;
    if( parseNumber(tarifa_ofrecida) >= precio_total ;
        "ACEPTAR_CON_CAUTELA" ;
        "CONTRAOFERTA"
    )
)
```

Lógica:
- Si broker ofrece ≥ 110% de nuestro costo → ACEPTAR (hay margen)
- Si broker ofrece ≥ 100% de nuestro costo → ACEPTAR_CON_CAUTELA (marginal)
- Si broker ofrece < 100% → CONTRAOFERTA (perdemos dinero)

Luego puedes enviar `recomendacion_oferta` y `precio_total` a Vapi para que la IA responda con un contraoferta si es necesario.

---

## 🧮 Cómo se usa en el flujo final

Una vez tengas los 7 precios, el resultado que envías a Vapi es:

```json
{
  "camion_ganador": "BOX-001",
  "operador": "Juan García",
  "precio_calculado": 736.25,
  "desglose": {
    "millas_cargadas_costo": 592.50,
    "deadhead_costo": 56.25,
    "servicios": 75.00,
    "peajes": 12.50
  },
  "tarifa_ofrecida": 850,
  "recomendacion": "ACEPTAR",
  "margen": 113.75
}
```

Vapi responde algo como:
```text
"Tenemos el Box Truck BOX-001 disponible a las 3 horas.
El costo total que calculamos es $736.25 (millas, deadhead, servicios y peajes).
El broker ofrece $850, así que hay buen margen.
¿Aceptamos?"
```

---

## ⚠️ Errores comunes

### "tarifa_por_milla_real está vacío"

- **Causa:** La búsqueda en Airtable no devolvió resultados.
- **Solución:** 
  1. Verifica que la tabla de Tarifas exista en Airtable.
  2. Verifica que haya un registro con `truck_type` = `tipo_camion`.
  3. Usa un fallback con `if(isempty(...); default_value; ...)`.

### "costo_millas_cargadas da 0"

- **Causa:** `viaje_millas` o `tarifa_por_milla_real` es texto, no número.
- **Solución:** Envuelve en `parseNumber()`:
  ```
  parseNumber(viaje_millas) * parseNumber(tarifa_por_milla_real)
  ```

### "costo_peajes_real da error de símbolos"

- **Causa:** Google Maps devolvió "$15.75" en vez de 15.75.
- **Solución:**
  ```
  parseFloat(replace(costo_peajes; "$"; ""))
  ```

### "Precio final tiene 10 decimales"

- **Causa:** No usaste `round()`.
- **Solución:** Siempre termina con `round(subtotal; 2)`.

### "servicios_especiales no detecta ninguno"

- **Causa:** Vapi envió como "white_glove" pero tú buscas "white-glove" (guión vs underscore).
- **Solución:** Normaliza los nombres en el prompt de Vapi para que siempre use snake_case.

---

## ✅ Checklist

```text
[ ] Tabla de Tarifas creada en Airtable (o Inventario de Flota con tarifa_por_milla)
[ ] Airtable Search records configurado antes del Set Variable
[ ] Caja 1: tarifa_por_milla_real creada con fallback
[ ] Caja 2: costo_millas_cargadas creada y probada
[ ] Caja 3: costo_deadhead creada y probada
[ ] Caja 4: costo_servicios_especiales creada (fijo o lookup)
[ ] Caja 5: costo_peajes_real creada y limpiada
[ ] Caja 6: subtotal_costo creada
[ ] Caja 7: precio_total creada y redondeada
[ ] Caja 8: recomendacion_oferta creada
[ ] Probado con bundle box-truck-normal → precio coherente
[ ] Probado con bundle edge-case (con servicios) → precio mayor
```

---

## 🧪 Valores esperados con los bundles de prueba

### carga-box-truck-normal.json
```
tipo_camion = box_truck, viaje_millas = 237, deadhead_millas = 45
tarifa_box = $2.50/milla, sin servicios, peajes = $12.50
costo_millas = 237 × 2.50 = 592.50
costo_deadhead = 45 × 2.50 × 0.5 = 56.25
costo_servicios = 0
subtotal = 592.50 + 56.25 + 0 + 12.50 = 661.25
precio_total = 661.25
```

### carga-van-expedited.json
```
tipo_camion = van, viaje_millas = 95, deadhead_millas = 30
tarifa_van = $1.50/milla, signature_required (sin costo extra), peajes = $5.00
costo_millas = 95 × 1.50 = 142.50
costo_deadhead = 30 × 1.50 × 0.5 = 22.50
costo_servicios = 0
subtotal = 142.50 + 22.50 + 0 + 5.00 = 170.00
precio_total = 170.00
```

### carga-edge-case-reefer-hazmat.json
```
tipo_camion = box_truck, viaje_millas = 156, deadhead_millas = 52
tarifa_box = $2.50/milla, white_glove $75 + inside_delivery $50 + hazmat $100 + reefer $50 = $275
peajes = $22.50
costo_millas = 156 × 2.50 = 390.00
costo_deadhead = 52 × 2.50 × 0.5 = 65.00
costo_servicios = 275.00
subtotal = 390.00 + 65.00 + 275.00 + 22.50 = 752.50
precio_total = 752.50
```

Usa estos números para verificar que tus fórmulas están bien. Si te dan distinto, hay un error en alguna caja.

---

## 🔗 Relación con variables anteriores

```
Google Maps:     viaje_millas, deadhead_millas, costo_peajes
Webhook Vapi:    tarifa_ofrecida, servicios_especiales, hazmat, refrigerado
Airtable:        tarifa_por_milla (lookup por truck_type)
Cajas precio:    tarifa_por_milla_real → costo_millas_cargadas → costo_deadhead → ... → precio_total
```

---

## 📚 Integración con Vapi

Una vez que tengas `precio_total` y `recomendacion_oferta`, crea variables de output en el webhook de Make que envíe estos datos a Vapi:

```json
{
  "precio_calculado": precio_total,
  "recomendacion": recomendacion_oferta,
  "desglose": {
    "millas_cargadas": costo_millas_cargadas,
    "deadhead": costo_deadhead,
    "servicios": costo_servicios_especiales,
    "peajes": costo_peajes_real
  }
}
```

Vapi los recibe y forma su respuesta verbal al broker.
