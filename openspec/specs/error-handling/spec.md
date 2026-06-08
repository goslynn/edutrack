# Error Handling Specification

## Purpose

Un único formato JSON de error para todo MS y un mecanismo de excepciones de dominio con códigos
estables, de modo que clientes (y otros MS) discriminen condiciones sin parsear mensajes. Implementa
`FE-GEN-012` (mensajes sin detalles técnicos) y el contrato de error que `inter-service-communication`
propaga.

Fuentes: `commons/.../infrastructure/exception/{ErrorResponse,DomainException,GlobalExceptionMappers,
ConflictException,NotFoundException,ForbiddenException,BadRequestException,UnauthorizedException}.java`.

## Requirements

### Requirement: Envelope ErrorResponse único

Toda excepción que escape de un recurso SHALL serializarse con el envelope `ErrorResponse` con los
campos: `timestamp` (Instant ISO-8601 UTC), `status` (int, también en el body porque algunos clientes
pierden el status al loguear), `error` (reason phrase del status, o `"HTTP <n>"` si no estándar),
`code` (string de dominio opcional), `message` (legible), `metadata` (`Map<String,Object>` opcional)
y `trace` (opcional). Los campos vacíos/null SHALL omitirse (`@JsonInclude(NON_EMPTY)`).

#### Scenario: Conflicto de dominio
- **WHEN** se lanza un conflicto de email duplicado en Auth
- **THEN** la respuesta es `409` con `{timestamp, status:409, error:"Conflict", code:"AUTH.USER.EMAIL_EXISTS", message, metadata:{email}}`

#### Scenario: Envelope mínimo
- **WHEN** un error no tiene `code` ni `metadata`
- **THEN** esos campos se omiten del JSON, manteniéndolo mínimo

### Requirement: Excepciones de dominio con código estable

Las reglas de negocio SHALL lanzarse con `DomainException` o su sugar (`ConflictException`=409,
`NotFoundException`=404, `ForbiddenException`=403, `BadRequestException`=400,
`UnauthorizedException`=401). El `code` SHALL seguir la convención `<MS>.<ENTIDAD>.<CONDICION>` en
SCREAMING_SNAKE y ser **estable** entre versiones aunque cambie el `message`. La metadata se agrega
encadenada con `.with(k, v)` preservando orden de inserción. Para status no cubiertos por el sugar
(p. ej. `422`), SHALL instanciarse `DomainException` directamente.

#### Scenario: Lanzamiento con contexto
- **WHEN** `throw new ConflictException("AUTH.USER.EMAIL_EXISTS", "Email already in use").with("email", email)`
- **THEN** el envelope lleva `status=409`, ese `code`, ese `message` y `metadata={"email": ...}`

#### Scenario: Status fuera del sugar
- **WHEN** una validación de dominio requiere `422 Unprocessable`
- **THEN** se lanza `DomainException(422, code, message)` directamente

### Requirement: Prohibido construir errores HTTP a mano en dominio

NO se PERMITE lanzar `new WebApplicationException(Response.status(...).entity(Map.of(...)).build())`
para errores de dominio. Esos casos SHALL usar el sugar de `DomainException`. (Excepción: guards de
autenticación/identidad como `requireUserId()` que lanza `401` directo.)

#### Scenario: Reemplazo del patrón antiguo
- **WHEN** se necesita devolver un `409` con contexto
- **THEN** se usa `ConflictException(...).with(...)`, no un `WebApplicationException` ad-hoc

### Requirement: Handler global por tipo

`GlobalExceptionMappers` SHALL mapear con `@ServerExceptionMapper` por tipo: `DomainException`,
`ConstraintViolationException` (→ `400`), `WebApplicationException` (preserva su status) y `Throwable`
(→ `500`). Toda respuesta de error pasa por aquí y produce el mismo envelope.

#### Scenario: Bean Validation a 400
- **WHEN** un `@Valid` falla con `ConstraintViolationException`
- **THEN** el handler responde `400` con el envelope estándar

#### Scenario: Excepción no controlada a 500
- **WHEN** escapa un `Throwable` inesperado
- **THEN** el handler responde `500` con el envelope (sin filtrar internals salvo config explícita)

### Requirement: Stacktrace solo bajo configuración explícita

El campo `trace` (≤ 25 frames) SHALL incluirse únicamente cuando
`edutrack.errors.expose-stacktrace=true`. El default SHALL ser `false` para no filtrar
paquetes/clases internas ni inflar el body en prod.

#### Scenario: Producción sin trace
- **WHEN** `expose-stacktrace=false` (default) y ocurre un `500`
- **THEN** el envelope no incluye `trace`

#### Scenario: Debug con trace
- **WHEN** `expose-stacktrace=true`
- **THEN** el envelope incluye `trace` con hasta 25 frames
