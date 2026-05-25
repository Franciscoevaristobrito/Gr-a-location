# Airtable — Schema

Documentación del schema de cada tabla. Un archivo por tabla.

## Para qué sirve

Saber qué campos existen, qué tipo son, sus fórmulas y vistas, sin tener que abrir Airtable.

## Plantilla

Archivo: `nombre-tabla.md`

```markdown
# Tabla: Nombre

**Última actualización:** YYYY-MM-DD
**Propósito:** Qué representa esta tabla en el sistema.

## Campos

| Campo | Tipo | Descripción | Fórmula / Origen |
|---|---|---|---|
| Field A | Single line text | ... | (manual) |
| Field B | Formula | ... | `IF(...)` |
| Field C | Link | Link a Otra Tabla | - |

## Vistas

| Vista | Filtro | Para qué |
|---|---|---|
| Activos | `estado = "Activo"` | Operadores disponibles ahora |

## Relaciones
- Field C → Tabla "Otra Tabla"

## Fórmulas críticas (con explicación)

### Estado_Temporal_Cita

```text
IF(Hora_Fin_Estimada < NOW(), "Pasada",
  IF(AND(Hora_Inicio <= NOW(), Hora_Fin_Estimada >= NOW()), "Actual",
    "Futura"))
```

Por qué: clasifica la cita en función del momento actual.

## Cambios importantes
- 2026-05-23 — versión inicial
```
