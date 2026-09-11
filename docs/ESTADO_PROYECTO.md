# 🚛 Estado del Proyecto — AI Dispatch Brain

**Última actualización:** 2026-07-19
**Estado:** MVP completo y funcional. Listo para primer cliente (flotas de brokers).

Snapshot de todo lo construido. Para detalles de cada pieza, ver los docs
enlazados.

---

## ✅ LO QUE ESTÁ CONSTRUIDO Y FUNCIONA

### 1. Motor de despacho (el core)
- Recibe solicitud del broker vía Vapi (voz) → webhook a Make
- Filtra operadores viables: HOS, tipo de camión, estado, disponibilidad
- Calcula deadhead + viaje con Google Maps (Routes API, con peajes)
- Buffers modulares (carga, carretera, descarga)
- Validación de regreso a casa (`puede_regresar`) y próxima cita
- Score multifactor → elige el mejor candidato
- Búsqueda en tabla **Turnos** por día solicitado + hora alternativa si no hay

### 2. Cotización y confirmación
- Precio: `Tarifa_Milla` × millas + deadhead 50% + peajes × `Factor_Peaje` + servicios
- Respuesta a Vapi en formato `results[{toolCallId, result}]` con `_internal`
- `confirm_booking`: pasa NOMBRES (no IDs, que el LLM alucina); Make resuelve IDs
- Rate Confirmation por email al confirmar

### 3. Cancelaciones y reprogramaciones
- Squad de Vapi: triaje → reserva / cambios (handoff)
- `cancel_load` + `confirm_cancellation`: cotiza TONU → acepta → cancela
- `reschedule_load` + `confirm_reschedule`: Detention/Layover + reasignación
- Escalera de 3 niveles: mismo camión → otro camión → hora alternativa
- Tarifas configurables en tabla **Config**

### 4. Facturación (ciclo del dinero, modelo broker)
- Factura al entregar (con POD adjunto), NET-30
- Seguimiento de cuentas por cobrar + recordatorios automáticos
- Vistas "Por Cobrar" y "Vencidas" con total

### 5. Airtable — la base operacional
- HOS automático (Rollups + Formulas, respeta canceladas)
- Detección de conflictos de cita (script) + notificación
- Tabla **Turnos** con disponibilidad en tiempo real (NOW), días libres,
  turnos nocturnos, y control manual (Forzar_Descanso/Activo/Mantenimiento)
- Override de horas manual (el dueño pone la hora que quiera)
- Zona horaria: America/Chicago (base DFW)

### 6. Interface para el dueño de flota (el cliente que paga)
- 6 paneles: Inicio (KPIs + gráfica), Flota (galería), Citas, Cobros,
  Conflictos, Turnos (calendario editable)
- Solo se comparte el Interface (no la base) → no ve fórmulas ni automatizaciones
- Puede crear/editar citas y camiones, gestionar turnos y descansos

---

## 🎯 DECISIONES DE ARQUITECTURA CLAVE

| Decisión | Razón |
|---|---|
| Minutos como unidad, no horas | Consistencia en cálculos (decisión 0001) |
| Lógica de HOS/conflictos en Airtable | Funciona para IA y humanos por igual |
| NOMBRES en vez de IDs desde Vapi | El LLM alucina los `recXXX` |
| Reglas de negocio en tabla Config | El dueño ajusta sin tocar Make; multi-cliente |
| Brokers primero (no cliente final) | Adopción rápida, volumen de datos, sin Stripe |
| Quedarse en Vapi (no Bland) | El problema era arquitectónico, no de plataforma |

---

## 🔲 PENDIENTES (solo detalles / mejoras)

- Verificar que `Mensaje_Conflicto` se llena con conflictos reales
- Confirmar consistencia `_real` en todas las fórmulas de Turnos
- Pulir campos visibles en cada panel del Interface (esconder técnicos)
- Publicar el Interface y compartir con el primer cliente

## 🚀 FUTURO (documentado, para cuando haya demanda)

- Reposicionamiento/backhaul avanzado (decisión 0003)
- Precio por mercado destino (factor_retorno)
- Entregas a cliente final con depósito/Stripe/ePOD (guía 06)
- Multi-cliente (una base o filtros por flota)

---

## 📂 DOCUMENTACIÓN

```
docs/make-escenarios/
   00-manual-completo-despacho.md      → el escenario completo explicado
   01-crear-variables-carga.md
   02-crear-buffers.md
   03-crear-precios.md
   06-entregas-deposito-epod.md        → capa de entrega (cliente final, futuro)
docs/airtable/
   hos-automatico.md
   automation-conflictos-citas.md
   tabla-turnos.md
docs/decisiones/
   0001-minutos-en-vez-de-horas.md
   0002-modelo-de-buffers.md
   0003-reposicionamiento-parada-final.md
```

---

## Cambios

- 2026-07-19 — snapshot inicial. MVP completo: motor de despacho, cotización,
  confirmación, cancelaciones/reprogramaciones, facturación, y el Interface
  de 6 paneles para el dueño de flota.
