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

### FASE 2b — Precio por mercado destino (headhaul/backhaul pricing) (PENDIENTE)

La misma tabla `Ciudades_Estrategicas` alimenta el PRECIO: cobrar según qué
tan probable es conseguir carga de regreso en el destino.

```
Entrega en ciudad CALIENTE (Houston): casi seguro hay backhaul
   → el regreso "se paga solo" → cotizar MÁS BARATO → ganar la carga
Entrega en ciudad FRÍA / pueblo: regreso vacío casi seguro
   → el precio debe cubrir parte del regreso → cotizar MÁS ALTO
```

**Campo adicional en Ciudades_Estrategicas:**

| ciudad | mercado | factor_retorno |
|---|---|---|
| Houston, TX | Caliente | 0.0 |
| San Antonio, TX | Caliente | 0.1 |
| Oklahoma City, OK | Medio | 0.3 |
| Little Rock, AR | Frío | 0.5 |
| (no está en la tabla) | — | 0.7 (default pueblo) |

`factor_retorno` = qué fracción del costo del viaje de regreso se le cobra a
esta carga. El nivel de mercado es el PROXY de la probabilidad de backhaul
(Caliente ≈ 80%+, Medio ≈ 50%, Frío ≈ 20%, fuera de tabla ≈ 0%).

**En Make (cadena de precios):**

```
[Airtable Search: Ciudades_Estrategicas WHERE ciudad ≈ delivery_city, Max 1]
[Array Aggregator anti-rotura]
[Set var: factor_retorno = ifempty(first(Array.factor_retorno); 0.7)]
[Set var: costo_reposicionamiento =
    viaje_millas_real × tarifa_por_milla_real × factor_retorno × 0.5]
[subtotal += costo_reposicionamiento]
```

Ejemplo (283 mi, tarifa $2.50): Houston (0.0) → +$0. Pueblo (0.7) → +$248.
Misma distancia, la diferencia es la realidad económica del regreso vacío.

### FASE 4 — Probabilidad de backhaul con datos reales (FUTURO LEJANO)

NO modelar probabilidades sin historial: serían números inventados.
Cuando Bookings tenga 100+ viajes completados, calcular por ciudad:

```
prob_backhaul(ciudad) =
    viajes donde se consiguió carga de regreso desde esa ciudad
  / viajes que entregaron en esa ciudad
```

Y derivar el factor con datos: `factor_retorno = (1 − prob_backhaul) × ajuste`.
Hasta entonces, los niveles Caliente/Medio/Frío ajustados a mano por el
dueño son suficientes (así operan los dispatchers humanos).

## Árbol de decisión completo (referencia de implementación)

**Al COTIZAR (afecta el precio):**

```
¿delivery_city está en Ciudades_Estrategicas?
   Caliente → factor_retorno 0.0 → precio competitivo
   Frío/no  → factor 0.5-0.7    → el precio cubre el regreso vacío
```

**Al CONFIRMAR/ENTREGAR (decide la Parada_final):**

```
¿Le queda tiempo al operador para regresar a base?
│
├─ SÍ (viaje ≤ max_horas_regreso y el turno alcanza)
│    ¿Ciudad de entrega es Caliente y su perfil permite esperar?
│       SÍ → Parada_final = delivery_city (espera carga, max_horas_espera)
│       NO → Parada_final = Base_Operacional (regresa)
│
└─ NO (HOS/turno acabándose)
     ¿Perfil = Pernocta_OK?
        NO → nunca debió ganar este viaje (tipo_viabilidad ya lo filtra)
        SÍ → ¿Entrega en/cerca de ciudad grande (< radio_min_cercania)?
              SÍ → Parada_final = la ciudad grande (duerme donde hay mercado)
              NO → Parada_final = delivery_city (duerme ahí, mañana decide)
```

Las citas futuras siempre quedan protegidas por `respeta_proxima_cita`.

**Nota sobre los 2 campos de Bookings:** `dirección-de-entrega` (a dónde va
la CARGA, contrato, nunca cambia) y `Parada-final` (dónde termina el CAMIÓN,
decisión operacional). Hoy casi siempre coinciden; este árbol es lo que los
hará diferir cuando convenga (entrega en Flint, dormir en Detroit).

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
- 2026-07-14 — decisión: Parada_final se usa como la siguiente ubicación
  del operador (Fase 1, simple). Se agregan al diseño futuro: Fase 2b
  (precio por mercado destino con factor_retorno), Fase 4 (probabilidad
  de backhaul con datos reales) y el árbol de decisión completo.
  Corrección importante: la ubicación se actualiza al DELIVERED (Automation),
  no al reservar — reservar solo escribe en la agenda (Bookings).
