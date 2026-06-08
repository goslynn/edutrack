# Estructura de paquetes estándar — EduTrack Backend

Documento normativo para todos los microservicios del monorepo. Define la jerarquía de paquetes canónica, justifica cada decisión y establece equivalencias para migrar servicios que divergen del estándar.

---

## Estructura canónica

```
cl.duocuc.edutrack.ms.<servicio>/
├── infrastructure/               ← código de librería compartido (ver sección aparte)
│   ├── context/
│   ├── exception/
│   ├── jackson/
│   ├── security/
│   └── validation/
│
├── model/
│   ├── entity/                   ← entidades JPA + enums de dominio
│   └── dto/                      ← XxxRequest + XxxResponse + vistas/validaciones propias
│
├── repository/                   ← interfaces/clases Panache
├── resource/                     ← endpoints JAX-RS
├── security/                     ← contratos de seguridad específicos del MS
└── service/                      ← lógica de negocio
```

Paquete base: `cl.duocuc.edutrack.ms.<servicio>` (p. ej. `…ms.auth`, `…ms.attendance`).

---

## El paquete `infrastructure` — código de librería

`infrastructure` es código transversal copiado intencionalmente en cada MS hasta que se extraiga como artefacto `edutrack-commons`. **No importa ningún paquete `ms.<servicio>.*`; no contiene reglas de negocio.**

| Subpaquete | Responsabilidad |
|---|---|
| `infrastructure.context` | `InternalHeader` (enum), `RequestHeaders` (record), `RequestContext` (@RequestScoped), `HeaderValidationMode` |
| `infrastructure.exception` | `GlobalExceptionMappers`, `ErrorResponse`, `DomainException` + sugar (`ConflictException`, `NotFoundException`, `ForbiddenException`) |
| `infrastructure.jackson` | `Views` (jerarquía estándar de `@JsonView`), `JacksonCustomConfig` (vista por defecto = `Views.Base`) |
| `infrastructure.security` | `@RequirePermission`, `Permission` (enum de bits), `ResourceIds`, contrato `PermissionEvaluator`, `RequirePermissionFilter` |
| `infrastructure.validation` | `Validations` (`OnCreate`, secuencia `Create`) |

---

## El paquete `model`

### `model/entity/`

- Entidades JPA con Panache Active Record (`PanacheEntityBase`), PKs UUID, fetch LAZY.
- Superclases `@MappedSuperclass` también viven aquí (`CreatableEntity`, `AuditableEntity extends CreatableEntity`, etc.).
- **Enums de estado y tipo que pertenecen al ciclo de vida de la entidad van aquí** (p. ej. `AttendanceStatus`, `SessionStatus`). Son parte del modelo de datos, no del contrato HTTP.

### `model/dto/`

- Máximo **dos records por entidad/recurso**: `XxxRequest` y `XxxResponse`. No se crean records adicionales para granularidad diferente — se usa `@JsonView`.
- `XxxResponse` expone `fromEntity(XxxEntity)` (o `of(...)` si no respalda una entidad). La lógica de ensamble vive en el DTO, no en los callers.
- Enums que forman parte explícita del contrato HTTP/API (usados como campos en los DTOs, visibles en la documentación) también pueden vivir aquí.
- Vistas propias del dominio extienden la jerarquía de `infrastructure.jackson.Views` y se declaran aquí (p. ej. `AttendanceViews`, `AuthViews`).
- Grupos de validación específicos del dominio se declaran aquí (p. ej. `AttendanceValidations`, `AuthValidations`).

---

## El paquete `repository`

- Repositorios Panache: clases que extienden `PanacheRepository<E>` o métodos de query estáticos en la entidad.
- Convención de nombre: `XxxRepository`.
- **Vive en `repository/` directo bajo el servicio**, no dentro de `model/`. La razón: `model/` agrupa los objetos del modelo de datos (entidades + DTOs); los repositorios son el contrato de acceso a datos, una responsabilidad diferente que merece su propio nivel de visibilidad en el árbol de paquetes.

