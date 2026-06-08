# Student Service Specification

## Purpose

Gestión de alumnos y apoderados: CRUD de alumnos con RUT chileno, registro de apoderados, traslado
entre cursos conservando historial, borrado lógico y publicación de eventos de ciclo de vida. Schema
BD `student`. Path raíz `/student`. Implementa `BE-STU-001..004`.

Fuentes: `student/src/main/java/.../domain/{Student,Guardian,StudentStatus,RelationType}.java`,
`resource/StudentResource.java`, `service/{StudentService,GuardianService,StudentEventPublisher}.java`.

> Nota de divergencia: este servicio aún lee identidad con un filtro propio (`GatewayHeaderFilter` +
> `UserContext`) y excepciones propias (`StudentNotFoundException`, etc.) en lugar de inyectar
> `RequestContext` y usar el sugar de `commons`. La spec describe el **comportamiento esperado**; la
> alineación con los contratos de plataforma (`request-context`, `error-handling`, `authorization`)
> es deuda técnica conocida.

## Requirements

### Requirement: Registro de alumnos con RUT válido y único

Student SHALL exponer CRUD de alumnos (`/student/students`) con campos `rut`, `firstName`,
`lastName`, `birthDate` y `courseId`. El `rut` SHALL validarse como RUT chileno y ser único. Un RUT
duplicado SHALL producir `409`; un RUT inválido SHALL producir `422`.

#### Scenario: Alta válida
- **WHEN** se crea un alumno con RUT válido y no usado
- **THEN** la respuesta es `201` con el alumno creado en estado `ACTIVE`

#### Scenario: RUT duplicado
- **WHEN** se crea un alumno con un RUT ya registrado
- **THEN** la respuesta es `409`

#### Scenario: RUT inválido
- **WHEN** se crea un alumno con un RUT que no pasa la validación chilena
- **THEN** la respuesta es `422`

### Requirement: Estados y borrado lógico

Un alumno SHALL tener estado `ACTIVE`, `TRANSFERRED` o `DELETED`. El borrado SHALL ser lógico
(`softDelete`: estado `DELETED` + `deleted_at`), preservando el registro y su historial. Un alumno
borrado no SHALL aparecer en el listado activo.

#### Scenario: Borrado lógico
- **WHEN** se elimina un alumno
- **THEN** su estado pasa a `DELETED`, se registra `deleted_at` y deja de aparecer en el listado activo

### Requirement: Traslado entre cursos conservando historial

`PATCH /student/students/{id}/transfer` SHALL reasignar el `courseId` del alumno conservando su
historial académico (notas y asistencia previas permanecen asociadas al alumno). El traslado SHALL
publicar un evento `STUDENT_TRANSFERRED`.

#### Scenario: Traslado preserva historial
- **WHEN** se traslada un alumno a otro curso
- **THEN** su `courseId` cambia, su historial sigue asociado y se publica `STUDENT_TRANSFERRED` con `targetCourseId`

### Requirement: Registro de apoderados

`POST /student/students/{id}/guardians` SHALL asociar apoderados a un alumno con su `RelationType`
(`PARENT`, `GUARDIAN`, `TUTOR`). `GET .../guardians` SHALL listarlos. Al crear un apoderado SHALL
publicarse un evento de bienvenida (`GUARDIAN_REGISTERED`) consumible por Notification.

#### Scenario: Alta de apoderado dispara evento
- **WHEN** se registra un apoderado de un alumno
- **THEN** se persiste el apoderado y se publica `GUARDIAN_REGISTERED` async

### Requirement: Publicación de eventos de ciclo de vida

Student SHALL publicar eventos de creación, eliminación y traslado de alumno para consistencia en
servicios dependientes (ver spec `async-events`). Tras eliminar un alumno, los servicios dependientes
(p. ej. Attendance) dejan de aceptar registros para ese alumno.

#### Scenario: Eventos emitidos
- **WHEN** se crea / elimina / traslada un alumno
- **THEN** se publica `STUDENT_CREATED` / `STUDENT_DELETED` / `STUDENT_TRANSFERRED` respectivamente

### Requirement: Segregación de datos de apoderados

Los datos de apoderados SHALL tratarse como sensibles: accesibles solo por el flujo de notificación,
no por otros MS directamente (`TRV-PRI-002`). El acceso a datos de menores SHALL ser auditable
(`TRV-SEC-004`, Ley 19.628).

#### Scenario: Otro MS no accede a apoderados directamente
- **WHEN** un MS distinto de Notification intenta leer datos de apoderados
- **THEN** no tiene acceso directo al schema/datos de apoderados
