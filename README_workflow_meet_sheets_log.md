# n8n – Captura de Leads con Calendly + Validación y CORS

Este workflow de **n8n** recibe leads desde un **formulario web o chatbot (Christal)**, valida y normaliza los datos, y responde con un **link personalizado de Calendly** para agendar una reunión.  
Incluye compatibilidad **CORS completa**, sanitización de datos y validación básica de `name` y `email`.

> Flujo basado en `My workflow 5 (15).json`, activo en producción (`/webhook/leads-meet-zz`).

---

## 🧭 Diagrama (alto nivel)

```
Webhook (POST /leads-meet-zz) → Sanitizar & Enriquecer → IF Validar
  ├─ ✅ Válido:
  │    → Generar URL de Calendly (Set)
  │         → Responder 200 OK (con JSON y bookingUrl)
  └─ ❌ Inválido:
       → Responder 400 (no válido)
```

---

## 🚀 Qué hace hoy (features)
- **Webhook principal:** `POST /webhook/leads-meet-zz`
- **Sanitización:** elimina espacios (`trim`), convierte el `email` a minúsculas, y agrega:
  - `source` (por defecto `"web"`)
  - `userAgent`, `ip` y `receivedAt`
- **Validación:** comprueba que `name` no esté vacío y `email` tenga un formato correcto (regex estándar).
- **Calendly dinámico:** construye un link como:
  ```
  https://calendly.com/alvaro-quiroga-tw/30min?name={{name}}&email={{email}}
  ```
- **Respuesta OK:** devuelve un JSON con:
  ```json
  {
    "ok": true,
    "status": "accepted",
    "message": "Lead procesado exitosamente",
    "bookingUrl": "https://calendly.com/alvaro-quiroga-tw/30min?name=..."
  }
  ```
- **Respuesta 400:** si el lead no es válido, devuelve JSON con error.

---

## ✅ Requisitos previos
- **n8n** en ejecución (Docker, EC2, o local).
- Webhook expuesto en HTTPS (por Caddy o dominio):
  ```
  https://n8n.3-134-22-156.sslip.io/webhook/leads-meet-zz
  ```
- Frontend (React/Christal) configurado para enviar los leads con `fetch` o `axios`.

---

## ⚙️ Configuración paso a paso
1) **Importar** `My workflow 5 (15).json` en tu n8n.
2) Asegurarte que los nodos estén activos:
   - `Webhook POST1` → `path: leads-meet-zz`
   - `Sanitizar & Enriquecer1`
   - `Validar lead1`
   - `Calendly (URL)1`
   - `Respond 200 OK1`
   - `Respond 400 (no válido)1`
   - `Webhook OPTIONS (CORS)` + `Respond 204 (CORS)1`
3) En `Respond 200 OK1`, revisar que el **header CORS** sea:
   ```
   Access-Control-Allow-Origin: *
   Access-Control-Allow-Methods: GET,POST,OPTIONS
   Access-Control-Allow-Headers: Content-Type,Authorization
   ```

---

## 🧪 Pruebas

### 1️⃣ Lead válido
```bash
curl -X POST "https://n8n.3-134-22-156.sslip.io/webhook/leads-meet-zz"   -H "Content-Type: application/json"   -d '{"name":"Yanina Spina","email":"yanina_05_196@hotmail.com"}'
```
**Respuesta esperada:**
```json
{
  "ok": true,
  "status": "accepted",
  "message": "Lead procesado exitosamente",
  "bookingUrl": "https://calendly.com/alvaro-quiroga-tw/30min?name=Yanina%20Spina&email=yanina_05_196@hotmail.com"
}
```

### 2️⃣ Lead inválido
```bash
curl -X POST "https://n8n.3-134-22-156.sslip.io/webhook/leads-meet-zz"   -H "Content-Type: application/json"   -d '{"name":"","email":"correo_invalido"}'
```
**Respuesta esperada:**
```json
{
  "ok": false,
  "status": "error",
  "message": "Lead no válido"
}
```

---

## 🧯 Troubleshooting
- **CORS error en frontend** → confirmar cabeceras `Access-Control-*` en ambos `Respond` (200 y 400) y en `OPTIONS`.
- **Campos vacíos en respuesta** → revisar que el `Webhook POST` tenga `Response Mode: responseNode` y reciba body como JSON.
- **Calendly sin datos** → asegurarse que `$json.name` y `$json.email` están en el nivel raíz (no en `$json.body`).

---

## 📄 Licencia
MIT (uso libre y personalizable).
