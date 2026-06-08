# Auth Service Specification

## Purpose

Servicio de identidad y autorización de EduTrack: registra y autentica usuarios, emite JWT RS256 con
roles, gestiona refresh tokens revocables, CRUD de roles y de grants Unix-style por rol, asignación de
roles a usuarios, expone `/auth/access` y el JWKS público. Schema BD `auth`. Implementa `BE-AUTH-001..008`.

Path raíz: `/auth` (`@ApplicationPath`). Ver también specs `authorization` (modelo de permisos) y
`request-context` (identidad).

Fuentes: `auth/src/main/java/.../resource/*`, `service/*`, `model/entity/*`, `db/migration/*`.

## Requirements

### Requirement: Autenticación y emisión de JWT

`POST /auth/login` SHALL autenticar con credenciales (email + password) y, si son válidas, emitir un
par de tokens. El access token SHALL ser un JWT firmado **RS256** con claims `sub` (UUID del usuario),
`roles` (lista de UUID de rol como strings), `iss="edutrack-auth"` y `exp`. Las credenciales inválidas
SHALL producir `401`. Los roles se resuelven **en la emisión** del token.

#### Scenario: Login exitoso
- **WHEN** se postean credenciales válidas
- **THEN** la respuesta trae access token (JWT RS256 con `sub`, `roles`, `exp`) y refresh token

#### Scenario: Credenciales inválidas
- **WHEN** el email no existe o el password no coincide
- **THEN** la respuesta es `401`

#### Scenario: Roles en el token reflejan la asignación actual
- **WHEN** un usuario tiene roles `[ADMIN]` al momento del login
- **THEN** el JWT emitido lleva `roles=[<uuid ADMIN>]`

### Requirement: Refresh tokens revocables con rotación

`POST /auth/refresh` SHALL emitir un nuevo par de tokens a partir de un refresh token válido. Los
refresh tokens SHALL persistirse **hasheados** (SHA-256), tener expiración (default 7 días) y ser
revocables. Al rotar, el refresh token usado SHALL marcarse revocado. Un refresh token revocado o
expirado SHALL producir `403`. El access token tiene expiración corta (default 900 s).

#### Scenario: Refresh válido
- **WHEN** se postea un refresh token vigente y no revocado
- **THEN** se emite un nuevo par de tokens y el refresh usado queda revocado

#### Scenario: Refresh revocado o expirado
- **WHEN** se postea un refresh token revocado o vencido
- **THEN** la respuesta es `403`

#### Scenario: Almacenamiento hasheado
- **WHEN** se inspecciona `auth.refresh_tokens`
- **THEN** solo se guarda el hash del token (nunca el valor en claro)

### Requirement: Logout / revocación de sesiones

`POST /auth/logout` SHALL revocar las sesiones (refresh tokens) de la identidad propagada.
`DELETE /auth/users/{id}/sessions` SHALL permitir a un administrador revocar todas las sesiones de un
usuario. La revocación invalida el refresh; el access token vigente expira por su `exp` (TTL corto).

#### Scenario: Logout revoca refresh
- **WHEN** un usuario autenticado hace logout
- **THEN** sus refresh tokens quedan revocados y un refresh posterior da `403`

### Requirement: CRUD dinámico de roles

Auth SHALL permitir crear, leer, actualizar y eliminar roles (`name` único, `description`) bajo
`/auth/roles`, protegidos por `@RequirePermission(resource="auth.roles")`. Los roles base SUPERUSER,
ADMIN, DOCENTE vienen como seed mutable. Un rol eliminado no SHALL ser asignable. NO se SHALL poder
dejar el sistema sin SUPERUSER activo.

#### Scenario: Crear rol
- **WHEN** un ADMIN crea un rol con nombre único
- **THEN** el rol queda disponible para asignación

#### Scenario: Nombre duplicado
- **WHEN** se crea un rol con un `name` ya existente
- **THEN** la respuesta es `409`

### Requirement: CRUD de grants Unix-style por rol

