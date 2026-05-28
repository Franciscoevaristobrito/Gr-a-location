# Documentación — AI Dispatch Brain

Índice maestro. Si no sabes dónde está algo o dónde guardar algo nuevo, empieza aquí.

---

## 📚 Archivos principales

| Archivo | Para qué sirve |
|---|---|
| [`PROJECT_OVERVIEW.md`](./PROJECT_OVERVIEW.md) | Visión global del proyecto. El "mapa mental" completo. |
| [`glosario.md`](./glosario.md) | Diccionario de variables y términos. Fuente única de verdad. |
| [`webhook-vapi-schema.md`](./webhook-vapi-schema.md) | Contrato Vapi → Make. Qué campos recibe el sistema. |
| [`vapi-variable-descriptions.md`](./vapi-variable-descriptions.md) | Descripciones paste-ready para configurar Vapi (cómo extrae cada variable). 26 variables completas. |
| [`vapi-variables-carga.md`](./vapi-variables-carga.md) | Subset enfocado: solo las 12 variables de carga (sin ciudad/hora/broker). |
| [`vapi-prompt-cargo.md`](./vapi-prompt-cargo.md) | Fragmento modular de prompt para descubrimiento de carga. Se inserta dentro de un prompt principal ya existente en Vapi. |
| [`rutina_diaria.md`](./rutina_diaria.md) | Cheat sheet de la rutina de trabajo. |
| [`pizarron_template.html`](./pizarron_template.html) | Plantilla imprimible para el pizarrón. |

---

## 📁 Carpetas

| Carpeta | Qué guarda | Cuándo se usa |
|---|---|---|
| [`decisiones/`](./decisiones/) | Decisiones importantes con su justificación | Cada vez que decides algo que afecta el sistema |
| [`make-escenarios/`](./make-escenarios/) | Cómo está armado cada escenario de Make | Al terminar o modificar un escenario |
| [`airtable/`](./airtable/) | Schema, fórmulas y vistas de cada tabla | Al crear o modificar una tabla/campo |
| [`bundles-ejemplo/`](./bundles-ejemplo/) | JSON reales de pruebas (Vapi, Maps, etc.) | Cuando recibes un payload útil para reusar |
| [`pizarron-fotos/`](./pizarron-fotos/) | Foto del pizarrón al cierre de cada semana | Viernes antes de borrar el pizarrón |
| [`bitacora/`](./bitacora/) | Una entrada por día/semana: qué se hizo y aprendió | Al final del día o de la semana |

---

## 🧭 Cómo saber dónde guardar algo nuevo

```text
¿Es una decisión sobre el sistema? → decisiones/
¿Documentas un escenario de Make? → make-escenarios/
¿Es un campo/fórmula de Airtable? → airtable/
¿Es un JSON real para reusar?    → bundles-ejemplo/
¿Es una variable nueva?          → glosario.md
¿Es lo que hiciste hoy/esta semana? → bitacora/
¿Es una foto del pizarrón?       → pizarron-fotos/
```

Si dudas → ábrelo en `bitacora/` primero. Si después ves que es importante, lo mueves al sitio correcto.

---

## ⚠️ Regla de oro

```
Antes de borrar algo del pizarrón:
  ¿Esto vive ya en el repo, o se pierde para siempre?
```

Si vale la pena conservarlo → al repo antes de borrar.
