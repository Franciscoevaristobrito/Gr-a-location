# Guía — Crear los precios (Tarifa × Millas + Peajes + Servicios)

Cómo construir `tarifa_por_milla_real`, `factor_peaje_real`, `costo_millas_cargadas`, `costo_deadhead`, `costo_peajes_real`, `costo_servicios_especiales` y `precio_total` usando los datos de Google Maps, Airtable y servicios especiales.

**Principio:** igual que los buffers, dividimos cada precio en cajas pequeñas (Set Variable) que se suman al final.

---

## 📋 Tu tabla de Airtable (Inventario de Flota / Tarifas)

La tabla del camión tiene estas columnas para precio:

| Columna | Tipo | Ejemplo | Para qué |
|---|---|---|---|
| `Tarifa_Milla` | Currency | `$2.50` / `$3.50` | Cuánto cobra ese camión por milla |
| `Factor_Peaje` | Number | `1.2` / `2.0` | Multiplicador de peajes según tamaño del camión |
| `estado` | Select | `Disponible` | Si el camión está activo |

> 💡 **Por qué `Factor_Peaje`:** Google Maps Routes API devuelve el peaje estimado para un **carro normal**. Un camión paga más en las casetas. El factor ajusta ese estimado: 1.2 = camión chico (paga 20% más), 2.0 = camión grande (paga el doble).

---

## 🧱 Estructura general

```
[Ganador seleccionado (después Array Aggregator)]
    ↓
[Airtable: Search records → trae Tarifa_Milla y Factor_Peaje del camión ganador]
    ↓
[Set var: tarifa_por_milla_real]        ← Tarifa_Milla del lookup
    ↓
[Set var: factor_peaje_real]            ← Factor_Peaje del lookup
    ↓
[Set var: costo_millas_cargadas]        ← viaje_millas × tarifa_por_milla_real
    ↓
[Set var: costo_deadhead]               ← deadhead_millas × tarifa_por_milla_real × 0.5
    ↓
[Set var: costo_peajes_real]            ← costo_peajes (Google) × factor_peaje_real
    ↓
[Airtable: Search records → Servicios_Especiales que pidió el broker]
    ↓
[Array Aggregator: costo_servicios_especiales]  ← suma costo_servicio de los resultados
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

## CAJA 1 — tarifa_por_milla_real y factor_peaje_real

**Lookup en Airtable.** Como ya tienes el camión ganador seleccionado, el módulo
**Airtable → Search records** que trae ese camión también te trae `Tarifa_Milla`
y `Factor_Peaje`. Solo extráelos en dos Set Variables.

> Si ya tienes en el bundle del ganador los campos `Tarifa_Milla` y `Factor_Peaje`
> (porque vienen del Array Aggregator), puedes usarlos directo sin otro Search.

### tarifa_por_milla_real

```text
Variable name:  tarifa_por_milla_real
Variable value: parseNumber( {Tarifa_Milla del camión ganador} )
```

### factor_peaje_real

```text
Variable name:  factor_peaje_real
Variable value: parseNumber( {Factor_Peaje del camión ganador} )
```

**⚠️ Importante:** el módulo Airtable Search debe ejecutar ANTES de estos Set
Variables. Si lo pones después, no tendrá el dato.

### Fallback si llega vacío

Si por alguna razón el campo viene vacío, usa un default según el tipo de camión:

```text
tarifa_por_milla_real:
  if( isempty({Tarifa_Milla}) ;
      if( tipo_camion = "box_truck" ; 3.50 ; 2.50 ) ;
      parseNumber({Tarifa_Milla}) )
```

```text
factor_peaje_real:
  if( isempty({Factor_Peaje}) ; 1.2 ; parseNumber({Factor_Peaje}) )
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

**Recomendación: crea una tabla separada `Servicios_Especiales`.** Es la mejor
opción por 4 razones:

