# SAR — Error Handler

> Workflow 3 de 3 · [View in English](./README.md)

Workflow centralizado de manejo de errores para el sistema SAR. Detecta automáticamente fallas en cualquier workflow del SAR y envía una alerta por email con los detalles necesarios para diagnosticar y resolver el problema de inmediato.

---

## Descripción

Este workflow funciona como red de seguridad de todo el sistema SAR. Cuando cualquier nodo del **Workflow 1 (SAR)** o del **Workflow 2 (SAR - Loop de Rechazo)** falla — ya sea por un error de conexión con Google Sheets, un problema con la API de Telegram, o cualquier falla inesperada — este workflow se activa automáticamente y envía un email de alerta estructurado.

Esto es especialmente relevante en un contexto de producción donde el sistema opera 24/7 sin supervisión humana constante.

---

## Cómo Funciona

```
Error Trigger → Gmail (alerta al administrador)
```

| Paso | Nodo | Descripción |
|------|------|-------------|
| 1 | Error Trigger | Se activa automáticamente cuando cualquier workflow vinculado falla |
| 2 | Send a message (Gmail) | Envía un email de alerta estructurado con los detalles del error |

---

## Contenido del Email de Alerta

El email incluye:

- **Nombre del workflow** — cuál falló (SAR o SAR - Loop de Rechazo)
- **Nombre del nodo** — el nodo específico donde ocurrió la falla
- **Mensaje de error** — la descripción del error si está disponible
- **Fecha y hora** — cuándo ocurrió la falla
- **Link directo** — URL al log de ejecución en n8n para debugging inmediato

---

## Configuración

### 1. Importar el workflow
Importá `workflow-sar-error-handler.json` en tu instancia de n8n.

### 2. Configurar credenciales de Gmail
Conectá tu cuenta de Gmail en n8n y actualizá la referencia de credencial en el nodo Gmail.

### 3. Configurar el email destinatario
Actualizá el campo `sendTo` en el nodo Gmail con el email del administrador.

### 4. Vincular a los otros workflows
En el **Workflow 1 (SAR)** y en el **Workflow 2 (SAR - Loop de Rechazo)**:
- Abrí la configuración del workflow (**Settings**)
- Configurá **Error Workflow** → `SAR - Error Handler`
- Guardá y publicá

### 5. Activar
Activá el workflow con el toggle de publicación.

---

## Requisitos

- n8n (self-hosted o cloud)
- Credenciales Gmail OAuth2 configuradas en n8n
- El Workflow 1 y el Workflow 2 deben tener este workflow configurado como Error Workflow en sus Settings

---

## Parte del Sistema SAR

| Workflow | Descripción |
|----------|-------------|
| [Workflow 1 — SAR](../wf1/) | Flujo principal: ingesta del formulario, filtrado de candidatos, notificación por Telegram |
| [Workflow 2 — SAR Loop de Rechazo](../wf2/) | Loop de escalado: recorre candidatos hasta encontrar cobertura |
| **Workflow 3 — SAR Error Handler** | **Este workflow: detección centralizada de errores y alertas** |

---

## Stack Tecnológico

`n8n` `Gmail API` `Error Trigger` `JavaScript`

---

*Parte del proyecto SAR — desarrollado por [Fabricio Garrido](https://github.com/fabrogarrido) · [Ineo Digital Hub](https://iteradigitalhub.com)*
