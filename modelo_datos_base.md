# Modelo de datos base

Este documento define un modelo de datos mínimo para operar agendamiento de turnos, cotizaciones asistidas por chat, catálogo y analítica de eventos para una bicicletería.

## 1) Tablas principales y campos obligatorios

> Convención sugerida de tipos: `UUID` para ids, `TIMESTAMPTZ` para fecha/hora, `TEXT` para texto libre, `NUMERIC(12,2)` para montos, `JSONB` para estructuras dinámicas.

### appointments

Registra solicitudes y seguimiento de turnos de servicio técnico.

| Campo | Tipo sugerido | Obligatorio | Descripción |
|---|---|---|---|
| id | UUID (PK) | Sí | Identificador único del turno. |
| nombre | TEXT | Sí | Nombre de la persona solicitante. |
| contacto | TEXT | Sí | Canal de contacto (teléfono, email o WhatsApp). |
| tipo_bici | TEXT | Sí | Tipo de bicicleta (urbana, ruta, MTB, e-bike, etc.). |
| servicio | TEXT | Sí | Servicio solicitado (mantenimiento, reparación, diagnóstico, etc.). |
| fecha_hora | TIMESTAMPTZ | Sí | Fecha y hora del turno. |
| estado | TEXT (enum) | Sí | Estado del turno (ver sección de estados). |
| created_at | TIMESTAMPTZ | Sí | Fecha/hora de creación del registro. |
| updated_at | TIMESTAMPTZ | Sí | Fecha/hora de última modificación del registro. |
| source | TEXT | Sí | Origen del dato (web, whatsapp, mostrador, teléfono, API, etc.). |
| consentimiento_datos | BOOLEAN | Condicional | Consentimiento para tratamiento de datos personales cuando aplique por canal/jurisdicción. |
| consentimiento_at | TIMESTAMPTZ | Condicional | Marca temporal del consentimiento cuando aplique. |

### quote_requests

Registra el flujo de cotización de bicicletas o indumentaria.

| Campo | Tipo sugerido | Obligatorio | Descripción |
|---|---|---|---|
| id | UUID (PK) | Sí | Identificador único de la solicitud. |
| canal_origen | TEXT | Sí | Canal de entrada (webchat, whatsapp, tienda, etc.). |
| tipo_cotizacion | TEXT (enum: bici/ropa) | Sí | Tipo de cotización solicitada. |
| respuestas_chat | JSONB | Sí | Respuestas capturadas en el flujo conversacional. |
| recomendacion | TEXT/JSONB | Sí | Recomendación emitida por reglas/asesor. |
| monto_estimado | NUMERIC(12,2) | Sí | Monto estimado de la cotización. |
| pdf_url | TEXT | Condicional | URL del PDF generado, cuando exista. |
| pdf_hash | TEXT | Condicional | Hash de integridad del PDF, cuando exista. |
| whatsapp_clicked_at | TIMESTAMPTZ | Condicional | Momento en que se derivó a WhatsApp por click. |
| estado | TEXT (enum) | Sí | Estado del flujo de cotización (ver sección de estados). |
| created_at | TIMESTAMPTZ | Sí | Fecha/hora de creación del registro. |
| updated_at | TIMESTAMPTZ | Sí | Fecha/hora de última modificación del registro. |
| source | TEXT | Sí | Fuente/origen técnico del evento (frontend, bot, CRM, API). |
| consentimiento_marketing | BOOLEAN | Condicional | Consentimiento para contacto comercial, cuando aplique. |
| consentimiento_at | TIMESTAMPTZ | Condicional | Marca temporal del consentimiento cuando aplique. |

> Nota: el requisito `pdf_url/hash` puede implementarse como dos columnas (`pdf_url`, `pdf_hash`) para facilitar validación y auditoría.

### catalog_items

Catálogo de productos disponibles para recomendación o venta.

