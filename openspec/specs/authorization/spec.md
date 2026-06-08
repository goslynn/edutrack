# Authorization Specification

## Purpose

Modelo de autorización **Unix-style por tipo de recurso** (ADR-002, ADR-003). Auth responde "¿este
rol puede hacer la acción X sobre el *tipo* Y?"; el MS dueño del dato responde "¿sobre *esta
instancia*?". Es el contrato de seguridad transversal que todo MS comparte y el más implícito entre
servicios. Implementa `BE-AUTH-005`, `BE-AUTH-007` y el lado autorización de `BE-GW-002`.

Fuentes: `commons/.../infrastructure/security/{Permission,ResourceIds,RequirePermission,
PermissionEvaluator,RequirePermissionFilter,RemotePermissionEvaluator}.java`,
`auth/.../resource/AccessResource.java`, `auth/.../service/PermissionService.java`,
`auth/.../security/AuthResourceId.java`, `doc/permisos.md`.

## Requirements

### Requirement: Bits de permiso Unix-style

Los permisos SHALL representarse con los tres bits clásicos `READ=4`, `WRITE=2`, `EXECUTE=1`,
combinables en un `SMALLINT` (0–7). La evaluación de un bit requerido SHALL ser
`(effectiveFlags & requested.bit) == requested.bit`. Semántica: READ = GET/listados; WRITE =
POST/PUT/PATCH/DELETE; EXECUTE = acciones no-CRUD (disparar procesos, emitir tokens, marcar asistencia).

#### Scenario: Flag de lectura habilita GET
- **WHEN** un rol tiene flags `4` (r--) sobre un recurso
- **THEN** un endpoint que exige `READ` pasa, y uno que exige `WRITE` da `403`

#### Scenario: Combinación de bits
- **WHEN** un rol tiene flags `6` (rw-) sobre un recurso
- **THEN** satisface `READ` y `WRITE` por separado, pero no `EXECUTE`

### Requirement: resource_key de texto como contrato

Un recurso SHALL identificarse con una **clave estable de texto** `"<servicio>.<recurso>"`
(p. ej. `auth.users`), opaca para Auth y comparada por igualdad. El string **es** el contrato:
idéntico en ambos lados de un grant, sin UUIDs que coordinar. Cada MS define sus claves como
`public static final String` en su propio paquete `security/` (p. ej. `AuthResourceId`); Auth las
persiste como `varchar` en `auth.role_permissions.resource_key` sin conocer su significado.

#### Scenario: Grant y anotación usan la misma clave
- **WHEN** un endpoint declara `@RequirePermission(resource = "auth.users", value = WRITE)` y existe
  un grant `(ADMIN, "auth.users", 7)`
- **THEN** un ADMIN pasa el filtro porque ambos lados comparan el literal `"auth.users"`

#### Scenario: Clave sin grant
- **WHEN** se evalúa un `resource_key` para el que ningún rol del usuario tiene fila
- **THEN** los flags efectivos sobre esa clave son `0` (antes de aplicar el comodín)

### Requirement: Comodín ALL

`ResourceIds.ALL = "*"` SHALL ser un recurso comodín reservado. Todo evaluador SHALL computar los
flags efectivos como el OR entre los flags del recurso pedido y los flags sobre `ALL`:
`effective = flags(roles, resourceKey) | flags(roles, ALL)`. El comodín se usa al **consultar**, no
al **anotar**: ningún endpoint REST debe declarar `@RequirePermission(resource = ALL, ...)`.

#### Scenario: Grant comodín cubre cualquier recurso
- **WHEN** un rol tiene `(SUPERUSER, "*", 7)` y se evalúa cualquier `resourceKey`
- **THEN** los flags efectivos incluyen `7` ⇒ READ/WRITE/EXECUTE concedidos sobre todo recurso

#### Scenario: OR entre recurso y comodín
- **WHEN** un rol tiene `(DOCENTE, "course.asignatura", 2)` y `(DOCENTE, "*", 4)`
- **THEN** los flags efectivos sobre `course.asignatura` son `2 | 4 = 6` (rw-)

### Requirement: Evaluador único reutilizado (sin duplicar bits)

La lógica de bits y el comodín SHALL vivir en **una sola** implementación de `PermissionEvaluator`
por MS, reutilizada tanto por el filtro interno como por cualquier consulta en código. Auth la
implementa contra `auth.role_permissions`; los demás MS usan `RemotePermissionEvaluator` (HTTP a
`/auth/access`). El contrato del evaluador es
`hasPermission(List<UUID> roleIds, String resourceKey, short requiredBits)`.

#### Scenario: Filtro y endpoint comparten algoritmo
- **WHEN** se evalúa el mismo `(roles, resource, bit)` vía `@RequirePermission` y vía `/auth/access`
- **THEN** ambos devuelven el mismo resultado (misma implementación subyacente)

### Requirement: Anotación @RequirePermission

