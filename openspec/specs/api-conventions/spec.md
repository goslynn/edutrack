# API Conventions (DTOs, Views, Validation) Specification

## Purpose

Convenciones transversales para la forma de la API HTTP: dos DTOs por recurso, granularidad de campos
con `@JsonView` sobre una jerarquía estándar, granularidad de validación con validation groups, y el
contrato `fromEntity`/`of`. Garantiza consistencia de payloads entre MS e implementa `TRV-DOC-001`
(OpenAPI), `TRV-SEC-003` (validación de inputs).

Fuentes: `commons/.../infrastructure/jackson/{Views,JacksonCustomConfig}.java`,
`commons/.../infrastructure/validation/Validations.java`, `doc/estructura_paquetes.md`, CLAUDE.md.

## Requirements

### Requirement: Máximo dos DTOs por recurso

Cada entidad/recurso SHALL exponer a lo sumo **un `XxxRequest` y un `XxxResponse`**. La granularidad
(qué campos viajan en qué endpoint) NO se modela creando records adicionales sino con `@JsonView`.

#### Scenario: Distintas vistas, un solo Response
- **WHEN** un recurso necesita un payload reducido en listados y completo en `GET /{id}`
- **THEN** usa un único `XxxResponse` con campos anotados por vista, no dos records

### Requirement: Jerarquía estándar de vistas

La granularidad de campos SHALL modelarse con `@JsonView` sobre la jerarquía compartida `Views`:
`Base`, `Extra`, `Detailed extends Base`, `Create extends Base`, `List extends Base,Extra`,
`Patch extends Base,Extra`, `Update extends Base`, `Admin extends Base,Extra`, `Internal`. Cada MS
PUEDE extender con vistas de dominio (p. ej. `AuthViews.Login extends Views.Base`).

#### Scenario: Herencia de vistas
- **WHEN** un endpoint se anota `@JsonView(Views.Detailed.class)`
- **THEN** también serializa los campos marcados `Views.Base` (por herencia)

### Requirement: Base como vista por defecto, sin inclusión implícita

`JacksonCustomConfig` SHALL fijar `Views.Base` como vista por defecto del `ObjectMapper` y
deshabilitar `DEFAULT_VIEW_INCLUSION`. Consecuencia: un campo **sin** `@JsonView` NO se serializa ni
deserializa; los endpoints cuya vista es `Base` NO necesitan anotación, los demás sí (override
per-request en método y/o parámetro de body).

#### Scenario: Campo sin vista se omite
- **WHEN** un DTO tiene un campo sin `@JsonView`
- **THEN** ese campo no aparece en la serialización ni se lee en deserialización

#### Scenario: Tolerant reader inter-servicio
- **WHEN** un consumidor declara un DTO mínimo para leer una respuesta ajena
- **THEN** cada campo lleva `@JsonView(Views.Base.class)` (si no, Jackson no lo deserializa)

### Requirement: Contrato fromEntity / of

Todo `XxxResponse` SHALL exponer un factory estático que sabe construirse desde su fuente, confinando
la instanciación al propio DTO: `fromEntity(XxxEntity)` cuando respalda una entidad (con colaborador
extra como parámetro si la entidad no basta), u `of(...)` cuando es un resultado computado
(`AuthResponse`, `AccessResponse`). La lógica de ensamble (labels derivados, constantes de protocolo)
SHALL vivir en el DTO, no en los call sites.

#### Scenario: Construcción desde entidad
- **WHEN** un endpoint devuelve un recurso a partir de su entidad
- **THEN** llama `XxxResponse.fromEntity(entity)`, no `new XxxResponse(...)`

#### Scenario: Resultado computado
- **WHEN** se devuelve un resultado sin entidad de respaldo (tokens emitidos, acceso evaluado)
- **THEN** se usa `XxxResponse.of(...)` y el DTO deriva los campos calculados (p. ej. label rwx)

### Requirement: Toda validación de request por Bean Validation

La validación de datos del request SHALL declararse con anotaciones de Jakarta Bean Validation
(`@NotBlank`, `@Email`, `@Size`, `@Min`/`@Max`, `@Pattern`) sobre los componentes del record y
dispararse con `@Valid`. Está **prohibido** validar datos del request con `if` en endpoints o en la
capa de servicio. La `ConstraintViolationException` se mapea a `400` automáticamente. (Quedan fuera:
guards de autenticación/identidad y reglas de negocio como unicidad/invariantes de dominio.)

#### Scenario: Campo inválido
- **WHEN** llega un body con un email malformado y el campo tiene `@Email`
- **THEN** la respuesta es `400` sin ningún `if` manual en el endpoint

### Requirement: Validación condicional con validation groups

La validación condicional por endpoint sobre un único record (obligatorio en `Create`, opcional en
`Update`) SHALL modelarse con validation groups, no con checks manuales: restricciones de **formato**
sin grupo (siempre, null-safe); restricciones de **presencia** (`@NotNull`/`@NotBlank`) con
`groups = Validations.OnXxx.class`. El endpoint dispara la secuencia con
`@Valid @ConvertGroup(from = Default.class, to = Validations.Xxx.class)`. La `@GroupSequence` ejecuta
formato primero y presencia después (cortocircuita al primer fallo).

#### Scenario: Campo obligatorio solo en creación
- **WHEN** un `POST` de creación usa `@ConvertGroup(to = Validations.Create.class)` y falta un campo marcado `OnCreate`
- **THEN** la respuesta es `400`; el mismo record en `PATCH` (solo `@Valid`) acepta su ausencia

### Requirement: OpenAPI por servicio

Cada MS SHALL exponer su contrato OpenAPI 3.0 (SmallRye OpenAPI) con Swagger UI en `/q/swagger-ui` en
dev (`TRV-DOC-001`). Los endpoints y DTOs SHOULD anotarse con metadatos OpenAPI relevantes.

#### Scenario: Contrato disponible en dev
- **WHEN** un MS corre en dev mode
- **THEN** su OpenAPI está disponible y Swagger UI en `/q/swagger-ui`
