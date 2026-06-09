# SAR — Workflow 1: Flujo Principal

Parte del Sistema Autónomo de Reemplazos (SAR) — ver README principal

---

## ¿Qué hace este workflow?

Se dispara cuando un profesional envía el Google Form para solicitar una licencia. Procesa el rango de fechas, identifica al candidato disponible de mayor prioridad y le envía una consulta interactiva por Telegram. Si el candidato rechaza o acepta parcialmente, delega al WF2 para continuar la escalada.

---

## Flujo de nodos

| Nodo                  | Tipo               | Función                                                                                                      |
| --------------------- | ------------------ | ------------------------------------------------------------------------------------------------------------ |
| Google Sheets Trigger | Trigger            | Detecta una nueva fila en la pestaña de respuestas del formulario                                            |
| Expand Dates          | Code JS            | Calcula qué fechas de guardia caen dentro del rango solicitado                                               |
| Read Professionals    | Google Sheets      | Lee todos los profesionales desde la pestaña Profesionales                                                   |
| Filter Candidates     | Code JS            | Filtra por día disponible, estado ACTIVO y excluye al solicitante. Ordena por prioridad                      |
| Split Dates           | Code JS            | Formatea fechas a DD/MM/AAAA y prepara el array `pendingShifts`                                              |
| Reset Index           | Google Sheets      | Escribe 0 en la pestaña Estado para comenzar desde el candidato de mayor prioridad                           |
| Ask and Wait          | Telegram           | Envía un Custom Form con checkboxes de fechas dinámicos al candidato                                         |
| Process Response      | Code JS            | Calcula `acceptedDates` y `remainingDates` en base a la selección                                            |
| IF (canCover)         | IF                 | Evalúa si el profesional aceptó al menos una fecha                                                           |
| Replacement Confirmed | Telegram           | Confirma al profesional qué fechas va a cubrir                                                               |
| Find Requester        | Google Sheets      | Busca el Telegram ID del profesional solicitante                                                             |
| You Are Covered       | Telegram           | Notifica al solicitante el estado de la cobertura                                                            |
| Notify Management     | Gmail              | Envía un resumen a la jefatura y administración                                                              |
| Expand Accepted Dates | Code JS            | Genera un ítem por cada fecha aceptada                                                                       |
| Log Replacement       | Google Sheets      | Escribe una fila por fecha en la pestaña Reemplazos                                                          |
| IF Any Dates Left?    | IF                 | Verifica si `remainingDates.length > 0`                                                                      |
| All Covered           | Telegram           | Mensaje final al solicitante cuando todas las fechas están cubiertas                                         |
| Prepare Escalation    | Code JS            | Prepara el payload con las fechas restantes y los candidatos                                                 |
| HTTP Request          | HTTP               | POST al webhook de WF2 para continuar la escalada                                                            |

---

## Configuración requerida

### 1. Google Sheets

- Abrir el nodo Google Sheets Trigger y seleccionar tu hoja SAR
- Repetir para los nodos Read Professionals, Reset Index, Find Requester y Log Replacement
- Verificar que los nombres de las pestañas coincidan: `Respuestas de formulario 1`, `Profesionales`, `Estado`, `Reemplazos`

### 2. Telegram

- En n8n → Credenciales → crear una credencial de Telegram API con el token de tu bot
- Asignarla a todos los nodos de Telegram del workflow

### 3. Gmail

- En n8n → Credenciales → conectar tu cuenta de Gmail via OAuth2
- En el nodo Notify Management, reemplazar `your-email@gmail.com` con los emails reales de jefatura y administración

### 4. URL del webhook de WF2

- En el nodo HTTP Request, reemplazar `https://TU-DOMINIO-N8N.com/webhook/sar-rejection` con la URL real del webhook de WF2

---

## Notas técnicas

- El nodo Filter Candidates excluye automáticamente al solicitante comparando `p.NOMBRE === solicitud.solicitante`
- Reset Index al inicio garantiza que cada proceso nuevo comience siempre desde el candidato de prioridad 1
- El timeout del nodo Ask and Wait está configurado en 6 horas; si el profesional no responde, la ejecución se detiene
- Los checkboxes del Custom Form se generan dinámicamente usando `.map()` sobre el array `pendingShifts`
