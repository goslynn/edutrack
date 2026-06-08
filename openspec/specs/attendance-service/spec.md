# Attendance Service Specification

## Purpose

Registro de asistencia por clase, **agnóstico al mecanismo de captura**: sesiones de asistencia que el
docente abre y cierra, y registros por alumno (presente/ausente/justificado). Schema BD `attendance`.
Path raíz `/attendance`. Implementa `BE-ATT-001..004`.

Fuentes: `attendance/src/main/java/.../domain/{AttendanceSession,AttendanceRecord,SessionStatus,
AttendanceStatus}.java`, `resource/{AttendanceSessionResource,AttendanceRecordResource}.java`,
`security/AttendencesResourcesId.java`.

> Nota de divergencia: las claves de recurso de este MS (`attendance_sessions`,
> `attendance_records`) NO siguen la convención `"<servicio>.<recurso>"` con punto (sería
> `attendance.sessions`, `attendance.records`). Para que la autorización funcione, los grants
> sembrados en Auth deben usar exactamente estas mismas claves. Alinear a la convención con punto es
> deuda técnica conocida (ver spec `authorization`, requirement "resource_key de texto como contrato").

## Requirements

### Requirement: Sesiones de asistencia

`POST /attendance/sessions` SHALL crear una sesión de asistencia para una clase, protegida por
`@RequirePermission(resource="attendance_sessions", value=WRITE)`. Una sesión SHALL tener estado
`OPEN` o `CLOSED`, partiendo en `OPEN`.

#### Scenario: Crear sesión
- **WHEN** un docente con permiso crea una sesión
- **THEN** la respuesta es `201` con la sesión en estado `OPEN`

#### Scenario: Sin permiso de escritura
- **WHEN** un usuario sin WRITE sobre `attendance_sessions` intenta crear una sesión
- **THEN** la respuesta es `403`

### Requirement: Registro de asistencia por alumno

`POST /attendance/sessions/{id}/records` SHALL registrar la asistencia de un alumno en una sesión,
protegido por `@RequirePermission(resource="attendance_records", value=WRITE)`. El estado SHALL ser
`PRESENT`, `ABSENT` o `JUSTIFIED`. Un registro duplicado para la misma sesión/clase y alumno SHALL
producir `409`.

#### Scenario: Registro presente
- **WHEN** se registra a un alumno como `PRESENT` en una sesión abierta
- **THEN** la respuesta es `201`

#### Scenario: Registro duplicado
- **WHEN** se registra dos veces al mismo alumno en la misma sesión
- **THEN** la segunda da `409`

### Requirement: Cierre de sesión inmuta el registro

`PATCH /attendance/sessions/{id}/close` SHALL cerrar la sesión (estado `CLOSED`), impidiendo
modificaciones posteriores. Cualquier intento de registrar o modificar asistencia en una sesión
cerrada SHALL producir `403`. Cerrar una sesión ya cerrada SHALL ser rechazado.

#### Scenario: Cerrar sesión
- **WHEN** el docente cierra una sesión abierta
- **THEN** la sesión pasa a `CLOSED`

#### Scenario: Registro en sesión cerrada
- **WHEN** se intenta registrar asistencia en una sesión `CLOSED`
- **THEN** la respuesta es `403`

#### Scenario: Cierre de sesión ya cerrada
- **WHEN** se cierra una sesión que ya está `CLOSED`
- **THEN** la operación es rechazada (`403`/error de estado)

### Requirement: Contrato agnóstico al origen de captura

El payload de registro SHALL aceptar `student_id` (UUID), `class_id`/sesión (UUID), `timestamp`
(ISO-8601), `status` (PRESENT|ABSENT|JUSTIFIED), un `capture_method` opcional (string informativo, NO
procesado ni validado por el MS) y `metadata` opcional (mapa clave-valor, almacenado tal cual sin
interpretación). El MS NO SHALL tener conocimiento ni dependencia de sistemas externos de captura
(biometría, RFID): cualquier integración es un módulo cliente externo que consume este contrato como
cualquier HTTP. El contrato SHALL estar documentado en OpenAPI 3.0 (`BE-ATT-004`).

#### Scenario: capture_method ignorado en lógica
- **WHEN** llega un registro con `capture_method="biometric"`
- **THEN** el registro se acepta y `capture_method` no altera la lógica de negocio

#### Scenario: metadata almacenada sin interpretar
- **WHEN** llega un registro con `metadata={device:"X"}`
- **THEN** la metadata se almacena tal cual, sin interpretación

### Requirement: Reporte de inasistencias acumuladas

Attendance SHOULD calcular el porcentaje de asistencia acumulada por alumno y período (hasta 3
decimales) y señalar alerta cuando es < 85% (`BE-ATT-003`).

#### Scenario: Alerta por baja asistencia
- **WHEN** un alumno acumula < 85% de asistencia en el período
- **THEN** el reporte marca la alerta correspondiente

### Requirement: Integridad referencial con alumnos eliminados

Attendance SHALL dejar de aceptar registros para un alumno eliminado, consumiendo el evento
`STUDENT_DELETED` (ver spec `async-events`).

#### Scenario: Registro para alumno eliminado
- **WHEN** se intenta registrar asistencia para un alumno marcado `DELETED` en Student
- **THEN** Attendance lo rechaza