`@RequirePermission(resource, value, selfParam?)` SHALL declarar el permiso mínimo de un endpoint
JAX-RS vía name binding. `RequirePermissionFilter` (prioridad `AUTHORIZATION`) SHALL ejecutarse
**antes** de la deserialización del body y de Bean Validation: un `403` no cuesta parsear el payload.
El filtro SHALL buscar la anotación primero en el método y luego en la clase, leer roles de
`RequestContext`, delegar en `PermissionEvaluator` y, si retorna `false`, lanzar `ForbiddenException`
(code `SECURITY.PERMISSION.DENIED`, con `resource`/`requiredPermission`/`userId` en metadata).

#### Scenario: Permiso suficiente
- **WHEN** un request a un endpoint anotado lleva roles que satisfacen el bit requerido
- **THEN** el filtro deja pasar y el endpoint ejecuta

#### Scenario: Permiso insuficiente
- **WHEN** los roles no satisfacen el bit requerido
- **THEN** el filtro lanza `403` con code `SECURITY.PERMISSION.DENIED` antes de leer el body

#### Scenario: Endpoint sin anotación no paga el filtro
- **WHEN** un endpoint no porta `@RequirePermission` (ni su clase)
- **THEN** el filtro no se invoca para ese endpoint (name binding)

### Requirement: Excepción "self"

Cuando `@RequirePermission` declara `selfParam`, el filtro SHALL autorizar sin consultar permisos si
la identidad propagada coincide con el path-param indicado (comparado como string contra
`userId.toString()`). Útil para "un usuario siempre puede leer su propio perfil".

#### Scenario: Usuario accede a su propio recurso
- **WHEN** `GET /auth/users/{id}` con `selfParam="id"` y `X-User-Id` igual a `{id}`
- **THEN** el filtro autoriza sin evaluar permisos

#### Scenario: Usuario accede al recurso de otro
- **WHEN** `{id}` no coincide con `X-User-Id`
- **THEN** el filtro evalúa permisos normalmente

### Requirement: Reparto propio vs ajeno (instancia la decide el MS dueño)

Auth SHALL decidir solo a nivel de **tipo** de recurso. La regla de pertenencia ("es mío") SHALL
vivir en el MS dueño del dato como cláusula SQL o check post-fetch (`WHERE owner = :userId`). Si la
instancia no es del usuario, el MS responde `403` (o `404` si oculta existencia), aunque Auth haya
concedido el verbo sobre el tipo.

#### Scenario: Verbo concedido pero instancia ajena
- **WHEN** un docente con WRITE sobre `course.asignatura` edita una asignatura que no es suya
- **THEN** Auth dice "permitido" sobre el tipo, pero Course responde `403`/`404` por la cláusula de pertenencia

#### Scenario: Super-usuario omite la pertenencia
- **WHEN** un SUPERUSER (grant `*`=7) opera sobre cualquier instancia
- **THEN** el MS dueño puede omitir el `AND owner = :userId` para roles administrativos

### Requirement: Endpoint GET /auth/access para consumo entre servicios

Auth SHALL exponer `GET /auth/access?resourceKey={key}&permission={READ|WRITE|EXECUTE}` aplicando el
mismo algoritmo que `@RequirePermission` sobre la identidad propagada. El endpoint SHALL ser
**público tras el Gateway** (sin `@RequirePermission`): sin identidad no hay grants que sumar ⇒
responde "denegado", no `403` (evita el meta-guard circular). SHALL negociar contenido:
- `text/plain` (default): `"1"` / `"0"`.
- `application/json`: `AccessResponse` con `allowed`, `resourceKey`, `required`, `effectiveFlags`,
  `effectiveLabel` (rwx). `effectiveFlags` es siempre el OR completo (incluye comodín);
  `permission` no lo altera.

#### Scenario: Consulta texto plano permitida
- **WHEN** otro MS llama `GET /auth/access?resourceKey=course.asignatura&permission=WRITE` con identidad que la tiene
- **THEN** Auth responde `200` con cuerpo `"1"`

#### Scenario: Consulta sin identidad
- **WHEN** se consulta `/auth/access` sin cabeceras de identidad propagadas
- **THEN** Auth responde `"0"` / `allowed=false` (no `403`)

#### Scenario: resourceKey ausente
- **WHEN** se llama `/auth/access` sin `resourceKey`
- **THEN** Auth responde `400`

#### Scenario: JSON con flags efectivos
- **WHEN** se pide `application/json` y el usuario tiene flags `6` sobre el recurso
- **THEN** la respuesta trae `effectiveFlags=6` y `effectiveLabel="rw-"`

### Requirement: Fail-closed remoto

`RemotePermissionEvaluator` SHALL ser fail-closed: cortocircuita a `false` si no hay roles, y traduce
a `false` (logueando WARN) cualquier excepción de la llamada a `/auth/access` (timeout, 5xx,
indisponibilidad, error de deserialización). Una falla de Auth nunca SHALL escalar a permiso concedido.

#### Scenario: Auth caído
- **WHEN** un MS evalúa un permiso y `/auth/access` falla con timeout
- **THEN** el evaluador devuelve `false` (acceso denegado) y loguea WARN

#### Scenario: Usuario sin roles
- **WHEN** `roleIds` viene vacío
- **THEN** el evaluador devuelve `false` sin llamada remota
