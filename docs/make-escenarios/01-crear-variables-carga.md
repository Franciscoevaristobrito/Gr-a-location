# Guía — Crear las 12 variables de carga en Make

Tutorial paso a paso para que Make reciba las 12 variables de carga desde Vapi y las pueda usar en el resto del escenario.

**Nombres en español** (consistencia con el resto del proyecto).

---

## 🎯 Lo que vamos a hacer

```
Vapi (llamada de voz)
    ↓ envía JSON al webhook
Make → recibe las 12 variables
    ↓ las parsea según una estructura de datos
Make → ya puede usarlas en módulos posteriores
       (Set Variable, Airtable Search, Google Maps, etc.)
```

---

## 📋 Las 12 variables que vamos a crear

| # | Nombre | Tipo en Make | Requerida |
|---|---|---|---|
| 1 | `tipo_camion` | Text | ✅ Sí |
| 2 | `tipo_carga` | Text | ✅ Sí |
| 3 | `cantidad_pallets` | Number | ✅ Sí |
| 4 | `peso_total_libras` | Number | ✅ Sí |
| 5 | `refrigerado` | Boolean | ✅ Sí |
| 6 | `hazmat` | Boolean | ✅ Sí |
| 7 | `servicios_especiales` | Text | ❌ No |
| 8 | `tipo_recogida` | Text | ❌ No |
| 9 | `metodo_carga` | Text | ❌ No |
| 10 | `requiere_liftgate` | Boolean | ❌ No |
| 11 | `hazmat_clase` | Text | ❌ No |
| 12 | `temperatura_requerida_f` | Number | ❌ No |

---

## PASO 1 — Crear el webhook en Make

Si ya tienes un webhook recibiendo de Vapi, salta al **Paso 2**.

1. Abre tu escenario de Make.
2. Click en el **círculo grande con el `+`** (módulo inicial).
3. Busca: **Webhooks**.
4. Selecciona **Custom webhook**.
5. Click en **Add** para crear un webhook nuevo.
6. Ponle un nombre descriptivo:
   ```
   vapi_solicitud_carga
   ```
7. **Save**.
8. Copia la URL que te muestra Make. La vas a usar en Vapi después.

Ejemplo de URL:
```
https://hook.us2.make.com/abcd1234efgh5678
```

---

## PASO 2 — Definir la estructura de datos

Aquí es donde "creas" las 12 variables. Make necesita saber qué campos esperar y de qué tipo, para que aparezcan como variables seleccionables en los módulos siguientes.

1. Click sobre el módulo **Webhook** ya creado.
2. Click en **Add** al lado de **Data structure** (o "Show advanced settings" si no aparece).
3. Ponle nombre a la estructura:
   ```
   carga_desde_vapi
   ```

4. Click en **Add item** y crea cada variable, una por una:

### Variable 1: tipo_camion

```text
Name:     tipo_camion
Type:     Text
Required: yes
```

### Variable 2: tipo_carga

```text
Name:     tipo_carga
Type:     Text
Required: yes
```

### Variable 3: cantidad_pallets

```text
Name:     cantidad_pallets
Type:     Number
Required: yes
```

### Variable 4: peso_total_libras

```text
Name:     peso_total_libras
Type:     Number
Required: yes
```

### Variable 5: refrigerado

```text
Name:     refrigerado
Type:     Boolean
Required: yes
```

### Variable 6: hazmat

```text
Name:     hazmat
Type:     Boolean
Required: yes
```

### Variable 7: servicios_especiales

```text
Name:     servicios_especiales
Type:     Text
Required: no
```

### Variable 8: tipo_recogida

```text
Name:     tipo_recogida
Type:     Text
Required: no
```

### Variable 9: metodo_carga

```text
Name:     metodo_carga
Type:     Text
Required: no
```

### Variable 10: requiere_liftgate

```text
Name:     requiere_liftgate
Type:     Boolean
Required: no
```

### Variable 11: hazmat_clase

```text
Name:     hazmat_clase
Type:     Text
Required: no
```

