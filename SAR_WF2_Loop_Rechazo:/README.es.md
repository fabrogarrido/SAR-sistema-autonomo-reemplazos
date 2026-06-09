# SAR — Workflow 2: Loop de Rechazo

Parte del Sistema Autónomo de Reemplazos (SAR) — ver README principal

---

## ¿Qué hace este workflow?

Maneja la escalada automática de candidatos. Se dispara por un HTTP Request desde WF1 (cuando el primer candidato rechaza o acepta parcialmente) o desde sí mismo (cuando un candidato dentro del loop también rechaza). Avanza al siguiente candidato en la lista de prioridad hasta que todas las fechas estén cubiertas o se agoten los candidatos disponibles.

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

- Activar el workflow en producción para obtener la URL del webhook
- Copiar esa URL y pegarla en el nodo HTTP Request de WF1 y en el nodo HTTP Request 2 de este workflow
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
- Timeout de 6 horas en Ask and Wait 2; si el profesional no responde, esa ejecución se detiene (v2 pendiente: reintento automático con el siguiente candidato)
