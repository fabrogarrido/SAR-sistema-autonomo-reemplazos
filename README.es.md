# SAR — Sistema Autónomo de Reemplazos

**Sistema de automatización end-to-end en producción** para el Servicio de Salud Mental de un hospital público en Buenos Aires, Argentina.

Reemplaza un proceso crítico que dependía de llamadas telefónicas y mensajes de WhatsApp por un sistema algorítmico, completamente autónomo y disponible las 24 horas — con costo de infraestructura cero.

> **Estado:** v1 en producción · Costo de infraestructura: $0

---

## El problema que resuelve

Cuando un psicólogo no podía cubrir su guardia, el proceso era el siguiente:

1. El profesional avisa que no puede asistir
2. La jefa del servicio empieza a llamar a sus colegas uno por uno hasta que alguien atiende
3. Pueden pasar horas sin confirmación de cobertura
4. Los acuerdos se cierran en chats privados, sin registro trazable
5. Siempre se llama a las mismas personas — generando conflictos de distribución sostenidos

**En un servicio de salud mental, una guardia descubierta tiene consecuencias reales para los pacientes.**

---

## La solución

SAR actúa como coordinador imparcial que opera sin intervención humana:

```
El profesional completa un Google Form
          ↓
SAR calcula qué fechas de guardia quedan descubiertas
          ↓
Contacta a los candidatos disponibles por Telegram (en orden de prioridad)
          ↓
El profesional selecciona fechas usando checkboxes interactivos
          ↓
Si rechaza o acepta parcialmente → escala al siguiente candidato
          ↓
Registra cada cobertura en Google Sheets + notifica a la jefatura por email
```

---

## Stack tecnológico

| Componente              | Herramienta                           |
| ----------------------- | ------------------------------------- |
| Motor de automatización | n8n (self-hosted en VPS con Docker)   |
| Base de datos           | Google Sheets                         |
| Formulario de entrada   | Google Forms                          |
| Canal de comunicación   | Telegram Bot API                      |
| Notificaciones admin    | Gmail API                             |
| Lenguaje                | JavaScript (ES6+)                     |
| Infraestructura         | Contabo VPS · Docker                  |

---

## Arquitectura

El sistema corre en tres workflows independientes. Dos manejan la lógica principal de reemplazos, y uno actúa como manejador centralizado de errores.

```
        Google Forms
              │
              ▼
┌─────────────────────────────────┐
│   Workflow 1 — Flujo principal  │
│                                 │
│  · Expansión del rango de fechas│
│  · Filtrado y ordenamiento de   │
│    candidatos por prioridad     │
│  · Formulario dinámico con      │
│    checkboxes (Telegram)        │
│  · Registro de aceptaciones     │
│  · Notificaciones multicanal    │
└──────────────┬──────────────────┘
               │
               │ HTTP POST (rechazo / aceptación parcial)
               ▼
┌─────────────────────────────────┐
│  Workflow 2 — Loop de rechazo   │
│                                 │
│  · Lee el índice del candidato  │
│    actual (Google Sheets)       │
│  · Calcula el siguiente         │
│    candidato                    │
│  · Repite el flujo de contacto  │
│  · Se llama a sí mismo por HTTP │
│    POST (loop stateless)        │
└─────────────────────────────────┘

        ↕ ante cualquier falla en WF1 o WF2

┌─────────────────────────────────┐
│  Workflow 3 — Error Handler     │
│                                 │
│  · Se activa automáticamente    │
│    ante cualquier falla         │
│  · Envía email de alerta con    │
│    workflow, nodo, error y link │
│    directo al log de ejecución  │
└─────────────────────────────────┘
```

### ¿Por qué dos workflows separados para la lógica principal?

n8n es stateless por diseño: cada ejecución de webhook es independiente y no comparte memoria con ejecuciones anteriores. Un único workflow con múltiples puntos de entrada ejecutaría todos sus nodos una vez por cada conexión entrante, generando registros duplicados.

**La solución:** separar los flujos garantiza un único punto de entrada limpio por workflow. El índice del candidato actual se persiste en una pestaña `Estado` en Google Sheets, actuando como *external state store* — el mismo patrón conceptual que Redis o DynamoDB a mayor escala, implementado a costo cero.

---

## Decisiones técnicas

| Decisión                             | Alternativa descartada    | Por qué                                                                         |
| ------------------------------------ | ------------------------- | ------------------------------------------------------------------------------- |
| Dos workflows separados              | Loop dentro de un único WF | Evita doble ejecución de nodos con múltiples puntos de entrada                 |
| Google Sheets como state store       | `$getWorkflowStaticData`  | La variable interna de n8n no persiste entre ejecuciones de webhooks separados  |
| Checkboxes dinámicos vía Custom Form | Botones Aceptar / Rechazar | Permite seleccionar múltiples fechas en una sola interacción                   |
| Dropdown en Google Forms             | Campo de texto libre      | Garantiza que el nombre coincida exactamente con el registro en la hoja         |
| Workflow dedicado para errores       | Try/catch por nodo        | Alertas centralizadas sin modificar la lógica principal                         |

