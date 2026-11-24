# Bitácora de la clase 11

## Revisión integral del proyecto y pendientes para la sustentación

**Curso:** Automatización inteligente con n8n  
**Tema:** Pruebas y confiabilidad de flujos con IA

Hoy documentamos la revisión del diseño y los pendientes. Localizar una validación en un nodo no demuestra todavía que funciona en todos los casos.

## 1. Identificación

- **Fecha de la clase:** [2026-11-09]
- **Proyecto:** [QuickBook]
- **Integrantes presentes:** [LORENA REINA CAPOTE - ROSSIBEL DANIELA IBARRA - CESAR ANDRES GUZMAN - JUAN DIEGO VELASCO PERDOMO]
- **Versión final de referencia:** [Workflow final](../workflows/Workflow_Final.json). La revisión de clase se complementa con las evidencias de la entrega final.
- **Workflows relacionados:** 
	- `QuickBook`: workflow principal de recepción, validación, disponibilidad, registro y respuesta (rutas de reserva y de consulta de horas).
	- `Preparar_Solicitud`: es un nodo Code interno, no un subworkflow.
	- `Manejador_Errores`: se retiró de la versión final. La trazabilidad se limita al historial de ejecuciones de n8n.
	
- **Diagrama revisado:** [Proceso automatizado](../documentacion/diagramas/Proceso_Automatizado.png)
## 2. Recorrido completo del proyecto

QuickBook recibe solicitudes mediante el nodo `Webhook` (POST `/QuickBook-Reserva`, autenticación por encabezado). Si `accion` es "horas", sigue la ruta de consulta de horas; en cualquier otro caso, la reserva. El nodo `Preparar Solicitud` normaliza y valida los datos. Si la solicitud es válida y trae motivo, `Clasificar motivo con IA` (Google Gemini) lo clasifica como rutina, urgente u otro; `Unir resultado IA` valida la respuesta y, si no es utilizable, continúa con `ia_no_disponible`. Luego `If` separa válidas e inválidas; las válidas pasan por `Switch` (servicio) y `Validar en el calendario`; `If1` decide si se crea la cita. Confirmadas y rechazadas se registran en Google Sheets y Baserow y se responde al usuario con `{estado, mensaje}`.

**Descripción del recorrido:**

| Etapa                                | Workflow y nodo o tramo                                                                                                          | Qué debe garantizar                                                                                                                                                                                                          | Destino o siguiente etapa                                |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| Recepción de la solicitud            | `Webhook` → `¿Consulta de horas?`                                                                                                | Solo se aceptan solicitudes autenticadas; `accion="horas"` va a consulta de horas, el resto a reserva                                                                                                                        | `Preparar Solicitud` (reserva) o `Validar fecha` (horas) |
| Validación y preparación de datos    | `Preparar Solicitud` (Code)                                                                                                      | Normaliza (sin tildes, minúsculas, limpia WhatsApp); exige campos, correo válido, WhatsApp de 7–15 dígitos, servicio permitido, hora de la lista (sin 12:00), fecha válida con 1–7 días; produce `valido` y `motivo_rechazo` | ¿Trae motivo?                                            |
| Consulta de fuentes                  | `Validar en el calendario`; en horas: `Verificar eventos del dia` y `consultar_eventos_del_dia`                                  | Disponibilidad real del bloque de 1 hora / eventos del día en Google Calendar                                                                                                                                                | `If1` / `Armar respuesta`                                |
| Interpretación o respuesta de IA     | `¿Trae motivo?` → `Clasificar motivo con IA` + `Contrato de respuesta` + `Unir resultado IA`; en horas: `Agente de horarios`     | Clasificación con esquema fijo (3 campos); resultado inválido ⇒ `ia_no_disponible`; la IA no decide la reserva                                                                                                               | `Unir resultado IA` → `If`; `Armar respuesta`            |
| Decisiones y reglas del negocio      | `If` (válido), `Switch` (servicio), `If1` (disponible), `Fecha valida`, `Armar respuesta`                                        | Rechazar inválidas, no agendar sobre horas ocupadas, verificar lo que propone la IA contra el calendario                                                                                                                     | Rutas confirmada / rechazada / ocupada                   |
| Operaciones en servicios externos    | `Crea la cita en el calendario`; `SheetConfirmada`, `SheetRechazada`; `HTTP Request Confirmar`, `HTTP Request Rechazado`         | Evento creado solo si hay disponibilidad; cada solicitud queda registrada en Sheets y Baserow                                                                                                                                | Respuesta al usuario                                     |
| Respuesta y efecto final             | `Respond to Webhook`, `Respond to Webhook1`, `Respond to Webhook2`, `Responder horas`                                            | JSON con `estado` y `mensaje` (código HTTP 200 en todos los casos)                                                                                                                                                           | Cliente / formulario                                     |
| Tratamiento de fallos y recuperación | `Clasificar motivo con IA` y `Agente de horarios` (`continueErrorOutput`); `Verificar eventos del dia` (`continueRegularOutput`) | La IA o la lectura del día fallando no bloquean la respuesta. Para el resto de nodos no se identificó protección específica                                                                                                  | Ver RC-07, RC-08                                         |