> **Nota de decisión:** Auth tenía los repositorios en `model/repository/`. El estándar los promueve un nivel arriba (`repository/` directo). Esto sigue la convención más habitual de Spring/Quarkus en proyectos con múltiples capas visibles y hace que la estructura de primer nivel sea simétrica: `model`, `repository`, `resource`, `service` son cuatro pilares al mismo nivel.

---

## El paquete `resource`

- Clases JAX-RS (`@Path`): una por entidad/recurso principal.
- Sólo coordinan: reciben request validado, delegan al `service/`, devuelven response con el `@JsonView` correcto.
- **No contienen lógica de negocio** ni acceden directamente a repositorios.
- Inicializadores/seeders de datos que deben ejecutarse en el arranque del servidor pueden vivir aquí como `DataInitializer` (anotado `@Startup` o `@Observes StartupEvent`). Alternativa aceptable: `service/XxxSeeder` si la lógica de inicialización es sustantiva.

---

## El paquete `service`

- Un `@ApplicationScoped` por aggregate root o agrupación funcional cohesiva.
- Las excepciones de dominio se lanzan directamente con el sugar de `infrastructure.exception`:

```java
throw new NotFoundException("ATTENDANCE.SESSION.NOT_FOUND", "Session %s not found".formatted(id));
throw new ConflictException("ATTENDANCE.RECORD.DUPLICATE", "Record already exists").with("sessionId", sessionId);
```

- **No se crean clases de excepción propias** para condiciones que ya cubren `ConflictException`, `NotFoundException` y `ForbiddenException`. Si el dominio necesita semántica adicional no cubierta por el código HTTP (p. ej. `422 Unprocessable`), se extiende `DomainException` directamente en el servicio — sin crear un paquete `exception/` propio.

---

## Subpaquetes por feature dentro de `service/`

Cuando una pieza de lógica (estrategia, factory, policy, calculadora, validador de invariantes, builder complejo) existe **sólo para que un service haga su trabajo**, vive en un subpaquete de `service/` nombrado por la **feature**, no por el patrón.

```
service/
├── AttendanceService.java
└── grading/                       ← subpaquete por feature, no por patrón
    ├── GradingService.java
    ├── GradingStrategy.java
    ├── WeightedAverageStrategy.java
    └── GradingStrategyFactory.java
```

**Reglas:**

- El subpaquete se nombra por el *qué* (`grading`, `enrollment`, `scheduling`), no por el *cómo* (`strategy/`, `factory/`, `helpers/`).
- Las clases internas del subpaquete son **package-private** siempre que se pueda. Si sólo la factory instancia las estrategias, nadie fuera del subpaquete debería verlas.
- Test de pertenencia: *¿esta clase tiene sentido fuera de la feature?* Si no, va en el subpaquete. Si sí y no tiene dependencias de dominio, candidata a `infrastructure.*` (con su propio subpaquete técnico, p. ej. `infrastructure.paging`). En la duda, ponela en la feature — promoverla después es trivial.
- Value objects o cálculos puros que forman parte del **vocabulario del dominio** (p. ej. `Grade`, `WeightedScore`) pueden vivir en `model/` si son lenguaje común del servicio; si son específicos de una feature, en su subpaquete.

**Prohibido:** crear paquetes `util/`, `helpers/`, `common/`, `misc/` ni equivalentes. Describen el *cómo* (cosas misceláneas) en lugar del *qué*, no acotan visibilidad, y crecen sin criterio hasta volverse un cajón de sastre. Si aparece la tentación, casi siempre el código pertenece a un subpaquete de feature o a `infrastructure.*`.

---

## El paquete `security`

- Contiene **código de seguridad específico del MS**, no la infraestructura transversal.
- Artefactos típicos:
  - `XxxResourceId` — constantes `String`/`UUID` de los recursos registrados en Auth (p. ej. `AuthResourceId`).
  - Implementación concreta de `PermissionEvaluator` si el MS la necesita.
- **No** contiene filtros de cabeceras, contextos de request ni lógica de JWT: eso pertenece a `infrastructure.context` e `infrastructure.security`.

