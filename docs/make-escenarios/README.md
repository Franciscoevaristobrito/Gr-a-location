# Make — Escenarios

Documentación de cada escenario de Make.com. Un archivo por escenario.

## Para qué sirve

Cuando un escenario falle dentro de 3 meses, o quieras modificarlo, este archivo te dice **qué hace, en qué orden y por qué**. Sin abrir Make.

## Plantilla

Archivo: `NN-nombre-escenario.md`

```markdown
# Escenario NN — Nombre

**Última actualización:** YYYY-MM-DD
**Trigger:** (webhook Vapi / Airtable / programado / etc.)
**Output:** (qué produce al final)

## Propósito
Una línea: qué resuelve.

## Flujo (cajas y flechas)

[Webhook Vapi]
    ↓
[Tools: parseDate pickup_time]
    ↓
[Airtable: search operadores viables]
    ↓
...

## Módulos clave

| # | Módulo | Qué hace | Variable que produce |
|---|---|---|---|
| 1 | Webhook | Recibe request | `pickup_city`, `pickup_time`, ... |
| 2 | Tools | parseDate | `pickup_time_formateado` |

## Variables que produce
- `pickup_time_formateado`
- `deadhead_minutos`
- ...

## Errores conocidos
- Si Google Maps devuelve 0 → ...
- Si no hay operadores activos → ...

## Cambios importantes
- 2026-05-23 — versión inicial
```
