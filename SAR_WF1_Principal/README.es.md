# SAR — Workflow 1: Setup

Parte del Sistema Autónomo de Reemplazos (SAR) — ver README principal

---

## ¿Qué hace este workflow?

Se dispara cuando un profesional envía el Google Form para solicitar una licencia. Expande el rango de fechas, filtra y ordena los candidatos disponibles por prioridad, resetea el índice de candidatos a `-1` en Google Sheets, y delega inmediatamente al WF2 vía HTTP POST.

WF1 **no contacta candidatos**. Toda la lógica de interacción — mensajes de Telegram, parseo de respuestas, registro de aceptaciones, notificaciones — vive exclusivamente en el WF2.

---

## Flujo de nodos

| Nodo                  | Tipo          | Función                                                                                                      |
| --------------------- | ------------- | ------------------------------------------------------------------------------------------------------------ |
| Google Sheets Trigger | Trigger       | Detecta una nueva fila en la pestaña de respuestas del formulario                                            |
| Expandir Fechas       | Code JS       | Calcula qué fechas de guardia caen dentro del rango solicitado                                               |
| Leer Profesionales    | Google Sheets | Lee todos los profesionales desde la pestaña Profesionales                                                   |
| Filtrar Candidatos    | Code JS       | Filtra por día disponible, estado ACTIVO y excluye al solicitante. Ordena por prioridad                      |
| Split Fechas          | Code JS       | Formatea fechas a DD/MM/AAAA y prepara el array `guardiasPendientes`                                         |
| Reset Indice          | Google Sheets | Escribe `-1` en la pestaña Estado — WF2 incrementa a `0` en el primer llamado, cayendo correctamente en el candidato 0 |
| HTTP Request          | HTTP          | POST al webhook de WF2 con el payload completo (fechas, candidatos, info del solicitante)                    |

---

## Configuración requerida

### 1. Google Sheets

- Abrir el nodo Google Sheets Trigger y seleccionar tu hoja SAR
- Repetir para los nodos Leer Profesionales y Reset Indice
- Verificar que los nombres de las pestañas coincidan: `Respuestas de formulario 1`, `Profesionales`, `Estado`

### 2. URL del webhook de WF2

- Activar primero el WF2 para obtener su URL de webhook
- En el nodo HTTP Request, pegar la URL del webhook de WF2

---

## Notas técnicas

- **Reset Indice** escribe `-1` (no `0`) para que el primer incremento del WF2 caiga en el candidato `0`. Esto es lo que permite que WF2 maneje todos los candidatos — incluyendo el primero — sin ningún caso especial en WF1.
- El body del HTTP Request referencia `$('Split Fechas').first().json.*` explícitamente, porque **Reset Indice** devuelve la fila actualizada del Sheet, no los datos de candidatos. La expresión alcanza hacia atrás al nodo Split Fechas para obtener el payload completo.
- No existe lógica de Telegram, Gmail ni contacto de candidatos en este workflow. Este workflow pasó de 20 nodos (v1) a 7 nodos (v2) al mover toda esa lógica al WF2.