| Campo | Tipo sugerido | Obligatorio | Descripción |
|---|---|---|---|
| id | UUID (PK) | Sí | Identificador único del producto. |
| categoria | TEXT | Sí | Categoría principal (bicicleta, indumentaria, accesorio, etc.). |
| disciplina | TEXT | Sí | Disciplina de uso (urbano, ruta, MTB, gravel, etc.). |
| talla | TEXT | Sí | Talle/tamaño del producto. |
| precio_lista | NUMERIC(12,2) | Sí | Precio de lista vigente. |
| activo | BOOLEAN | Sí | Indicador de disponibilidad en catálogo. |
| created_at | TIMESTAMPTZ | Sí | Fecha/hora de alta del ítem. |
| updated_at | TIMESTAMPTZ | Sí | Fecha/hora de última actualización. |
| source | TEXT | Sí | Sistema de origen (ERP, backoffice, carga manual, API). |

### analytics_events

Eventos de analítica para trazabilidad del embudo y comportamiento.

| Campo | Tipo sugerido | Obligatorio | Descripción |
|---|---|---|---|
| id | UUID (PK) | Sí | Identificador único del evento. |
| user_session_id | TEXT | Sí | Identificador de sesión del usuario. |
| event_name | TEXT | Sí | Nombre normalizado del evento. |
| event_props_json | JSONB | Sí | Propiedades del evento en formato JSON. |
| timestamp | TIMESTAMPTZ | Sí | Fecha/hora de ocurrencia del evento. |
| created_at | TIMESTAMPTZ | Sí | Fecha/hora de persistencia del registro. |
| updated_at | TIMESTAMPTZ | Sí | Fecha/hora de última corrección/enriquecimiento. |
| source | TEXT | Sí | Origen del evento (webapp, chatbot, pixel server-side, etc.). |
| consentimiento_analytics | BOOLEAN | Condicional | Consentimiento para tracking no esencial cuando aplique. |
| consentimiento_at | TIMESTAMPTZ | Condicional | Marca temporal del consentimiento cuando aplique. |

---

## 2) Estados permitidos y transiciones

### Flujo de `appointments`

Estados permitidos:
- `iniciada`
- `confirmada`
- `atendida`
- `cancelada`
- `no_show`

Transiciones válidas:
1. `iniciada` → `confirmada`
2. `confirmada` → `atendida`
3. `confirmada` → `cancelada`
4. `confirmada` → `no_show`

Reglas sugeridas:
- No permitir transición directa `iniciada` → `atendida`.
- No permitir cambios desde estados terminales (`atendida`, `cancelada`, `no_show`) salvo por proceso administrativo explícito y auditado.

### Flujo de `quote_requests`

Estados permitidos:
- `iniciada`
- `recomendacion_emitida`
- `pdf_generado`
- `whatsapp_derivado`

Transiciones válidas:
1. `iniciada` → `recomendacion_emitida`
2. `recomendacion_emitida` → `pdf_generado`
3. `pdf_generado` → `whatsapp_derivado`

Reglas sugeridas:
- Si `estado = pdf_generado`, exigir `pdf_url` y/o `pdf_hash` no nulos.
- Si `estado = whatsapp_derivado`, exigir `whatsapp_clicked_at` no nulo.

---

## 3) Auditoría y cumplimiento

Campos mínimos transversales en todas las tablas:
- `created_at` (no nulo)
- `updated_at` (no nulo)
- `source` (no nulo)

Buenas prácticas adicionales:
- Registrar consentimientos de forma explícita (`consentimiento_*`) cuando haya datos personales o tracking sujeto a regulación.
- Guardar `consentimiento_at` para trazabilidad temporal y evidencias de cumplimiento.
- Versionar o historizar cambios de estado relevantes (por ejemplo, en tabla auxiliar `status_history`) si se requiere auditoría avanzada.
- En `analytics_events`, evitar guardar PII en `event_props_json`; usar identificadores seudonimizados cuando sea posible.

