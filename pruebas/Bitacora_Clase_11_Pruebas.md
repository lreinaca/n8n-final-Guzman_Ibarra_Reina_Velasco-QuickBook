# Bitácora de la clase 11

## Revisión integral del proyecto y pendientes para la sustentación

**Curso:** Automatización inteligente con n8n  
**Tema:** Pruebas y confiabilidad de flujos con IA

Hoy documentamos la revisión del diseño y los pendientes. Localizar una validación en un nodo no demuestra todavía que funciona en todos los casos.

## 1. Identificación

- **Fecha de la clase:** [2026-11-09]
- **Proyecto:** [QuickBook]
- **Integrantes presentes:** [LORENA REINA CAPOTE - ROSSIBEL DANIELA IBARRA - CESAR ANDRES GUZMAN - JUAN DIEGO VELASCO PERDOMO]
- **Versión revisada:** [[Guzman_Velasco_Reina_Ibarra/02_Workflows/QuickBook (7).json|QuickBook (10)]] - workflow único, 33 nodos, activo
- **Workflows relacionados:** 
	- `QuickBook`: workflow principal de recepción, validación, disponibilidad, registro y respuesta (rutas de reserva y de consulta de horas).
	- `Preparar_Solicitud`: es un nodo Code interno, no un subworkflow.
	- `Manejador_Errores`: se retiró de la versión final. La trazabilidad se limita al historial de ejecuciones de n8n.
	
- **Diagrama revisado:** documentacion/diagramas/Proceso_Automatizado.png
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

Etapas o situaciones que faltan por revisar: ninguna adicional. Las brechas RC-07 y RC-08 se reconocen como limitaciones del diseño actual.

## 4. Casos previstos para comprobar el comportamiento

Precondición común: workflow con credenciales propias, calendario, hojas y tablas de prueba. Todos los datos son ficticios.

**Solicitud base:** nombre: Laura Gómez, whatsapp: 3001234567, correo: [laura.gomez@ejemplo.com](mailto:laura.gomez@ejemplo.com), servicio: consulta medica, fecha_preferida: fecha a 3 días (AAAA-MM-DD), hora_preferida: 10:00, motivo: Control general.

### Caso CP-01: Reserva válida con disponibilidad

- **Revisión asociada:** RC-09
- **Qué se comprueba:** Reserva normal con motivo, hora libre en Calendar
- **Entrada:** Solicitud base completa
- **Resultado esperado:** Respuesta `Confirmado`, evento creado 10:00–11:00, fila en Sheets (Confirmado) y Baserow
- **Efecto que no debe ocurrir:** Evento duplicado; fila en "Rechazado"
- **Resultado observado:** Confirmado. Evento creado, fila registrada en ambos destinos
- **Conclusión:** Cumple
- **Evidencia:** Captura de ejecución en n8n y formulario con confirmación

***
### Caso CP-02: Campos obligatorios ausentes

- **Revisión asociada:** RC-02
- **Qué se comprueba:** Solicitud sin correo
- **Entrada:** Solicitud base sin `correo`
- **Resultado esperado:** Rechazado con "Faltan campos obligatorios: correo"; sin evento
- **Efecto que no debe ocurrir:** Evento creado; llamada a IA
- **Resultado observado:** Rechazado con el campo faltante indicado
- **Conclusión:** Cumple
- **Evidencia:** Captura de respuesta del formulario

***
### Caso CP-03: Fecha inválida

- **Revisión asociada:** RC-03
- **Qué se comprueba:** Fecha de hoy y fecha anterior
- **Entrada:** Solicitud base con `fecha_preferida` = fecha de hoy, y otra con fecha pasada
- **Resultado esperado:** Rechazado por la regla de 1 a 7 días; sin evento
- **Efecto que no debe ocurrir:** Evento creado
- **Resultado observado:** Rechazado en ambos casos
- **Conclusión:** Cumple
- **Evidencia:** Capturas de respuesta

***
### Caso CP-04: Hora o servicio inválido

- **Revisión asociada:** RC-03, RC-04
- **Qué se comprueba:** hora_preferida = 12:00 y servicio = Odontología
- **Entrada:** Solicitud base con hora 12:00; otra con servicio Odontología
- **Resultado esperado:** Rechazado por horario o servicio no reconocido
- **Efecto que no debe ocurrir:** Evento creado
- **Resultado observado:** Rechazado en ambos casos con el motivo correspondiente
- **Conclusión:** Cumple
- **Evidencia:** Capturas de respuesta

***
### Caso CP-05: Horario ocupado

- **Revisión asociada:** RC-05
- **Qué se comprueba:** Solicitud para hora que ya tiene cita
- **Entrada:** Solicitud base para una hora ya reservada
- **Resultado esperado:** Rechazado por fecha ocupada; sin segundo evento
- **Efecto que no debe ocurrir:** Dos eventos a la misma hora
- **Resultado observado:** Rechazado por ocupada
- **Conclusión:** Cumple
- **Evidencia:** Calendar antes/después, respuesta del formulario

***
### Caso CP-06: Solicitud repetida (duplicado)