---

## Funcionalidades — v1 vs v2

| Funcionalidad                                        | v1 | v2 |
| ---------------------------------------------------- | --- | --- |
| Detección automática de licencias vía Forms          | ✅  |    |
| Filtrado por disponibilidad y prioridad              | ✅  |    |
| Solicitante excluido de la lista de candidatos       | ✅  |    |
| Selección de múltiples fechas con checkboxes         | ✅  |    |
| Escalada automática al siguiente candidato           | ✅  |    |
| Notificaciones en tiempo real (Telegram + Gmail)     | ✅  |    |
| Registro automático en Google Sheets                 | ✅  |    |
| Alerta a jefatura cuando no se encuentra cobertura   | ✅  |    |
| Timeout de inactividad de 6 horas                    | ✅  |    |
| Manejo centralizado de errores + alertas por email   | ✅  |    |
| Reintento automático tras timeout                    |    | 📋  |
| Regla: el último en cubrir pasa al final de la lista |    | 📋  |
| Límite de coberturas por persona por período         |    | 📋  |
| Botones inline de Telegram sin link externo          |    | 📋  |

---

## Estructura del repositorio

```
SAR/
├── README.md                             ← versión en inglés
├── README.es.md                          ← este archivo
├── SAR_WF1_Principal/
│   ├── SAR_WF1_Principal.json            ← workflow exportado de n8n
│   ├── README.md                         ← documentación del WF1 (inglés)
│   └── README.es.md                      ← documentación del WF1 (español)
├── SAR_WF2_Loop_Rechazo/
│   ├── SAR_WF2_Loop_Rechazo.json         ← workflow exportado de n8n
│   ├── README.md                         ← documentación del WF2 (inglés)
│   └── README.es.md                      ← documentación del WF2 (español)
└── SAR_WF3_Error_Handler/
    ├── workflow-sar-error-handler.json   ← workflow exportado de n8n
    ├── README.md                         ← documentación del WF3 (inglés)
    └── README.es.md                      ← documentación del WF3 (español)
```

---

## Instalación

### Requisitos

- n8n instalado (self-hosted o cloud)
- Google Sheets con la estructura descripta abajo
- Bot de Telegram creado con @BotFather
- Cuenta de Gmail conectada a n8n via OAuth2
- Google Forms vinculado a la pestaña `Respuestas de formulario 1`

### Pasos

1. **Importar los workflows en n8n** — Menú → Workflows → Importar desde archivo → importar WF1, WF2 y WF3

2. **Configurar credenciales en n8n**
   - Google Sheets: OAuth2
   - Telegram: token del bot
   - Gmail: OAuth2

3. **Actualizar referencias en ambos workflows**
   - URL del webhook de WF2 en los nodos HTTP Request
   - Email de destino en los nodos de Gmail
   - ID de la hoja de cálculo en los nodos de Google Sheets

4. **Vincular el error handler**
   En Settings del WF1 y WF2 → Error Workflow → seleccionar `SAR - Error Handler`

5. **Activar primero WF2 y WF3**, luego activar WF1

6. **Cargar los profesionales** en la pestaña `Profesionales` de la hoja

7. **Probar** enviando una respuesta de ejemplo desde Google Forms

### Estructura de Google Sheets

**Pestaña: Profesionales**

| PRIORIDAD | NOMBRE | CATEGORÍA           | ESTADO | DISPONIBILIDAD | TELEGRAM ID  | NOTAS       |
| --------- | ------ | ------------------- | ------ | -------------- | ------------ | ----------- |
| 1–15      | Nombre | Planta / G / C / R  | ACTIVO | LUN, MAR...    | ID numérico  | Texto libre |

**Pestaña: Reemplazos** (generada automáticamente)

| FECHA      | DÍA    | CUBIERTO_POR  | SOLICITANTE   | MOTIVO   |
| ---------- | ------ | ------------- | ------------- | -------- |
| DD/MM/AAAA | MARTES | Profesional X | Profesional Y | Licencia |

**Pestaña: Estado** (memoria persistente del sistema)

| CLAVE           | VALOR |
| --------------- | ----- |
| indiceCandidato | 0     |

---

## Autor

Desarrollado por **Fabricio Garrido** — [LinkedIn](https://linkedin.com/in/fabriciogarrido) · [GitHub](https://github.com/fabrogarrido)
