# 0001 — Trabajar en minutos en vez de horas

**Fecha:** 2026-05-23
**Estado:** Aceptada

## Contexto

El sistema necesita calcular HOS, duración de viajes, buffers y deadhead. Google Maps Routes API devuelve duración en segundos (ej. `14209s`). Airtable maneja mal los decimales cuando se trabaja en horas (`2.45h` se vuelve confuso para comparar).

## Decisión

Todas las variables de tiempo internas del sistema se manejan en **minutos enteros**.

Las horas solo se usan en la salida final hacia el humano (Vapi habla "1 hora y 20 minutos", el broker ve horas en pantalla).

## Alternativas consideradas

- **Horas decimales (2.45h):** se descarta por errores de redondeo y dificultad para comparar fórmulas en Airtable.
- **Segundos:** demasiado granular, números grandes innecesarios.

## Consecuencias

- Toda fórmula de Airtable / Make trabaja en minutos.
- Conversión obligatoria al recibir datos de Google Maps:
  `valor_en_segundos / 60` y redondeo.
- HOS de 11 horas = 660 minutos en todo el sistema.
- La capa de presentación (Vapi, pantallas) convierte minutos → "Xh Ym".
