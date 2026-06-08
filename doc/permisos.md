# Permisos en EduTrack — cómo funcionan

Documento de referencia para el equipo. Explica el modelo de autorización del backend de EduTrack: qué decide Auth, qué decide cada microservicio dueño del dato, y cómo se reparte la responsabilidad cuando aparece la distinción "recurso propio" vs "recurso de otro".

---

## 1. El modelo en una frase

Autorización **Unix-style por tipo de recurso**: cada par `(rol, tipo_de_recurso)` lleva un flag numérico `r=4 / w=2 / x=1` (0–7); la decisión es OR de flags sobre los roles del usuario, más un comodín `ALL` que cubre cualquier recurso.

Auth contesta **"¿este rol puede hacer la acción X sobre el *tipo* Y?"**. **No** contesta "¿sobre *esta instancia* concreta?" — eso es asunto del MS dueño del dato.

---

## 2. Las piezas

### 2.1. Identidad (la propaga el API Gateway)

- El usuario se autentica contra **Auth Service**, que emite un **JWT RS256** con `sub` (UUID del usuario) y `roles[]` (UUIDs de rol).
- El **API Gateway** es el único punto de entrada: valida la firma del JWT y propaga la identidad ya autenticada a cada MS como **cabeceras internas**:
  - `X-User-Id` — UUID del usuario.
  - `X-User-Roles` — UUIDs de rol separados por coma.
- Los MS **confían** en esas cabeceras porque solo el Gateway puede inyectarlas. **No vuelven a consultar a Auth en cada request** para autenticar — solo lo consultan en login/refresh/revocación y, opcionalmente, para preguntar por un permiso puntual.

### 2.2. Lectura de cabeceras dentro de cada MS

Cada MS expone **un único intérprete** de esas cabeceras en `infrastructure.context` — no se leen a mano en endpoints ni filtros.

| Tipo | Rol |
|---|---|
| `InternalHeader` (enum) | Única fuente de verdad de los nombres `X-User-Id`, `X-User-Roles`. |
| `RequestHeaders` (record) | Value object inmutable: `Optional<UUID> userId`, `List<UUID> roleIds`. Helpers `hasIdentity()` y `requireUserId()` (⇒ `401` si falta). |
| `RequestContext` | Bean `@RequestScoped` que interpreta y valida **una sola vez por request** (`@PostConstruct`) y expone el record vía `headers()`. |

Reglas:

- Cabecera **ausente** ⇒ valor vacío. No es error de validación; la decisión de exigir identidad la toma el consumidor (`requireUserId()` ⇒ `401`).
- Cabecera **presente pero malformada** ⇒ depende de `edutrack.headers.validation.mode`:
  - `EAGER` (default): aborta con `400`.
  - `WARN`: loguea y trata como ausente.

### 2.3. Recursos (cada MS define los suyos)

- Cada MS declara sus tipos de recurso como UUIDs hardcoded en un enum local. En Auth: `cl.duocuc.edutrack.ms.auth.security.AuthResourceId` (`USERS`, `ROLES`, `PERMISSIONS`).
- En un futuro Course tendría su `CourseResourceId.ASIGNATURA = <uuid>`, etc.
- El UUID es **opaco para Auth**: Auth no sabe que "asignatura" existe, solo guarda el string del UUID en la tabla de grants.

El **comodín** `ResourceIds.ALL = 00000000-0000-0000-0000-000000000000` vive en `infrastructure.security` porque es transversal a todos los MS: un grant sobre `ALL` cubre cualquier recurso, presente o futuro.

### 2.4. Grants (la tabla que materializa quién puede qué)

Tabla `auth.role_permissions`:

```
role_id            | resource_uuid                          | flags
-------------------|----------------------------------------|------
<SUPERUSER-uuid>   | 00000000-0000-0000-0000-000000000000   | 7   (rwx sobre ALL)
<ADMIN-uuid>       | <USERS-uuid>                           | 7   (rwx sobre USERS)
<DOCENTE-uuid>     | <ASIGNATURA-uuid>                      | 5   (r-x sobre ASIGNATURA)
```

`flags` es `SMALLINT` 0–7. Constraint único `(role_id, resource_uuid)`.

### 2.5. Decisión (la lógica vive en un solo lugar)

`PermissionService` implementa el contrato compartido `infrastructure.security.PermissionEvaluator`:

