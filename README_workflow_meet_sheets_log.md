# n8n – Ingesta de Leads con Cita en Google Meet + Registro en Google Sheets + Log de Errores 400

Este workflow de **n8n** recibe leads por **Webhook**, valida y normaliza los datos, **responde 202** de inmediato, **crea una reunión con Google Meet**, realiza **deduplicación** contra Google Sheets y **persiste** el lead si no existe. Además, **envía un email de confirmación** con el link de Meet y **registra los errores 400** en una pestaña `Errors` del mismo spreadsheet.

> Este README acompaña el flujo que subiste (`My workflow (3).json`). Los nombres de nodos que se mencionan abajo coinciden con tu JSON (p. ej., *“Introducción de webhook”*, *“Sanitizar & Enriquecer”*, etc.).

---

## 🧭 Diagrama (alto nivel)

```
Webhook (POST) → Sanitizar & Enriquecer → IF Validar
  ├─ ✅ Válido:
  │    → Respondedor 202 (rápido) → Calcular Slot de Cita
  │         → Crear Evento (Google Meet)
  │         → Hojas de cálculo - Búsqueda por correo (Lookup, Always Output Data)
  │         → ¿Existe en Sheets? → IF Deduplicar
  │               ├─ No existe → Hojas - Guardar Lead (Append) → Enviar Email con Meet
  │               └─ Ya existe → Enviar Email con Meet (opcional mantener)
  └─ ❌ Inválido:
       → Responder 400 (no válido) → Preparar Log Error → Sheets - Log Error (Append)
```

---

## 🚀 Qué hace hoy (features)
- **Webhook**: recibe `name`, `email`, `source` (y opcional `meeting_at`).
- **Sanitización**: `trim` de `name`, `lowercase` de `email`, captura `IP`, `UserAgent`, `receivedAt`.
- **Validación**: `name` no vacío y `email` con regex estándar.
- **Respuesta temprana**: `Respondedor 202` retorna `{"status":"accepted","message":"Lead recibido"}` **sin bloquear** la ejecución restante.
- **Cita y Meet**: calcula un **slot de 30 min** (próxima media hora o `meeting_at`) y **crea evento** en Google Calendar con **Google Meet** y **attendee** el email del lead.
- **Deduplicación**: hace **Lookup por email** en `Leads!A:K`; si **no existe**, hace **Append**.
- **Persistencia**: guarda `Name, Email, Source, IP, UserAgent, ReceivedAt, ProcessedAt, EventId, MeetLink, MeetingStart, Status`.
- **Confirmación al cliente**: envía **Email con el link de Meet** y la hora programada.
- **Trazabilidad de errores**: si el lead es inválido, responde **400** y registra una fila en la pestaña **`Errors`**.

---

## ✅ Requisitos previos
- **n8n** en ejecución (Docker/EC2).
- Credenciales en **Settings → Credentials**:
  - **Google Calendar OAuth2** (lectura/escritura del calendario).
  - **Google API OAuth** (Google Sheets).
  - **SMTP** (Email Send).
- **Spreadsheet** con dos pestañas:
  - `Leads` con encabezados, por ejemplo:  
    `Name | Email | Source | IP | UserAgent | ReceivedAt | ProcessedAt | EventId | MeetLink | MeetingStart | Status`
  - `Errors` con encabezados:  
    `Name | Email | Source | IP | UserAgent | ReceivedAt | ProcessedAt | ErrorCode | ErrorReason`

---

## ⚙️ Configuración paso a paso
1) **Importar** el JSON del workflow en n8n.  
2) Abrir los nodos y **asignar credenciales**:
   - **Google Calendar** (nodo *“Google Calendar”* o *“Crear Evento (Google Meet)”*).
   - **Google Sheets** (nodos *“Hojas de cálculo - Búsqueda por correo”*, *“Hojas - Guardar Lead”*, *“Sheets - Log Error”*).
   - **Email** (nodo *“Enviar Email con Meet”*).
3) En cada nodo de **Google Sheets**:
   - Seleccionar el **Spreadsheet** (documentId) y la **Hoja** (`Leads` o `Errors`).
   - En el **Lookup**, activar **Options → Always Output Data = true**.
4) En **Enviar Email con Meet**:
   - Definir **From** (`no-reply@tu-dominio.com`) y revisar el **To** (usa el email del lead).
5) En el **Webhook** (*“Introducción de webhook”*):
   - Confirmar la **ruta** (`/webhook/leads-meet`). Cambiala por una menos predecible si querés.

> **Importante sobre expresiones**: Si renombrás nodos, actualizá las **expresiones** que hacen referencia a otros nodos (deben coincidir 1:1 con el nombre del nodo).

---

## 🧪 Pruebas
- **Lead válido (nuevo)**  
  POST al webhook con:
  ```json
  {"name":"Yanina","email":"yani+meet@example.com","source":"web"}
  ```
  **Esperado**: HTTP 202 inmediato; evento en Calendar con Meet; fila nueva en `Leads`; correo de confirmación al cliente.

- **Lead duplicado** (mismo `email`)  
  **Esperado**: HTTP 202; **no** agrega fila; se envía (o no) el email dependiendo de cómo conectaste la rama TRUE de “IF Deduplicar”.

- **Lead inválido** (sin email o nombre vacío)  
  **Esperado**: HTTP 400 con JSON; nueva fila en `Errors`.

---

## 🧯 Troubleshooting
- **No se crea el evento** → revisá credencial de Calendar y que el nodo reciba `startISO/endISO` desde *“Calcular Slot de Cita”*.
- **No hace dedupe** → confirmar `lookupColumn = B`, `range = Leads!A:K` y **Always Output Data**.
- **No persiste en Sheets** → setear correctamente `documentId`/`sheetName` y permisos de la cuenta.
- **No llega el email** → credencial **SMTP**, `fromEmail` válido y/o puertos TLS/SSL correctos.
- **Expresiones “no encuentran” datos** → revisá nombres exactos de nodos que usas en `{{$node["..."].json...}}`.

---

## 📌 Próximamente (roadmap)
- **Recordatorio por email 1 día antes** para **la dueña de la página** y **el cliente** (basado en los `attendees` del evento).  
  - Sugerencia técnica: workflow aparte con **Cron** diario, `Google Calendar → Get Many` (rango de mañana) y `Email Send` a cada `attendee.email` + una copia a la **dueña**.
- (Opcional) **Resumen diario** de todas las reuniones a la dueña (tabla HTML).

> Estos módulos se pueden agregar como **workflows adicionales**, sin tocar el de ingesta de leads.

---

## 📄 Licencia
MIT (o la que prefieras).

---

## ✍️ Notas
- Si necesitás que los duplicados **actualicen** la misma fila en lugar de ignorarse, podemos cambiar *“Append”* por *“Update (By Key)”* tomando `Email` como clave.