Auth SHALL gestionar los grants `(role_id, resource_key, flags)` en `auth.role_permissions`:
`GET /auth/roles/{roleId}/permissions` (lista), `PUT /auth/roles/{roleId}/permissions/{resourceKey}`
(upsert), `DELETE .../{resourceKey}` (revoca), `GET .../permissions/effective?resourceKey=` (flags
efectivos). `flags` SHALL estar en 0–7 y `(role_id, resource_key)` ser único. Un cambio de permiso
SHALL reflejarse en el siguiente token/consulta. La administración (`auth.permissions`) y la consulta
efectiva (`auth.permissions.effective`) son recursos distintos: la consulta es legible por DOCENTE, la
administración no.

#### Scenario: Upsert de grant
- **WHEN** un ADMIN hace `PUT /auth/roles/{id}/permissions/course.asignatura` con flags 6
- **THEN** se crea/actualiza el grant y la consulta efectiva del rol sobre `course.asignatura` devuelve 6

#### Scenario: Flags inválidos
- **WHEN** se intenta upsert con flags fuera de 0–7
- **THEN** la operación es rechazada (`400`/constraint)

### Requirement: Asignación y revocación de roles a usuarios

Auth SHALL permitir `GET /auth/users/{userId}/roles`, `POST /auth/users/{userId}/roles/{roleId}`
(asignar) y `DELETE .../{roleId}` (revocar), protegidos por `auth.user-roles`. La asignación SHALL
reflejarse en el siguiente JWT emitido; la revocación invalida sesiones tras el TTL. Los permisos
granulares por usuario individual (qué docente accede a qué curso) NO son responsabilidad de Auth: se
delegan al MS de dominio.

#### Scenario: Rol asignado aparece en el próximo token
- **WHEN** se asigna un rol a un usuario y este vuelve a loguearse
- **THEN** el nuevo JWT incluye ese rol

### Requirement: CRUD de usuarios con self-read

Auth SHALL exponer `/auth/users` (list/create/get/update/disable), protegidos por `auth.users`
(READ/WRITE). `GET /auth/users/{id}` SHALL permitir `selfParam="id"`: un usuario siempre puede leer su
propio perfil sin permiso explícito. El borrado SHALL ser lógico (disable). El `email` es único y el
password se almacena hasheado (bcrypt).

#### Scenario: Usuario lee su propio perfil
- **WHEN** un usuario hace `GET /auth/users/{su-id}`
- **THEN** obtiene su perfil aunque no tenga READ sobre `auth.users`

#### Scenario: Email duplicado
- **WHEN** se crea un usuario con un email ya registrado
- **THEN** la respuesta es `409` con code `AUTH.USER.EMAIL_EXISTS`

### Requirement: Verificación de acceso y JWKS públicos

Auth SHALL exponer `GET /auth/access` (ver spec `authorization`) y `GET /auth/.well-known/jwks.json`
con el set público de claves para validar JWT. Ambos SHALL ser públicos tras el Gateway.

#### Scenario: JWKS disponible
- **WHEN** un cliente o validador pide el JWKS
- **THEN** obtiene la(s) clave(s) pública(s) RS256 sin autenticación

### Requirement: Seed de roles base y administrador inicial

La migración Flyway `V2__seed.sql` SHALL sembrar los roles SUPERUSER/ADMIN/DOCENTE y sus grants base
(SUPERUSER `*`=7; ADMIN rwx sobre cada recurso de auth; DOCENTE r sobre `auth.roles` y
`auth.permissions.effective`). El usuario administrador inicial NO se siembra por SQL: `AdminSeeder`
lo crea al primer arranque si `auth.users` está vacía, con password hasheado y rol SUPERUSER, desde
config (`auth.seed.admin.*`).

#### Scenario: Primer arranque sin usuarios
- **WHEN** el servicio arranca y no existe el admin con rol SUPERUSER
- **THEN** `AdminSeeder` crea el admin con el password configurado y le asigna SUPERUSER

#### Scenario: Arranques sucesivos
- **WHEN** el admin SUPERUSER ya existe
- **THEN** el seeder no crea duplicados (idempotente)

### Requirement: Rendimiento de autenticación

El endpoint de login SHOULD responder en p95 < 500 ms con 100 usuarios concurrentes (`BE-AUTH-008`).

#### Scenario: Login bajo carga
- **WHEN** 100 usuarios concurrentes hacen login
- **THEN** el p95 de latencia del endpoint es < 500 ms
