# QuickBook – Sistema automatizado de reservas

## Descripción del proyecto

QuickBook es un sistema de automatización de reservas desarrollado con n8n, que permite recibir, validar y gestionar solicitudes de citas para diferentes servicios.

La solución consulta la disponibilidad en Google Calendar, aplica las reglas del negocio, registra los resultados en Google Sheets y Baserow y devuelve al usuario una respuesta de confirmación o rechazo.

El proyecto incorpora inteligencia artificial mediante Google Gemini para clasificar el motivo de la cita, generar un resumen e identificar solicitudes que requieren revisión.

## Información académica

- **Institución:** EAM
- **Asignatura:** Electiva IV
- **Periodo académico:** 2026-II
- **Profesor:** Robert García Rey
- **Estudiantes:** Rossibel Ibarra - Cesar Guzman - Lorena Reina - Juan Diego Velasco

## Problema

La gestión manual de reservas requiere revisar la disponibilidad de los horarios, registrar los datos de los clientes y comunicar el resultado de cada solicitud. Este proceso puede generar errores de registro, reservas duplicadas, demoras en la atención y dificultades para consultar el historial de solicitudes.

QuickBook propone automatizar estas actividades mediante la integración de diferentes servicios, manteniendo reglas de validación que permitan gestionar las reservas de manera organizada.

## Objetivo

Desarrollar un sistema automatizado de gestión de reservas que permita recibir solicitudes, validar los datos, consultar la disponibilidad, crear citas y registrar los resultados, incorporando inteligencia artificial para clasificar el motivo de la solicitud.

## Alcance

QuickBook contempla la gestión de reservas para los siguientes servicios:

- Consulta médica.
- Consulta veterinaria.
- Peluquería.

El sistema permite:

- Recibir solicitudes mediante un Webhook.
- Validar los datos obligatorios de la reserva.
- Gestionar solicitudes con una anticipación de entre uno y siete días.
- Consultar la disponibilidad de horarios.
- Crear citas de una hora de duración en Google Calendar.
- Clasificar el motivo de la solicitud mediante inteligencia artificial.
- Registrar solicitudes confirmadas y rechazadas.
- Informar al usuario el resultado de su solicitud.

La inteligencia artificial cumple una función de clasificación y apoyo. No realiza diagnósticos médicos ni tiene autorización para confirmar, modificar o cancelar citas de manera autónoma.

## Funcionamiento general

QuickBook automatiza el proceso de gestión de reservas mediante un workflow desarrollado en n8n.

El proceso comienza cuando el usuario envía una solicitud con sus datos personales, el servicio requerido, la fecha y la hora de preferencia.

El workflow realiza las siguientes actividades:

1. Recibe la solicitud mediante un Webhook.
2. Prepara los datos para su procesamiento.
3. Clasifica el motivo de la solicitud mediante Google Gemini, cuando el usuario proporciona esta información.
4. Valida los campos obligatorios y el rango de fechas permitido.
5. Identifica el servicio solicitado.
6. Consulta la disponibilidad del horario en Google Calendar.
7. Si el horario está disponible, crea la cita. En caso contrario, genera una respuesta de rechazo.
8. Registra el resultado en Google Sheets y Baserow.
9. Devuelve al usuario una respuesta con el estado de la solicitud.

### Integración de inteligencia artificial

Google Gemini se utiliza para analizar el motivo de la reserva y generar tres elementos:

- Categoría del motivo: rutina, urgente u otro.
- Resumen breve del motivo.
- Indicador de si la solicitud requiere revisión.

La clasificación se realiza únicamente cuando el usuario proporciona un motivo. Si el servicio de inteligencia artificial falla o devuelve una respuesta inválida, el workflow continúa con el procesamiento de la reserva y marca el motivo como no clasificado.

Las decisiones relacionadas con la disponibilidad y la confirmación de citas se realizan mediante reglas del negocio, independientemente del resultado de la inteligencia artificial.

## Tecnologías utilizadas

| Tecnología | Función dentro del proyecto |
|---|---|
| n8n | Automatización y coordinación del proceso de reservas. |
| Google Calendar | Consulta de disponibilidad y creación de citas. |
| Google Sheets | Registro de solicitudes confirmadas y rechazadas. |
| Baserow | Almacenamiento estructurado de las solicitudes. |
| Google Gemini | Clasificación y resumen del motivo de la reserva. |
| Webhook HTTP | Recepción de solicitudes y devolución de respuestas. |
| GitHub | Control de versiones y publicación de los entregables del proyecto. |

## Requisitos para ejecutar el proyecto

Para configurar y ejecutar QuickBook se requiere:

- n8n versión 2.35.7, utilizada durante el desarrollo del proyecto.
- Una cuenta de Google con acceso a Google Calendar y Google Sheets.
- Una cuenta de Baserow con permisos para crear tablas y registrar información.
- Una clave de API de Google Gemini para la clasificación de los motivos de reserva.
- Credenciales configuradas en n8n para cada uno de los servicios externos.
- Un cliente HTTP, como Postman, para enviar solicitudes al Webhook y verificar las respuestas.

### Recursos externos necesarios

| Servicio | Recursos requeridos |
|---|---|
| Google Calendar | Un calendario destinado a la gestión de citas. |
| Google Sheets | Una hoja de cálculo con pestañas para solicitudes confirmadas, rechazadas y registro de errores. |
| Baserow | Una base de datos con tablas para solicitudes confirmadas y rechazadas. |
| Google Gemini | Una clave de API y acceso al modelo de inteligencia artificial configurado en el workflow. |

Las credenciales deben configurarse en la instancia de n8n de cada usuario. Por seguridad, el repositorio no incluye claves de API, contraseñas ni tokens de acceso.

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

Las citas tienen una duración de una hora. Ambos nodos deben utilizar el mismo calendario para garantizar que la consulta de disponibilidad corresponda al calendario donde se crean las reservas.

### 2. Google Sheets

Google Sheets se utiliza para registrar las solicitudes procesadas por QuickBook y conservar el historial de reservas.

#### Configuración

1. Crear una hoja de cálculo en Google Sheets.
2. Crear las siguientes pestañas:

| Pestaña | Propósito |
|---|---|
| Confirmado | Registrar las solicitudes de reserva confirmadas. |
| Rechazado | Registrar las solicitudes de reserva rechazadas. |
| Errores | Registrar los errores técnicos durante la ejecución. |

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

#### Estructura de la pestaña Errores

| Columna | Descripción |
|---|---|
| fecha_registro | Fecha y hora del error. |
| workflow | Nombre del workflow. |
| nodo_error | Nodo donde ocurrió el error. |
| mensaje_error | Descripción del error. |
| url_error | Enlace a la ejecución. |
| id_ejecucion | Identificador de la ejecución. |

#### Configuración en n8n

1. Crear una credencial de tipo Google Sheets OAuth2 API.
2. Autorizar el acceso a Google Sheets.
3. Seleccionar la hoja de cálculo creada en los nodos correspondientes.
4. Configurar el nodo `SheetConfirmada` para utilizar la pestaña `Confirmado`.
5. Configurar el nodo `SheetRechazada` para utilizar la pestaña `Rechazado`.
6. Verificar que el mapeo de los campos coincida con los encabezados de cada pestaña.

Cada usuario debe utilizar su propia hoja de cálculo y sus propias credenciales. No es necesario tener acceso a los recursos originales del equipo de desarrollo.

