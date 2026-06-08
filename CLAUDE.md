# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

EduTrack es la plataforma de libro de clases digital del Colegio Bernardo O'Higgins (Coquimbo). Este repositorio es el monorepo que aloja cada microservicio del backend como subdirectorio independiente.

**Stack transversal:** Quarkus 3.x + Java 21, PostgreSQL (Shared DB con schemas separados por servicio), Flyway para migraciones, Docker + Fly.io, RabbitMQ para eventos asíncronos.

## Microservicios

Cada subdirectorio es un proyecto Maven independiente con su propio `pom.xml`:

| Directorio | Servicio | Schema BD |
|---|---|---|
| `auth/` | Auth Service — JWT (RS256), roles dinámicos, permisos Unix-style | `auth` |
| *(próximos)* | Course, Student, Content, Assessment, Attendance, Annotation, Notification, Report | schema propio |

Cada microservicio tiene su propio `CLAUDE.md` con instrucciones específicas del dominio.

## Arquitectura

- **API Gateway** (OpenResty) como único punto de entrada: valida JWT RS256, propaga `sub` y `roles[]` como headers internos (`X-User-Id`, `X-User-Roles`) y elimina `Authorization` antes de llegar a cada MS.
- **Comunicación:** REST síncrono para operaciones directas; RabbitMQ (eventos async) para operaciones no críticas (Student, Assessment, Annotation → Notification).
- **Base de datos:** Una instancia PostgreSQL compartida. Cada MS tiene credenciales exclusivas a su schema. Report Service tiene credenciales adicionales de solo lectura sobre todos los schemas.
- **Seguridad:** JWT RS256 emitidos por Auth Service. Modelo de permisos Unix-style: flags numéricos `(r=4, w=2, x=1)` por par `(role_uuid, resource_key)`. El `resource_key` es una clave estable de texto (`<servicio>.<recurso>`, p. ej. `auth.users`), opaca en Auth y comparada por igualdad — cada MS define las suyas en código. El string es el contrato: idéntico en ambos lados de un grant, sin UUIDs que coordinar.
- **Circuit Breaker:** SmallRye Fault Tolerance (Quarkus) en llamadas entre servicios.
- **Hosting:** Fly.io. Cada MS es una app independiente. Comunicación interna vía `*.fly.internal` (DNS privado de Fly.io — sin Consul, sin configuración adicional).

## Contrato del API Gateway

El gateway enruta dinámicamente extrayendo el **primer segmento del path** y resolviéndolo como nombre de servicio:

```
/auth/login      → edutrack-auth.fly.internal:8080
/courses/lista   → edutrack-courses.fly.internal:8080
```

**Regla no negociable:** el nombre del primer segmento del path de cada microservicio debe coincidir exactamente con el nombre de su app en Fly.io (sin el prefijo `edutrack-`). Si el MS se llama `edutrack-courses` en Fly.io, todos sus endpoints deben estar bajo `/courses/`.

Esto aplica tanto en local (Docker, el contenedor se llama `courses`) como en Fly.io (`edutrack-courses`). El gateway no tiene lista de servicios — el contrato de nombres es el único mecanismo de descubrimiento.

## Código compartido entre microservicios (`infrastructure`)

El paquete `cl.duocuc.edutrack.ms.infrastructure` de cada MS aloja **código transversal** — sin reglas de negocio ni nombres específicos del MS. En un futuro cercano se extraerá como un artefacto de librería interna (p. ej. `edutrack-commons`) consumido por todos los servicios; hasta entonces se duplica intencionalmente en cada MS, manteniendo la misma forma y nombres. Ver el `package-info.java` del paquete para el contrato detallado.

**Reglas que ordenan `infrastructure.*`:**

- **No importa paquetes `ms.<servicio>.*`.** Ningún archivo bajo `infrastructure` puede depender de código de un dominio concreto. Cuando se necesita un punto de extensión específico del MS host (p. ej. evaluación de permisos), se expone como contrato CDI (`PermissionEvaluator`) y cada MS aporta una implementación.
- **No referencia nombres específicos.** Los identificadores propios de un MS (UUIDs de recursos, vistas/grupos de validación de su dominio) viven en el paquete del MS — p. ej. `auth.security.AuthResourceId`, `auth.model.dto.AuthViews`, `auth.model.dto.AuthValidations`. Lo común vive bajo `infrastructure.*`: el wildcard `ResourceIds.ALL`, la jerarquía `Views.Base/Extra/Detailed/...`, el grupo `Validations.OnCreate` con su secuencia `Validations.Create`.
- **Subpaquetes por responsabilidad técnica:**
  - `infrastructure.security` — autorización Unix-style: anotación `@RequirePermission(resource = <resource-key>, value = Permission.READ|WRITE|EXECUTE)`, enum `Permission` (bits), `ResourceIds` (wildcard `ALL` = `"*"`), contrato `PermissionEvaluator` que el filtro consume, `RequirePermissionFilter`.
  - `infrastructure.context` — intérprete único de cabeceras internas del Gateway: enum `InternalHeader`, record `RequestHeaders`, bean `RequestContext` (request-scoped), `HeaderValidationMode`.
  - `infrastructure.exception` — handler global (`GlobalExceptionMappers`), envelope JSON único (`ErrorResponse`), jerarquía `DomainException` + sugar (`ConflictException`, `NotFoundException`, `ForbiddenException`).
  - `infrastructure.jackson` — interfaz `Views` (vistas estándar `@JsonView`) + `JacksonCustomConfig` que la fija como vista por defecto.
  - `infrastructure.validation` — interfaz `Validations` con el grupo transversal `OnCreate` y la secuencia `Create`. Los grupos específicos de un dominio (p. ej. `AuthValidations.OnLogin`) viven en el MS.