- **Revisión asociada:** RC-06
- **Qué se comprueba:** Misma solicitud enviada dos veces seguidas
- **Entrada:** Solicitud base dos veces con la misma hora
- **Resultado esperado:** Primera confirmada, segunda rechazada por ocupada
- **Efecto que no debe ocurrir:** Dos eventos
- **Resultado observado:** Primera confirmada, segunda rechazada
- **Conclusión:** Cumple (protección indirecta funciona en envíos secuenciales; duplicados simultáneos no se probaron)
- **Evidencia:** Dos respuestas del formulario, Calendar

***
### Caso CP-07: Falla de Calendar

- **Revisión asociada:** RC-07
- **Qué se comprueba:** Qué pasa cuando Calendar no está disponible
- **Entrada:** Solicitud base con credencial de Calendar inválida
- **Resultado esperado:** No se confirma una cita inexistente
- **Efecto que no debe ocurrir:** Confirmación falsa
- **Resultado observado:** La ejecución falla en el nodo de Calendar; n8n registra el error en su historial
- **Conclusión:** No hay control explícito; se declara como limitación
- **Evidencia:** Historial de ejecuciones de n8n

***
### Caso CP-08: Falla de almacenamiento después de crear la cita

- **Revisión asociada:** RC-08
- **Qué se comprueba:** Evento creado pero Sheets o Baserow falla
- **Entrada:** Solicitud base con ID de tabla Baserow inexistente
- **Resultado esperado:** Documentar el estado de cada destino
- **Efecto que no debe ocurrir:** Dar por cerrado sin documentar la inconsistencia
- **Resultado observado:** Evento creado en Calendar, fila en Sheets, Baserow falla. La respuesta al usuario dice Confirmado porque sale en paralelo
- **Conclusión:** Operación parcial confirmada como limitación; sin reversión automática
- **Evidencia:** Calendar, Sheets, error en n8n para Baserow

***
### Caso CP-09: Motivo de rutina

- **Revisión asociada:** RC-09
- **Qué se comprueba:** Clasificación de un motivo claro
- **Entrada:** Solicitud base con motivo "Control de rutina anual"
- **Resultado esperado:** Categoría `rutina`, resumen breve, `requiere_revision = false`
- **Efecto que no debe ocurrir:** Que la IA decida la reserva
- **Resultado observado:** Categoría `rutina`, resumen coherente
- **Conclusión:** Cumple
- **Evidencia:** Salida del nodo `Unir resultado IA`

***
### Caso CP-10: Motivo urgente

- **Revisión asociada:** RC-10
- **Qué se comprueba:** Clasificación de un motivo urgente
- **Entrada:** Solicitud base con motivo "Tengo dolor cerca del hígado"
- **Resultado esperado:** Categoría `urgente`, `requiere_revision = true`; la reserva sigue reglas normales
- **Efecto que no debe ocurrir:** Que la IA bloquee o autorice la reserva
- **Resultado observado:** Categoría `urgente`, resumen "Dolor cerca del hígado", `requiere_revision = true`, `motivo_estado = clasificado`
- **Conclusión:** Cumple
- **Evidencia:** Captura del nodo `Clasificar motivo con IA` (output) y `Unir resultado IA`

***
### Caso CP-11: Solicitud sin motivo

- **Revisión asociada:** RC-13
- **Qué se comprueba:** Que se omita la IA cuando no hay motivo
- **Entrada:** Solicitud base sin campo `motivo`
- **Resultado esperado:** Se omite la clasificación, el flujo continúa
- **Efecto que no debe ocurrir:** Llamada a Gemini
- **Resultado observado:** Nodo de IA no ejecutado; reserva confirmada
- **Conclusión:** Cumple
- **Evidencia:** Ejecución en n8n mostrando que el nodo de IA no se activó

***
### Caso CP-12: IA no disponible

- **Revisión asociada:** RC-12
- **Qué se comprueba:** Que la reserva continúe si Gemini falla
- **Entrada:** Solicitud base con credencial de Gemini inválida
- **Resultado esperado:** `motivo_estado = ia_no_disponible`; la reserva se confirma si hay disponibilidad
- **Efecto que no debe ocurrir:** Fallo total de la ejecución
- **Resultado observado:** Reserva confirmada con `ia_no_disponible`
- **Conclusión:** Cumple
- **Evidencia:** Ejecución en n8n, evento en Calendar

***
### Caso CP-13: Registro de confirmación en Sheets y Baserow

- **Revisión asociada:** RC-05 (coherencia)
- **Qué se comprueba:** Que los campos y el estado coincidan entre Calendar, Sheets y Baserow
- **Entrada:** Solicitud base con hora libre
- **Resultado esperado:** Campos coherentes en los tres destinos
- **Efecto que no debe ocurrir:** Datos distintos entre destinos
- **Resultado observado:** Nombre, fecha, hora, servicio y estado coinciden en Calendar, Sheets y Baserow
- **Conclusión:** Cumple
- **Evidencia:** Capturas de Calendar, Sheets y Baserow

***
### Caso CP-14: Registro de rechazo

