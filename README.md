# QuickBook – Sistema automatizado de reservas

## Descripción del proyecto

QuickBook es un sistema de automatización de reservas desarrollado con n8n, que permite recibir, validar y gestionar solicitudes de citas para diferentes servicios.

La solución consulta la disponibilidad en Google Calendar, aplica las reglas del negocio, registra los resultados en Google Sheets y Baserow y devuelve al usuario una respuesta de confirmación o rechazo.

El proyecto incorpora inteligencia artificial mediante Google Gemini para clasificar el motivo de la cita, generar un resumen e identificar solicitudes que requieren revisión. También permite consultar las horas libres de un día y recibir hasta tres recomendaciones, verificadas contra Google Calendar.

## Información académica

- **Institución:** EAM
- **Asignatura:** Electiva IV
- **Periodo académico:** 2026-II
- **Profesor:** Robert García Rey
- **Estudiantes:** Rossibel Daniela Ibarra - Cesar Andres Guzman - Lorena Reina Capote - Juan Diego Velasco Perdomo

## Problema

La gestión manual de reservas requiere revisar la disponibilidad de los horarios, registrar los datos de los clientes y comunicar el resultado de cada solicitud. Este proceso puede generar errores de registro, reservas duplicadas, demoras en la atención y dificultades para consultar el historial de solicitudes.

QuickBook propone automatizar estas actividades mediante la integración de diferentes servicios, manteniendo reglas de validación que permitan gestionar las reservas de manera organizada.

## Objetivo

Desarrollar un sistema automatizado de gestión de reservas que permita recibir solicitudes, validar los datos, consultar la disponibilidad, crear citas y registrar los resultados, incorporando inteligencia artificial para clasificar el motivo de la solicitud y recomendar horarios disponibles.

## Alcance

QuickBook contempla la gestión de reservas para los siguientes servicios:

- Consulta médica.
- Consulta veterinaria.
- Peluquería.

El sistema permite:

- Recibir solicitudes desde un formulario web propio mediante un Webhook autenticado.
- Validar los datos obligatorios de la reserva.
- Gestionar solicitudes con una anticipación de entre uno y siete días.
- Consultar la disponibilidad de horarios y recomendar hasta tres horas libres.
- Crear citas de una hora de duración en Google Calendar.
- Clasificar el motivo de la solicitud mediante inteligencia artificial.
- Registrar solicitudes confirmadas y rechazadas.
- Informar al usuario el resultado de su solicitud.

La inteligencia artificial cumple una función de clasificación y apoyo. No realiza diagnósticos médicos ni tiene autorización para confirmar, modificar o cancelar citas de manera autónoma.

## Funcionamiento general

QuickBook automatiza el proceso de gestión de reservas mediante un workflow desarrollado en n8n.

El proceso comienza cuando el usuario envía una solicitud con sus datos personales, el servicio requerido, la fecha y la hora de preferencia.

El workflow tiene dos rutas:

**Reserva de citas:**

1. Recibe la solicitud mediante el Webhook autenticado.
2. Normaliza y valida los campos obligatorios, el servicio, la fecha y la hora. Si los datos son inválidos, dirige la solicitud a la rama de rechazo.
3. Si la solicitud es válida y contiene motivo, Google Gemini lo clasifica.
4. Consulta la disponibilidad del bloque de una hora en Google Calendar.
5. Si está disponible, crea la cita; en caso contrario, genera el rechazo.
6. Registra el resultado en Google Sheets y luego en Baserow. La respuesta al usuario se genera en una rama independiente de los registros.

**Consulta de horarios:** cuando la solicitud contiene `accion: "horas"`, el flujo valida la fecha, consulta los eventos del día mediante el agente de horarios y vuelve a verificar la disponibilidad de forma independiente. Devuelve las horas libres y hasta tres recomendaciones. Esta ruta no crea reservas.

### Integración de inteligencia artificial

Google Gemini se utiliza para analizar el motivo de la reserva y generar tres elementos:

- Categoría del motivo: rutina, urgente u otro.
- Resumen breve del motivo.
- Indicador de si la solicitud requiere revisión.

La clasificación se realiza únicamente cuando la solicitud es válida y el usuario proporciona un motivo. Si el servicio de inteligencia artificial falla o devuelve una respuesta inválida, el workflow continúa con el procesamiento de la reserva y marca el motivo como no clasificado.

El agente de horarios utiliza una herramienta de lectura de Google Calendar. El nodo `Armar respuesta` recalcula las horas libres y filtra las recomendaciones. Si Gemini falla y la consulta independiente del calendario funciona, el código puede responder sin la IA.

