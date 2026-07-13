# 0003 — Reposicionamiento y parada final dinámica

**Fecha:** 2026-07-13
**Estado:** Fase 1 aceptada e implementada. Fases 2-3 diseñadas, pendientes.

## Contexto

Cuando un camión entrega en otra ciudad, hay una decisión de negocio real:
¿regresa vacío a la base (gasta diésel sin generar) o se queda en la ciudad
de entrega esperando una carga de regreso (backhaul)?

Las flotas reales deciden según el **mercado de carga** de la ciudad destino:
- Ciudad caliente (Houston, San Antonio) → quedarse, saldrá carga pronto.
- Ciudad fría / pueblo → regresar, esperar ahí es perder tiempo.

Un camión que regresa CARGADO gana casi el doble por el mismo diésel.

## Decisión

Implementar en 3 fases, empezando por la más simple (que da ~70% del valor):

### FASE 1 — Parada final al confirmar (IMPLEMENTADA)

Al confirmar un booking, Make actualiza el camión:

```
[Airtable Update Record: Inventario de Flota]
   Ubicacion_Operacional_Actual = delivery_city
   Parada_final                 = delivery_city
   Disponible_Desde             = eta_delivery + buffer_descarga
```

Y la Automation de "Delivered" (ver hos-automatico.md) mantiene el ciclo.

**Por qué esto ya da el backhaul "gratis":** el camión queda registrado EN la
ciudad de entrega. Cuando mañana llegue una carga desde esa ciudad, el Search
de candidatos lo encuentra ahí con deadhead ≈ 0 → score altísimo → gana.
El reposicionamiento inteligente emerge del score sin lógica adicional.

### FASE 2 — Tabla Ciudades_Estrategicas (PENDIENTE)

La decisión "quedarse vs regresar" como DATO configurable, no código.

**Tabla `Ciudades_Estrategicas`:**

| Campo | Tipo | Ejemplo |
|---|---|---|
| ciudad | Text | Houston, TX |
| mercado | Select | Caliente / Medio / Frío |
| max_horas_espera | Number | 12 |
| radio_min_cercania | Number | 45 (minutos de manejo) |

Valores iniciales sugeridos (base Dallas):

```
Dallas, TX        | Caliente | —  (es base) | 45
Houston, TX       | Caliente | 12           | 45
San Antonio, TX   | Caliente | 10           | 30
Austin, TX        | Caliente | 8            | 30
Oklahoma City, OK | Medio    | 6            | 30
Little Rock, AR   | Frío     | 3            | 30
```

**Regla de decisión** (en el escenario de confirmación, tras crear el booking):

```
¿La ciudad de entrega (o una estratégica a < radio_min via Google Maps)
 está en la tabla?
   NO → parada_final = Base_Operacional (regresa)
   SÍ → ¿el perfil del operador permite quedarse?
        (Pernocta_OK, o Solo_Local con mucho turno restante)
      NO → regresa
      SÍ → ¿cabe la espera sin romper su próxima cita futura?
           (hora_cita_futura − fin_entrega) >
              (max_horas_espera + viaje_a_cita + buffer)
         NO → regresa (la cita programada manda)
         SÍ → parada_final = ciudad estratégica
              Estado_Camion = "Esperando_Carga"
              Espera_Hasta = fin_entrega + max_horas_espera
```

**Pueblo cercano a ciudad grande:** si la entrega es en un pueblo, 1 llamada
a Google Maps (pueblo → estratégica más cercana). Si < radio_min_cercania,
la parada final se mueve a la ciudad grande (mejor probabilidad de reload).
Costo: 1 op por booking confirmado — barato porque solo corre en
confirmaciones, no en cotizaciones.

### FASE 3 — Timeout de espera (PENDIENTE)

Escenario de Make programado cada hora:

```
[Scheduler 60 min]
   ↓
[Search: Estado_Camion = "Esperando_Carga" AND Espera_Hasta < NOW()]
   ↓ por cada camión vencido:
[Update: Estado_Camion = "Regresando_A_Base"
         Parada_final = Base_Operacional
         Disponible_Desde = now + tiempo_regreso (Google Maps)]
   ↓ (opcional)
[Notificación al dispatcher: "BOX-07 no consiguió carga en Houston,
 regresando a Dallas"]
```

La protección de citas futuras NO necesita lógica nueva: la Fase 2 ya validó
que espera + regreso caben antes de la próxima cita (mismo patrón
`respeta_proxima_cita` del escenario de cotización).

## Alternativas consideradas

- **Módulo de "espera de carga" explícito desde el día 1:** descartado.
  El score por deadhead ya premia al camión que quedó en la ciudad de la
  nueva carga — la Fase 1 sola produce el comportamiento de backhaul.
- **Decidir quedarse/regresar con IA (LLM):** descartado. Es una regla de
  negocio determinística; una tabla la resuelve mejor, más barato y auditable.
- **Mercados como código en Make:** descartado. Como tabla en Airtable, el
  dueño ajusta ciudades y tiempos sin tocar la automatización (mismo
  principio que la tabla Config).

## Consecuencias

- Fase 1 agrega 1 módulo Update al escenario de confirmación.
- Fases 2-3 requieren: tabla nueva, 2 campos en Inventario de Flota
  (`Estado_Camion`, `Espera_Hasta`), 1 escenario programado, ~1 op extra
  por confirmación.
- El orden de implementación importa: Fase 2 solo tiene sentido con volumen
  real de cargas. Construirla antes del primer cliente es pulir un motor
  que no ha corrido su primera carrera.

## Cambios

- 2026-07-13 — versión inicial. Fase 1 implementada en el escenario de
  confirmación; Fases 2-3 quedan diseñadas para cuando haya volumen.
