# 🚛🧠 AI Dispatch Brain — Sistema Inteligente de Despacho Logístico

## Resumen del Proyecto

Estamos construyendo un sistema de despacho inteligente para flotas de camiones. El objetivo es que una IA pueda recibir una solicitud de carga, analizar automáticamente qué camión y conductor pueden hacer el viaje, validar horas disponibles, ubicación real, citas actuales/futuras, distancia, tiempo de llegada, deadhead y disponibilidad operacional, para luego ofrecer el mejor vehículo disponible.

El sistema usa principalmente:

```text
Airtable     = base de datos operacional
Make.com     = motor de automatización y cálculo
Google Maps  = cálculo de distancia, tiempo, tráfico y peajes (Routes API)
Vapi / IA    = conversación con broker o cliente
```

La IA no debe hacer los cálculos pesados. La IA solo debe conversar y presentar el resultado. Los cálculos reales viven en Make.com y Airtable.

---

## 🎯 Objetivo Principal

El sistema debe responder esta pregunta:

```text
¿Qué camión puede aceptar esta carga sin romper su agenda,
sin exceder HOS y llegando a tiempo?
```

No buscamos solo si un camión está "libre". Buscamos si el camión es **operacionalmente viable**.

Eso significa analizar:

1. Tipo de camión solicitado
2. Estado del conductor
3. Horas disponibles HOS
4. Ubicación operacional real del camión
5. Tiempo vacío hacia el pickup
6. Tiempo del viaje cargado
7. Buffer operacional
8. Próxima cita futura
9. Parada final del viaje actual o anterior
10. Rentabilidad futura con millas y peajes

---

## 🧱 Estructura de Tablas en Airtable

### 1. `Bookings and Operations`

Guarda las citas/viajes.

Campos importantes:

```text
Company Name
Vehicle_assigned
Operador_Asignado
Dirección-de-recogida
Dirección-de-entrega
Parada_final
Hora_Inicio
Hora_Fin_Estimada
Trip-Status
Estado_Temporal_Cita
Minutos_Consumidos
Quoted Rate
```

La columna `Parada_final` es muy importante porque representa dónde termina realmente el camión operacionalmente. Puede ser igual a la dirección de entrega, pero en el futuro también puede representar una base, hotel, truck stop, almacén o ciudad de descanso.

### 2. `Operadores`

Representa el estado del conductor.

Campos importantes:

```text
Nombre_del_Operador
Estado_del_Conductor
Camion_Asignado
Turno_Asignado
Perfil_de_Viaje
Minutos_Consumidos_Hoy
Minutos_Disponibles_HOS
Ubicación de Inicio de Jornada
Hora_Inicio_Turno
Hora_Final_Turno
```

Aquí se calcula la disponibilidad legal del conductor. El HOS se recomienda manejar en minutos, no en horas.

Ejemplo:

```text
11 horas de manejo = 660 minutos
```

### 3. `Inventario de Flota`

Representa los camiones/unidades.

Campos importantes:

```text
Nombre_de_Unidad
truck_type
estado
Operador_Asignado
Ubicacion_Operacional_Actual
Disponible_Desde
Base_Operacional
```

La ubicación operacional del camión debe vivir preferiblemente aquí, porque pertenece a la unidad.

---

## 🔗 Relación Recomendada entre Tablas

```text
Bookings and Operations → Vehicle_assigned → Inventario de Flota
Bookings and Operations → Operador_Asignado → Operadores
Inventario de Flota → Operador_Asignado → Operadores
```

Cuando Make crea una cita nueva, debe llenar automáticamente:

```text
Vehicle_assigned = camión ganador
Operador_Asignado = operador vinculado a ese camión
```

Esto es necesario para calcular correctamente HOS del operador, agenda del camión, citas futuras y disponibilidad operacional.

---

## 🚦 Estado Temporal de Citas

Campo: `Estado_Temporal_Cita` en `Bookings and Operations`.

Clasifica cada cita como:

```text
Pasada
Actual
Futura
```

Lógica:

