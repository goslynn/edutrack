# Async Events Specification

## Purpose

Comunicación asíncrona entre servicios vía RabbitMQ para operaciones no críticas (ciclo de vida del
alumno, anotaciones → notificación). Define la forma del evento y las garantías de entrega. Implementa
`INF-MSG-001/002`, `BE-STU-002/004`, `BE-ANN-002`, `BE-NOT-004`.

Estado: el **contrato** de eventos está definido; el publisher actual en Student es un **stub de log**
(`StudentEventPublisher` escribe el evento al log). El broker RabbitMQ real y los consumidores están
pendientes. Fuente: `student/.../service/StudentEventPublisher.java`.

## Requirements

### Requirement: Broker central RabbitMQ con persistencia

Los eventos asíncronos SHALL transitar por un broker RabbitMQ central. Los mensajes SHALL persistir
ante reinicio del broker (`INF-MSG-001`).

#### Scenario: Mensaje sobrevive reinicio
- **WHEN** se publica un evento y el broker se reinicia antes de ser consumido
- **THEN** el mensaje sigue disponible tras el reinicio

### Requirement: Forma estándar del evento

Todo evento de dominio SHALL incluir al menos `type` (string en SCREAMING_SNAKE, p. ej.
`STUDENT_CREATED`), el identificador del agregado afectado y `timestamp` (ISO-8601 UTC). Campos
adicionales específicos del evento SHALL agregarse según el tipo (p. ej. `targetCourseId` en
`STUDENT_TRANSFERRED`, `guardianId` en `GUARDIAN_REGISTERED`).

#### Scenario: Evento de creación de alumno
- **WHEN** se crea un alumno
- **THEN** se publica `{type:"STUDENT_CREATED", studentId, timestamp}`

#### Scenario: Evento con datos extra
- **WHEN** se traslada un alumno
- **THEN** se publica `STUDENT_TRANSFERRED` con `studentId`, `timestamp` y `targetCourseId`

### Requirement: Eventos de ciclo de vida del alumno

Student Service SHALL publicar eventos de creación, eliminación y traslado de alumno, y de registro de
apoderado, para consistencia en servicios dependientes (`BE-STU-004`). Los consumidores reaccionan
(p. ej. Attendance deja de aceptar registros para un alumno eliminado; Notification envía bienvenida
al registrar apoderado).

#### Scenario: Bienvenida a apoderado
- **WHEN** se registra un apoderado (`BE-STU-002`)
- **THEN** se publica `GUARDIAN_REGISTERED` que Notification consume para enviar la bienvenida async

#### Scenario: Alumno eliminado
- **WHEN** se elimina un alumno
- **THEN** se publica `STUDENT_DELETED` y Attendance deja de aceptar registros para ese alumno

### Requirement: Notificación de anotación negativa

Annotation Service SHALL publicar un evento async a Notification al registrar una anotación negativa,
de modo que el apoderado reciba la notificación (`BE-ANN-002`, criterio: < 5 min).

#### Scenario: Anotación negativa notificada
- **WHEN** se registra una anotación de tipo negativo
- **THEN** se publica un evento que Notification consume para notificar al apoderado

### Requirement: Reintentos con backoff y Dead Letter Queue

El consumo SHALL reintentar con backoff exponencial ante fallo transitorio del proveedor (p. ej. SMTP)
y, tras N reintentos (criterio: 3), enviar el mensaje a una **DLQ** con alerta (`INF-MSG-002`,
`BE-NOT-004`).

#### Scenario: Fallo persistente a DLQ
- **WHEN** un mensaje falla su procesamiento tras 3 reintentos
- **THEN** se mueve a la DLQ y se dispara una alerta

### Requirement: Eventos para operaciones no críticas

La mensajería async SHALL reservarse para operaciones **no críticas**; las operaciones directas
SHALL usar REST síncrono. Un fallo del broker no SHALL bloquear la operación de negocio principal del
publisher.

#### Scenario: Operación principal independiente del broker
- **WHEN** se crea un alumno y el broker está indisponible
- **THEN** el alumno se persiste correctamente; la publicación del evento se maneja de forma resiliente (reintento/log), sin fallar la creación
