# Inter-Service Communication Specification

## Purpose

Cómo un MS-x le habla a un MS-y de forma síncrona (este–oeste): clients declarativos tipados que
retornan `Response`, propagación automática de identidad, propagación transparente de errores de
dominio y resiliencia (ADR-007, ADR-008). Las llamadas son directas app-a-app por DNS de Fly.io, no
pasan por el Gateway.

Fuentes: `commons/.../clients/{AuthClient,AttendanceClient,package-info}.java`,
`commons/.../infrastructure/discovery/{IdentityHeadersFactory,HTTPClientUtils}.java`.

## Requirements

### Requirement: Clients declarativos por servicio

Cada MS-y SHALL exponer su SDK de consumo como interfaz MicroProfile REST Client **declarativa** en
`cl.duocuc.edutrack.ms.clients.<servicio>`, anotada con `@RegisterRestClient(configKey = ServiceIds.X)`
y `@RegisterClientHeaders(IdentityHeadersFactory.class)`. NO se escribe productor CDI ni construcción
manual: se inyecta con `@Inject @RestClient`. La URL la deriva `DiscoveryConfigSourceFactory` (ver
spec `service-discovery`). Los paths SHALL prefijarse con `"/"+ServiceIds.X` (el MS monta su
`@ApplicationPath` bajo ese segmento).

#### Scenario: Inyección sin configuración
- **WHEN** un MS inyecta `@Inject @RestClient AuthClient`
- **THEN** el client resuelve su URL desde el patrón de discovery sin que el MS declare nada más

### Requirement: Estilo Response + tolerant reader

Los métodos de client SHALL retornar `jakarta.ws.rs.core.Response` y, cuando lleven body de request,
recibirlo como `Object`: el client se mantiene genérico y no se acopla a los DTOs de dominio del MS-y
(que viven en su propio MS, fuera de la librería). La **forma** de la respuesta la define el
**consumidor** con su propio DTO `@JsonIgnoreProperties(ignoreUnknown = true)` y campos
`@JsonView(Views.Base.class)`. Así el productor evoluciona su payload sin romper consumidores.

#### Scenario: Consumidor bindea su propio slice
- **WHEN** dos MS distintos consumen el mismo endpoint de Course
- **THEN** cada uno declara su propio DTO mínimo (tolerant reader) sin compartir un DTO canónico

### Requirement: Propagación automática de identidad

`IdentityHeadersFactory` SHALL reenviar `X-User-Id`/`X-User-Roles` desde el `RequestContext` del
request entrante hacia toda llamada saliente, sin `@HeaderParam("X-...")` manuales. Si el request no
trae identidad, no se agregan headers (la llamada sale anónima y el upstream decide).

#### Scenario: Identidad reenviada en cadena
- **WHEN** un request autenticado en MS-x dispara una llamada a MS-y vía client
- **THEN** MS-y recibe `X-User-Id`/`X-User-Roles` idénticos a los que recibió MS-x

#### Scenario: Llamada anónima
- **WHEN** no hay identidad en el request entrante
- **THEN** la llamada saliente no incluye cabeceras de identidad

### Requirement: Propagación transparente de errores de dominio

`HTTPClientUtils.readOrThrow(Response, type)` SHALL extraer el cuerpo en 2xx, o en no-2xx
**reconstruir** una `DomainException` desde el envelope `ErrorResponse` del upstream, preservando su
`code`, `status` y `metadata` de dominio. Como un método que retorna `Response` no dispara los
`ResponseExceptionMapper`, el chequeo de status SHALL centralizarse aquí. Si el cuerpo no es el
envelope estándar, SHALL caer a una `DomainException` genérica con el status HTTP.

#### Scenario: Code de dominio preservado
- **WHEN** MS-y responde `409` con `code="AUTH.USER.EMAIL_EXISTS"` y MS-x usa `readOrThrow`
- **THEN** MS-x recibe una `DomainException` con el mismo `code` y `status`, que su handler global vuelve a serializar idéntica

#### Scenario: Respuesta no-envelope
- **WHEN** el upstream devuelve un cuerpo no-2xx que no es `ErrorResponse`
- **THEN** `readOrThrow` lanza `DomainException` genérica con "Upstream service returned HTTP <status>"

#### Scenario: Acceso a status/headers crudos
- **WHEN** un endpoint necesita el status o headers de la respuesta del upstream
- **THEN** usa la `Response` cruda directamente, no `readOrThrow`

### Requirement: Resiliencia en llamadas inter-servicio

Los métodos de client que lo ameriten SHOULD declarar SmallRye Fault Tolerance (`@Timeout`,
`@CircuitBreaker`) para acotar latencia y abrir circuito ante fallas repetidas del upstream. La
política de degradación SHALL ser fail-closed cuando afecte una decisión de autorización (ver spec
`authorization`).

#### Scenario: Circuito abierto
- **WHEN** un upstream supera el umbral de fallas configurado en el client
- **THEN** el circuit breaker abre y las llamadas fallan rápido sin esperar el timeout completo