1. Cuando cambies un precio, lo cambias en Airtable — **no tocas Make**.
2. Puedes agregar servicios nuevos sin editar el escenario.
3. Misma lógica que ya usas con `Tarifa_Milla` (eres consistente).
4. Una sola fuente de verdad para los precios.

### Tabla `Servicios_Especiales`

| nombre_servicio | costo_servicio | activo |
|---|---|---|
| white_glove | 75 | ✅ |
| inside_delivery | 50 | ✅ |
| signature_required | 0 | ✅ |
| liftgate_delivery | 40 | ✅ |
| residential | 30 | ✅ |
| appointment_only | 25 | ✅ |

> ⚠️ **Crítico:** `nombre_servicio` debe coincidir EXACTO con lo que Vapi manda
> en `servicios_especiales` (snake_case: `white_glove`, NO `White Glove`).

### Cómo sumarlos en Make (el truco con FIND)

`servicios_especiales` llega como string separado por comas, ej:
`"white_glove,inside_delivery"`. Para sumar solo los que pidió el broker:

1. **Airtable → Search records** en la tabla `Servicios_Especiales` con este
   **filtro de fórmula**:
   ```
   FIND({nombre_servicio}, "{{servicios_especiales}}") > 0
   ```
   `FIND` devuelve la posición donde aparece el nombre dentro del string. Si el
   servicio NO fue pedido, devuelve 0 y ese registro se descarta. Resultado:
   solo trae los servicios que el broker realmente pidió.

2. **Array Aggregator (tipo Numeric)** sobre el output del Search → suma el
   campo `costo_servicio`:
   ```text
   Variable name:  costo_servicios_especiales
   Variable value: sum( {array de resultados}[].costo_servicio )
   ```
   Si no pidió ningún servicio → el array viene vacío → la suma da 0. ✅

### Alternativa rápida (sin tabla, todo en Make)

Si por ahora no quieres crear la tabla, puedes hardcodear con `contains()`. Más
rápido de montar, pero los precios viven dentro de Make:

```
sum(
  if(contains(servicios_especiales; "white_glove"); 75; 0);
  if(contains(servicios_especiales; "inside_delivery"); 50; 0);
  if(contains(servicios_especiales; "signature_required"); 0; 0);
  if(hazmat = true; 100; 0);
  if(refrigerado = true; 50; 0)
)
```

> Recomendación: empieza con esta alternativa para probar el flujo completo,
> y cuando funcione migra a la tabla `Servicios_Especiales`.

---

## CAJA 5 — costo_peajes_real (Google Maps × Factor_Peaje) ⭐

Google Maps te da el peaje para **carro normal**. Lo multiplicas por tu
`Factor_Peaje` para ajustarlo al tamaño del camión.

```text
Variable name:  costo_peajes_real
Variable value:
```

Fórmula principal:
```
parseNumber(costo_peajes) * factor_peaje_real
```

Ejemplo: Google dice $10 (carro) → camión box: $10 × 1.2 = **$12.00**
         camión grande: $10 × 2.0 = **$20.00**

### Si `costo_peajes` llega "sucio"

Si Google devolvió string con símbolo (ej: "$15.75"):
```
parseFloat(replace(costo_peajes; "$"; "")) * factor_peaje_real
```

Si puede venir vacío o null:
```
if(isempty(costo_peajes); 0; parseNumber(costo_peajes) * factor_peaje_real)
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
  parseFloat(replace(costo_peajes; "$"; "")) * factor_peaje_real
  ```

### "factor_peaje_real da 0 y el peaje se vuelve 0"

- **Causa:** El campo `Factor_Peaje` llegó vacío o como texto.
- **Solución:** Usa el fallback `if(isempty({Factor_Peaje}); 1.2; parseNumber({Factor_Peaje}))`.

### "Precio final tiene 10 decimales"

- **Causa:** No usaste `round()`.
- **Solución:** Siempre termina con `round(subtotal; 2)`.

