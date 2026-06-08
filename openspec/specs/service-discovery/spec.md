# Service Discovery Specification

## Purpose

Define cómo un microservicio resuelve la URL de otro **sin** un registry (Consul/Eureka): un único
**patrón de discovery** más un catálogo de nombres lógicos materializa, en build/arranque, una URL
de REST Client por servicio. Es el sustento este–oeste del sistema y el contrato de naming que el
Gateway también usa norte–sur (ADR-005).

Fuentes: `commons/.../infrastructure/discovery/ServiceIds.java`,
`DiscoveryConfigSourceFactory.java`, `commons/.../clients/package-info.java`.

## Requirements

### Requirement: Nombre lógico bare como única fuente de verdad

El catálogo `ServiceIds` SHALL contener el **nombre lógico sin prefijo ni puerto** de cada MS
(`auth`, `course`, `student`, `content`, `assessment`, `attendance`, `annotation`, `notification`,
`report`). Ese valor DEBE ser idéntico al primer segmento del path del Gateway, al `configKey` del
REST Client y al token `{service}` del patrón. El prefijo (`edutrack-`) y el dominio
(`.fly.internal:8080`) los aporta el patrón, no la constante — así un mismo valor sirve en local y
en Fly.io.

#### Scenario: Mismo id en los tres planos
- **WHEN** se agrega el servicio `course`
- **THEN** `ServiceIds.COURSE = "course"`, el Gateway enruta `/course/...` y el client usa
  `@RegisterRestClient(configKey = ServiceIds.COURSE)` — todos con el literal `"course"`

### Requirement: Derivación de URLs de REST Client desde el patrón

`DiscoveryConfigSourceFactory` SHALL publicar, por cada id de `ServiceIds.ALL`, una propiedad
`quarkus.rest-client.<id>.url` expandiendo `{service}` en el patrón `edutrack.discovery.pattern`
(default `edutrack-{service}.fly.internal:8080`) con el esquema `edutrack.discovery.scheme`
(default `http`). Así un MS no escribe a mano la URL de cada uno de los ~9 servicios; el patrón es
la única fuente de verdad.

#### Scenario: URL derivada en producción
- **WHEN** el patrón es el default y el esquema `http`
- **THEN** se genera `quarkus.rest-client.auth.url=http://edutrack-auth.fly.internal:8080`

#### Scenario: Override local profile-aware
- **WHEN** corre con perfil `dev` y `%dev.edutrack.discovery.pattern={service}:8080`
- **THEN** se genera `quarkus.rest-client.auth.url=http://auth:8080` (el perfil gana sobre el default)

### Requirement: Precedencia configurable por debajo del MS

El ConfigSource generado SHALL publicarse con ordinal `150`, **debajo** del `application.properties`
del MS (250). Un MS PUEDE pisar la URL de un servicio puntual declarando explícitamente
`quarkus.rest-client.<id>.url`; lo derivado solo cubre lo que el MS no fijó a mano.

#### Scenario: Override explícito de un servicio
- **WHEN** un MS declara `quarkus.rest-client.report.url=http://host-especial:9000` en su `application.properties`
- **THEN** ese valor gana sobre el derivado del patrón para `report`, sin afectar a los demás

### Requirement: Registro automático vía ServiceLoader

El factory SHALL descubrirse por `ServiceLoader`
(`META-INF/services/io.smallrye.config.ConfigSourceFactory`) incluido en el JAR de `commons`.
Cualquier MS que dependa de la librería lo obtiene automáticamente sin declarar ni registrar nada.

#### Scenario: MS nuevo sin configuración de discovery
- **WHEN** un MS agrega `edutrack-ms-commons` como dependencia y no configura discovery
- **THEN** los REST Clients declarativos resuelven su URL desde el patrón sin código adicional

### Requirement: Catálogo completo obligatorio

`ServiceIds.ALL` SHALL contener todos los ids del catálogo. Al agregar un MS, su constante DEBE
sumarse tanto como constante como en `ALL`; de lo contrario su `quarkus.rest-client.<id>.url` no se
genera y el client declarativo falla en runtime con "no URL configured".

#### Scenario: Id ausente de ALL
- **WHEN** se agrega `ServiceIds.COURSE` pero no se incluye en `ALL`
- **THEN** un client `configKey=COURSE` falla en runtime por URL no configurada (defecto detectable en arranque/test)
