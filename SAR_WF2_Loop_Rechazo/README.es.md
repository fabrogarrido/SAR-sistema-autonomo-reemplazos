# SAR — Workflow 2: Loop de Rechazo

Parte del Sistema Autónomo de Reemplazos (SAR) — ver README principal

---

## ¿Qué hace este workflow?

Maneja **toda la lógica de contacto de candidatos** — desde el primero hasta el último. Se dispara por un HTTP Request desde WF1 en cada nueva solicitud de reemplazo (no solo ante rechazos). Avanza por la lista de prioridad enviando un formulario interactivo de Telegram a cada candidato, hasta que todas las fechas estén cubiertas o se agoten los candidatos disponibles.

WF2 es la única fuente de verdad para: contacto de candidatos, parseo de respuestas, registro de aceptaciones, notificación al solicitante y email a jefatura. En la v1, esta lógica estaba duplicada entre WF1 y WF2. En la v2, vive únicamente aquí.

**Cómo funciona el índice:** WF1 resetea la pestaña Estado a `-1`. En el primer llamado, WF2 lee `-1`, incrementa a `0`, y contacta al candidato `0` (el de mayor prioridad). En cada rechazo subsiguiente, WF2 se llama a sí mismo y vuelve a incrementar.

---

## ¿Por qué un workflow separado?

n8n es stateless por diseño: cada ejecución de webhook es independiente y no comparte memoria con ejecuciones anteriores. Si el loop de rechazo estuviera dentro de WF1, un nodo con múltiples conexiones entrantes se ejecutaría dos veces — una por cada conexión — generando datos duplicados e inconsistentes.

Separar en dos workflows garantiza que cada uno tenga un único punto de entrada limpio, eliminando el problema de doble ejecución. El índice del candidato actual se persiste en la pestaña Estado de Google Sheets, actuando como *external state store* entre ejecuciones independientes.

---

## Flujo de nodos

| Nodo                            | Tipo          | Función                                                                                               |
| ------------------------------- | ------------- | ----------------------------------------------------------------------------------------------------- |
| Webhook                         | Webhook       | Recibe el payload con fechas pendientes y candidatos desde WF1 o desde sí mismo                       |
| Read Index                      | Google Sheets | Lee `candidateIndex` desde la pestaña Estado                                                          |
| Next Candidate                  | Code JS       | Calcula el siguiente candidato y prepara `pendingShifts`                                              |
| IF1 (noCandidates)              | IF            | Verifica si todavía hay candidatos disponibles                                                        |
| Notify Management — No Coverage | Gmail         | Email de alerta cuando se agotaron todos los candidatos                                               |
| Save Index                      | Google Sheets | Escribe el nuevo índice en la pestaña Estado                                                          |
| Ask and Wait 2                  | Telegram      | Envía un Custom Form con checkboxes dinámicos al siguiente candidato                                  |
| Process Response 2              | Code JS       | Calcula `acceptedDates` y `remainingDates`                                                            |
| IF (canCover)                   | IF            | Evalúa si el profesional aceptó al menos una fecha                                                    |
| Replacement Confirmed 2         | Telegram      | Confirma al profesional qué fechas va a cubrir                                                        |
| Find Requester 2                | Google Sheets | Busca el Telegram ID del solicitante                                                                  |
| You Are Covered 2               | Telegram      | Notifica al solicitante el estado actualizado de la cobertura                                         |
| Notify Management 2             | Gmail         | Email a jefatura y administración con el resumen                                                      |
| Expand Accepted Dates 2         | Code JS       | Genera un ítem por cada fecha aceptada                                                                |
| Log Replacement 2               | Google Sheets | Escribe una fila por fecha en la pestaña Reemplazos                                                   |
| IF Any Dates Left? 2            | IF            | Verifica si `remainingDates.length > 0`                                                               |
| All Covered 2                   | Telegram      | Mensaje final al solicitante cuando todas las fechas están cubiertas                                  |
| Prepare Escalation 2            | Code JS       | Prepara el payload para el siguiente candidato                                                        |
| HTTP Request 2                  | HTTP          | POST al mismo webhook — reinicia el loop con el siguiente candidato                                   |

---

## Configuración requerida

### 1. Webhook

- Activar este workflow primero para obtener la URL del webhook
- Copiar esa URL y pegarla en:
  - El nodo HTTP Request de WF1 (delegación inicial)
  - Los nodos HTTP Request e HTTP Request 2 de este workflow (auto-loop ante rechazo)
- Reemplazar todas las instancias de `https://TU-DOMINIO-N8N.com/webhook/sar-rejection`

### 2. Google Sheets

- Asignar tu hoja SAR a los siguientes nodos: Read Index, Save Index, Find Requester 2, Log Replacement 2
- Verificar que los nombres de las pestañas coincidan: `Estado`, `Profesionales`, `Reemplazos`

### 3. Telegram

- Usar la misma credencial de Telegram API configurada en WF1
- Asignarla a todos los nodos de Telegram de este workflow

### 4. Gmail

- Usar la misma credencial de Gmail configurada en WF1
- En los nodos Notify Management — No Coverage y Notify Management 2, reemplazar `your-email@gmail.com` con los emails reales

---

## Notas técnicas

- El nodo Next Candidate accede al body del webhook con `JSON.parse(rawBody[''])` — n8n envuelve el body en una clave vacía cuando el Content-Type es `application/json`; el código maneja ambos casos con un `try/catch`
- El loop es autolimitante: cuando `candidateIndex >= candidates.length`, IF1 enruta al nodo de notificación a jefatura y el flujo termina limpiamente
- El índice se incrementa antes de enviar el mensaje, por lo que el valor en la pestaña Estado siempre refleja el candidato que está siendo contactado en ese momento
- En el **primer llamado desde WF1**, la pestaña Estado contiene `-1`. `Siguiente Candidato` lo lee, incrementa a `0`, y contacta a `candidatos[0]` — el profesional de mayor prioridad
- Timeout de 6 horas en Ask and Wait 2; si el profesional no responde, esa ejecución se detiene (pendiente: reintento automático con el siguiente candidato)
