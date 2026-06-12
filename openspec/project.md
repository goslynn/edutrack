# EduTrack — Project Context

> Contexto del proyecto para humanos y agentes que trabajan sobre las especificaciones
> de este monorepo. Este archivo describe **qué** es EduTrack, **cómo** está construido y
> las **decisiones de arquitectura** que las specs de `openspec/specs/` formalizan en
> requisitos verificables.

## Purpose

EduTrack es la **plataforma de libro de clases digital** para instintuciones educativas de Chile.
Reemplaza el libro de clases físico por un backend de microservicios que
gestiona identidad, alumnos, cursos, contenido, evaluaciones, asistencia, anotaciones,
notificaciones y reportes.

Es a la vez un sistema real y el entregable de la asignatura **DSY1106 — Desarrollo
Fullstack III** (Duoc UC). La matriz de requisitos vive en
`doc/requisitos_libro_clases.csv` (IDs `BE-*`, `FE-*`, `INF-*`, `TRV-*`); cada spec en
`openspec/specs/` referencia los IDs que materializa.

## Tech Stack

| Capa | Tecnología |
|---|---|
| Lenguaje / runtime | Java 21 |
| Framework | Quarkus 3.x (Quarkus REST / JAX-RS, ArC CDI, Panache) |
| Persistencia | PostgreSQL — **instancia única compartida, schema-per-service** |
| Migraciones | Flyway (`src/main/resources/db/migration/`) |
| Auth | JWT **RS256** (SmallRye JWT), permisos Unix-style |
| Mensajería | RabbitMQ (eventos asíncronos; hoy stub de log en Student) |
| Resiliencia | SmallRye Fault Tolerance (`@Timeout`, `@CircuitBreaker`) |
| Cliente HTTP | MicroProfile REST Client declarativo |
| API Gateway | OpenResty (nginx + Lua) |
| Observabilidad | SmallRye Health, OpenTelemetry, logging con `X-Correlation-ID` |
| Frontend | React + Vite + TypeScript + Tailwind (`front/`) |
| Empaquetado | Docker (multi-stage, Dockerfile.jvm) |
| Hosting | Fly.io (cada MS es una app; DNS privado `*.fly.internal`) |

## Monorepo layout

Cada subdirectorio de primer nivel es un proyecto Maven independiente con su propio
`pom.xml` y su propio `CLAUDE.md` de dominio.

```
edutrack/
├── auth/          MS Auth — JWT, usuarios, roles, permisos Unix-style, /auth/access  (schema auth)
├── student/       MS Student — alumnos, apoderados, eventos de ciclo de vida          (schema student)
├── attendance/    MS Attendance — sesiones y registros de asistencia                  (schema attendance)
├── commons/       Librería edutrack-ms-commons — infrastructure.* + clients.*  (instalada en ~/.m2)
├── infra/         API Gateway (OpenResty) + fly.toml
├── front/         SPA React/Vite
├── doc/           Matriz de requisitos, permisos.md, estructura_paquetes.md
├── .certs/        Llaves RS256 (gitignored): privateKey.pem + publicKey.pem
├── docker-compose.yml   ÚNICA fuente de verdad del stack local
└── openspec/      ← estas especificaciones
```

Microservicios planificados aún no implementados (ya reservados en `ServiceIds`):
**Course, Content, Assessment, Annotation, Notification, Report**.

## Architecture (vista de 10.000 pies)

```
                          ┌────────────────────────────────────────────┐
   Cliente (SPA / app) ──▶│ API Gateway (OpenResty)                     │
   Authorization: Bearer  │  · valida firma JWT RS256 (clave pública)   │
                          │  · enruta por 1er segmento del path         │
                          │  · propaga X-User-Id / X-User-Roles         │
                          │  · elimina Authorization                    │
                          │  · rate-limit, X-Correlation-ID             │
                          └───────┬──────────┬──────────┬──────────────┘
                                  │          │          │   (DNS Fly.io interno)
                            ┌─────▼───┐ ┌────▼────┐ ┌───▼──────┐
                            │  auth   │ │ student │ │attendance│  … (más MS)
                            └────┬────┘ └────┬────┘ └────┬─────┘
                                 │           │           │
       inter-service REST  ◀─────┼───────────┘           │   (app→app, NO pasa por gateway,
       (commons clients,         │  GET /auth/access     │    reenvía X-User-* automáticamente)
        fail-closed)             ▼                       ▼
                          ┌──────────────────────────────────────┐
                          │ PostgreSQL — 1 instancia, schema/MS   │  RabbitMQ (eventos async)
                          │ credenciales exclusivas por schema    │
                          └──────────────────────────────────────┘
```

**Dos planos de llamada coexisten:**
- **Norte–sur (cliente → sistema):** siempre por el Gateway. El cliente manda `Bearer <JWT>`;
  el Gateway valida y traduce a cabeceras internas.
- **Este–oeste (MS → MS):** directo app-a-app por DNS de Fly.io, **sin** pasar por el Gateway.
  La identidad se reenvía con las mismas cabeceras internas vía `IdentityHeadersFactory`.

## Architectural Decision Records (ADR)

Decisiones cerradas que las specs asumen como invariantes:

