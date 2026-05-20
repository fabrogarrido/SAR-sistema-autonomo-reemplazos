# SAR — Sistema Autónomo de Reemplazos

**Automatización end-to-end en producción** para el Servicio de Salud Mental del Hospital Público de la Provincia de Buenos Aires.

Reemplaza un proceso crítico que dependía de llamadas telefónicas y mensajes de WhatsApp por un sistema algorítmico disponible las 24 horas — con costo de infraestructura cero.

> **Estado:** v1 en producción · Costo de infraestructura: $0

---

## El problema que resuelve

Cuando un psicólogo no puede cubrir su guardia, el proceso era:

1. El profesional avisa que no puede ir
2. La jefa empieza a llamar uno por uno hasta que alguien conteste
3. Pueden pasar horas sin cobertura confirmada
4. Los acuerdos quedan en chats privados, sin registro
5. Siempre se llama a los mismos — generando conflictos de distribución

**En un servicio de salud mental, una guardia sin cobertura ademas de un inconveniente administrativo, es un problema asistencial.**

---

## La solución

El SAR actúa como coordinador imparcial que opera sin intervención humana:

```
Profesional completa Google Forms
          ↓
SAR calcula las fechas de guardia sin cubrir
          ↓
Contacta candidatos disponibles por Telegram (orden de prioridad)
          ↓
Profesional selecciona fechas con checkboxes interactivos
          ↓
Si rechaza o acepta parcialmente → escala al siguiente candidato
          ↓
Registra cada cobertura en Google Sheets + notifica a jefatura por email
```

---

## Stack

| Componente | Herramienta |
|---|---|
| Motor de automatización | n8n (self-hosted en VPS con Docker) |
| Base de datos | Google Sheets |
| Formulario de entrada | Google Forms |
| Canal de comunicación | Telegram Bot API |
| Notificaciones administrativas | Gmail API |
| Lenguaje | JavaScript (ES6+) |
| Infraestructura | Contabo VPS · Docker |

---

## Arquitectura

El sistema opera en dos workflows independientes comunicados por webhooks HTTP.

```
          Google Forms
                │
                ▼
┌─────────────────────────────────┐
│   Workflow 1 — Flujo principal  │
│                                 │
│  · Expansión de fechas de rango │
│  · Filtrado y ordenamiento      │
│    de candidatos por prioridad  │
│  · Custom Form dinámico         │
│    con checkboxes (Telegram)    │
│  · Registro de aceptaciones     │
│  · Notificaciones multi-canal   │
└──────────────┬──────────────────┘
               │ HTTP POST (rechazo / aceptación parcial)
               ▼
┌─────────────────────────────────┐
│  Workflow 2 — Loop de rechazo   │
│                                 │
│  · Lee índice del candidato     │
│    actual (Google Sheets)       │
│  · Calcula siguiente candidato  │
│  · Repite el flujo de contacto  │
│  · Se llama a sí mismo via      │
│    HTTP POST (loop stateless)   │
└─────────────────────────────────┘
```

### ¿Por qué dos workflows separados?

n8n es stateless por diseño: cada ejecución de webhook es independiente y no comparte memoria con ejecuciones anteriores. Un único workflow con múltiples entradas ejecutaría todos sus nodos una vez por cada conexión entrante, generando registros duplicados.

**La solución:** separar los flujos garantiza una única entrada limpia por workflow. El índice del candidato actual se persiste en una pestaña `Estado` de Google Sheets, que actúa como *external state store* — el mismo patrón conceptual que Redis o DynamoDB en sistemas de mayor escala, implementado con costo cero.

---

## Decisiones técnicas

| Decisión | Alternativa descartada | Por qué |
|---|---|---|
| Dos workflows separados | Loop dentro del mismo WF | Evita doble ejecución de nodos con múltiples entradas |
| Google Sheets como state store | `$getWorkflowStaticData` | La variable interna de n8n no persiste entre ejecuciones de webhooks separados |
| Custom Form con checkboxes dinámicos | Botones Acepto / Rechazo | Permite selección de múltiples fechas en una sola interacción |
| Dropdown en Google Forms | Campo de texto libre | Garantiza que el nombre coincida exactamente con el registro en el Sheet |

---

## Funcionalidades — v1 vs v2

| Funcionalidad | v1 | v2 |
|---|---|---|
| Detección automática de licencias via Forms | ✅ | |
| Filtrado por disponibilidad y prioridad | ✅ | |
| Exclusión del solicitante de los candidatos | ✅ | |
| Selección múltiple de fechas con checkboxes | ✅ | |
| Escalado automático al siguiente candidato | ✅ | |
| Notificaciones en tiempo real (Telegram + Gmail) | ✅ | |
| Registro automático en Google Sheets | ✅ | |
| Alerta a jefatura cuando no hay cobertura | ✅ | |
| Timeout de 6 horas por inactividad | ✅ | |
| Reintento automático tras timeout | | 📋 |
| Regla: el último que cubrió va al final | | 📋 |
| Límite de guardias por período por persona | | 📋 |
| Botones inline sin enlace externo en Telegram | | 📋 |

---

## Estructura del repositorio

```
SAR/
├── README.md
├── SAR_WF1_Principal/
│   ├── SAR_WF1_Principal.json    ← workflow exportado de n8n
│   └── README.md                 ← documentación del WF1
└── SAR_WF2_Loop_Rechazo/
    ├── SAR_WF2_Loop_Rechazo.json ← workflow exportado de n8n
    └── README.md                 ← documentación del WF2
```

---

## Instalación

### Requisitos

- n8n instalado (self-hosted o cloud)
- Google Sheets con la estructura descripta abajo
- Bot de Telegram creado con @BotFather
- Cuenta de Gmail conectada a n8n vía OAuth2
- Google Forms vinculado a la pestaña `Respuestas de formulario 1`

### Pasos

1. **Importar los workflows en n8n**
   Menú → Workflows → Import from file → importar primero WF1, luego WF2

2. **Configurar credenciales en n8n**
   - Google Sheets: OAuth2
   - Telegram: token del bot
   - Gmail: OAuth2

3. **Actualizar referencias en ambos workflows**
   - URL del webhook del WF2 en los nodos HTTP Request
   - Email de destino en los nodos Gmail
   - ID del spreadsheet en los nodos Google Sheets

4. **Activar el WF2 primero** para obtener su URL de webhook, luego el WF1

5. **Cargar los profesionales** en la pestaña `Profesionales` del Sheet

6. **Testear** enviando una respuesta de prueba desde el Google Forms

### Estructura de Google Sheets

**Pestaña: Profesionales**

| PRIORIDAD | NOMBRE | CATEGORIA | ESTADO | DISPONIBILIDAD | ID TELEGRAM | OBSERVACIONES |
|---|---|---|---|---|---|---|
| 1–15 | Nombre | Planta / G / C / R | ACTIVO | LUNES, MARTES... | ID numérico | Texto libre |

**Pestaña: Reemplazos** (generada automáticamente)

| FECHA | DIA | CUBIERTO_POR | SOLICITANTE | MOTIVO |
|---|---|---|---|---|
| DD/MM/YYYY | MARTES | Profesional X | Profesional Y | Vacaciones |

**Pestaña: Estado** (memoria persistente del sistema)

| CLAVE | VALOR |
|---|---|
| indiceCandidato | 0 |

---

## Autor

Desarrollado por **Fabricio Garrido** — [LinkedIn](https://linkedin.com/in/fabriciogarrido) · [GitHub](https://github.com/fabrogarrido)