## Convenciones de desarrollo

- **Paquete base:** `cl.duocuc.edutrack.ms.<servicio>`
- **Modelo de datos:** `cl.duocuc.edutrack.ms.<servicio>.model` — entidades JPA con Panache Active Record (`PanacheEntityBase`), PKs UUID, fetch LAZY en todas las asociaciones.
- **Herencia en entidades (DRY):** cuando un conjunto de columnas se repite en todas o la gran mayoría de las tablas de un servicio (típicamente `id`, `createdAt`, `updatedAt` y otros campos de auditoría), se extraen a una superclase `@MappedSuperclass` en el paquete `model.entity` en lugar de duplicarlas. Las callbacks JPA (`@PrePersist`, `@PreUpdate`) viajan con la superclase y se heredan automáticamente. Diseñar la jerarquía de superclases por capas de responsabilidad (p. ej. `CreatableEntity` con `id + createdAt` → `AuditableEntity extends CreatableEntity` agrega `updatedAt`) para que cada entidad herede solo lo que aplica a su semántica (entidades inmutables como tokens heredan únicamente la capa "creatable"). El DDL no cambia: las columnas siguen mapeándose con los mismos nombres, así que `database.generation=validate` sigue cuadrando sin migraciones adicionales.
- **Migraciones:** Flyway en `src/main/resources/db/migration/`. Hibernate con `database.generation=none` en prod; `validate` en dev para verificar coherencia con el DDL.
- **Variables de entorno** con defaults de desarrollo en `application.properties` (ej: `${DB_PASSWORD:dev_pass}`). No usar `.env` ni archivos de secrets versionados.
- **DTOs:** máximo **2 DTOs por entidad/recurso** — un único `XxxRequest` y un único `XxxResponse`. La granularidad (qué campos viajan en qué endpoint) se modela con `@JsonView` de Jackson, no creando records adicionales.
  - La jerarquía estándar de vistas vive en `infrastructure.jackson.Views` (compartida por todos los MS): `Base`, `Extra`, `Detailed extends Base`, `Create extends Base`, `List extends Base, Extra`, `Patch extends Base, Extra`, `Update extends Base`, `Admin extends Base, Extra`, `Internal`. Cada servicio aporta vistas propias del dominio en su propio paquete `model/dto/` (p. ej. `AuthViews.Login extends Views.Base`, `AuthViews.Refresh extends Views.Base` en Auth).
  - Los componentes de cada record DTO se anotan con `@JsonView(...)` indicando en qué vistas son visibles. Los endpoints JAX-RS se anotan con `@JsonView(Views.XXX.class)` tanto sobre el método (response) como sobre el parámetro de body (request) cuando aplique — salvo cuando la vista deseada es `Base`: cada servicio configura `Views.Base` como vista por defecto vía un `ObjectMapperCustomizer` en `infrastructure.jackson` (más `MapperFeature.DEFAULT_VIEW_INCLUSION=false`), por lo que **no es necesario anotar** `@JsonView(Views.Base.class)` explícitamente; los `@JsonView` que sí se anotan se aplican como override per-request.
  - **Contrato `*Response.fromEntity(...)`.** Todo `XxxResponse` expone un método estático que sabe construirse desde su fuente — la instanciación con `new XxxResponse(...)` queda confinada al propio DTO; los call sites siempre pasan por el factory. La signatura canónica es `public static XxxResponse fromEntity(XxxEntity entity)`. Cuando la entidad no basta (un colaborador externo aporta datos que no son columna, p. ej. el `roleIds` de `UserResponse` que vive en `UserRoleRepository`), el factory acepta ese colaborador como parámetro adicional. Cuando el DTO no respalda una entidad (resultados computados como `AuthResponse` tras emitir tokens, o `AccessResponse` tras evaluar permisos), se usa `public static XxxResponse of(...)` con los datos fuente; el principio es el mismo — la lógica de ensamble (etiquetas derivadas, constantes de protocolo como `tokenType="Bearer"`) vive en el DTO, no se duplica en cada caller.
  - **Toda validación de datos del request pasa por la API de Bean Validation de Jakarta.** Está prohibido validar datos del request con sentencias `if` dentro de un endpoint o en la capa de servicio (nada de `if (campo == null/blank/fuera-de-rango) throw 4xx`). Las restricciones (`@NotBlank`, `@Email`, `@Size`, `@Min`/`@Max`, …) se declaran sobre los componentes del record y se disparan con `@Valid`; la `ConstraintViolationException` se mapea a `400` automáticamente. La validación condicional por endpoint sobre un único record compartido (un campo obligatorio en `Create` pero opcional en `Update`) se modela con **validation groups** (`@GroupSequence` + `@ConvertGroup` en el parámetro del recurso), no con checks manuales. Quedan fuera de esta regla los guards de autenticación/identidad y las reglas de negocio (unicidad, invariantes de dominio), que siguen siendo checks explícitos en su capa.