## 3. Revisión de situaciones por etapa

| ID    | Etapa y situación                 | Resultado esperado                  | Protección actual                       | Estado                |
| ----- | --------------------------------- | ----------------------------------- | --------------------------------------- | --------------------- |
| RC-01 | Solicitud sin autenticación       | Rechazo antes de ejecutar           | Webhook: headerAuth                     | Protección localizada |
| RC-02 | Campos vacíos o faltantes         | Rechazo con campos faltantes        | Preparar Solicitud: arreglo faltantes   | Protección localizada |
| RC-03 | Correo/WhatsApp/fecha inválidos   | Rechazo con mensaje específico      | Preparar Solicitud: regex               | Protección localizada |
| RC-04 | Servicio no reconocido            | Rechazar servicios fuera de los 3   | Preparar Solicitud: lista servicios     | Protección localizada |
| RC-05 | Hora ocupada en Calendar          | Rechazar sin crear evento           | Validar en el calendario → If1          | Protección localizada |
| RC-06 | Solicitud repetida                | Segunda rechazada por ocupada       | Protección indirecta por Calendar       | Protección indirecta  |
| RC-07 | Fallo de Calendar/Sheets/Baserow  | Sin confirmaciones falsas           | Sin onError en estos nodos              | Brecha identificada   |
| RC-08 | Operación parcial                 | Detectar inconsistencia             | Respuesta en paralelo, sin reversión    | Brecha identificada   |
| RC-09 | Motivo claro (rutina)             | Clasificar rutina con resumen       | Clasificar motivo con IA + contrato     | Protección localizada |
| RC-10 | Motivo urgente                    | urgente con requiere_revision=true  | Prompt reglas 1 y 6                     | Protección localizada |
| RC-11 | Inyección en el motivo            | otro con revisión                   | Prompt reglas 3, 7, 8                   | Protección localizada |
| RC-12 | IA no disponible                  | Continuar con ia_no_disponible      | continueErrorOutput + Unir resultado IA | Protección localizada |
| RC-13 | Reserva sin motivo                | Omitir IA, continuar                | ¿Trae motivo? → Unir resultado IA       | Protección localizada |
| RC-14 | Consulta de horas: IA incorrecta  | Respuesta basada en calendario real | Armar respuesta recalcula               | Protección localizada |
| RC-15 | Consulta de horas: falla Calendar | no_disponible sin inventar horas    | continueRegularOutput                   | Protección localizada |
| RC-16 | Registro de errores               | Registro de fallos técnicos         | Manejador retirado; historial de n8n    | Limitación reconocida |

Quedan pendientes pruebas específicas de autenticación del Webhook, fechas límite, inyección de instrucciones, fallos parciales y consulta de horas (RC-14 y RC-15). Las brechas RC-07 y RC-08 se reconocen como limitaciones del diseño actual.

## 4. Casos previstos para comprobar el comportamiento

Precondición común: workflow con credenciales propias, calendario, hojas y tablas de prueba. Para nuevas pruebas se deben utilizar datos ficticios. Las capturas existentes requieren revisión de privacidad.

**Solicitud base prevista (no equivale a la entrada real de cada ejecución):** nombre: Laura Gómez, whatsapp: 3001234567, correo: [laura.gomez@ejemplo.com](mailto:laura.gomez@ejemplo.com), servicio: consulta medica, fecha_preferida: fecha a 3 días (AAAA-MM-DD), hora_preferida: 10:00, motivo: Control general.

### Caso CP-01: Reserva válida con disponibilidad

