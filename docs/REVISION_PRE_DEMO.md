# 🔍 Revisión Pre-Demo + Visión del Sistema — AI Dispatch Brain

**Propósito:** documentar cómo está construido TODO el sistema (la visión) y, al mismo
tiempo, revisar cada parte antes del demo — anotando problemas y mejoras SIN arreglarlos
todavía. Primero encontrar, luego arreglar, luego vender.

**Última actualización:** 2026-09-05

---

## 🧭 Cómo usar este documento

Recorre cada sección EN ORDEN. Por cada una:
1. Lee **"Cómo está construido"** → confirma que entiendes esa pieza.
2. Haz la **checklist de "Probar"** → como lo haría un cliente.
3. Escribe en **"Notas"** cada problema/mejora, con su severidad.
4. Marca `[x] Revisado` cuando termines la sección.

### La regla de oro
> **HOY solo se ENCUENTRA y se ANOTA. No se arregla nada.**
> Si te dan ganas de arreglar → anótalo y sigue. Arreglar en el momento = te atascas
> y nunca terminas la revisión.

### Sistema de severidad (ponlo al momento de anotar)
```
🔴 BLOQUEA DEMO   → un cliente diría "no" si ve esto → arréglalo antes de salir
🟡 DESPUÉS        → se ve mal pero no bloquea → backlog post-demo
🟢 NICE TO HAVE   → detalle menor → algún día
```
La mayoría será 🟡/🟢. **Solo los 🔴 detienen el demo.**

---

## 🗺️ El mapa del sistema (visión general)

```
                          ┌──────────────────────────┐
   Broker llama  ──voz──> │  VAPI (IA de voz)        │
                          │  cotiza · confirma ·      │
                          │  cancela · estado         │
                          └────────────┬─────────────┘
                                       │ webhook
                                       ▼
                          ┌──────────────────────────┐
                          │  MAKE (automatización)   │
                          │  motor de despacho:       │
                          │  HOS · deadhead · score   │
                          │  → elige el mejor camión  │
                          └────────────┬─────────────┘
                                       │ lee/escribe
                                       ▼
                          ┌──────────────────────────┐
                          │  AIRTABLE (base de datos) │
                          │  Reservas · Operadores ·  │
                          │  Turnos · Brokers · Config│
                          │  + automatizaciones       │
                          └────────────┬─────────────┘
                                       │ comparte (no la base)
                                       ▼
                          ┌──────────────────────────┐
                          │  INTERFACES (el dueño)    │
                          │  Dashboard · Dispatch ·   │
                          │  Loads · Drivers · etc.   │
                          └──────────────────────────┘

   Operador (chofer)  ──form/link──> "Ya salí", entrega POD, accidente
```

**En una frase:** el broker habla con la IA → la IA cotiza y despacha usando el motor →
todo vive en Airtable → el dueño lo controla desde interfaces → el chofer reporta por links.

---

## 1. 📞 Recepción por voz — Vapi

**Cómo está construido:**
- Squad de Vapi: un asistente de TRIAJE identifica la intención y hace handoff a:
  reserva, cambios (cancelar/reprogramar), y estado/tracking.
- Tool `calcular_ruta_y_precio` (cotizar) → webhook a Make → responde en formato
  `results[{toolCallId, result}]` con bloque `_internal` para los IDs.
- Pasa NOMBRES, no IDs (el LLM alucina los recXXX).
- Captura el MC del broker para la verificación anti-fraude.

**Probar:**
- [ ] Broker llama → la IA pide origen/destino/carga → cotiza con precio
- [ ] Confirmar la carga → se crea la cita
- [ ] Pide el MC + company del broker
- [ ] Handoff del triaje a reserva / cambios / estado funciona
- [ ] La IA lee de vuelta el precio y confirma

**Estado:** [ ] Revisado
**Notas:**
```
(severidad · qué pasó)

```

---

## 2. 🛡️ Verificación de brokers (anti-fraude)