- **Verificación de acceso entre servicios.** Auth expone `GET /auth/access?resourceKey={key}&permission={READ|WRITE|EXECUTE}` para que otros MS pregunten si el usuario propagado por el Gateway tiene un permiso sobre un recurso. Aplica exactamente el mismo algoritmo Unix-style que la anotación `@RequirePermission` (flags efectivos OR comodín `ALL`), centralizado en una única implementación reusada por el filtro interno y el endpoint — nunca se duplica la lógica de bits. Respuesta pensada para ser barata: `text/plain` ⇒ `"1"`/`"0"` (default); `application/json` ⇒ objeto con flags efectivos y label. El consumidor envía la identidad vía las cabeceras internas (no un `roleId` explícito).
- **Manejo de errores: handler global + excepciones de dominio.** Cada MS expone un único formato JSON de error vía `cl.duocuc.edutrack.ms.infrastructure.exception.GlobalExceptionMappers` (`@ServerExceptionMapper` por tipo: `DomainException`, `ConstraintViolationException`, `WebApplicationException`, `Throwable`). El envelope `ErrorResponse` lleva `timestamp` (Instant ISO-8601 UTC), `status` (int, también va en el body para que clientes que pierden el status en logs lo tengan), `error` (reason phrase), `code` (string opcional de dominio, e.g. `AUTH.USER.EMAIL_EXISTS`), `message`, `path`, `metadata` (`Map<String,Object>` opcional con contexto estructurado) y `trace` (opcional, solo cuando `edutrack.errors.expose-stacktrace=true`; default `false` en prod para no filtrar paquetes/clases internas ni inflar el body). Las reglas de negocio se lanzan con `DomainException` (sugar: `ConflictException`, `NotFoundException`, `ForbiddenException` con sus 409/404/403); el `code` es estable y switcheable por clientes, mientras el `message` es legible y la `metadata` carga el contexto (`.with("userId", id)`). Para reemplazar `new WebApplicationException(Response.status(409).entity(Map.of(...)).build())`: usar `throw new ConflictException("AUTH.<DOMINIO>.<CONDICION>", message).with("k", v)`. Convención de `code`: `<MS>.<ENTIDAD>.<CONDICION>` en SCREAMING_SNAKE; estable, no cambia entre versiones aunque cambie el `message`.
- **Cabeceras internas: un único intérprete request-scoped.** El API Gateway propaga la identidad ya autenticada como cabeceras internas `X-...` (`X-User-Id`, `X-User-Roles`, …). Cada MS expone **un único componente** en `infrastructure.context` que las interpreta: un enum `InternalHeader` (única fuente de verdad de los nombres de cabecera), un `record` inmutable con los valores ya tipados/validados, y un bean `@RequestScoped` proxyable (`RequestContext`) que computa y valida **una sola vez por request** (`@PostConstruct`) y expone el record vía `headers()`. **Prohibido** leer cabeceras `X-...` a mano en endpoints, filtros o servicios (`@HeaderParam("X-...")`, `getHeaderString("X-...")`): se inyecta `RequestContext`. Una cabecera **ausente** se modela como valor vacío (`Optional.empty()` / colección vacía), nunca `null`, y **no** es fallo de validación. Una cabecera **presente pero malformada** se rige por `edutrack.headers.validation.mode`: `EAGER` (default) aborta con `400`; `WARN` loguea y la trata como ausente dejando pasar el request. Nota CDI/ArC: un productor de bean normal-scoped no puede devolver un `record` (son `final`, no proxiables) — por eso el bean inyectable es el holder `RequestContext`, no `@Produces @RequestScoped` del record.

## Comandos por microservicio

Desde el directorio del servicio (ej: `auth/`):

```bash
# Dev mode con hot reload
./mvnw quarkus:dev

# Build
./mvnw package

# Tests unitarios
./mvnw test

# Tests de integración
./mvnw verify

# Un solo test
./mvnw test -Dtest=NombreDelTest
```