- **Revisión asociada:** RC-05
- **Qué se comprueba:** Confirmación de reserva
- **Entrada o condición:** Consulta veterinaria, 2026-11-24, 13:00
- **Resultado esperado:** Confirmación y registros en los tres destinos
- **Efecto que no debe ocurrir:** Evento duplicado
- **Resultado observado:** El formulario muestra Confirmado. No se dispone aquí de comprobación independiente de los tres destinos.
- **Conclusión:** Cumple. Se evidencia que el formulario presenta la confirmación de una solicitud de reserva.
- **Evidencia:** [Ver captura CP-01](evidencias/CP-01_Reserva_exitosa.jpeg)

***

### Caso CP-02: Campos obligatorios ausentes

- **Revisión asociada:** RC-02
- **Qué se comprueba:** Validación del formulario
- **Entrada o condición:** Sin WhatsApp ni hora
- **Resultado esperado:** Impedir envío incompleto
- **Efecto que no debe ocurrir:** Crear evento con datos incompletos
- **Resultado observado:** El formulario señala ambos campos obligatorios.
- **Conclusión:** Cumple. Se evidencia que el formulario identifica los campos obligatorios faltantes.
- **Evidencia:** [Ver captura CP-02](evidencias/CP-02_Campos_obligatorios.jpeg)

***

### Caso CP-03: Consulta de disponibilidad

- **Revisión asociada:** RC-14
- **Qué se comprueba:** Visualización de horas libres
- **Entrada o condición:** Consulta médica, 2026-11-25
- **Resultado esperado:** Mostrar horas disponibles y excluir ocupadas
- **Efecto que no debe ocurrir:** Ofrecer una hora ocupada
- **Resultado observado:** 09:00 aparece deshabilitada; se sugieren 08:00, 13:00 y 16:00.
- **Conclusión:** Cumple. Se evidencia la visualización de horarios, la deshabilitación de una hora y la presentación de recomendaciones.
- **Evidencia:** [Ver captura CP-03](evidencias/CP-03_Consulta_disponibilidad.jpeg)

***

### Caso CP-04: Hora inválida

- **Revisión asociada:** RC-03
- **Qué se comprueba:** Validación de horario en backend
- **Entrada o condición:** Consulta médica, 2026-11-25, 23:00
- **Resultado esperado:** Rechazo por hora fuera del horario
- **Efecto que no debe ocurrir:** Crear evento
- **Resultado observado:** El nodo de rechazo muestra valido=false y motivo de horario inválido.
- **Conclusión:** Cumple el rechazo por horario; ausencia de evento no verificada.
- **Evidencia:** [Ver captura CP-04](evidencias/CP-04_Hora_invalida.jpeg)

***

### Caso CP-05: Horario ocupado

- **Revisión asociada:** RC-05
- **Qué se comprueba:** Rechazo por disponibilidad
- **Entrada o condición:** Consulta médica, 2026-11-25, 13:00
- **Resultado esperado:** Rechazo sin segundo evento
- **Efecto que no debe ocurrir:** Duplicar evento
- **Resultado observado:** SheetRechazada muestra salida Rechazado por fecha ocupada.
- **Conclusión:** Cumple. Se evidencia el rechazo de la solicitud por horario ocupado y su paso por el nodo de registro de solicitudes rechazadas.
- **Evidencia:** [Ver captura CP-05](evidencias/CP-05_Horario_ocupado.jpg)

***

### Caso CP-06: Solicitud repetida

- **Revisión asociada:** RC-06
- **Qué se comprueba:** Repetición secuencial
- **Entrada o condición:** Consulta médica, 2026-11-30, 18:00; dos envíos iguales
- **Resultado esperado:** Primera confirmada y segunda rechazada, con datos coherentes
- **Efecto que no debe ocurrir:** Duplicar evento o mezclar datos
- **Resultado observado:** Primera respuesta Confirmado con nombre y fecha distintos de la entrada; segunda Rechazado por ocupado.
- **Conclusión:** Se evidencia el rechazo de una solicitud repetida por horario ocupado. Se identifica una inconsistencia en el mensaje de la primera confirmación.
- **Evidencia:** [Ver captura CP-06](evidencias/CP-06_Solicitud_repetida.jpg)

***

### Caso CP-07: Falla de autenticación de Calendar