Las decisiones relacionadas con la disponibilidad y la confirmación de citas se realizan mediante reglas del negocio, independientemente del resultado de la inteligencia artificial.

## Tecnologías utilizadas

| Tecnología | Función dentro del proyecto |
|---|---|
| n8n | Automatización y coordinación del proceso de reservas. |
| Google Calendar | Consulta de disponibilidad y creación de citas. |
| Google Sheets | Registro de solicitudes confirmadas y rechazadas. |
| Baserow | Almacenamiento estructurado de las solicitudes. |
| Google Gemini | Clasificación y resumen del motivo, y recomendaciones de horarios. |
| Webhook HTTP | Recepción de solicitudes y devolución de respuestas. |
| GitHub | Control de versiones y publicación de los entregables del proyecto. |

## Requisitos para ejecutar el proyecto

Para configurar y ejecutar QuickBook se requiere:

- n8n versión 2.35.7, utilizada durante el desarrollo del proyecto.
- Una cuenta de Google con acceso a Google Calendar y Google Sheets.
- Una cuenta de Baserow con permisos para crear tablas y registrar información.
- Una clave de API de Google Gemini con acceso a los modelos `models/gemini-3.5-flash-lite` (clasificador) y `models/gemini-3.5-flash` (agente de horarios), configurados en el JSON final.
- Credenciales configuradas en n8n para cada uno de los servicios externos.
- Un cliente HTTP, como curl o Postman, para enviar solicitudes al Webhook y verificar las respuestas.

### Recursos externos necesarios

| Servicio | Recursos requeridos |
|---|---|
| Google Calendar | Un calendario destinado a la gestión de citas. |
| Google Sheets | Una hoja de cálculo con pestañas para solicitudes confirmadas y rechazadas. |
| Baserow | Una base de datos con tablas para solicitudes confirmadas y rechazadas. |
| Google Gemini | Una clave de API y acceso a los dos modelos de inteligencia artificial configurados en el workflow. |

Las credenciales deben configurarse en la instancia de n8n de cada usuario. No se deben publicar claves de API, contraseñas, tokens ni datos personales reales.

## Configuración de los servicios externos

Antes de ejecutar el workflow, se deben crear los recursos externos y configurar las credenciales correspondientes en n8n.

### 1. Google Calendar

Google Calendar se utiliza para consultar la disponibilidad de los horarios y crear las citas confirmadas.

#### Configuración

1. Ingresar a Google Calendar.
2. Crear un calendario llamado `n8nQuickBook`.
3. Configurar la zona horaria `America/Bogota`.
4. En n8n, crear una credencial de tipo Google Calendar OAuth2 API.
5. Autorizar el acceso a la cuenta de Google.
6. Seleccionar el calendario creado en los siguientes nodos del workflow:

| Nodo | Función |
|---|---|
| Validar en el calendario | Consultar la disponibilidad del horario solicitado. |
| Crea la cita en el calendario | Crear el evento correspondiente a la reserva confirmada. |
| consultar_eventos_del_dia | Consultar los eventos del día mediante la herramienta del agente. |
| Verificar eventos del dia | Consultar los eventos de forma independiente para verificar la disponibilidad. |

Las citas tienen una duración de una hora. Los cuatro nodos deben utilizar el mismo calendario para garantizar que la consulta de disponibilidad corresponda al calendario donde se crean las reservas.

### 2. Google Sheets

Google Sheets se utiliza para registrar las solicitudes procesadas por QuickBook y conservar el historial de reservas.

#### Configuración

1. Crear una hoja de cálculo en Google Sheets.
2. Crear las siguientes pestañas:

| Pestaña | Propósito |
|---|---|
| Confirmado | Registrar las solicitudes de reserva confirmadas. |
| Rechazado | Registrar las solicitudes de reserva rechazadas. |

#### Estructura de las pestañas Confirmado y Rechazado

Ambas pestañas deben contener los siguientes encabezados en la primera fila:

| Columna | Descripción |
|---|---|
| nombre | Nombre del usuario. |
| whatsapp | Número de contacto. |
| correo | Correo electrónico. |
| fecha_preferida | Fecha solicitada para la cita. |
| hora_preferida | Hora solicitada para la cita. |
| servicio | Servicio requerido. |
| estado | Resultado de la solicitud. |
| asunto | Asunto del mensaje generado. |
| cuerpo | Mensaje de confirmación o rechazo. |

