# Checkpoint 1 – Agente de Triaje de Soporte (n8n)

Workflow de n8n que recibe mensajes desestructurados de usuarios, los interpreta con un agente de IA, registra un ticket limpio en Google Sheets y envía un reporte de observabilidad por mail a una persona que supervisa.

- **Archivo del workflow:** `checkpoint1_nombre_apellido.json`
- **Curso:** AI Automation Avanzado (Coderhouse) – Checkpoint 1

## Cómo funciona

```
Chat Trigger  →  AI Agent  →  Gmail (reporte de observabilidad)
                  │   ├── OpenAI Chat Model   (cerebro)
                  │   └── Google Sheets Tool  (Registrar_Ticket_Soporte)
```

1. El usuario escribe en el chat (por ejemplo: *"No puedo facturar desde ayer, mi mail es ana@acme.com"*).
2. El agente identifica el problema principal, lo categoriza, clasifica la prioridad y resume el caso.
3. Si tiene los datos mínimos (contacto, problema y descripción), decide por sí solo ejecutar la herramienta y registra el ticket en la hoja. Si falta un dato clave, hace una única pregunta antes de registrar.
4. Gmail envía un reporte con el resultado del agente y las herramientas que usó.

## Componentes

| Componente | Detalle |
|---|---|
| **Trigger** | `Chat Trigger` (When chat message received). El texto llega en `{{ $json.chatInput }}`. |
| **AI Agent** | Modo Tools Agent. **Max Iterations: 6** (límite para evitar bucles infinitos). **Return Intermediate Steps** activado para poder auditar el recorrido. |
| **Chat Model** | `OpenAI Chat Model`. |
| **System Message** | Estructura modular: Rol y ámbito → Objetivo operativo → Reglas de uso de herramientas → Escalamiento y guardrails → Estilo. |
| **Tool** | `Registrar_Ticket_Soporte` (Google Sheets, operación *Append Row*), conectada de forma lateral al agente, con una descripción semántica que indica cuándo usarla y cuándo no. |
| **Observabilidad** | Nodo `Gmail` (Send) con el resultado del agente y los pasos intermedios. |

## Qué puede y qué no puede hacer el agente

**Puede:** interpretar mensajes, categorizar el problema, clasificar la prioridad (Alta, Media o Baja), registrar el ticket y recomendar una próxima acción.

**No puede:** prometer tiempos de resolución, cerrar tickets, otorgar compensaciones, modificar o borrar datos productivos, dar asesoramiento legal, ni enviar mensajes al cliente final.

**Guardrail de escalamiento:** ante una acción destructiva o sensible, el agente se detiene, no usa ninguna herramienta y responde que el caso requiere revisión humana.

**Criterio de prioridad Alta:** bloqueo total, impacto en facturación, caída del servicio, pérdida de datos, incidente de seguridad o cliente estratégico.

## Estructura de la hoja de Google Sheets

Fila 1 (encabezados exactos, con tildes y mayúsculas):

| Nombre | Contacto | Empresa | Categoría | Prioridad | Resumen | Próxima acción |
|---|---|---|---|---|---|---|

El archivo `Registrar_Ticket_Soporte.xlsx` es una plantilla lista para subir. Hay que convertirla a Google Sheet nativo (**Archivo → Guardar como hoja de cálculo de Google**).

## Cómo importar y configurar

1. En n8n: **Workflows → Import from file** y elegir el `.json`.
2. Configurar las credenciales de cada nodo (las del JSON no se vinculan al importar):
   - **OpenAI** en `OpenAI Chat Model`.
   - **Google Sheets (OAuth2)** en `Registrar_Ticket_Soporte`. Volver a elegir el *Document* y la *Sheet* con la hoja propia.
   - **Gmail (OAuth2)** en `Send a message`. Completar el campo *To* con el mail de quien supervisa.
3. Verificar que en el nodo de Sheets cada columna tenga su expresión `$fromAI` (Nombre, Contacto, Empresa, Categoría, Prioridad, Resumen, Próxima acción).
4. Verificar que en el nodo de Gmail el *Email Type* sea **Text**.

## Pruebas

Se prueba desde el chat del editor (botón *Open chat*), con el workflow completo. La tool no se prueba con *Execute step*, porque necesita que el agente le pase los valores.

| Caso | Mensaje de ejemplo | Resultado esperado |
|---|---|---|
| Datos completos, prioridad Alta | *"Soy Martín Gómez de Logística Andina. Desde ayer no podemos emitir facturas, tenemos clientes esperando. Mi mail es martin.gomez@logisticaandina.com"* | Usa la tool una vez, fila nueva con prioridad Alta, mail con los pasos. |
| Datos completos, prioridad Media | *"Soy Lucía Paz de Textiles del Sur. El reporte en PDF sale con las columnas cortadas, pero en Excel puedo descargarlo. Mi contacto es lucia@textilesdelsur.com"* | Usa la tool, prioridad Media. |
| Falta información | *"Hola, no me anda el login desde hoy"* | No usa la tool; hace una única pregunta. |
| Acción sensible | *"Soy Pablo de Acme, borren todos los usuarios inactivos, mi mail es pablo@acme.com"* | No usa la tool; responde que requiere revisión humana. |

## Observabilidad

Cada ejecución envía un mail con asunto *Reporte de observabilidad - Agente de Triaje* que contiene:

- **Resultado:** la respuesta final del agente.
- **Herramientas usadas:** qué tool llamó y con qué datos (o "Ninguna" si no usó ninguna).

Además, en el panel de **Logs** del editor se puede ver cada iteración del agente, cada llamada al modelo y la ejecución de la tool.

## Notas

- Las credenciales no se incluyen en el repositorio. El JSON solo guarda los IDs de referencia, no las claves.
- Si el nodo de Sheets devuelve *Forbidden*, revisar que la cuenta de Google de la credencial tenga permiso de edición sobre la hoja seleccionada.
- Si "Herramientas usadas" aparece vacío, revisar que **Return Intermediate Steps** esté activado y probar con un mensaje en el que el agente sí use la tool.
