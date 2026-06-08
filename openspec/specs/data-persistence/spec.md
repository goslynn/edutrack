# Data Persistence Specification

## Purpose

Cómo persiste cada MS: base de datos compartida con schema-per-service y aislamiento por credenciales
(ADR-001), entidades Panache con PK UUID y auditoría heredada, y migraciones Flyway con validación de
coherencia DDL. Implementa `INF-DB-001`, `BE-RPT-005`, `TRV-SEC-002`.

Fuentes: `commons/.../infrastructure/persistence/{CreatableEntity,AuditableEntity,AuditContext}.java`,
`auth/.../db/migration/V1__schema.sql`, `doc/estructura_paquetes.md`, `docker-compose.yml`.

## Requirements

### Requirement: Shared DB, schema-per-service, credenciales exclusivas

El sistema SHALL usar **una** instancia PostgreSQL compartida con **un schema por microservicio**
(`auth`, `student`, `attendance`, …). Cada MS SHALL conectarse con credenciales de BD **exclusivas a
su schema**; ningún MS PUEDE leer/escribir el schema de otro (restricción a nivel de usuario de BD).
El Report Service SHALL tener credenciales adicionales de **solo lectura** sobre todos los schemas
para reportes cross-dominio.

#### Scenario: Aislamiento entre schemas
- **WHEN** el MS de notas intenta leer la tabla de apoderados (otro schema)
- **THEN** la BD lo deniega por credenciales (aislamiento técnico, no de honor)

#### Scenario: Report read-only cross-schema
- **WHEN** Report Service genera un reporte consolidado
- **THEN** lee en read-only los schemas ajenos con sus credenciales adicionales, sin poder escribir

### Requirement: Entidades Panache con PK UUID y fetch LAZY

Las entidades SHALL ser Panache active record (`PanacheEntityBase`) con PK `UUID`
(`@GeneratedValue(strategy = UUID)`) y todas las asociaciones en fetch **LAZY**.

#### Scenario: PK generada
- **WHEN** se persiste una entidad nueva sin id explícito
- **THEN** se genera un UUID como PK

### Requirement: Auditoría heredada por capas

Las columnas de auditoría SHALL extraerse a superclases `@MappedSuperclass` en lugar de duplicarse:
- `CreatableEntity` (append-only): `id`, `created_at`, `creator_user` — `updatable=false`,
  `NOT NULL`. Para entidades inmutables (tokens, eventos, snapshots).
- `AuditableEntity extends CreatableEntity`: agrega `updated_at`, `updater_user`. Default para
  entidades mutables.

Las callbacks JPA SHALL viajar con la superclase: `@PrePersist` fija `created_at`/`creator_user`;
`@PreUpdate` refresca `updated_at`/`updater_user`; en creación, una `AuditableEntity` copia
created→updated para dejar la fila consistente. El `userId` lo aporta `AuditContext`.

#### Scenario: Entidad inmutable hereda solo creatable
- **WHEN** se modela un refresh token (inmutable)
- **THEN** extiende `CreatableEntity` (sin `updated_at`/`updater_user`)

#### Scenario: Creación de entidad auditable
- **WHEN** se persiste una entidad `AuditableEntity` por primera vez
- **THEN** `created_at == updated_at` y `creator_user == updater_user`

#### Scenario: Actualización
- **WHEN** se actualiza una entidad `AuditableEntity`
- **THEN** `updated_at`/`updater_user` se refrescan y `created_at`/`creator_user` quedan intactos (`updatable=false`)

### Requirement: Migraciones Flyway y validación de DDL

El esquema SHALL gestionarse con Flyway en `src/main/resources/db/migration/`. Hibernate SHALL correr
con `database.generation=validate` en dev (verifica coherencia entidad↔DDL) y `none` en prod. La
herencia por `@MappedSuperclass` NO cambia el DDL (las columnas conservan nombre), así que `validate`
sigue cuadrando sin migraciones extra.

#### Scenario: Desfase entidad/DDL en dev
- **WHEN** una entidad mapea una columna inexistente en el DDL Flyway en dev
- **THEN** el arranque falla en validación (detecta el desfase temprano)

### Requirement: Restricciones de integridad en el schema

El schema SHALL declarar las invariantes estructurales en DDL: unicidad de negocio (p. ej.
`users.email UNIQUE`, `role_permissions UNIQUE(role_id, resource_key)`), FKs con la política de borrado
correcta (`ON DELETE CASCADE` para hijos, `RESTRICT` donde corresponde) y checks de rango (p. ej.
`flags BETWEEN 0 AND 7`).

#### Scenario: Grant duplicado
- **WHEN** se intenta insertar un segundo `(role_id, resource_key)` igual
- **THEN** la BD lo rechaza por la constraint única

#### Scenario: Flags fuera de rango
- **WHEN** se intenta persistir `flags=8`
- **THEN** la BD lo rechaza por el `CHECK (flags >= 0 AND flags <= 7)`

### Requirement: Config por variables de entorno con defaults de dev

La configuración de datasource SHALL leerse de variables de entorno con defaults de desarrollo en
`application.properties` (`${DB_PASSWORD:dev_pass}`). NO se PERMITE `.env` versionado ni secretos en
el repo. En el stack local, `.env` es obligatorio y los secretos no tienen default (compose aborta si faltan).

#### Scenario: Arranque local sin secreto
- **WHEN** se levanta el stack sin `DB_PASSWORD` en `.env`
- **THEN** docker compose aborta exigiendo la variable (`${DB_PASSWORD:?...}`)