- `effectiveFlags(roleIds, resourceUuid)` → OR de los `flags` de todas las filas que matcheen los roles del usuario **y** (`resourceUuid` o `ALL`).
- `hasPermission(roleIds, resourceUuid, requiredBits)` → `(effectiveFlags & required) == required`.

Ejemplo: si Ana tiene rol DOCENTE y existe la fila `(DOCENTE, ASIGNATURA, 5)` más la global `(DOCENTE, ALL, 4)`, entonces `effectiveFlags([DOCENTE], ASIGNATURA) = 5 | 4 = 5`.

**No se duplica la lógica de bits ni el comodín**: hay un solo evaluador, y tanto la decisión interna como el endpoint público lo reutilizan.

### 2.6. Puntos de entrada a la decisión

Hay exactamente dos:

**(a) Interno al MS — `@RequirePermission` sobre un endpoint JAX-RS:**

```java
@POST
@Path("/users")
@RequirePermission(resource = AuthResourceId.Uuid.USERS, value = Permission.WRITE)
public UserResponse create(@Valid UserRequest req) { ... }
```

`RequirePermissionFilter` inspecciona la anotación, obtiene los roles vía `RequestContext`, consulta `PermissionEvaluator.hasPermission(...)`, y si la respuesta es `false` ⇒ `403`. Sin HTTP, sin red.

**(b) Externo — `GET /auth/access`** (lo consumen otros MS):

```
GET /auth/access?resourceUuid=<UUID>&permission=READ|WRITE|EXECUTE
X-User-Id: <ana>
X-User-Roles: <DOCENTE>
```

- Negociación de contenido:
  - `text/plain` (default) ⇒ body `"1"` o `"0"`.
  - `application/json` ⇒ `AccessResponse` con `allowed`, `resourceUuid`, `required`, `effectiveFlags`, `effectiveLabel`.
- Es **público tras el Gateway** (sin `@RequirePermission`): sin identidad propagada no hay grants que sumar ⇒ responde `"0"`, no `403`. Esto evita el meta-guard circular "necesitar permiso para preguntar por permisos".
- Aplica el mismo algoritmo que `@RequirePermission`. Pensado para ser barato y consumido frecuentemente.

---

## 3. Reparto de responsabilidades — "propio" vs "de otro"

Aquí está la pregunta que más confunde al integrar un MS nuevo.

**Auth contesta: tipo de recurso. El MS dueño contesta: instancia.**

- Auth dice "DOCENTE puede WRITE sobre ASIGNATURA": es una afirmación **genérica**, idéntica para Ana y para cualquier otro docente.
- El MS Course, cuando recibe `PATCH /asignaturas/{id}`, después de saber por Auth que el verbo está concedido, aplica su propia regla de pertenencia en SQL: `WHERE id = :id AND docente_id = :userId`. Si la fila no aparece ⇒ `403` (o `404` si prefiere ocultar la existencia).

Esto se sostiene porque el dato `docente_id` **solo existe en la BD de Course**. Auth no lo conoce.

---

## 4. Flujo completo de ejemplo

**Escenario:** Ana (rol DOCENTE) edita la asignatura **42**, que es suya. Luego intenta editar la **99**, que es de otro docente.

### 4.1. Caso feliz — asignatura propia

```
┌─────────┐          ┌──────────────┐           ┌──────┐         ┌───────────────┐
│ Cliente │          │ API Gateway  │           │ Auth │         │ Course (MS)   │
└────┬────┘          └──────┬───────┘           └──┬───┘         └───────┬───────┘
     │ PATCH /asign/42       │                     │                     │
     │ Authorization: Bearer <JWT>                 │                     │
     ├──────────────────────▶│                     │                     │
     │                       │ valida firma RS256  │                     │
     │                       │ extrae sub, roles   │                     │
     │                       │                     │                     │
     │                       │ PATCH /asign/42     │                     │
     │                       │ X-User-Id: <ana>    │                     │
     │                       │ X-User-Roles: <DOCENTE>                   │
     │                       ├────────────────────────────────────────▶  │
     │                       │                     │                     │
     │                       │                     │  RequestContext     │
     │                       │                     │  lee cabeceras 1×   │
     │                       │                     │                     │
     │                       │ GET /auth/access?resourceUuid=<ASIG>&permission=WRITE
     │                       │ X-User-Id: <ana>    │                     │
     │                       │ X-User-Roles: <DOCENTE>                   │
     │                       │                     │◀────────────────────┤
     │                       │                     │                     │
     │                       │ PermissionService.effectiveFlags(         │
     │                       │   [DOCENTE], <ASIG-uuid>) = 2 (-w-)       │
     │                       │ 2 & WRITE(2) == WRITE  ⇒ "1"              │
     │                       │                     ├────────────────────▶│
     │                       │                     │  body: "1"          │
     │                       │                     │                     │
     │                       │                     │  Course consulta su BD:
     │                       │                     │  SELECT * FROM asignatura
     │                       │                     │   WHERE id=42
     │                       │                     │     AND docente_id = <ana>
     │                       │                     │  ⇒ match ⇒ UPDATE permitido
     │                       │ 200 OK              │                     │
     │                       │◀────────────────────────────────────────  │
     │ 200 OK                │                     │                     │
     │◀──────────────────────┤                     │                     │
```

