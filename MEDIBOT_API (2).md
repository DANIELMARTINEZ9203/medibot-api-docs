# MediBot API — Documentación para Desarrolladores

> **Base URL:** `https://bots.quantumsmarttradeai.com`  
> **Autenticación:** Cookie de sesión activa (próximamente: API Key)  
> **Formato:** JSON  
> **Versión:** 1.0

---

## Índice

1. [Transcripción de Audio](#1-transcripción-de-audio)
2. [Generación de SOAP con IA](#2-generación-de-soap-con-ia)
3. [Agendar Cita](#3-agendar-cita)
4. [Enviar Mensaje WhatsApp](#4-enviar-mensaje-whatsapp)
5. [Consultar Expediente](#5-consultar-expediente)

---

## 1. Transcripción de Audio

Transcribe un archivo de audio de una consulta médica usando Whisper (OpenAI). El flujo es asíncrono: primero creas una sesión, luego subes el audio, luego haces polling para obtener el resultado.

### Paso 1 — Crear sesión de grabación

```
POST /medico/grabar-movil/sesion
```

**Body:**
```json
{
  "expediente_id": 123,
  "negocio_id": 1
}
```

**Respuesta:**
```json
{
  "token": "abc123xyz",
  "url": "https://bots.quantumsmarttradeai.com/medico/grabar-movil?session=abc123xyz"
}
```

---

### Paso 2 — Subir audio

```
POST /medico/grabar-movil/subir?session={token}
Content-Type: multipart/form-data
```

**Form data:**
| Campo | Tipo | Descripción |
|-------|------|-------------|
| `file` | `File` | Audio en formato WebM, MP4, MP3, WAV o M4A |
| `session` | `string` | Token obtenido en el paso 1 |

**Respuesta:**
```json
{
  "ok": true
}
```

**Notas:**
- Tamaño máximo recomendado: 25 MB
- Duración máxima: 60 minutos
- Idioma detectado automáticamente (optimizado para español)
- El tiempo de procesamiento es aproximadamente 1/5 de la duración del audio

---

### Paso 3 — Obtener transcripción (polling)

```
GET /medico/grabar-movil/resultado?session={token}
```

**Respuesta cuando está listo:**
```json
{
  "listo": true,
  "transcripcion": "Doctor: Buenos días. ¿Cómo se ha sentido esta semana?..."
}
```

**Respuesta cuando aún procesa:**
```json
{
  "listo": false,
  "transcripcion": null
}
```

**Ejemplo en Python:**
```python
import httpx, time

# Paso 1 — Crear sesión
r = httpx.post("https://bots.quantumsmarttradeai.com/medico/grabar-movil/sesion",
    json={"expediente_id": 123, "negocio_id": 1},
    cookies={"session": "tu_cookie"})
token = r.json()["token"]

# Paso 2 — Subir audio
with open("consulta.webm", "rb") as f:
    httpx.post(f"https://bots.quantumsmarttradeai.com/medico/grabar-movil/subir?session={token}",
        files={"file": ("consulta.webm", f, "audio/webm")},
        cookies={"session": "tu_cookie"})

# Paso 3 — Polling
while True:
    r = httpx.get(f"https://bots.quantumsmarttradeai.com/medico/grabar-movil/resultado?session={token}",
        cookies={"session": "tu_cookie"})
    data = r.json()
    if data["listo"]:
        transcripcion = data["transcripcion"]
        break
    time.sleep(3)

print(transcripcion)
```

**Ejemplo en JavaScript:**
```javascript
// Paso 1
const { token } = await fetch('/medico/grabar-movil/sesion', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ expediente_id: 123, negocio_id: 1 })
}).then(r => r.json());

// Paso 2
const form = new FormData();
form.append('file', audioBlob, 'consulta.webm');
await fetch(`/medico/grabar-movil/subir?session=${token}`, {
  method: 'POST', body: form
});

// Paso 3 — Polling
const poll = async () => {
  const { listo, transcripcion } = await fetch(
    `/medico/grabar-movil/resultado?session=${token}`
  ).then(r => r.json());
  if (listo) return transcripcion;
  await new Promise(r => setTimeout(r, 3000));
  return poll();
};
const transcripcion = await poll();
```

---

## 2. Generación de SOAP con IA

Convierte una transcripción de consulta médica en un expediente clínico estructurado usando Claude (con fallback a DeepSeek). Extrae automáticamente: motivo de consulta, SOAP completo, diagnóstico, medicamentos y signos vitales.

```
POST /medico/procesar-consulta
Content-Type: application/json
```

**Body:**
```json
{
  "transcripcion": "Doctor: Buenos días. Paciente: Buenos días doctor, vengo porque llevo 3 días con fiebre de 38.5..."
}
```

**Respuesta:**
```json
{
  "motivo_consulta": "Fiebre y dolor de garganta de 3 días de evolución",
  "subjetivo": "Paciente refiere fiebre, odinofagia y malestar general de 3 días de evolución. Niega tos ni dificultad respiratoria.",
  "objetivo": "Faringe hiperémica con exudado blanquecino bilateral. Adenopatías cervicales palpables. Sin datos de dificultad respiratoria.",
  "evaluacion": "Cuadro compatible con faringoamigdalitis bacteriana aguda.",
  "plan": "Antibioticoterapia por 10 días, antipiréticos, hidratación oral. Cita de seguimiento en 7 días si no mejora.",
  "diagnostico": "Faringoamigdalitis bacteriana aguda",
  "diagnostico_cie10_buscar": "faringitis amigdalitis bacteriana aguda",
  "medicamentos": [
    {
      "nombre": "Amoxicilina",
      "forma_farmaceutica": "Cápsula",
      "dosis": "500mg",
      "frecuencia": "cada 8 horas",
      "via_administracion": "Oral",
      "duracion": "10 días",
      "indicaciones_uso": "Tomar con alimentos",
      "presentacion": "Caja con 30 cápsulas"
    },
    {
      "nombre": "Paracetamol",
      "forma_farmaceutica": "Tableta",
      "dosis": "500mg",
      "frecuencia": "cada 6 horas",
      "via_administracion": "Oral",
      "duracion": "5 días",
      "indicaciones_uso": "Solo si hay fiebre mayor a 38°C",
      "presentacion": "Caja con 20 tabletas"
    }
  ],
  "indicaciones": "Reposo relativo, hidratación abundante, tomar antibiótico completo aunque mejore.",
  "signos_vitales": {
    "peso_kg": null,
    "temperatura_c": 38.5,
    "presion_sistolica": null,
    "presion_diastolica": null,
    "frecuencia_cardiaca": null,
    "frecuencia_respiratoria": null,
    "saturacion_oxigeno": null,
    "glucosa_mg_dl": null
  }
}
```

**Errores:**
| Código | Descripción |
|--------|-------------|
| `400` | No se recibió transcripción |
| `401` | No autorizado — sesión inválida |
| `402` | Créditos de IA insuficientes |
| `500` | Error al procesar con IA |

**Ejemplo en Python:**
```python
import httpx

r = httpx.post("https://bots.quantumsmarttradeai.com/medico/procesar-consulta",
    json={"transcripcion": "Doctor: ¿Cómo está? Paciente: Mal doctor, fiebre y garganta..."},
    cookies={"session": "tu_cookie"})

soap = r.json()
print(f"Diagnóstico: {soap['diagnostico']}")
print(f"Medicamentos: {len(soap['medicamentos'])}")
```

**Ejemplo en JavaScript:**
```javascript
const soap = await fetch('/medico/procesar-consulta', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  credentials: 'include',
  body: JSON.stringify({ transcripcion: "Doctor: ¿Cómo está?..." })
}).then(r => r.json());

console.log(soap.diagnostico);
console.log(soap.medicamentos);
```

---

## 3. Agendar Cita

Crea una cita médica para un paciente. Envía confirmación automática por WhatsApp al paciente y recordatorio 24 horas antes.

```
POST /medico/api/guardar_cita
Content-Type: application/json
```

**Body:**
```json
{
  "nombre_paciente": "Juan Pérez",
  "telefono": "5215544861234",
  "fecha": "2026-06-15",
  "hora": "10:00",
  "motivo": "Consulta general",
  "doctor_id": 1,
  "consultorio_id": 1,
  "duracion_minutos": 30,
  "enviar_confirmacion": true
}
```

**Respuesta:**
```json
{
  "ok": true,
  "cita_id": 456,
  "confirmacion_enviada": true,
  "mensaje": "Cita agendada para el 15 de junio a las 10:00 AM"
}
```

**Campos opcionales:**
| Campo | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `duracion_minutos` | `int` | `30` | Duración de la consulta |
| `enviar_confirmacion` | `bool` | `true` | Enviar WhatsApp de confirmación al paciente |
| `notas` | `string` | `null` | Notas internas de la cita |
| `tipo_consulta` | `string` | `"presencial"` | `presencial` o `teleconsulta` |

**Errores:**
| Código | Descripción |
|--------|-------------|
| `400` | Datos incompletos o fecha inválida |
| `401` | No autorizado |
| `409` | Conflicto — ya existe una cita en ese horario |

**Ejemplo en Python:**
```python
import httpx

r = httpx.post("https://bots.quantumsmarttradeai.com/medico/api/guardar_cita",
    json={
        "nombre_paciente": "María García",
        "telefono": "5215544861234",
        "fecha": "2026-06-20",
        "hora": "11:30",
        "motivo": "Control de hipertensión",
        "doctor_id": 1,
        "consultorio_id": 1
    },
    cookies={"session": "tu_cookie"})

print(r.json())
# {"ok": true, "cita_id": 789, "confirmacion_enviada": true}
```

---

## 4. Enviar Mensaje WhatsApp

Envía un mensaje de texto por WhatsApp a cualquier número usando la instancia del negocio. Ideal para notificaciones, recordatorios personalizados y comunicación directa con pacientes.

```
POST /api/soporte/enviar
Content-Type: application/json
```

**Body:**
```json
{
  "cliente_id": "5215544861234",
  "mensaje": "Hola Dr. García, le recordamos su cita mañana a las 10:00 AM."
}
```

**Respuesta:**
```json
{
  "ok": true,
  "mensaje_id": "BAE5F4E123456789"
}
```

**Notas:**
- El número debe incluir código de país sin `+` (ejemplo: `5215544861234` para México)
- Mensajes más de 4096 caracteres se truncan automáticamente
- El bot de respuesta automática se pausa 2 horas tras enviar un mensaje manual

**Ejemplo en Python:**
```python
import httpx

r = httpx.post("https://bots.quantumsmarttradeai.com/api/soporte/enviar",
    json={
        "cliente_id": "5215544861234",
        "mensaje": "Su resultado de laboratorio está listo. Puede verlo en: https://..."
    },
    cookies={"session": "tu_cookie"})

print(r.json())
# {"ok": true, "mensaje_id": "BAE5F4..."}
```

**Ejemplo en JavaScript:**
```javascript
const result = await fetch('/api/soporte/enviar', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  credentials: 'include',
  body: JSON.stringify({
    cliente_id: '5215544861234',
    mensaje: 'Su cita ha sido confirmada para mañana a las 10:00 AM.'
  })
}).then(r => r.json());
```

---

## 5. Consultar Expediente

Obtiene el historial clínico completo de un paciente incluyendo consultas, diagnósticos, medicamentos y signos vitales.

```
GET /medico/api/paciente/{expediente_id}/historial-chat
```

**Parámetros:**
| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `expediente_id` | `int` | ID del expediente médico |

**Respuesta:**
```json
{
  "expediente_id": 123,
  "paciente": {
    "nombre_completo": "Juan Pérez López",
    "fecha_nacimiento": "1985-03-15",
    "sexo": "M",
    "telefono": "5215544861234",
    "alergias": "Penicilina",
    "antecedentes": "Hipertensión arterial sistémica"
  },
  "consultas": [
    {
      "id": 456,
      "fecha": "2026-06-01",
      "motivo": "Control de hipertensión",
      "diagnostico": "Hipertensión arterial controlada",
      "subjetivo": "Paciente refiere sentirse bien, sin cefalea ni mareo.",
      "objetivo": "TA 125/80 mmHg. Frecuencia cardíaca 72 lpm.",
      "evaluacion": "Hipertensión arterial con adecuado control farmacológico.",
      "plan": "Continuar tratamiento actual. Control en 3 meses.",
      "medicamentos": ["Losartán 50mg cada 24 horas"],
      "signos_vitales": {
        "presion_sistolica": 125,
        "presion_diastolica": 80,
        "frecuencia_cardiaca": 72,
        "peso_kg": 78.5
      }
    }
  ],
  "total_consultas": 1
}
```

**Errores:**
| Código | Descripción |
|--------|-------------|
| `401` | No autorizado |
| `404` | Expediente no encontrado |

**Ejemplo en Python:**
```python
import httpx

r = httpx.get("https://bots.quantumsmarttradeai.com/medico/api/paciente/123/historial-chat",
    cookies={"session": "tu_cookie"})

expediente = r.json()
print(f"Paciente: {expediente['paciente']['nombre_completo']}")
print(f"Consultas: {expediente['total_consultas']}")
```

**Ejemplo en JavaScript:**
```javascript
const expediente = await fetch('/medico/api/paciente/123/historial-chat', {
  credentials: 'include'
}).then(r => r.json());

console.log(expediente.paciente.nombre_completo);
expediente.consultas.forEach(c => console.log(c.diagnostico));
```

---

## Autenticación

Actualmente los endpoints requieren una cookie de sesión activa (`session`). Para integraciones server-to-server, próximamente estarán disponibles **API Keys** con los siguientes niveles:

| Plan | Requests/mes | Precio |
|------|-------------|--------|
| Free | 100 transcripciones, 500 mensajes | Gratis |
| Startup | 1,000 transcripciones, 5,000 mensajes | $50 USD/mes |
| Scale | Ilimitado + SLA | $200 USD/mes + uso |

---

## Códigos de Error Globales

| Código | Descripción |
|--------|-------------|
| `400` | Bad Request — parámetros inválidos o faltantes |
| `401` | Unauthorized — sesión inválida o expirada |
| `402` | Payment Required — créditos de IA insuficientes |
| `404` | Not Found — recurso no encontrado |
| `409` | Conflict — conflicto de datos (ej. cita duplicada) |
| `500` | Internal Server Error — error del servidor |

---

## Soporte para Desarrolladores

- **Email:** soporte@medibot.mx
- **WhatsApp:** +52 55 XXXX XXXX
- **Documentación:** https://developers.medibot.mx *(próximamente)*

---

*MediBot API v1.0 — Junio 2026*
