# API Gateway Specification

## Purpose

El API Gateway (OpenResty: nginx + Lua) es el **único punto de entrada** norte–sur del sistema.
Centraliza la validación del JWT, descubre el upstream a partir del path, propaga la identidad
ya autenticada como cabeceras internas y aplica políticas transversales (rate-limit, correlación).
Implementa `BE-GW-001..004`, `BE-AUTH-002` (lado validación) y `BE-TRV-004` (correlación).

Fuentes: `infra/gateway/nginx.conf.template`, `infra/gateway/lua/jwt.lua`,
`infra/gateway/lua/upstream.lua`, `infra/gateway/entrypoint.sh`.

## Requirements

### Requirement: Enrutamiento por primer segmento del path

El Gateway SHALL resolver el microservicio destino extrayendo el **primer segmento del path**
(`^/([^/]+)`) y sustituyéndolo en `UPSTREAM_HOST_PATTERN` (placeholder `{service}`). El Gateway
NO MANTIENE una lista de servicios: el contrato de naming es el único mecanismo de descubrimiento.
El primer segmento DEBE coincidir con el nombre lógico del servicio en `ServiceIds`, con el nombre
del contenedor en local y con la app en Fly.io (sin el prefijo `edutrack-`).

#### Scenario: Ruta válida se enruta al upstream correcto
- **WHEN** llega `POST /auth/login`
- **THEN** el Gateway resuelve `service = "auth"` y hace `proxy_pass` a `auth:8080` (local) /
  `edutrack-auth.fly.internal:8080` (Fly.io) según `UPSTREAM_HOST_PATTERN`

#### Scenario: Path sin segmento de servicio
- **WHEN** llega un request cuyo path no matchea `^/([^/]+)` (p. ej. `/`)
- **THEN** el Gateway responde `404 Not Found` sin contactar ningún upstream

### Requirement: Validación de JWT RS256 en rutas protegidas

En toda ruta protegida el Gateway SHALL exigir un header `Authorization: Bearer <token>`, validar
la **firma RS256** con la clave pública (`JWT_PUBLIC_KEY`) y verificar que **no esté expirado**
(`exp`). Un token ausente, malformado, con firma inválida o expirado SHALL producir
`401 Unauthorized` con cuerpo JSON `{"status":401,"error":"Unauthorized","message":"..."}`.

#### Scenario: Token válido
- **WHEN** llega un request protegido con un Bearer token de firma válida y no expirado
- **THEN** el Gateway lo deja pasar al upstream con las cabeceras internas pobladas

#### Scenario: Sin header Authorization
- **WHEN** llega un request protegido sin `Authorization`
- **THEN** el Gateway responde `401` con `message="Missing Authorization header"`

#### Scenario: Formato no-Bearer
- **WHEN** el header `Authorization` no calza `^[Bb]earer\s+(.+)$`
- **THEN** el Gateway responde `401` con `message="Invalid Authorization format"`

#### Scenario: Token expirado o firma inválida
- **WHEN** el token está expirado o su firma no valida contra la clave pública
- **THEN** el Gateway responde `401` con `message="Invalid or expired token"`

#### Scenario: Clave pública no configurada
- **WHEN** `JWT_PUBLIC_KEY` no está disponible en el entorno del Gateway
- **THEN** el Gateway responde `500 Internal Server Error` (no degrada a "permitir")

### Requirement: Propagación de identidad como cabeceras internas

Tras validar el JWT el Gateway SHALL propagar al upstream `X-User-Id` (claim `sub`) y, cuando el
claim `roles` exista, `X-User-Roles` (UUIDs separados por coma). El Gateway SHALL **eliminar** el
header `Authorization` antes de reenviar, de modo que el token nunca llegue al MS.

#### Scenario: Claims propagados
- **WHEN** el JWT validado trae `sub=<uuid>` y `roles=[r1,r2]`
- **THEN** el upstream recibe `X-User-Id: <uuid>` y `X-User-Roles: r1,r2`, y **no** recibe `Authorization`

#### Scenario: Token sin roles
- **WHEN** el JWT no trae el claim `roles`
- **THEN** el upstream recibe `X-User-Id` pero `X-User-Roles` no se agrega

### Requirement: Rutas públicas sin JWT

El Gateway SHALL tratar como públicas (sin validación de JWT) las rutas de login, refresh y JWKS,
identificadas por patrón de path independiente del servicio:
`^/[^/]+/login$`, `^/[^/]+/refresh$`, `^/[^/]+/\.well-known/`. Estas resuelven upstream igual que
las protegidas pero **no** ejecutan la validación de token.

#### Scenario: Login es público
- **WHEN** llega `POST /auth/login` sin `Authorization`
- **THEN** el Gateway lo enruta a `auth` sin exigir token

#### Scenario: JWKS es público
- **WHEN** llega `GET /auth/.well-known/jwks.json`
- **THEN** el Gateway lo enruta a `auth` sin exigir token (la SPA y otros validadores leen las claves)

### Requirement: Rate limiting por IP

El Gateway SHALL limitar a **200 req/min por IP** (`limit_req_zone ... rate=200r/m`) con buckets de
burst por tipo de ruta, y responder `429 Too Many Requests` al excederse (`BE-GW-003`).

#### Scenario: Exceso de tasa
- **WHEN** una IP supera el límite configurado más su burst
- **THEN** el Gateway responde `429`

### Requirement: Correlation ID

El Gateway SHALL garantizar un `X-Correlation-ID` por request: respeta el header entrante si viene,
o genera uno desde el `request_id` de nginx. SHALL propagarlo al upstream y devolverlo al cliente
en la respuesta (`BE-TRV-004`).

#### Scenario: Cliente no envía correlation id
- **WHEN** el request no trae `X-Correlation-ID`
- **THEN** el Gateway genera uno, lo envía al upstream y lo incluye en la respuesta al cliente

#### Scenario: Cliente envía correlation id
- **WHEN** el request trae `X-Correlation-ID: abc`
- **THEN** el Gateway preserva `abc` hacia el upstream y en la respuesta

### Requirement: Provisión de la clave pública independiente del entorno

El Gateway SHALL obtener la clave pública RS256 de forma idéntica en lógica entre entornos: en local
desde un archivo montado (`JWT_PUBLIC_KEY_FILE` → cargado a `JWT_PUBLIC_KEY` por el entrypoint), en
Fly.io desde el secret `JWT_PUBLIC_KEY` ya presente. El Lua SHALL leer siempre `JWT_PUBLIC_KEY`.

#### Scenario: Arranque local con archivo de clave
- **WHEN** `JWT_PUBLIC_KEY_FILE` apunta a un archivo existente al iniciar el contenedor
- **THEN** el entrypoint carga su contenido en `JWT_PUBLIC_KEY` antes de arrancar nginx

#### Scenario: Archivo de clave declarado pero ausente
- **WHEN** `JWT_PUBLIC_KEY_FILE` está seteado pero el archivo no existe
- **THEN** el entrypoint aborta el arranque con un mensaje de error (no arranca sin poder validar)