### 4.2. Caso denegado — asignatura ajena

```
     │ PATCH /asign/99   ── ── (idéntico hasta llegar a Course) ── ──
     │
     │ Auth sigue diciendo "1": Ana SÍ puede WRITE sobre ASIGNATURA en general.
     │
     │ Pero Course:
     │      SELECT ... WHERE id=99 AND docente_id = <ana>
     │      ⇒ 0 filas  ⇒  Course responde 403 (o 404 si oculta existencia)
```

### 4.3. Variante — SUPERUSER pasa por encima de todo

Si en vez de DOCENTE el caller fuera SUPERUSER, el grant `(SUPERUSER, ALL, 7)` haría que `effectiveFlags` devuelva `7` para cualquier `resourceUuid`. Course recibiría `"1"` igualmente, y — como decisión propia del MS — omitiría el `AND docente_id = ...` para roles administrativos.

---

## 5. Implicancias prácticas para quien integra un MS nuevo

1. **Define tus recursos en un enum local** del MS, espejo de `AuthResourceId`. Cada constante = un UUID hardcoded. Eso es tu contrato con Auth.
2. **Inserta los grants iniciales** vía migración Flyway en `auth` (filas en `role_permissions` para cada par `(rol, resource_uuid)` que aplique).
3. **Protege los endpoints internos** con `@RequirePermission(resource = MiResourceId.Uuid.X, value = Permission.X)`. No escribas tu propio filtro.
4. **Si necesitas verificar acceso desde otro MS**, llama `GET /auth/access` — no reimplementes la lógica de bits.
5. **La regla de pertenencia ("es mío") vive en tu MS**, como cláusula `WHERE` o check posterior al fetch. No esperes que Auth la resuelva por ti.
6. **Nunca leas `X-User-Id` / `X-User-Roles` a mano**. Inyecta `RequestContext`.
7. **Nunca dupliques el evaluador de permisos**. Si necesitas evaluar en código (fuera de un endpoint), inyecta `PermissionEvaluator`.

---

## 6. Tabla rápida de referencia

| Pregunta | Lo responde | Cómo |
|---|---|---|
| ¿El token es válido? ¿Quién es el usuario? | API Gateway | Valida JWT RS256 emitido por Auth |
| ¿Este rol puede hacer la acción X sobre el tipo Y? | Auth (`PermissionService`) | OR de flags + comodín `ALL` |
| ¿Este usuario puede modificar *esta instancia* concreta? | MS dueño del dato | `WHERE owner = :userId` (o equivalente) |
| ¿Cómo pregunto desde otro MS si Ana puede WRITE sobre asignatura? | Auth | `GET /auth/access?resourceUuid=<ASIG>&permission=WRITE` |
| ¿Cómo protejo un endpoint dentro de mi MS? | El propio MS | Anotación `@RequirePermission` + `RequirePermissionFilter` |

---

## 7. Configuración relevante

| Propiedad | Default | Descripción |
|---|---|---|
| `edutrack.headers.validation.mode` | `EAGER` | `EAGER` ⇒ cabecera malformada = `400`; `WARN` ⇒ se loguea y se trata como ausente |
| `edutrack.errors.expose-stacktrace` | `false` | `true` ⇒ incluye `ErrorResponse.trace[]` (≤ 25 frames). Apagar en prod. |
