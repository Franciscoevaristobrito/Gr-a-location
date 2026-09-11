# Guía — Entregas con depósito, checklist de sitio y ePOD

**Última actualización:** 2026-07-19
**Estado:** Plan de construcción. Fase 1 recomendada primero.
**Aplica a:** entregas a cliente final (gabinetes, materiales, obra/condominios)

Guía para construir la capa de ENTREGA y COBRO encima del motor de dispatch
existente. El motor ya resuelve "qué camión y cuándo"; esto agrega "asegurar
el sitio, activar la cita con depósito, probar la entrega y cobrar".

---

## 🔑 Principio central: la cita nace TENTATIVA, se activa con el DEPÓSITO

```
NINGÚN camión se despacha hasta que:
   1. El cliente firmó el checklist de sitio (elevador, parking, supervisor)
   2. El depósito está PAGADO (Stripe confirma)

Sin esas 2 cosas → la cita existe pero NO reserva camión (queda Tentativa).
```

Esto ataca la causa #1 de millas vacías: cliente no listo / no aparece.

---

## 🔄 CICLO DE VIDA DE LA CITA (los estados)

Campo `Estado_Cita` (Single select) en Bookings. Orden de vida:

```
Solicitada
   │  cliente pide entrega (IA recolecta datos + checklist)
   ▼
Tentativa  ← se manda el link de pago+firma, esperando al cliente
   │  ┌─ depósito pagado + checklist firmado ──────────┐
   │  │                                                 ▼
   │  │                                            Confirmada  ← AQUÍ se despacha camión
   │  │                                                 │
   │  └─ NO paga / pago falla / no firma (timeout) ─┐   ▼
   │                                                │  Despachada
   ▼                                                │   │
Expirada / Cancelada  ←─────────────────────────────┘   ▼
   (nunca se activó — libera el hueco tentativo)      En_Sitio
                                                         │  ┌─ entrega OK (ePOD) ─┐
                                                         │  │                     ▼
                                                         │  │                Entregada
                                                         │  │                     │ cobra saldo
                                                         │  │                     ▼
                                                         │  │                Completada
                                                         │  └─ sin acceso 30min ─┐
                                                         ▼                        ▼
                                                    Intento_Fallido (Dry Run) ────┘
                                                       → cola de aprobación del dueño
```

### La transición clave: Tentativa → Confirmada

```
DISPARADOR: webhook de Stripe dice "depósito pagado"
            Y el checklist está firmado (elevador/parking/supervisor OK)
ACCIÓN:     Estado_Cita = Confirmada
            → SOLO AHORA corre el motor de dispatch (asigna camión + turno)
            → se reserva el hueco de verdad
```

Antes de eso, la cita es solo una intención. El camión no se compromete.

---

## 💳 QUÉ PASA CON UN PAGO FALLIDO / MALO

Un depósito puede fallar por: tarjeta rechazada, fondos insuficientes,
el cliente nunca completa el link, o abandona a mitad.

```
Stripe intenta el cobro del depósito
   │
   ├─ ÉXITO  → deposit_status = PAID → Estado_Cita = Confirmada → despacha
   │
   └─ FALLO / TIMEOUT:
        - deposit_status = FAILED (o sigue PENDING)
        - Estado_Cita se queda en Tentativa (NUNCA pasa a Confirmada)
        - NO se despacha camión, NO se reserva turno
        - La IA/SMS le avisa al cliente: "el pago no se completó, aquí está
          el link de nuevo" (reintento, máx 2 veces)
        - Si tras N horas sigue sin pagar → Estado_Cita = Expirada
          (se libera cualquier hueco tentativo que se hubiera apartado)
```

**Regla de oro del pago malo:** el sistema NUNCA despacha con un pago
pendiente o fallido. "Sin depósito confirmado por Stripe = no hay camión."
El estado Tentativa es la sala de espera; solo el webhook de Stripe con
`payment_intent.succeeded` abre la puerta a Confirmada.

> Diferencia importante Depósito vs Pre-Auth Hold:
> - Depósito: se COBRA ya. Si la entrega sale bien, se resta del saldo.
> - Pre-Auth Hold: solo se CONGELA. Se captura únicamente si hay penalización.
> Para empezar, usa DEPÓSITO (más simple). Pre-Auth es refinamiento futuro.

---

## 🚨 DÓNDE EL DUEÑO ES OBLIGATORIO (human-in-the-loop)

La IA puede pedir, cotizar, enviar links y registrar. EJECUTAR un cobro
disputable pasa SIEMPRE por el dueño (evita chargebacks).

```
✅ Automático (bajo riesgo):
   - Checklist, cotizar depósito, enviar link, registrar firma/fotos,
     cambiar estados, cobrar el SALDO cuando el cliente firmó conforme
     el ePOD (la firma ES la autorización)

🚨 Requiere ✅ del DUEÑO (alto riesgo / disputable):
   - Quedarse el depósito en no-show / cancelación tardía
   - Cobrar Dry Run / Detention
   - Capturar un Pre-Auth Hold como penalización
   - Facturar cuando el ePOD muestra POSIBLE DAÑO
   - Cualquier cargo > umbral (Config, ej. $300)
   - Reembolsos (siempre)
```

**Patrón: COLA DE APROBACIÓN** (no un cobro automático):
```
Evento cobrable → Estado_Reclamo = "X_Pendiente" + evidencia
   → vista Airtable "💰 Aprobaciones Pendientes"
   → dueño marca checkbox Aprobado_Por_Dueno
   → UN escenario "ejecutor" detecta el ✅ → cobra en Stripe → Cobrado
```