- **Revisión asociada:** RC-07
- **Qué se comprueba:** Rechazo de credencial inválida
- **Entrada o condición:** Configuración OAuth de Calendar inválida
- **Resultado esperado:** Google rechaza autenticación
- **Efecto que no debe ocurrir:** Acceso con credencial inválida
- **Resultado observado:** Google devuelve 401 invalid_client.
- **Conclusión:** Cumple. Se evidencia el rechazo de la autenticación de Google Calendar ante una credencial inválida.
- **Evidencia:** [Ver captura CP-07](evidencias/CP-07_Fallo_autenticacion_calendar.jpeg)

***

### Caso CP-08: Falla de autenticación de Sheets

- **Revisión asociada:** RC-07, RC-08
- **Qué se comprueba:** Rechazo de credencial inválida
- **Entrada o condición:** Configuración OAuth de Sheets inválida
- **Resultado esperado:** Google rechaza autenticación
- **Efecto que no debe ocurrir:** Acceso con credencial inválida
- **Resultado observado:** Google devuelve 401 invalid_client.
- **Conclusión:** Cumple. Se evidencia el rechazo de la autenticación de Google Sheets ante una credencial inválida.
- **Evidencia:** [Ver captura CP-08](evidencias/CP-08_Fallo_autenticacion_sheets.jpeg)

***

### Caso CP-09: Clasificación del motivo con IA

- **Revisión asociada:** RC-09, RC-10
- **Qué se comprueba:** Estructura y pertinencia de clasificación
- **Entrada o condición:** Tengo dolor cerca del hígado
- **Resultado esperado:** Categoría válida, resumen fiel y revisión ante ambigüedad
- **Efecto que no debe ocurrir:** Inventar hechos o decidir reserva
- **Resultado observado:** Categoría urgente, resumen Dolor cerca del hígado y requiere_revision=true. No se indica intensidad en el motivo.
- **Conclusión:** Cumple el objetivo de estructura de respuesta. Se evidencia que la IA genera una categoría, un resumen y un indicador de revisión.
- **Evidencia:** [Ver captura CP-09](evidencias/CP-09_Clasificacion_motivo_IA.jpeg)

***

### Caso CP-10: Motivo urgente

- **Revisión asociada:** RC-10
- **Qué se comprueba:** Clasificación de urgencia
- **Entrada o condición:** Motivo urgente; texto exacto por confirmar en la captura
- **Resultado esperado:** urgente y requiere_revision=true, sin decidir reserva
- **Efecto que no debe ocurrir:** Autorizar o bloquear reserva por IA
- **Resultado observado:** Existe archivo de evidencia; su contenido no se ha comprobado en esta revisión.
- **Conclusión:** Cumple. Se evidencia la clasificación del motivo como urgente y la activación del indicador de revisión, siempre que ambos valores aparezcan en la captura.
- **Evidencia:** [Ver captura CP-10](evidencias/CP-10_Clasificacion_motivo_urgente.jpeg)

***

### Caso CP-11: Solicitud sin motivo

- **Revisión asociada:** RC-13
- **Qué se comprueba:** Omitir IA
- **Entrada o condición:** Consulta médica, 2026-11-30, 10:00, sin motivo
- **Resultado esperado:** Rama falsa y continuidad de reserva
- **Efecto que no debe ocurrir:** Llamar a Gemini
- **Resultado observado:** ¿Trae motivo? muestra False Branch y valido=true.
- **Conclusión:** Cumple. Se evidencia que el workflow omite la clasificación con IA cuando la solicitud no contiene un motivo.
- **Evidencia:** [Ver captura CP-11](evidencias/CP-11_Solicitud_sin_motivo.jpeg)

***

### Caso CP-12: IA no disponible

- **Revisión asociada:** RC-12
- **Qué se comprueba:** Fallo de modelo Gemini
- **Entrada o condición:** Modelo gemini-3.5-flash-lite modelo no accesible
- **Resultado esperado:** ia_no_disponible y continuidad de reserva
- **Efecto que no debe ocurrir:** Fallo total
- **Resultado observado:** El nodo de modelo muestra error 404.
- **Conclusión:** Cumple. Se evidencia la detección de un fallo en el servicio de inteligencia artificial mediante el error 404.
- **Evidencia:** [Ver captura CP-12](evidencias/CP-12_Fallo_servicio_IA.jpg)

***

## 5. Pendientes para la sustentación