### Variable 12: temperatura_requerida_f

```text
Name:     temperatura_requerida_f
Type:     Number
Required: no
```

5. Click **Save** cuando termines las 12.
6. Click **OK** para cerrar el módulo webhook.

---

## PASO 3 — Probar el webhook con un payload de ejemplo

Make necesita "ver" datos reales una vez para empezar a mostrarte las variables en los módulos siguientes. Tienes 2 formas:

### Opción A — Pegar JSON manualmente (más rápido)

1. En el módulo webhook, click en **Redetermine data structure**.
2. Make te dará una URL temporal para enviar datos.
3. Usa Postman, curl o cualquier herramienta para enviar este JSON:

```json
{
  "tipo_camion": "box_truck",
  "tipo_carga": "palletized",
  "cantidad_pallets": 8,
  "peso_total_libras": 4500,
  "refrigerado": false,
  "hazmat": false,
  "servicios_especiales": "",
  "tipo_recogida": "live_load",
  "metodo_carga": "forklift_dock",
  "requiere_liftgate": false,
  "hazmat_clase": "",
  "temperatura_requerida_f": 0
}
```

### Opción B — Llamada real desde Vapi

1. Configura Vapi con la URL del webhook (la del Paso 1).
2. Haz una llamada de prueba.
3. Make capturará la estructura automáticamente.

---

## PASO 4 — Verificar que Make reconoce las variables

1. Agrega un módulo nuevo después del webhook (por ejemplo, **Tools → Set variable** o el siguiente que ya tenías).
2. En el campo de valor, click para abrir el panel de variables.
3. Deberías ver las 12 variables disponibles bajo el nombre de tu webhook:

```
vapi_solicitud_carga
  ├─ tipo_camion          (Text)
  ├─ tipo_carga           (Text)
  ├─ cantidad_pallets     (Number)
  ├─ peso_total_libras    (Number)
  ├─ refrigerado          (Boolean)
  ├─ hazmat               (Boolean)
  ├─ servicios_especiales (Text)
  ├─ tipo_recogida        (Text)
  ├─ metodo_carga         (Text)
  ├─ requiere_liftgate    (Boolean)
  ├─ hazmat_clase         (Text)
  └─ temperatura_requerida_f (Number)
```

Si las ves: **listo**. Ya las puedes usar en cualquier módulo posterior.

Si NO las ves: revisa los errores comunes al final del doc.

---

## PASO 5 (opcional) — Crear Set Variable modules para derivados

Algunas variables se calculan A PARTIR de las que vienen de Vapi. No las recibes, las derivas.

Ejemplo claro: **`requiere_liftgate`** se puede deducir de `metodo_carga`.

### Cómo crear un Set Variable derivado

1. Después del webhook, agrega un módulo **Tools → Set variable**.
2. Configura:

```text
Variable name:  requiere_liftgate_calculado
Variable value:
  if( metodo_carga = "liftgate" ; true ;
  if( metodo_carga = "forklift_dock" ; false ;
      requiere_liftgate ))
```

Esto significa:
- Si el método es liftgate → true
- Si el método es forklift_dock → false
- Si no, usar lo que Vapi haya enviado en `requiere_liftgate`

### Otros derivados útiles

**`carga_es_viable_van`** (boolean — para filtrar Van rápidamente):

```text
Variable name:  carga_es_viable_van
Variable value:
  if( tipo_camion = "van"
      AND cantidad_pallets <= 6
      AND peso_total_libras <= 3000
      AND requiere_liftgate = false ;
      true ; false )
```

**`carga_es_viable_box_truck`**:

```text
Variable name:  carga_es_viable_box_truck
Variable value:
  if( tipo_camion = "box_truck"
      AND cantidad_pallets <= 12
      AND peso_total_libras <= 10000 ;
      true ; false )
```

Estos derivados te ahorran condiciones repetitivas más adelante en el filtro de operadores.

---

## 🚨 Errores comunes

### "No veo las variables en el panel"

