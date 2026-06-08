# Observability Specification

## Purpose

Salud, correlación de logs y trazabilidad distribuida transversales a todos los MS. Implementa
`BE-TRV-003` (health/readiness), `BE-TRV-004` (logging con correlationId), `BE-TRV-005`
(OpenTelemetry), `INF-MON-001/002`.

Estado: health checks y `X-Correlation-ID` (gateway) listos; OpenTelemetry/tracing parcial.

## Requirements

### Requirement: Health y readiness por servicio

Cada MS SHALL exponer endpoints de salud (SmallRye Health): liveness y readiness. El readiness SHALL
reflejar dependencias críticas (p. ej. la BD): un MS con BD caída SHALL reportarse NotReady. El stack
local usa `/q/health/ready` como healthcheck de contenedor.

#### Scenario: Readiness con BD caída
- **WHEN** la BD no responde
- **THEN** `/q/health/ready` reporta DOWN y el orquestador marca el pod/contenedor NotReady

#### Scenario: Liveness independiente de upstreams
- **WHEN** un upstream del MS está caído pero el proceso vive
- **THEN** el liveness sigue UP (no se reinicia el pod por una dependencia externa)

### Requirement: Correlation ID de extremo a extremo

El Gateway SHALL garantizar un `X-Correlation-ID` por request (respeta el entrante o genera uno) y
propagarlo al upstream y al cliente. Los MS SHALL incluir el `correlationId` en sus logs para que una
búsqueda por ese id recupere todos los eventos del request (`BE-TRV-004`).

#### Scenario: Trazar un request por logs
- **WHEN** se busca por un `X-Correlation-ID` concreto
- **THEN** aparecen los eventos de log de ese request a través de los servicios que tocó

### Requirement: Logging estructurado

Los logs SHOULD ser estructurados (JSON) y centralizables (p. ej. CloudWatch), incluyendo
`correlationId` y nivel. Los errores de dominio se loguean con su `code`; los fallos fail-closed de
autorización se loguean como WARN.

#### Scenario: Fallo de autorización logueado
- **WHEN** `RemotePermissionEvaluator` cae a fail-closed por error de `/auth/access`
- **THEN** se emite un log WARN con el `resourceKey` y los bits requeridos

### Requirement: Trazabilidad distribuida (OpenTelemetry)

Los MS SHOULD emitir trazas OpenTelemetry de modo que un request se siga a través de los servicios en
una herramienta APM (`BE-TRV-005`). Los REST Clients declarativos heredan instrumentación OTel.

#### Scenario: Traza distribuida completa
- **WHEN** un request atraviesa Gateway → MS-x → MS-y
- **THEN** la traza correlacionada es visible en la herramienta APM

### Requirement: Retención y alertas de monitoreo

El entorno productivo SHALL retener logs (criterio: 365 días) y configurar alertas sobre métricas
clave (CPU > 80%, error rate > 5%, DLQ con mensajes) (`INF-MON-001/002`).

#### Scenario: DLQ con mensajes dispara alerta
- **WHEN** la DLQ acumula mensajes
- **THEN** se dispara una alerta de monitoreo
