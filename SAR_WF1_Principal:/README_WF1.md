# SAR — Workflow 1: Flujo Principal

> Parte del **Sistema Autónomo de Reemplazos (SAR)** — ver [README principal](../README.md)

## ¿Qué hace este workflow?

Se activa cuando un profesional completa el Google Form solicitando una licencia. Procesa el rango de fechas, identifica al candidato de mayor prioridad disponible y le envía una consulta interactiva por Telegram. Si hay rechazo o aceptación parcial, delega al [WF2](../SAR_WF2_Loop_Rechazo/) para continuar el escalado.

## Flujo de nodos

| Nodo | Tipo | Función |
|------|------|---------|
| Google Sheets Trigger | Trigger | Detecta nueva fila en Respuestas del formulario |
| Expandir Fechas | Code JS | Calcula las fechas de guardia que caen en el rango solicitado |
| Leer Profesionales | Google Sheets | Lee todos los profesionales de la pestaña Profesionales |
| Filtrar Candidatos | Code JS | Filtra por día disponible, estado ACTIVO y excluye al solicitante. Ordena por prioridad |
| Split Fechas | Code JS | Formatea fechas a DD/MM/YYYY y prepara el array `guardiasPendientes` |
| Reset Indice | Google Sheets | Escribe `0` en la pestaña Estado para iniciar desde el candidato de mayor prioridad |
| Preguntar y Esperar | Telegram | Envía Custom Form con checkboxes dinámicos por fecha al candidato |
| Procesar Respuesta | Code JS | Calcula `fechasAceptadas` y `fechasRestantes` según la selección |
| IF (puedeCubrir) | IF | Evalúa si el profesional aceptó al menos una fecha |
| Reemplazo Registrado | Telegram | Confirma al profesional las fechas que cubrirá |
| Buscar Solicitante | Google Sheets | Busca el ID de Telegram del profesional solicitante |
| Ya Estas Cubierto | Telegram | Notifica al solicitante el estado de cobertura |
| Notificar Jefatura | Gmail | Envía resumen a jefatura y secretaría |
| Expandir Fechas Aceptadas | Code JS | Genera un item por cada fecha aceptada |
| Registrar Reemplazo | Google Sheets | Escribe una fila por fecha en la pestaña Reemplazos |
| IF ¿Quedan Fechas? | IF | Verifica si `fechasRestantes.length > 0` |
| Todas Cubiertas | Telegram | Mensaje final al solicitante cuando todas las fechas están cubiertas |
| Preparar Escalado | Code JS | Prepara el payload con las fechas restantes y los candidatos |
| HTTP Request | HTTP | POST al webhook del WF2 para continuar el escalado |

## Configuración requerida

### 1. Google Sheets
- Abrí el nodo **Google Sheets Trigger** y seleccioná tu spreadsheet SAR
- Repetí para los nodos **Leer Profesionales**, **Reset Indice**, **Buscar Solicitante** y **Registrar Reemplazo**
- Verificá que los nombres de pestañas coincidan: `Respuestas de formulario 1`, `Profesionales`, `Estado`, `Reemplazos`

### 2. Telegram
- En n8n → Credenciales → creá una credencial **Telegram API** con el token de tu bot
- Asignala a todos los nodos de tipo Telegram del workflow

### 3. Gmail
- En n8n → Credenciales → conectá tu cuenta de Gmail vía OAuth2
- En el nodo **Notificar Jefatura**, reemplazá `tu-email@gmail.com` con el email real de jefatura y secretaría

### 4. Webhook URL del WF2
- En el nodo **HTTP Request**, reemplazá `https://TU-N8N-DOMINIO.com/webhook/sar-rechazo` con la URL real del webhook del WF2

## Notas técnicas

- El nodo **Filtrar Candidatos** excluye automáticamente al solicitante comparando `p.NOMBRE === solicitud.solicitante`
- El **Reset Indice** al inicio garantiza que cada proceso nuevo empiece siempre desde el candidato de prioridad 1
- El timeout del nodo **Preguntar y Esperar** está configurado en 6 horas; si el profesional no responde, la ejecución se detiene
- Los checkboxes del Custom Form se generan dinámicamente con `.map()` sobre el array `guardiasPendientes`