---

## 📋 PLAN DE CONSTRUCCIÓN (fases en orden, con checkboxes)

### 🟢 FASE 1 — Checklist de sitio (sin dinero, $0 herramientas)

```
AIRTABLE (Bookings):
[ ] tipo_entrega            Single select: comercial / residencial / obra
[ ] elevador_reservado      Checkbox
[ ] permiso_parking         Checkbox
[ ] contacto_supervisor     Phone
[ ] sitio_listo             Formula: AND(elevador, parking, contacto no vacío)

CONFIG:
[ ] dry_run_fee              = 75
[ ] ventana_gracia_min       = 30
[ ] deposito_default         = 150
[ ] umbral_aprobacion_dueno  = 300

VAPI (prompt reservas — Flow de descubrimiento):
[ ] Si tipo_entrega = residencial/obra → preguntar OBLIGATORIO:
    "Is the freight elevator reserved and parking cleared for that window?"
    "Who's the site contact — name and number?"
[ ] Anunciar política: "If our truck can't get access within 30 minutes,
    a $75 dry run fee applies."

REGLA:
[ ] confirm_booking NO corre si sitio_listo = false (para resid./obra)
```
✅ **Entregable Fase 1:** ninguna entrega especializada se despacha sin
el sitio confirmado. Elimina la causa #1 de deadhead. Cero herramientas nuevas.

### 🟡 FASE 2 — Estados de cita + Dry Run + cola de aprobación

```
AIRTABLE:
[ ] Estado_Cita             Single select con TODOS los estados del ciclo
[ ] Estado_Reclamo          agregar: DryRun_Pendiente
[ ] Aprobado_Por_Dueno      Checkbox
[ ] Vista "💰 Aprobaciones Pendientes"
    filtro: Estado_Reclamo termina en _Pendiente AND Aprobado = false

MAKE:
[ ] Al marcar Intento_Fallido → registrar DryRun_Pendiente + monto
    (patrón TONU) → alerta al dueño (email/SMS)
```
🚨 **Dueño:** revisa la vista y aprueba/rechaza cada cargo.

### 🟠 FASE 3 — Depósito con Stripe (activa la cita)

```
STRIPE (decisión de negocio del dueño 🚨):
[ ] Cuenta Stripe + API key guardada en Make (conexión segura)

AIRTABLE (Bookings):
[ ] deposit_status          Single select: NONE / PENDING / PAID / FAILED
[ ] stripe_payment_intent   Single line text
[ ] remaining_balance       Currency
[ ] deposito_monto          Currency

VAPI:
[ ] Tool send_booking_link (SMS con Stripe Payment Link + form de firma)
    → la IA lo llama cuando el cliente acepta la política

MAKE — escenario "Activar cita con depósito":
[ ] Webhook de Stripe (payment_intent.succeeded)
[ ] Buscar la cita por metadata (appointment_id / booking id)
[ ] Si pagó Y checklist firmado:
      Estado_Cita = Confirmada → CORRE EL MOTOR DE DISPATCH
      (asigna camión + reserva turno — recién aquí)
[ ] Si pago falla → deposit_status = FAILED, Estado_Cita sigue Tentativa,
    reintento del link (máx 2), luego Expirada

MAKE — escenario "Ejecutor de cobros aprobados":
[ ] Trigger: Aprobado_Por_Dueno = true (Airtable watch)
[ ] Cobra en Stripe → Estado_Reclamo = Cobrado
```
🚨 **Dueño:** aprueba TODO cargo de penalización antes de ejecutarse;
decide si Stripe se usa.

### 🔴 FASE 4 — ePOD (fotos + firma del chofer)

```
[ ] Form simple (Fillout / Airtable form) en el celular del chofer:
    3 fotos + firma del cliente + notas + checkbox "posible daño"
[ ] Al enviarse → Estado_Cita = Entregada → generar comprobante
[ ] Cobrar el SALDO automático (la firma del cliente ES la autorización)
[ ] 🚨 EXCEPCIÓN: checkbox "posible daño" marcado → NO facturar →
    cola de aprobación del dueño con las fotos
[ ] SMS "llega en 20 min": versión simple = el chofer pica "En camino"
    en su form → Make manda el SMS (geo-fencing real = futuro)
```

---

## ⚠️ La decisión que define el alcance: broker vs cliente final

```
Negocio BROKER (factura NET-30, sin tarjetas):
   Fases 1-2 SÍ. Fase 3 (depósitos Stripe) NO aplica. Fase 4 (ePOD) SÍ
   — el POD es lo que te deja COBRARLE al broker.

Negocio CLIENTE FINAL (gabinetes a condominios, pago con tarjeta):
   Fases 1-4 completas, incluido el depósito que activa la cita.

AMBOS: Fases 1-2-4 comunes + Fase 3 solo para el lado cliente directo.
```

---

## ✅ Orden recomendado

```
1. Terminar el reschedule flow (no dejar módulos a medias)
2. Fase 1 (checklist) — 1 tarde, cero herramientas, alto impacto
3. Fase 2 (cola de aprobación) — formaliza el patrón humano
4. Fases 3-4 — solo con modelo definido y demanda real
```

---

## Cambios

- 2026-07-19 — versión inicial. Plan de la capa de entrega/cobro:
  ciclo de vida de la cita (Tentativa→Confirmada por depósito), manejo
  de pago fallido, cola de aprobación del dueño, y las 4 fases de build.