Se recomienda configurar las columnas de WhatsApp, fecha y hora como texto para conservar el formato de los datos recibidos.

#### Configuración en n8n

1. Crear una credencial de tipo Google Sheets OAuth2 API.
2. Autorizar el acceso a Google Sheets.
3. Seleccionar la hoja de cálculo creada en los nodos correspondientes.
4. Configurar el nodo `SheetConfirmada` para utilizar la pestaña `Confirmado`.
5. Configurar el nodo `SheetRechazada` para utilizar la pestaña `Rechazado`.
6. Verificar que el mapeo de los campos coincida con los encabezados de cada pestaña.

Cada usuario debe utilizar su propia hoja de cálculo y sus propias credenciales. No es necesario tener acceso a los recursos originales del equipo de desarrollo.

### 3. Baserow

Crear una base de datos con dos tablas, una para reservas confirmadas y otra para rechazadas. Ambas deben contener los siguientes campos (respetar las mayúsculas y minúsculas):

| Campo | Tipo recomendado |
|---|---|
| Nombre | Texto |
| Whatsapp | Texto |
| Correo | Texto |
| Fecha_preferida | Texto |
| Hora_preferida | Texto |
| Servicio | Texto |
| asunto | Texto |
| Cuerpo | Texto largo |
| Estado | Texto |

1. Crear un token de base de datos en Baserow con permisos de creación de filas en las dos tablas.
2. En n8n, crear una credencial de tipo Header Auth con nombre de encabezado `Authorization` y valor `Token` seguido de un espacio y el token propio.
3. Asignar esta credencial a `HTTP Request Confirmar` y `HTTP Request Rechazado`.
4. En cada nodo, reemplazar el identificador de tabla en la URL por el de la tabla correspondiente:

   `https://api.baserow.io/api/database/rows/table/<ID_TABLA>/?user_field_names=true`

5. Mantener el método `POST`, el contenido JSON y comprobar el mapeo de los campos.

### 4. Google Gemini

1. Generar una clave de API propia de Google Gemini en Google AI Studio.
2. Crear en n8n una credencial de tipo Google Gemini (PaLM) API con esa clave.
3. Asignar la credencial a los dos nodos:

| Nodo | Modelo configurado |
|---|---|
| `Modelo Gemini - seleccionar credencial` | `models/gemini-3.5-flash-lite` |
| `Modelo Gemini horarios` | `models/gemini-3.5-flash` |

Verificar que los modelos estén disponibles para la cuenta utilizada. Si se necesita sustituir un modelo, volver a ejecutar las pruebas de clasificación y consulta de horarios y documentar el cambio.

### 5. Webhook

1. Crear una credencial Header Auth independiente de la credencial de Baserow.
2. Configurar el nombre del encabezado como `X-QuickBook-Key` y asignar una clave propia.
3. Seleccionar esa credencial en el nodo `Webhook`.
4. Mantener el método `POST`, la ruta `QuickBook-Reserva` y el modo de respuesta mediante nodos Respond to Webhook.
5. Si se usa un formulario web local, ajustar los orígenes permitidos al origen del formulario. El origen configurado durante las pruebas fue `http://127.0.0.1:5500`.

No compartir la clave del Webhook. Si se despliega un formulario para clientes reales, la clave no debe quedar expuesta en JavaScript del navegador. Para reproducir el workflow sin el formulario se pueden utilizar las solicitudes HTTP de la siguiente sección.

## Importación y ejecución

1. Descargar el [workflow final](./workflows/Workflow_Final.json).
2. Abrir n8n y crear un workflow. En el menú del editor, seleccionar Import from File e importar el JSON.
3. Preparar los recursos y asignar las credenciales propias siguiendo la sección anterior. Seleccionar el calendario en los cuatro nodos, el documento y las pestañas en los dos nodos de Sheets, y reemplazar los ID de las tablas de Baserow.
4. En la configuración del workflow, seleccionar la zona horaria `America/Bogota` y guardar.
5. Para probar sin activar el workflow, abrir el nodo `Webhook`, copiar la Test URL y seleccionar Listen for test event. Enviar la solicitud HTTP. Volver a habilitar la escucha antes de cada nueva solicitud de prueba.
6. Para utilizar la Production URL, guardar y activar o publicar el workflow, según la interfaz de n8n. Usar la URL que muestra el nodo Webhook.
7. Consultar la ejecución en n8n y revisar los resultados en los servicios externos. Al terminar, desactivar el workflow si se publicó para las pruebas. No hay disparadores programados.