**Cómo está construido:**
- Tabla `Brokers` (MC_Number, Company, Verificado, Verificacion_API…).
- En el escenario de cotización: Upsert por MC (crea/actualiza sin duplicar).
- `Verificacion_API` = lo que dice la API/máquina; `Verificado` = decisión del dueño.
- Gate: la carga no pasa a "Programada" hasta que el broker esté verificado.

**Probar:**
- [ ] Broker conocido y verificado → cita nace "Programada"
- [ ] Broker nuevo → cita "Pendiente_Verificacion" + aviso al dueño
- [ ] El Upsert NO duplica al correr 2 veces con el mismo MC
- [ ] Aprobar el broker en el panel → sus citas pasan a Programada (pieza #3)

**Estado:** [ ] Revisado
**Notas:**
```

```

---

## 3. 🚛 Motor de despacho — Make

**Cómo está construido:**
- Filtra operadores viables (HOS, tipo de camión, estado, disponibilidad).
- Deadhead + viaje con Google Routes (peajes incluidos).
- Buffers modulares; validación de regreso a casa / próxima cita.
- Score multifactor → elige el mejor candidato.
- (Avanzado) deadhead desde la `Parada_Final_Ubicacion` de la última cita.

**Probar:**
- [ ] Elige el camión más cercano / mejor score
- [ ] Respeta HOS (no asigna a quien no tiene horas)
- [ ] Respeta el tipo de camión pedido
- [ ] El deadhead se calcula bien (millas razonables)
- [ ] Si no hay camión → responde "no disponible" sin romperse

**Estado:** [ ] Revisado
**Notas:**
```

```

---

## 4. 🚚 Flujo del operador (chofer)

**Cómo está construido:**
- "Ya salí" (link/form) → cita a `En_Route`.
- Entrega: form (Fillout o Airtable) → sube POD_Fotos, marca `Entregado`.
- Links con `?prefill_...` prellenan la carga; formato `[texto](url)` en correos.

**Probar:**
- [ ] Correo al operador cuando la cita pasa a "Programada"
- [ ] Link "Ya salí" → la cita pasa a En_Route
- [ ] (opcional) aviso al cliente "on the way"
- [ ] Entrega: sube fotos → Trip-Status = Entregado (sin crear cita nueva)
- [ ] Freno de daño: Posible_Dano ✔ → no factura auto (cola del dueño)

**Estado:** [ ] Revisado
**Notas:**
```

```

---

## 5. 🚨 Accidente / incidente

**Cómo está construido:**
- Tabla `Incidentes` + form del chofer (tipo, ubicación, chofer_estado, fotos…).
- Automatización: congela la carga (Accident_Hold), saca el camión de servicio,
  alerta al dueño (con estado del chofer), avisa al cliente.

**Probar:**
- [ ] El chofer reporta → la carga se congela (Accident_Hold)
- [ ] El camión queda Fuera_de_Servicio
- [ ] Llega el email al dueño (con el estado del chofer)
- [ ] Llega el email al cliente (proactivo)

**Estado:** [ ] Revisado
**Notas:**
```

```

---

## 6. 💰 Facturación (ciclo del dinero)

**Cómo está construido:**
- Factura al entregar (con POD), NET-30.
- Monto_Factura, Estado_Pago, Fecha_Vencimiento (fórmulas).
- Vistas "Por Cobrar" / "Vencidas" con total.

**Probar:**
- [ ] Al entregar → se genera la factura con el monto correcto
- [ ] Fecha de vencimiento (NET-30) se calcula bien
- [ ] Vistas de cobros muestran los totales correctos

**Estado:** [ ] Revisado
**Notas:**
```

```

---

## 7. 🎙️ Consulta de estado por voz

**Cómo está construido:**
- Tool `consultar_estado_carga` → webhook Make → busca por ID-travel →
  traduce el Trip-Status a frase humana (o devuelve el estado y GPT lo arma).

**Probar:**
- [ ] Dueño/broker llama, da el número → la IA lee el estado correcto
- [ ] Estado inexistente → "no encuentro esa carga"
- [ ] Tolera el formato del número dicho por voz

**Estado:** [ ] Revisado
**Notas:**
```

```

---

## 8. 🖥️ Interfaces del dueño

**Cómo está construido:**
- Se comparte el Interface (no la base) → el dueño no ve fórmulas/automatizaciones.
- Páginas: Dashboard, Dispatch Board, Loads, Drivers, Schedule, Fleet, Billing,
  Conflicts, Approvals, History.

**Probar (una por una):**
- [ ] Dashboard: KPIs correctos (Total Charged, Delivered, Cancelled)
- [ ] Dispatch Board: disponibilidad (huecos) + filtro truck type/fecha + botón New Load
- [ ] Loads: lista de cargas
- [ ] Drivers: perfil + costos editables
- [ ] Schedule: turnos editables + filtro por operador
- [ ] Fleet / Billing / Conflicts / Approvals / History
- [ ] Nada técnico visible (minutos, fórmulas) — solo lo del dueño
- [ ] Se ve bien en CELULAR

**Estado:** [ ] Revisado
**Notas:**
```

```

---

## 9. ⚙️ Automatizaciones de Airtable

**Cómo está construido:**
- Generación de turnos (script, ventana rodante, no duplica — por op.id + fecha UTC).
- Detección de conflictos (script overlap por operador).
- Detección de duplicados de cita (clave_cita).
- Parada-final (dónde descansa/espera el camión) para el próximo deadhead.
- Auto-enlace del turno a la cita (para restar minutos → disponibilidad de la IA).

**Probar:**
- [ ] Turnos se generan solos (solo días laborales, sin duplicar al correr 2 veces)
- [ ] Conflicto: 2 citas del mismo operador que chocan → ⚠️ Mensaje_Conflicto
- [ ] Duplicado de cita → se marca / avisa
- [ ] Parada_Final_Ubicacion siempre tiene una ubicación real
- [ ] La cita resta minutos del turno correcto (disponibilidad correcta)

**Estado:** [ ] Revisado
**Notas:**
```

```

---

## 10. ➕ Creación de cita manual (dueño/asistente)

**Cómo está construido:**
- Form profesional (estructura de industria: Customer → Pickup → Delivery →
  Freight → Rate → Dispatch).
- Trip-Status = Programada por prefill (no default global → no rompe verificación).
- Broker nuevo: se crea aparte con MC + email, luego se enlaza.

**Probar:**
- [ ] Crear cita manual → nace "Programada" → entra al flujo normal
- [ ] Broker nuevo → se crea completo y se enlaza
- [ ] Labels en inglés / términos de industria (se ve pro)
- [ ] Campos requeridos no dejan crear cita a medias

**Estado:** [ ] Revisado
**Notas:**
```

```

---

## 📋 Tracker consolidado (llénalo al final)

Después de revisar todo, junta aquí lo anotado, ordenado por severidad:

### 🔴 Bloquea demo (arreglar ANTES de salir)
```
| # | Área | Qué pasó | Estado |
|---|------|----------|--------|
| 1 |      |          |        |
```

### 🟡 Después (post-demo)
```
| # | Área | Qué pasó |
|---|------|----------|
| 1 |      |          |
```

### 🟢 Nice to have
```
| # | Área | Qué pasó |
|---|------|----------|
| 1 |      |          |
```

---

## ✅ Definición de "listo para el demo"

```
[ ] Todos los 🔴 arreglados
[ ] Los flujos principales corren de punta a punta sin caerse:
      cotizar por voz → confirmar → despachar → En_Route → entregar → facturar
[ ] Las interfaces del dueño se ven limpias y profesionales (y en celular)
[ ] La demo se puede MOSTRAR sin vergüenza (los 🟡/🟢 pueden esperar)
```

Cuando esto se cumpla → **sales al demo.** Los detalles restantes se pulen con
feedback real y dinero entrando.

---

## Cambios
- 2026-09-05 — versión inicial. Visión del sistema + guía de revisión pre-demo
  por componente, sistema de severidad, tracker consolidado y definición de "listo".