1. **ADR-001 — Shared DB, schema-per-service.** Una instancia PostgreSQL/RDS, un schema por
   MS, credenciales exclusivas por schema (aislamiento *técnico*, no de honor). Report Service
   tendrá credenciales read-only sobre todos los schemas para reportes cross-dominio.
   Trade-off aceptado: migraciones coordinadas, menor autonomía de despliegue de schema.
   (Reemplaza database-per-service. Reqs `BE-RPT-005`, `INF-DB-001`.)
2. **ADR-002 — Autorización Unix-style por *tipo* de recurso.** Auth decide "¿este rol puede
   hacer X sobre el tipo Y?"; el MS dueño del dato decide "¿sobre *esta instancia*?". Flags
   `r=4/w=2/x=1`, OR sobre roles + comodín `ALL`. Ver spec `authorization`.
3. **ADR-003 — `resource_key` de texto como contrato (no UUID).** El recurso se nombra con una
   clave estable `"<servicio>.<recurso>"` (p. ej. `auth.users`), opaca para Auth y comparada
   por igualdad. El string **es** el contrato: idéntico en ambos lados de un grant, sin UUIDs
   que coordinar. (Evolución desde el modelo `resource_uuid` que aún describe `doc/permisos.md`.)
4. **ADR-004 — Identidad propagada por cabeceras internas.** El Gateway es el único que valida
   el JWT; los MS **confían** en `X-User-Id`/`X-User-Roles` porque solo el Gateway puede
   inyectarlas. Los MS no re-validan el token en cada request. Ver spec `request-context`.
5. **ADR-005 — Contrato de naming = descubrimiento.** El primer segmento del path debe coincidir
   con el nombre lógico del servicio (= nombre de la app en Fly.io sin prefijo `edutrack-`). El
   Gateway no tiene lista de servicios; el naming es el único mecanismo de discovery. Ver specs
   `api-gateway` y `service-discovery`.
6. **ADR-006 — `commons` duplicado intencionalmente → futuro artefacto.** `infrastructure.*` y
   `clients.*` viven en la librería `edutrack-ms-commons`; `infrastructure.*` **no** importa
   `ms.<servicio>.*` (sin dominio). Puntos de extensión específicos del MS se exponen como
   contratos CDI (`PermissionEvaluator`, `SuperUserResolver`).
7. **ADR-007 — Clients tipados que retornan `Response` + tolerant reader.** El client declara
   path/verbo/headers tipados pero retorna `jakarta.ws.rs.core.Response`; cada consumidor extrae
   con su propio DTO `@JsonIgnoreProperties(ignoreUnknown=true)`. El productor puede evolucionar
   su payload sin romper consumidores. Ver spec `inter-service-communication`.
8. **ADR-008 — Fail-closed en autorización inter-servicio.** Cualquier fallo evaluando permisos
   (timeout, 5xx, Auth caído) se traduce a "denegado", nunca a "permitido".

## Conventions

> Las convenciones de detalle (paquetes, DTOs, validación, errores) están normadas en
> `doc/estructura_paquetes.md`, `doc/permisos.md` y los `CLAUDE.md`. Resumen de lo que las
> specs dan por sentado:

- **Paquete base:** `cl.duocuc.edutrack.ms.<servicio>`.
- **Estructura canónica:** `model/{entity,dto}`, `repository/`, `resource/`, `service/`,
  `security/`, más `infrastructure/` (copia de `commons`). Sin paquetes `util/`/`helpers/`/`common/`.
- **Entidades:** Panache active record, PK UUID, fetch LAZY, superclases `@MappedSuperclass`
  (`CreatableEntity` → `AuditableEntity`) para auditoría.
- **DTOs:** máximo **2 por recurso** (`XxxRequest` + `XxxResponse`); granularidad de campos con
  `@JsonView` sobre la jerarquía `Views`; granularidad de validación con validation groups.
  `XxxResponse.fromEntity(...)` / `of(...)` confina la instanciación al propio DTO.
- **Validación:** solo Bean Validation de Jakarta (`@Valid` + grupos). Prohibido validar datos
  del request con `if` en endpoint o servicio. Reglas de negocio (unicidad, invariantes) sí son
  checks explícitos en su capa.
- **Errores:** handler global único + envelope `ErrorResponse`; reglas de negocio lanzan
  `DomainException` (sugar `ConflictException`/`NotFoundException`/`ForbiddenException`) con `code`
  estable `<MS>.<ENTIDAD>.<CONDICION>`.
- **Migraciones:** Flyway; `quarkus.hibernate-orm.database.generation=validate` en dev, `none` en prod.
- **Config:** variables de entorno con defaults de desarrollo en `application.properties`
  (`${DB_PASSWORD:dev_pass}`). Sin `.env` versionado ni secretos en el repo.
- **Comandos por MS:** `./mvnw quarkus:dev` · `./mvnw package` · `./mvnw test` · `./mvnw verify`
  · `./mvnw test -Dtest=NombreDelTest`.
- **Stack local:** `docker compose up` desde la raíz (única fuente de verdad). Requiere `.env`
  (copiar de `.env.example`) y llaves RS256 en `./.certs`.

## How to read these specs

Ver `openspec/README.md` para la convención de formato. En resumen: cada capability tiene un
`spec.md` con `## Requirements`; cada `### Requirement` enuncia una obligación con verbos
RFC-2119 (SHALL/MUST/SHOULD) y se acompaña de uno o más `#### Scenario` en formato
WHEN/THEN verificables.
