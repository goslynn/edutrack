# Planned Services Specification

## Purpose

Servicios ya reservados en `ServiceIds` pero aún **no implementados**. Esta spec fija sus contratos de
alto nivel para que, cuando se construyan, honren los contratos de plataforma (gateway, discovery,
authorization, request-context, error-handling, data-persistence, api-conventions, async-events) sin
re-derivar decisiones. Cada uno SHALL: montarse bajo `/<service>`, tener su propio schema BD,
proteger endpoints con `@RequirePermission` sobre sus `resource_key`, sembrar sus grants en Auth, y
exponer OpenAPI. Implementan `BE-CRS-*`, `BE-CNT-*`, `BE-ASS-*`, `BE-ANN-*`, `BE-NOT-*`, `BE-RPT-*`.

## Requirements

### Requirement: Course Service — cursos y permiso granular docente-curso

Course (`/course`, schema `course`) SHALL exponer CRUD de cursos (borrado lógico) y gestionar la
relación **docente-curso con nivel de acceso** (lectura/escritura). Este es el permiso granular **por
usuario e instancia** que complementa el modelo de roles de Auth: Auth concede el verbo sobre el tipo
`course.asignatura`; Course decide la pertenencia con su propia regla (`WHERE docente_id = :userId`).
El listado SHALL filtrarse por el docente autenticado.

#### Scenario: Docente no ve cursos ajenos
- **WHEN** el docente A lista cursos y no tiene asignación al curso del docente B
- **THEN** el curso de B no aparece en su listado (`BE-CRS-003`)

#### Scenario: Solo lectura no escribe
- **WHEN** un docente con acceso solo-lectura a un curso intenta registrar notas en él
- **THEN** la respuesta es `403` (`BE-CRS-002`)

### Requirement: Content Service — jerarquía configurable y archivos S3

Content (`/content`, schema `content`) SHALL modelar la estructura de contenido como un **árbol de
nodos configurables globalmente** (cada nodo: nombre, descripción, orden; todos los niveles
configurables, incluidos raíz y hoja; seed Semestre > Asignatura > Unidad > Clase). El CRUD de nodos
SHALL respetar la relación padre-hijo del árbol activo. Los archivos SHALL subirse en nodos hoja a S3
y descargarse vía URL pre-firmada con expiración; el tamaño máximo SHALL ser 500 MB (>500 MB → `413`).

#### Scenario: Validación padre-hijo
- **WHEN** se crea un nodo de nivel N sin padre de nivel N-1
- **THEN** la operación es rechazada (`BE-CNT-002`)

#### Scenario: Archivo demasiado grande
- **WHEN** se sube un archivo > 500 MB
- **THEN** la respuesta es `413` con mensaje descriptivo (`BE-CNT-004`)

#### Scenario: Descarga con URL pre-firmada
- **WHEN** se solicita la descarga de un archivo
- **THEN** se entrega una URL pre-firmada que expira en el tiempo configurado; sin URL válida el archivo es inaccesible

### Requirement: Assessment Service — notas, promedios y auditoría inmutable

Assessment (`/assessment`, schema `assessment`) SHALL registrar evaluaciones (nombre, fecha,
ponderación) y notas en escala chilena **1.0–7.0** por alumno y asignatura (nota fuera de rango →
`422`). SHALL calcular el promedio ponderado por asignatura y período. SHALL mantener un log de
auditoría **inmutable** de cambios de notas (usuario, timestamp, valor anterior, valor nuevo), legible
por ADMIN/SUPERUSER y no borrable por docente. SHALL respetar integridad referencial con Student (nota
para alumno inexistente → `404`).

#### Scenario: Nota fuera de rango
- **WHEN** se registra una nota < 1.0 o > 7.0
- **THEN** la respuesta es `422` (`BE-ASS-001`)

#### Scenario: Alumno inexistente
- **WHEN** se registra una nota para un alumno que no existe en Student
- **THEN** la respuesta es `404` (`BE-ASS-004`)

#### Scenario: Log de notas inmutable
- **WHEN** un docente intenta borrar el log de cambios de notas
- **THEN** la operación es denegada; el log permanece para ADMIN/SUPERUSER (`BE-ASS-003`)

### Requirement: Annotation Service — anotaciones con notificación async

Annotation (`/annotation`, schema `annotation`) SHALL registrar anotaciones positivas o negativas
vinculadas a alumno, docente y fecha (anotación sin tipo → `422`). Al registrar una anotación
**negativa** SHALL publicar un evento async a Notification (ver spec `async-events`).

#### Scenario: Anotación sin tipo
- **WHEN** se registra una anotación sin tipo
- **THEN** la respuesta es `422` (`BE-ANN-001`)

#### Scenario: Negativa notifica al apoderado
- **WHEN** se registra una anotación negativa
- **THEN** se publica un evento que Notification consume para notificar al apoderado (`BE-ANN-002`)

### Requirement: Notification Service — contrato genérico (Strategy) y EMAIL_HTML

Notification (`/notification`, schema `notification`) SHALL exponer un contrato de entrada
estandarizado `NotificationRequest` con `notification_type` (enum), `recipient_id`, `template_id` y
`payload` (mapa). El `notification_type` SHALL determinar la estrategia internamente (Strategy
Pattern); el contrato NO SHALL cambiar al agregar tipos. Un tipo desconocido SHALL producir `422`. En
v1 el único tipo soportado es `EMAIL_HTML` (correo HTML con plantillas interpoladas). Los tipos SHALL
ser registrables en BD sin redeploy. SHALL tener cola de reintento con backoff y DLQ tras N fallos.

#### Scenario: Tipo soportado
- **WHEN** llega un request con `notification_type=EMAIL_HTML`
- **THEN** lo procesa `EmailStrategy` (`BE-NOT-002`)

#### Scenario: Tipo desconocido
- **WHEN** llega un `notification_type` no registrado
- **THEN** la respuesta es `422` (`BE-NOT-001`)

#### Scenario: Nuevo tipo sin redeploy
- **WHEN** se registra un nuevo tipo con su estrategia/plantilla en BD
- **THEN** queda disponible sin modificar el contrato ni redeployar (`BE-NOT-003`)

### Requirement: Report Service — definiciones, formatos y acceso read-only cross-schema

Report (`/report`, schema `report`) SHALL gestionar definiciones de reporte (nombre, descripción,
fuentes, parámetros, formatos JSON/CSV/PDF), administrables **solo por ADMIN/SUPERUSER**. SHALL generar
reportes en JSON (respuesta HTTP), CSV (descarga `Content-Disposition: attachment`, UTF-8 con BOM para
Excel) y PDF (`application/pdf`, byte[]). SHALL usar las credenciales **read-only cross-schema**
(ADR-001) para consolidar datos de múltiples MS. Toda ejecución SHALL quedar en log de auditoría
(usuario, reporte, timestamp, formato); sin permiso → `403`.

#### Scenario: Solo roles autorizados definen reportes
- **WHEN** un usuario sin rol ADMIN/SUPERUSER intenta crear/modificar una definición de reporte
- **THEN** la respuesta es `403` (`BE-RPT-001`, `BE-RPT-006`)

#### Scenario: Generación CSV compatible Excel
- **WHEN** se ejecuta un reporte en formato CSV
- **THEN** se descarga un `.csv` UTF-8 con BOM y `Content-Disposition: attachment` (`BE-RPT-003`)

#### Scenario: Ejecución auditada
- **WHEN** se ejecuta cualquier reporte
- **THEN** queda un registro de auditoría con usuario, reporte, timestamp y formato (`BE-RPT-006`)