```text
Si Hora_Fin_Estimada < NOW()                              → Pasada
Si Hora_Inicio <= NOW() y Hora_Fin_Estimada >= NOW()      → Actual
Si Hora_Inicio > NOW()                                    → Futura
```

---

## 📍 Cómo Determinamos la Ubicación Real del Camión

Regla actual:

```text
Si el camión tiene cita actual:
    usar Parada_final de la cita actual

Si no tiene cita actual:
    usar Ubicacion_Operacional_Actual del camión

Si no tiene ubicación guardada:
    usar Base_Operacional
```

Recomendación profesional: guardar en `Inventario de Flota`:

```text
Ubicacion_Operacional_Actual
Disponible_Desde
Fuente_Ubicacion
Ultima_Actualizacion_Ubicacion
```

Cuando una cita se marca como `Delivered`, Make debe actualizar el camión:

```text
Ubicacion_Operacional_Actual = Parada_final
Disponible_Desde             = Hora_Fin_Estimada
Fuente_Ubicacion             = Última cita completada
```

---

## 🧠 Lógica Principal del Despacho

Cuando entra una solicitud del cliente, la IA/Vapi envía algo así a Make:

```json
{
  "pickup_city": "Detroit, MI",
  "delivery_city": "Chicago, IL",
  "pickup_time": "2026-05-15 14:00",
  "truck_type": "Box Truck"
}
```

Make empieza el análisis.

---

## ✅ Lo Que Ya Construimos / Resolvimos

### 1. Conversión de tiempo de Google Maps

Google Routes API devuelve duración así:

```text
14209s
```

Eso es texto, no número. Hay que limpiar la `s`, convertir a número y dividir entre 60.

```text
14209s → 237 minutos
```

Conclusión:

```text
La máquina trabaja en minutos.
La IA/humano puede escuchar horas.
```

### 2. Buffer operacional

Buffer dinámico según duración del viaje:

```text
0 a 3 horas      → 30 minutos
3 a 6 horas      → 60 minutos
6 a 10 horas     → 90 minutos
más de 10 horas  → 120 minutos
```

Evita errores por tráfico, carga lenta, fuel stop, espera en dock, break DOT o retrasos.

### 3. Filtro de operadores viables

Airtable busca operadores/camiones usando fórmula dinámica desde Make.

Filtro base:

```text
truck_type = tipo solicitado
Estado_del_Conductor = Activo
Minutos_Disponibles_HOS >= tiempo requerido
```

Nota importante: los operadores como `>=` deben escribirse manualmente como texto normal en la fórmula de Airtable, no usando el botón verde visual de Make, porque eso causaba errores.

### 4. Ubicación operacional real

La ubicación para calcular rutas viene de:

```text
Parada_final de cita actual
o
Ubicacion_Operacional_Actual del camión
o
Base_Operacional como fallback
```

### 5. Cálculo de deadhead

Deadhead = trayecto vacío del camión hasta el pickup.

```text
ubicacion_operacional_real → pickup_city
```

Guardamos:

```text
deadhead_minutos
deadhead_millas
```

---

## 🚧 Fase Actual

Estamos en la parte donde ya sabemos cómo calcular dónde está operacionalmente el camión, cuánto tarda en llegar al pickup y cuántas millas vacías recorrerá.

Trabajando en completar y ordenar:

```text
deadhead_minutos
deadhead_millas
viaje_minutos
viaje_millas
tiempo_total_operacional
```

Todo debe quedar en minutos para cálculos.

---

## 🚀 Próximos Pasos Técnicos

### Paso 1. Calcular Deadhead

Google Maps:

```text
ubicacion_operacional_real → pickup_city
```

Guardar:

```text
deadhead_minutos
deadhead_millas
```

### Paso 2. Calcular Viaje Principal

Google Maps:

```text
pickup_city → delivery_city
```

Guardar:

```text
viaje_minutos
viaje_millas
```

### Paso 3. Calcular Tiempo Total Operacional

```text
tiempo_total_operacional =
    deadhead_minutos
  + viaje_minutos
  + buffer_operacional
```