Si Google Calendar o Google Sheets requieren una configuración OAuth propia, habilitar las API correspondientes en el proyecto de Google Cloud, crear el cliente OAuth con la URL de redirección que muestra n8n y autorizar la cuenta de prueba con permisos sobre los recursos creados.

## Ejemplos reproducibles

### Reserva de una cita

Condiciones iniciales: credenciales válidas, recursos configurados y un horario libre en el calendario de prueba. La fecha debe estar entre uno y siete días después de hoy, en la zona `America/Bogota`.

Crear un archivo local `solicitud.json` con datos ficticios. Sustituir `AAAA-MM-DD` por una fecha válida, por ejemplo tres días después de hoy:

```json
{
  "nombre": "Usuario Prueba",
  "whatsapp": "3000000000",
  "correo": "prueba@example.com",
  "servicio": "consulta medica",
  "fecha_preferida": "AAAA-MM-DD",
  "hora_preferida": "10:00",
  "motivo": "Control general"
}
```

Desde Git Bash, con n8n local y el Webhook a la espera de una solicitud de prueba:

```bash
read -r -s -p "Clave del Webhook: " QUICKBOOK_KEY
printf '\n'
curl -i -X POST 'http://localhost:5678/webhook-test/QuickBook-Reserva' \
  -H 'Content-Type: application/json' \
  -H "X-QuickBook-Key: $QUICKBOOK_KEY" \
  --data-binary @solicitud.json
unset QUICKBOOK_KEY
```

Ajustar la URL si la instancia utiliza otro host, puerto o ruta. No subir al repositorio claves ni archivos con datos reales.

Resultado esperado: respuesta HTTP 200 con `estado: "Confirmado"` y un mensaje de confirmación. Verificar un evento de 10:00 a 11:00 en Google Calendar, una fila en la pestaña `Confirmado` y una fila en la tabla correspondiente de Baserow. La respuesta HTTP no garantiza por sí sola que los registros posteriores se hayan completado.

Si se repite la solicitud después de finalizar la primera ejecución, el resultado esperado es `Rechazado` por horario ocupado, sin un segundo evento. Verificar el calendario y los registros de rechazo.

### Consulta de disponibilidad

Utilizar el mismo procedimiento HTTP, sustituyendo el contenido de `solicitud.json` por:

```json
{
  "accion": "horas",
  "fecha_preferida": "AAAA-MM-DD",
  "servicio": "consulta medica"
}
```

Con una fecha válida, el resultado esperado contiene `estado`, `horas_disponibles`, `recomendadas`, `mensaje`, `verificado` e `ia`. Comparar las horas disponibles con los eventos del calendario y verificar que las recomendaciones pertenezcan a las horas libres. Esta consulta no debe crear eventos ni registros de reserva.

## Limitaciones conocidas

- Los tres servicios comparten un único calendario. Las citas duran una hora, con inicios permitidos a las 08:00, 09:00, 10:00, 11:00, 13:00, 14:00, 15:00, 16:00, 17:00 y 18:00.
- La consulta previa de disponibilidad reduce el riesgo de reservas repetidas secuenciales, pero no evita todas las solicitudes simultáneas.
- Los registros en Sheets y Baserow no son transaccionales; no hay reversión automática ante operaciones parciales.
- La versión final no incluye el manejador externo de errores. Los fallos técnicos se consultan en el historial de ejecuciones de n8n.
- La IA no decide la reserva ni realiza diagnósticos. La categoría y el indicador de revisión no se almacenan en Sheets ni Baserow.
- No se incluye cancelación, modificación, pagos ni envío de correos, mensajes de WhatsApp o Telegram.
- El formulario se utilizó localmente durante el desarrollo. Las solicitudes HTTP permiten probar el workflow sin depender del código del formulario.

Consultar la documentación y la bitácora para conocer el alcance de las pruebas y las verificaciones pendientes.

## Entregables del proyecto

- [Workflow final de n8n](./workflows/Workflow_Final.json)
- [Documentación del proyecto](./documentacion/Documentacion_Proyecto_Final.md)
- [Presentación final](./presentacion/Presentacion_Proyecto_Final.pdf)
- [Bitácora de pruebas](./pruebas/Bitacora_Clase_11_Pruebas.md)

### Diagramas del proceso

- [Proceso manual](./documentacion/diagramas/Proceso_Manual.png)
- [Proceso automatizado](./documentacion/diagramas/Proceso_Automatizado.png)

### Evidencias de pruebas

- [Consultar evidencias de ejecución](./pruebas/evidencias/)

