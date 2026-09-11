# Bundles de ejemplo

JSON reales capturados durante pruebas. Sirven para:

- Reproducir bugs sin tener que llamar a Vapi otra vez.
- Probar nuevos módulos de Make con datos reales.
- Documentar qué forma tiene cada payload.

## Convención

Nombre de archivo descriptivo:

```text
vapi-request-detroit-chicago.json
google-maps-deadhead-response.json
airtable-search-operadores.json
```

## Cómo capturar un bundle

En Make: ejecutas el escenario, click derecho en el módulo → **Copy output bundle**. Pegas el JSON en un archivo nuevo aquí.

## ⚠️ Datos sensibles

Antes de subir un bundle al repo:
- Quitar números de teléfono reales
- Quitar direcciones de clientes reales (reemplazar por genéricas)
- Quitar tokens / API keys (no deberían estar en bundles, pero por si acaso)
