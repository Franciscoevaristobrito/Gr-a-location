# 0002 — Modelo de buffers separados por segmento

**Fecha:** 2026-05-23
**Estado:** Aceptada

## Contexto

El sistema necesita estimar el tiempo total de una operación (deadhead + carga + viaje + descarga). Hay varias formas de modelar los buffers (márgenes de tolerancia):

- Un solo buffer general para toda la operación.
- Un buffer por etapa (deadhead, carga, viaje, descarga).
- Un buffer dinámico aprendido del historial por cliente.

## Decisión

Usar **buffers separados por segmento**, calculados con fórmulas distintas, pero sin perfiles por cliente todavía:

```text
tiempo_jornada_completa =
    deadhead_minutos
  + buffer_carga          (depende de método + pallets + extras)
  + viaje_minutos
  + buffer_carretera      (depende de duración del viaje)
  + buffer_descarga       (similar a buffer_carga, ~80% del tamaño)
```

## Alternativas consideradas

- **Un solo buffer global:** descartado porque mezcla retrasos de carretera con esperas de dock, dos cosas muy distintas. Imposible afinar sin separarlas.
- **Buffer por cliente (perfil histórico):** descartado para esta fase. Requiere histórico de viajes que aún no tenemos. Se retoma en Fase 2 cuando haya datos.
- **Buffer fijo de 2 horas estándar industria:** desperdicia capacidad. Solo serviría como worst-case para SLA, no para despacho.

## Consecuencias

- Se necesitan campos adicionales del webhook Vapi: `tipo_carga`, `metodo_carga`, `cantidad_pallets`, `tipo_recogida`, `servicios_especiales`. Documentado en [`webhook-vapi-schema.md`](../webhook-vapi-schema.md).
- La fórmula `buffer_operacional` actual se renombra a `buffer_carretera` para claridad.
- Aparecen dos variables nuevas: `buffer_carga` y `buffer_descarga`.
- Cuando haya 50-100 viajes históricos, se debería evolucionar a perfil por cliente (ver Fase 2 en webhook-vapi-schema).

## Tabla base (Box Truck y Van)

Tiempos base por método de carga (minutos):

| Método | Carga | Descarga |
|---|---|---|
| `forklift_dock` | 20 | 15 |
| `liftgate` | 40 | 30 |
| `manual` | 60 | 50 |
| `driver_assist` | 90 | 70 |

Por pallet (minutos):

| Método | Min/pallet |
|---|---|
| `forklift_dock` | 2 |
| `liftgate` | 3 |
| `manual` / `driver_assist` | 5 |

Extras:

| Servicio | Suma |
|---|---|
| `white_glove` | +30 |
| `inside_delivery` | +20 |
| `appointment_only` sin cita | +30 |
| `hazmat` | +30 |
| `refrigerado` | +15 |