**Causa típica:** Make todavía no ha recibido datos reales o de prueba.
**Solución:** Ejecuta el escenario una vez con datos de prueba (Paso 3).

### "Las variables aparecen pero son todas Text, incluso los números"

**Causa típica:** Cuando ejecutaste el webhook, mandaste el JSON con números entre comillas.
**Solución:** En el JSON de prueba, los números van SIN comillas:
- ✅ `"cantidad_pallets": 8`
- ❌ `"cantidad_pallets": "8"`

### "Refrigerado/hazmat me llegan como texto 'true'/'false' en vez de boolean"

**Causa típica:** Vapi a veces serializa booleans como strings.
**Solución:** En el JSON, asegúrate que sea boolean nativo:
- ✅ `"refrigerado": false`
- ❌ `"refrigerado": "false"`

Si Vapi te manda string y no se puede cambiar, crea un Set Variable para convertirlo:

```text
Variable name:  refrigerado_bool
Variable value: if( refrigerado = "true" ; true ; false )
```

### "El nombre de la variable no coincide con lo que Vapi envía"

**Causa típica:** Mayúsculas o guiones diferentes.
**Solución:** El nombre en Make y en Vapi debe ser **idéntico**:
- ✅ `tipo_camion` ↔ `tipo_camion`
- ❌ `tipo_camion` ↔ `tipoCamion`
- ❌ `tipo_camion` ↔ `Tipo_Camion`

### "Make recibe la llamada pero ignora algunos campos"

**Causa típica:** Esos campos no están definidos en la Data Structure.
**Solución:** Vuelve al Paso 2 y agrégalos a la estructura.

---

## ✅ Checklist final

```text
[ ] Webhook creado y URL copiada
[ ] Data Structure creada con las 12 variables (tipo correcto cada una)
[ ] Webhook probado con JSON de ejemplo
[ ] Variables visibles en el panel de selección
[ ] (Opcional) Set Variable de requiere_liftgate_calculado creado
[ ] (Opcional) Set Variable de carga_es_viable_van creado
[ ] (Opcional) Set Variable de carga_es_viable_box_truck creado
[ ] URL del webhook configurada en Vapi
[ ] Llamada de prueba real desde Vapi → JSON guardado en docs/bundles-ejemplo/
```

---

## 📦 JSON de prueba completo (para copiar)

### Ejemplo 1 — Box Truck, carga normal

```json
{
  "tipo_camion": "box_truck",
  "tipo_carga": "palletized",
  "cantidad_pallets": 8,
  "peso_total_libras": 4500,
  "refrigerado": false,
  "hazmat": false,
  "servicios_especiales": "",
  "tipo_recogida": "live_load",
  "metodo_carga": "forklift_dock",
  "requiere_liftgate": false,
  "hazmat_clase": "",
  "temperatura_requerida_f": 0
}
```

### Ejemplo 2 — Van, expedited

```json
{
  "tipo_camion": "van",
  "tipo_carga": "boxes",
  "cantidad_pallets": 0,
  "peso_total_libras": 800,
  "refrigerado": false,
  "hazmat": false,
  "servicios_especiales": "signature_required",
  "tipo_recogida": "live_load",
  "metodo_carga": "manual",
  "requiere_liftgate": false,
  "hazmat_clase": "",
  "temperatura_requerida_f": 0
}
```

### Ejemplo 3 — Refrigerado + hazmat (edge case)

```json
{
  "tipo_camion": "box_truck",
  "tipo_carga": "refrigerated_small",
  "cantidad_pallets": 6,
  "peso_total_libras": 3200,
  "refrigerado": true,
  "hazmat": true,
  "servicios_especiales": "white_glove,inside_delivery",
  "tipo_recogida": "live_load",
  "metodo_carga": "liftgate",
  "requiere_liftgate": true,
  "hazmat_clase": "9",
  "temperatura_requerida_f": 38
}
```

Estos JSONs guárdalos en `docs/bundles-ejemplo/` para reusarlos cuando hagas cambios en el escenario.