- **Revisión asociada:** RC-02 (coherencia)
- **Qué se comprueba:** Que un rechazo se registre correctamente sin crear evento
- **Entrada:** Solicitud base con correo inválido
- **Resultado esperado:** Datos correctos en Sheets y Baserow (Rechazado); sin evento nuevo
- **Efecto que no debe ocurrir:** Evento creado
- **Resultado observado:** Fila en Rechazado con el motivo correcto; Calendar sin cambios
- **Conclusión:** Cumple
- **Evidencia:** Capturas de Sheets y Calendar

***
### Caso CP-15: Respuesta al usuario frente a efectos reales

- **Revisión asociada:** RC-05, RC-08 (coherencia)
- **Qué se comprueba:** Que la respuesta JSON coincida con lo que ocurrió en cada sistema
- **Entrada:** Una solicitud confirmada y una rechazada
- **Resultado esperado:** La respuesta coincide con el estado real en Calendar, Sheets y Baserow
- **Efecto que no debe ocurrir:** Respuesta "Confirmado" sin evento creado
- **Resultado observado:** Confirmación con evento y registros; rechazo sin evento y con registros en Rechazado
- **Conclusión:** Cumple
- **Evidencia:** Respuestas del formulario + estado de cada destino

***
### Caso CP-16: Fallo técnico y registro de errores

- **Revisión asociada:** RC-16
- **Qué se comprueba:** Mecanismo de registro de errores de la versión final
- **Entrada:** Solicitud que provoca un fallo técnico en un nodo
- **Resultado esperado:** Documentar el mecanismo disponible
- **Efecto que no debe ocurrir:** Error sin registro alguno
- **Resultado observado:** n8n conserva la ejecución fallida en su historial con el nodo, el mensaje de error y la fecha. No hay workflow de manejo de errores externo
- **Conclusión:** Funcional pero limitado; se reconoce como limitación
- **Evidencia:** Historial de ejecuciones de n8n

***

## 5. Pendientes para la sustentación

| ID | Revisión | Pendiente y consecuencia | Prioridad | Responsable | Criterio de cierre | Estado |
|---|---|---|---|---|---|---|
| P-01 | RC-07 | Fallo de servicios externos sin control; posible confirmación falsa | Alta | Equipo | Documentado como limitación | Cerrado |
| P-02 | RC-08 | Operación parcial sin reversión; cita sin registro posible | Alta | Equipo | Documentado como limitación | Cerrado |
| P-03 | RC-06 | Duplicados simultáneos sin control por solicitud | Alta | Equipo | Protección indirecta verificada en CP-06 | Cerrado |
| P-04 | RC-16 | Manejador de errores retirado | Media | Equipo | Trazabilidad por historial de n8n | Cerrado |

**Cambios realizados durante esta clase:**
se revisó el diseño del JSON completo, se identificaron protecciones y brechas, y se definieron los 16 casos de prueba con entradas y criterios concretos.

## 6. Revisión entre integrantes del grupo

| Integrante                 | Etapas o situaciones revisadas                  | Observación o duda identificada                      | Acuerdo del grupo                                                |
| -------------------------- | ----------------------------------------------- | ---------------------------------------------------- | ---------------------------------------------------------------- |
| Lorena Reina Capote        | Componente de IA (RC-09 a RC-13)                | La categoría de la IA no se guarda en Sheets/Baserow | Se acepta como limitación; el resumen sí va a Calendar           |
| Rossibel Daniela Ibarra    | Coherencia entre destinos (RC-05, RC-07, RC-08) | La respuesta sale en paralelo con los registros      | Se documenta como limitación; CP-13 a CP-15 verifican coherencia |
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
- [x] Los resultados de las pruebas corresponden a ejecuciones reales.
- [x] Las capturas y los ejemplos no exponen contraseñas, tokens ni datos personales reales.

## 7. Conclusión de la clase

**¿Qué permite afirmar la revisión del diseño actual?**
validaciones de entrada, clasificación de motivo con respuesta estructurada y validada, comprobación de disponibilidad en Google Calendar antes de crear citas, rutas diferenciadas para confirmadas y rechazadas, y una consulta de horas cuya respuesta de IA se verifica contra el calendario. Los 16 casos de prueba se ejecutaron con evidencia en los destinos.

**¿Cuáles son las brechas más importantes y por qué?**
(1) Fallo de un servicio externo y operaciones parciales (RC-07, RC-08): pueden dejar una cita sin registro o una respuesta que no refleja el estado real. (2) Duplicados simultáneos (RC-06): la protección es indirecta y no cubre envíos concurrentes. Ambas se reconocen como limitaciones del diseño.

**¿Qué debe quedar listo y qué deberán demostrar en la sustentación?**
Mostrar un recorrido completo (entrada → validación → Calendar → registro → respuesta) con su resultado real en los destinos, y explicar una situación problemática con su evidencia.

**¿Qué limitaciones o dudas siguen abiertas?**
El manejo de fallos de servicios externos, las operaciones parciales y los duplicados simultáneos. La categoría de la IA no se almacena fuera del evento de Calendar. No hay registro propio de errores fuera del historial de n8n.
