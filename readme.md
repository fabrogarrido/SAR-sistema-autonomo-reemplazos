# SAR — Sistema Autónomo de Reemplazos

Sistema de automatización end-to-end desarrollado para el Servicio de Salud Mental del **Hospital Público HIGA Eva Perón de San Martín**. Reemplaza el proceso manual de búsqueda de reemplazos de guardia — que dependía de llamadas telefónicas y mensajes de WhatsApp — por un flujo algorítmico, transparente y disponible las 24 horas.

**Stack:** n8n (self-hosted) · JavaScript · Google Sheets API · Telegram Bot API · Gmail API · Docker · VPS  
**Costo de infraestructura:** $0 (todas las herramientas son gratuitas)  
**Estado:** v1 en producción

---

## El problema

Cuando un psicólogo necesita un reemplazo para su guardia, el proceso era:

1. El profesional avisa que no puede ir
2. La jefa empieza a llamar uno por uno hasta que alguien conteste
3. Pueden pasar horas sin cobertura confirmada
4. Los acuerdos quedan en chats privados y se pierden
5. Siempre se llama a los mismos, generando conflictos de distribución

## La solución

El SAR actúa como un **coordinador imparcial** que:

- Recibe la solicitud de licencia vía Google Forms
- Calcula automáticamente qué fechas de guardia quedan sin cubrir
- Contacta a los candidatos disponibles **en orden de prioridad** vía Telegram
- Permite selección de múltiples fechas con checkboxes interactivos
- Si alguien rechaza o acepta parcialmente, escala automáticamente al siguiente
- Registra cada cobertura en Google Sheets para liquidación mensual
- Notifica a jefatura y secretaría por email en tiempo real

## Arquitectura

```
        Google Forms
              │
              ▼
┌─────────────────────────────────┐
│  Workflow 1 — Flujo Principal   │
│                                 │
│  Expandir fechas del rango      │
│  Filtrar candidatos por día     │
│  Ordenar por prioridad          │
│  Enviar Custom Form (Telegram)  │
│  Registrar aceptaciones         │
│  Notificar (Telegram + Gmail)   │
└──────────────┬──────────────────┘
               │ HTTP POST (rechazo / aceptación parcial)
               ▼
┌─────────────────────────────────┐
│  Workflow 2 — Loop de Rechazo   │
│                                 │
│  Leer índice (Google Sheets)    │
│  Calcular siguiente candidato   │
│  Enviar Custom Form (Telegram)  │
│  Registrar aceptaciones         │
│  Notificar (Telegram + Gmail)   │
│  HTTP POST a sí mismo (loop)    │
└─────────────────────────────────┘
```

### ¿Por qué dos workflows separados?

n8n es **stateless**: cada ejecución de webhook es independiente y no comparte memoria con la anterior. Un único workflow con múltiples entradas ejecutaría los nodos una vez por cada conexión entrante, generando datos duplicados.

La separación garantiza que cada workflow tenga **una única entrada limpia**. El índice del candidato actual se persiste en Google Sheets, que actúa como *external state store* — la misma solución conceptual que Redis o DynamoDB en sistemas de mayor escala.

## Estructura del repositorio

```
SAR/
├── README.md                        ← este archivo
├── SAR_WF1_Principal/
│   ├── SAR_WF1_Principal.json       ← workflow exportado de n8n
│   └── README.md                    ← documentación del WF1
└── SAR_WF2_Loop_Rechazo/
    ├── SAR_WF2_Loop_Rechazo.json    ← workflow exportado de n8n
    └── README.md                    ← documentación del WF2
```

## Estructura de datos — Google Sheets

**Pestaña: Profesionales**
| PRIORIDAD | NOMBRE | CATEGORIA | ESTADO | DISPONIBILIDAD | ID TELEGRAM | OBSERVACIONES |
|-----------|--------|-----------|--------|----------------|-------------|---------------|
| 1–15 | Nombre | Planta G/C/R | ACTIVO | LUNES, MARTES... | ID numérico | Texto libre |

**Pestaña: Reemplazos** (registro generado automáticamente)
| FECHA | DIA | CUBIERTO_POR | SOLICITANTE | MOTIVO |
|-------|-----|--------------|-------------|--------|
| DD/MM/YYYY | MARTES | Profesional X | Profesional Y | Vacaciones |

**Pestaña: Estado** (memoria persistente del sistema)
| CLAVE | VALOR |
|-------|-------|
| indiceCandidato | 0 |

## Instalación

### Requisitos
- n8n instalado (self-hosted o cloud)
- Google Sheets con la estructura descripta arriba
- Bot de Telegram creado con [@BotFather](https://t.me/BotFather)
- Cuenta de Gmail conectada a n8n vía OAuth2
- Google Forms vinculado a la pestaña `Respuestas de formulario 1`

### Pasos

1. **Importar los workflows** en n8n:  
   `Menú → Workflows → Import from file` → importá primero el WF1, luego el WF2

2. **Configurar credenciales** en n8n:
   - Google Sheets: OAuth2 con acceso a tu spreadsheet
   - Telegram: token del bot creado con BotFather
   - Gmail: OAuth2 con tu cuenta

3. **Actualizar referencias** en ambos workflows:
   - URL del webhook del WF2 en los nodos `HTTP Request`
   - Email de destino en los nodos `Gmail`
   - IDs de tu spreadsheet en los nodos `Google Sheets`

4. **Activar el WF2 primero** para obtener su URL de webhook, luego el WF1

5. **Cargar los profesionales** en la pestaña `Profesionales` del Sheet

6. **Testear** enviando una respuesta de prueba desde el Google Forms

## Decisiones técnicas clave

| Decisión | Alternativa descartada | Razón |
|----------|----------------------|-------|
| Dos workflows separados | Loop dentro del mismo WF | Evita doble ejecución de nodos con múltiples entradas |
| Google Sheets como state store | `$getWorkflowStaticData` | La variable interna de n8n no persiste entre ejecuciones de webhooks separados |
| Custom Form con checkboxes dinámicos | Botones Acepto/Rechazo | Permite selección de múltiples fechas en una sola interacción |
| Dropdown en Google Forms | Campo de texto libre | Garantiza que el nombre del solicitante coincida exactamente con el registro en el Sheet |

## Alcance — v1 vs v2

| Funcionalidad | v1 | v2 |
|---------------|----|----|
| Detección automática de licencias via Forms | ✅ | |
| Filtrado por disponibilidad y prioridad | ✅ | |
| Selección múltiple de fechas con checkboxes | ✅ | |
| Escalado automático al siguiente candidato | ✅ | |
| Notificaciones en tiempo real (Telegram + Gmail) | ✅ | |
| Registro automático en Google Sheets | ✅ | |
| Timeout de 6 horas por inactividad | ✅ | |
| Reintento automático tras timeout | | 📋 |
| Regla: el último que cubrió va al final de la lista | | 📋 |
| Límite de guardias por período por persona | | 📋 |
| Botones inline sin enlace externo en Telegram | | 📋 |

---

*Desarrollado por [Fabricio Garrido](https://linkedin.com/in/fabriciogarrido) — [Itera Digital Hub](https://github.com/fabrogarrido)*