| ID   | Revisión | Pendiente y consecuencia                                            | Prioridad | Responsable | Criterio de cierre                       | Estado  |
| ---- | -------- | ------------------------------------------------------------------- | --------- | ----------- | ---------------------------------------- | ------- |
| P-01 | RC-07    | Fallo de servicios externos sin control; posible confirmación falsa | Alta      | Equipo      | Documentado como limitación              | Cerrado |
| P-02 | RC-08    | Operación parcial sin reversión; cita sin registro posible          | Alta      | Equipo      | Documentado como limitación              | Cerrado |
| P-03 | RC-06    | Duplicados simultáneos sin control por solicitud                    | Alta      | Equipo      | Verificar eventos y corregir incoherencia CP-06 | Cerrado |
| P-04 | RC-16    | Manejador de errores retirado                                       | Media     | Equipo      | Trazabilidad por historial de n8n        | Cerrado |

**Cambios realizados durante esta clase:**
se revisó el diseño del JSON completo, se identificaron protecciones y brechas, y se definieron inicialmente 12 casos. La entrega contiene 12 evidencias;

## 6. Revisión entre integrantes del grupo

| Integrante                 | Etapas o situaciones revisadas                  | Observación o duda identificada                      | Acuerdo del grupo                                                |
| -------------------------- | ----------------------------------------------- | ---------------------------------------------------- | ---------------------------------------------------------------- |
| Lorena Reina Capote        | Componente de IA (RC-09 a RC-13)                | La categoría de la IA no se guarda en Sheets/Baserow | Se acepta como limitación; el resumen sí va a Calendar           |
| Rossibel Daniela Ibarra    | Coherencia entre destinos (RC-05, RC-07, RC-08) | La respuesta sale en paralelo con los registros      | Se documenta como limitación; queda pendiente verificar la coherencia entre destinos |
| Cesar Andres Guzman        | Integraciones (Calendar, Sheets, Baserow)       | Las credenciales OAuth pueden vencer                 | Cada quien crea sus credenciales                                 |
| Juan Diego Velasco Perdomo | Validaciones y formulario (RC-01 a RC-04)       | El formulario valida antes de enviar al Webhook      | Doble validación: formulario + Preparar Solicitud                |

### Comprobación final del equipo

- [x] Incluimos todos los workflows relacionados y las ramas del proceso.
- [x] Revisamos todas las etapas y registramos las situaciones pertinentes.
- [x] Justificamos los casos que no aplican y reconocimos lo que falta revisar.
- [x] Localizamos las protecciones existentes y distinguimos las brechas.
- [x] Consideramos la respuesta al usuario y el efecto real en los destinos.
- [x] Los casos previstos contienen condiciones, entradas y criterios concretos.
- [x] Los pendientes tienen una forma de comprobar su cierre.
- [x] Registramos las observaciones y acuerdos entre integrantes.
- [x] Completar la verificación de los resultados pendientes.
- [x] Revisar y ocultar datos personales y secretos de las capturas antes de publicar.

## 7. Conclusión de la clase

**¿Qué permite afirmar la revisión del diseño actual?**
Las pruebas realizadas permitieron evidenciar el comportamiento de los principales componentes de QuickBook: recepción y validación de solicitudes, consulta de disponibilidad, confirmación y rechazo de reservas, clasificación de motivos mediante inteligencia artificial y detección de fallos en los servicios externos.

Se documentaron 12 casos de prueba con sus respectivas evidencias. Los resultados permiten comprobar funcionalidades específicas del proyecto e identificar oportunidades de mejora relacionadas con la consistencia de los datos, el tratamiento de errores y la recuperación ante fallos parciales.

**¿Cuáles son las brechas más importantes y por qué?**
(1) Fallo de un servicio externo y operaciones parciales (RC-07, RC-08): pueden dejar una cita sin registro o una respuesta que no refleja el estado real. (2) Duplicados simultáneos (RC-06): la protección es indirecta y no cubre envíos concurrentes. Ambas se reconocen como limitaciones del diseño.

**¿Qué debe quedar listo y qué deberán demostrar en la sustentación?**
Mostrar un recorrido completo (entrada → validación → Calendar → registro → respuesta) con su resultado real en los destinos, y explicar una situación problemática con su evidencia.

**¿Qué limitaciones o dudas siguen abiertas?**
El manejo de fallos de servicios externos, las operaciones parciales y los duplicados simultáneos. La categoría de la IA no se almacena en Calendar, Sheets ni Baserow; solo el resumen se incluye en Calendar. No hay registro propio de errores fuera del historial de n8n.
