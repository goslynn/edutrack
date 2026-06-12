# EduTrack — OpenSpec

Especificaciones **spec-driven** del backend de EduTrack en convención
[OpenSpec](https://github.com/Fission-AI/OpenSpec). Formalizan los **contratos implícitos
entre servicios** y las **decisiones de arquitectura y diseño** que hoy viven dispersas en
código, Javadoc, `CLAUDE.md` y `doc/`, y las convierten en requisitos verificables.

## Layout

```
openspec/
├── project.md                 Contexto: stack, arquitectura, ADRs, convenciones
├── README.md                  Este archivo
└── specs/
    ├── api-gateway/                  Punto de entrada único: routing, JWT, propagación, rate-limit
    ├── service-discovery/            Naming = discovery, ServiceIds, patrón Fly.io
    ├── request-context/              Cabeceras internas X-User-*, RequestContext, super-user
    ├── authorization/                Permisos Unix-style, resource_key, @RequirePermission, /auth/access
    ├── inter-service-communication/  Clients declarativos, propagación de identidad y errores, fail-closed
    ├── error-handling/               Envelope ErrorResponse, DomainException, códigos de dominio
    ├── data-persistence/             Shared DB schema-per-service, auditoría, Flyway, UUID PK
    ├── api-conventions/              DTOs, @JsonView Views, validation groups, fromEntity
    ├── async-events/                 Eventos RabbitMQ, forma del payload, DLQ
    ├── observability/                Health, X-Correlation-ID, OpenTelemetry, logging
    ├── auth-service/                 Dominio: identidad, JWT, refresh, roles, grants
    ├── attendance-service/           Dominio: sesiones y registros de asistencia
```

Las primeras 10 capabilities son **contratos de plataforma** (transversales, los que un MS
nuevo debe honrar). Las últimas son **dominios**.

## Formato de una spec

Cada `spec.md` sigue esta estructura:

```markdown
# <Capability> Specification

## Purpose
Una o dos frases sobre qué cubre esta capability y por qué existe.

## Requirements

### Requirement: <nombre corto e imperativo>
El sistema SHALL/MUST ... (una obligación clara, con verbos RFC-2119).

#### Scenario: <caso concreto>
- **WHEN** <condición / disparador>
- **THEN** <resultado observable y verificable>
- **AND** <resultado adicional>
```

Reglas:
- **SHALL / MUST** = obligación dura (rompe el contrato si falla). **SHOULD** = recomendación
  fuerte. **MAY** = opcional.
- Cada `### Requirement` lleva **al menos un** `#### Scenario`.
- Los escenarios son verificables (un test o una observación los confirma).
- Cuando aplica, cada requirement referencia el ID de `doc/requisitos_libro_clases.csv`
  (`BE-AUTH-005`, etc.) y/o la clase/archivo que lo implementa.
  