Ejemplo:

```text
80 + 237 + 60 = 377 minutos
```

### Paso 4. Validar HOS

```text
Minutos_Disponibles_HOS >= tiempo_total_operacional
```

Si cumple, el camión sigue como candidato. Si no, se descarta.

### Paso 5. Validar Si Llega al Pickup

Crear `hora_disponible_camion`:

```text
Si tiene cita actual:
    hora_disponible_camion = Hora_Fin_Estimada de la cita actual

Si no tiene cita actual:
    hora_disponible_camion = now o Disponible_Desde
```

Calcular:

```text
eta_llegada_pickup = hora_disponible_camion + deadhead_minutos
```

Validar:

```text
eta_llegada_pickup <= pickup_time_cliente
```

### Paso 6. Validar Próxima Cita Futura

```text
hora_estimada_entrega =
    pickup_time_cliente + viaje_minutos + buffer_operacional
```

Con Google Maps:

```text
delivery_city → pickup de próxima cita futura
```

Validar:

```text
hora_estimada_entrega + tiempo_a_proxima_cita <= hora_inicio_proxima_cita
```

Si no llega a tiempo, se descarta.

### Paso 7. Score de Candidatos

Cada camión viable recibirá puntos según:

```text
menos deadhead
más HOS disponible
menos millas vacías
sin próxima cita
perfil flexible
menos peajes
mejor rentabilidad
```

Make selecciona el candidato con mayor score.

---

## 🏁 Resultado Esperado del Sistema

Vapi/IA podrá responder al cliente algo como:

```text
Tengo un Box Truck disponible.

Puede llegar al pickup aproximadamente en 1 hora y 20 minutos.

El operador tiene suficiente disponibilidad operacional para completar
el viaje sin afectar sus compromisos futuros.
```

La IA no inventa. Solo comunica el resultado que Make ya calculó.

---

## 🧩 Resumen Para Otra IA

Sistema de despacho inteligente para flotas. Usa Airtable como base de datos, Make.com como motor de lógica, Google Maps Routes API para calcular tiempos/distancias/peajes, y Vapi como interfaz de voz.

El sistema recibe solicitudes de carga con pickup, delivery, hora y tipo de camión. Luego:

1. Filtra operadores activos con camiones compatibles y HOS suficiente.
2. Reconstruye la ubicación operacional real del camión usando la cita actual, la parada final, la ubicación guardada en flota o la base operacional.
3. Calcula deadhead desde ubicación real hasta pickup.
4. Calcula el viaje principal pickup → delivery.
5. Suma buffer operacional.
6. Valida HOS.
7. Valida llegada al pickup.
8. Valida que no rompa la próxima cita futura.
9. Asigna score y selecciona el mejor camión.

La regla central del sistema es:

```text
No preguntar "¿está libre el camión?"
Preguntar "¿puede ejecutar esta carga sin romper restricciones operacionales?"
```

El sistema trabaja internamente en minutos. Las horas solo se usan para mostrar al usuario o para que la IA hable de manera natural.

---

## 📌 Estado Actual del Proyecto

```text
Fase actual:
Cálculo operacional de rutas y disponibilidad real.
```

### Ya resuelto

- Estructura de tablas
- Relaciones básicas
- Fórmula de estado temporal
- Búsqueda de operadores viables
- Buffer operacional
- Uso de `Parada_final` como ubicación real
- Cálculo de deadhead
- Manejo de minutos en vez de horas

### Pendiente

- `viaje_minutos`
- `viaje_millas`
- `tiempo_total_operacional`
- Validación de llegada al pickup
- Validación de próxima cita futura
- Score final
- Creación automática de cita con camión y operador vinculados

---

## 🧠 Principio Arquitectónico Final

```text
Airtable      guarda la verdad operacional.
Make          toma decisiones.
Google Maps   calcula rutas.
Vapi / IA     conversa.
```

La decisión más importante del sistema:

```text
Make debe decidir el camión.
La IA solo debe explicar la decisión.
```
