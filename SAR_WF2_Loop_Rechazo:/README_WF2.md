# SAR — Workflow 2: Loop de Rechazo

> Parte del **Sistema Autónomo de Reemplazos (SAR)** — ver [README principal](../README.md)

## ¿Qué hace este workflow?

Maneja el escalado automático de candidatos. Se activa por HTTP Request desde el WF1 (cuando el primer candidato rechaza o acepta parcialmente) o desde sí mismo (cuando un candidato del loop también rechaza). Avanza al siguiente candidato en la lista de prioridad hasta cubrir todas las fechas o agotar los candidatos disponibles.

## ¿Por qué es un workflow separado?

n8n es **stateless por diseño**: cada ejecución del webhook es independiente y no comparte memoria con la anterior. Si el loop de rechazo estuviera dentro del WF1, un nodo con múltiples conexiones entrantes se ejecutaría dos veces — una por cada conexión — generando datos duplicados e inconsistentes.

La separación en dos workflows garantiza que cada uno tenga **una única entrada limpia**, eliminando el problema de doble ejecución. El índice del candidato actual se persiste en la pestaña `Estado` de Google Sheets, que actúa como **external state store** entre ejecuciones independientes.

## Flujo de nodos

| Nodo | Tipo | Función |
|------|------|---------|
| Webhook | Webhook | Recibe el payload con fechas pendientes y candidatos desde WF1 o desde sí mismo |
| Leer Indice | Google Sheets | Lee `indiceCandidato` de la pestaña Estado |
| Siguiente Candidato | Code JS | Calcula el próximo candidato y prepara `guardiasPendientes` |
| IF1 (sinCandidatos) | IF | Verifica si quedan candidatos disponibles |
| Notificar Rechazo Jefatura | Gmail | Email de alerta cuando se agotan todos los candidatos |
| Guardar Indice | Google Sheets | Escribe el nuevo índice en la pestaña Estado |
| Preguntar y Esperar 2 | Telegram | Envía Custom Form con checkboxes dinámicos al siguiente candidato |
| Procesar Respuesta 2 | Code JS | Calcula `fechasAceptadas` y `fechasRestantes` |
| IF (puedeCubrir) | IF | Evalúa si el profesional aceptó al menos una fecha |
| Reemplazo Registrado 2 | Telegram | Confirma al profesional las fechas que cubrirá |
| Buscar Solicitante 2 | Google Sheets | Busca el ID de Telegram del solicitante |
| Ya Estas Cubierto 2 | Telegram | Notifica al solicitante el estado actualizado |
| Notificar Jefatura 2 | Gmail | Email a jefatura y secretaría con el resumen |
| Expandir Fechas Aceptadas 2 | Code JS | Genera un item por cada fecha aceptada |
| Registrar Reemplazo 2 | Google Sheets | Escribe una fila por fecha en la pestaña Reemplazos |
| IF ¿Quedan Fechas? 2 | IF | Verifica si `fechasRestantes.length > 0` |
| Todas Cubiertas 2 | Telegram | Mensaje final al solicitante cuando todas las fechas están cubiertas |
| Preparar Escalado 2 | Code JS | Prepara el payload para el siguiente candidato |
| HTTP Request 2 | HTTP | POST al mismo webhook — reinicia el loop con el siguiente candidato |

## Configuración requerida

### 1. Webhook
- Activá el workflow en producción para obtener la URL del webhook
- Copiá esa URL y pegala en el nodo **HTTP Request** del WF1 y en el nodo **HTTP Request 2** de este mismo workflow
- Reemplazá todas las instancias de `https://TU-N8N-DOMINIO.com/webhook/sar-rechazo`

### 2. Google Sheets
- Asigná tu spreadsheet SAR en los nodos: **Leer Indice**, **Guardar Indice**, **Buscar Solicitante 2**, **Registrar Reemplazo 2**
- Verificá que los nombres de pestañas coincidan: `Estado`, `Profesionales`, `Reemplazos`

### 3. Telegram
- Usá la misma credencial **Telegram API** configurada en el WF1
- Asignala a todos los nodos Telegram del workflow

### 4. Gmail
- Usá la misma credencial Gmail configurada en el WF1
- En los nodos **Notificar Rechazo Jefatura** y **Notificar Jefatura 2**, reemplazá `tu-email@gmail.com` con los emails reales

## Notas técnicas

- El nodo **Siguiente Candidato** accede al body del webhook con `JSON.parse(rawBody[''])` — n8n encapsula el body en una clave vacía cuando el Content-Type es `application/json`; el código maneja ambos casos con try/catch
- El loop se auto-limita: cuando `indiceCandidato >= candidatos.length`, el IF1 deriva al nodo de notificación a jefatura y el flujo termina limpiamente
- El índice se incrementa **antes** de enviar el mensaje, por lo que el valor en la pestaña Estado siempre refleja el candidato que está siendo consultado en ese momento
- Timeout de 6 horas en **Preguntar y Esperar 2**; si el profesional no responde, esa ejecución se detiene (pendiente para v2: reintento automático con el siguiente candidato)
