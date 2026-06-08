# Request Context (Internal Headers) Specification

## Purpose

Define el **único intérprete** de las cabeceras internas que el Gateway propaga (`X-User-Id`,
`X-User-Roles`) dentro de cada MS, y la noción de identidad/super-usuario derivada de ellas. Es la
base de confianza de ADR-004: el MS no re-valida el JWT, confía en estas cabeceras porque solo el
Gateway puede inyectarlas.

Fuentes: `commons/.../infrastructure/context/{InternalHeader,RequestHeaders,RequestContext,
HeaderValidationMode,SuperUserResolver,RemoteSuperUserResolver}.java`,
`persistence/AuditContext.java`.

## Requirements

### Requirement: Catálogo único de nombres de cabecera

`InternalHeader` SHALL ser la única fuente de verdad de los nombres en el wire
(`USER_ID="X-User-Id"`, `USER_ROLES="X-User-Roles"`). Ningún otro archivo PUEDE referenciar el
string `"X-..."` directamente.

#### Scenario: Lectura siempre vía el catálogo
- **WHEN** un componente necesita el nombre de la cabecera de usuario
- **THEN** lo obtiene de `InternalHeader.USER_ID.wire`, nunca como literal `"X-User-Id"`

### Requirement: Prohibido leer cabeceras X- a mano

Endpoints, filtros y servicios NO PUEDEN leer las cabeceras internas con `@HeaderParam("X-...")` ni
`routingContext.request().getHeader("X-...")`. SHALL inyectar `RequestContext` y leer
`ctx.headers()`.

#### Scenario: Acceso a identidad en un endpoint
- **WHEN** un servicio necesita el `userId` del request
- **THEN** lo obtiene de `requestContext.headers().requireUserId()` / `.userId()`

### Requirement: Interpretación única por request

`RequestContext` SHALL ser `@RequestScoped` y proxyable, parsear cada cabecera del catálogo **una
sola vez** en `@PostConstruct` y exponer un `RequestHeaders` inmutable vía `headers()`. Inyecciones
sucesivas en el mismo request obtienen la misma instancia sin re-parsear. El record `RequestHeaders`
NO se inyecta directamente por CDI (los records son `final`, no proxiables).

#### Scenario: Costo fijo por request
- **WHEN** tres componentes inyectan `RequestContext` en el mismo request
- **THEN** las cabeceras se parsean una vez y los tres ven el mismo `RequestHeaders`

### Requirement: Ausencia ≠ error; valor vacío

Una cabecera **ausente o en blanco** SHALL modelarse como valor vacío (`Optional.empty()` para
`userId`, lista vacía para `roleIds`), **nunca** `null`, y **no** es fallo de validación. La
decisión de exigir identidad la toma el consumidor.

#### Scenario: Request anónimo (este–oeste sin identidad)
- **WHEN** llega un request sin `X-User-Id` ni `X-User-Roles`
- **THEN** `headers().userId()` es `Optional.empty()` y `headers().roleIds()` es lista vacía, sin error

#### Scenario: Identidad obligatoria ausente
- **WHEN** un endpoint llama `headers().requireUserId()` y no hay identidad
- **THEN** se lanza `401 Unauthorized` (la ausencia de identidad es fallo de autenticación, no de validación)

### Requirement: Política configurable para cabecera malformada

Una cabecera **presente pero malformada** (UUID inválido en `X-User-Id`, o cualquier token inválido
en `X-User-Roles`) SHALL regirse por `edutrack.headers.validation.mode`:
- `EAGER` (default): aborta el request con `400 Bad Request` y mensaje
  `"Cabecera interna malformada: <wire>"`.
- `WARN`: loguea y trata la cabecera como ausente (valor vacío).

`X-User-Roles` se separa por coma, se hace `trim`, se descartan tokens vacíos; basta **un** token
inválido para considerar toda la lista malformada (los tokens previos no se conservan).

#### Scenario: UUID inválido en modo EAGER
- **WHEN** `mode=EAGER` y llega `X-User-Id: not-a-uuid`
- **THEN** el request aborta con `400` y mensaje `"Cabecera interna malformada: X-User-Id"`

#### Scenario: Rol inválido en modo WARN
- **WHEN** `mode=WARN` y llega `X-User-Roles: <uuid-ok>,bad`
- **THEN** se loguea WARN y `roleIds()` queda como lista vacía, el request continúa

### Requirement: Resolución de super-usuario

`RequestContext.isSuper()` SHALL indicar si la identidad del request posee `rwx` (los tres bits)
sobre el comodín `ResourceIds.ALL`, delegando en `SuperUserResolver` (default
`RemoteSuperUserResolver`, sustituible por `@DefaultBean`). El resultado SHALL memoizarse a nivel de
request. SHALL ser fail-closed: cualquier fallo o ausencia de identidad ⇒ `false`.

#### Scenario: Identidad con grant ALL=7
- **WHEN** el usuario tiene un grant `(rol, *, 7)`
- **THEN** `isSuper()` devuelve `true` y solo invoca al resolver una vez por request

#### Scenario: Sin identidad
- **WHEN** el request no trae roles
- **THEN** `isSuper()` devuelve `false` sin llamadas remotas

### Requirement: Atribución de auditoría desde el contexto

`AuditContext` SHALL resolver el `userId` del request actual para poblar `creator_user`/`updater_user`
de las entidades. Fuera de un request HTTP (jobs, listeners) SHALL caer al usuario NOOP configurado
en `edutrack.defaults.noop-user-id`.

#### Scenario: Persistencia dentro de un request
- **WHEN** se persiste una entidad auditable durante un request con identidad
- **THEN** `creator_user` queda con el `userId` del request

#### Scenario: Persistencia fuera de request
- **WHEN** se persiste fuera de un request HTTP activo
- **THEN** `creator_user` queda con el UUID NOOP configurado (no `null` que viole el NOT NULL)