### "servicios_especiales no detecta ninguno"

- **Causa:** Vapi envió como "white_glove" pero tú buscas "white-glove" (guión vs underscore).
- **Solución:** Normaliza los nombres en el prompt de Vapi para que siempre use snake_case.

---

## ✅ Checklist

```text
[ ] Tabla con Tarifa_Milla y Factor_Peaje confirmada en Airtable
[ ] Airtable Search records (camión ganador) ejecuta antes de los Set Variables
[ ] Caja 1: tarifa_por_milla_real y factor_peaje_real creadas con fallback
[ ] Caja 2: costo_millas_cargadas creada y probada
[ ] Caja 3: costo_deadhead creada y probada
[ ] Caja 4: costo_servicios_especiales creada (tabla o alternativa con contains)
[ ] Caja 5: costo_peajes_real = costo_peajes × factor_peaje_real
[ ] Caja 6: subtotal_costo creada
[ ] Caja 7: precio_total creada y redondeada
[ ] Caja 8: recomendacion_oferta creada
[ ] (Opcional) Tabla Servicios_Especiales creada con nombre_servicio + costo_servicio
[ ] Probado con bundle box-truck-normal → precio coherente
[ ] Probado con bundle edge-case (con servicios) → precio mayor
```

---

## 🧪 Valores esperados con los bundles de prueba

### carga-box-truck-normal.json
```
tipo_camion = box_truck, viaje_millas = 237, deadhead_millas = 45
Tarifa_Milla = $2.50, Factor_Peaje = 1.2, sin servicios
peaje Google = $10.00
costo_millas = 237 × 2.50 = 592.50
costo_deadhead = 45 × 2.50 × 0.5 = 56.25
costo_peajes_real = 10.00 × 1.2 = 12.00
costo_servicios = 0
subtotal = 592.50 + 56.25 + 12.00 + 0 = 660.75
precio_total = 660.75
```

### carga-van-expedited.json
```
tipo_camion = van, viaje_millas = 95, deadhead_millas = 30
Tarifa_Milla = $2.50, Factor_Peaje = 1.2, signature_required (sin costo extra)
peaje Google = $5.00
costo_millas = 95 × 2.50 = 237.50
costo_deadhead = 30 × 2.50 × 0.5 = 37.50
costo_peajes_real = 5.00 × 1.2 = 6.00
costo_servicios = 0
subtotal = 237.50 + 37.50 + 6.00 + 0 = 281.00
precio_total = 281.00
```

### carga-edge-case-reefer-hazmat.json
```
tipo_camion = box_truck (grande), viaje_millas = 156, deadhead_millas = 52
Tarifa_Milla = $3.50, Factor_Peaje = 2.0
servicios: white_glove $75 + inside_delivery $50 + hazmat $100 + reefer $50 = $275
peaje Google = $15.00
costo_millas = 156 × 3.50 = 546.00
costo_deadhead = 52 × 3.50 × 0.5 = 91.00
costo_peajes_real = 15.00 × 2.0 = 30.00
costo_servicios = 275.00
subtotal = 546.00 + 91.00 + 30.00 + 275.00 = 942.00
precio_total = 942.00
```

> Ajusta los números a tus tarifas reales. Lo importante es que la fórmula sea
> coherente: cambia un input y el subtotal cambia en la misma dirección.

---

## 🔗 Relación con variables anteriores

```
Google Maps:     viaje_millas, deadhead_millas, costo_peajes
Webhook Vapi:    tarifa_ofrecida, servicios_especiales, hazmat, refrigerado
Airtable:        Tarifa_Milla, Factor_Peaje (lookup del camión ganador)
                 Servicios_Especiales (tabla: nombre_servicio + costo_servicio)
Cajas precio:    tarifa_por_milla_real + factor_peaje_real
                   → costo_millas_cargadas → costo_deadhead → costo_peajes_real
                   → costo_servicios_especiales → subtotal → precio_total
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
