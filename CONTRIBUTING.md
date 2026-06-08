# Guía de contribución — EduTrack

Este repositorio (`git@github.com:goslynn/edutrack.git`) es el **superproyecto** de
EduTrack: una *recolección* que agrupa cada microservicio y componente como
**submódulo de git**. El superproyecto **no contiene el código** de los módulos;
guarda una **referencia** (un commit fijo) al repositorio independiente de cada uno.

Lo que sí se versiona directamente aquí: `doc/`, `openspec/`, `CLAUDE.md`,
`docker-compose.yml` y `.env.example`. Los secretos (`.env`, `.certs/`, `*.pem`)
están en `.gitignore` y **nunca** se versionan — usa `.env.example` como plantilla.

## Módulos (submódulos)

Cada fila es un repositorio independiente con su propio historial, issues y PRs.
La columna **rama** es la rama de integración registrada en `.gitmodules` (la que
sigue `git submodule update --remote`).

| Path | Repositorio | Rama |
|---|---|---|
| `attendance` | `git@github.com:Ckrlos/attendance-ms.git` | `master` |
| `auth` | `git@github.com:goslynn/edutrack-ms-auth.git` | `master` |
| `commons` | `git@github.com:goslynn/edutrack-ms-commons.git` | `master` |
| `front` | `git@github.com:goslynn/edutrack-frontend.git` | `master` |
| `infra` | `git@github.com:goslynn/edutrack-infra.git` | `flyio` |

---

## 1. Clonar el repositorio

El superproyecto solo guarda referencias, así que hay que traer los submódulos
explícitamente. La forma recomendada, en un solo paso:

```bash
git clone --recurse-submodules git@github.com:goslynn/edutrack.git
cd edutrack
```

Si ya clonaste **sin** `--recurse-submodules` (verás carpetas de módulos vacías):

```bash
git submodule update --init --recursive
```

Para acelerar el clon de submódulos con historiales grandes puedes añadir
`--jobs 5` (clona varios en paralelo):

```bash
git clone --recurse-submodules --jobs 5 git@github.com:goslynn/edutrack.git
```

### Configuración recomendada (una sola vez)

Para que comandos como `git pull`, `git status` y `git checkout` tengan en cuenta
los submódulos automáticamente:

```bash
git config --global submodule.recurse true            # pull/checkout recursivos
git config --global status.submoduleSummary true      # status muestra cambios de submódulos
git config --global diff.submodule log                # diffs legibles de submódulos
```

---

## 2. Estrategia de branching: **git-flow**

La estrategia oficial y generalizada — tanto en el superproyecto como en cada
módulo — es **git-flow**. Ramas:

| Rama | Propósito | Vida |
|---|---|---|
| `main` (o `master`) | Código en producción. Solo recibe merges desde `release/*` y `hotfix/*`. | permanente |
| `develop` | Línea de integración del próximo release. Base de todo el trabajo nuevo. | permanente |
| `feature/<nombre>` | Una funcionalidad o cambio. Sale de `develop`, vuelve a `develop`. | temporal |
| `release/<version>` | Estabilización previa a publicar. Sale de `develop`, se mergea a `main` **y** `develop`. | temporal |
| `hotfix/<nombre>` | Corrección urgente sobre producción. Sale de `main`, se mergea a `main` **y** `develop`. | temporal |

Reglas de oro:

- **Nunca** se commitea directo sobre `main`/`master` ni sobre `develop`: todo
  entra vía Pull Request.
- El trabajo del día a día nace siempre de `develop` con una rama `feature/*`.
- Cada release se etiqueta con un tag semántico (`vMAJOR.MINOR.PATCH`) al
  mergear a `main`.

> **Nota sobre las ramas registradas en `.gitmodules`.** Hoy la rama de
> integración registrada por módulo es `master` (y `flyio` para `infra`). A medida
> que cada módulo adopte git-flow, su rama de integración pasará a ser `develop`;
> cuando eso ocurra se actualiza el `branch =` correspondiente en `.gitmodules`
> (ver §5).

Puedes usar la extensión [`git-flow`](https://github.com/petmongrels/gitflow) o
los comandos nativos de git; los ejemplos de abajo usan git nativo para no
depender de herramientas extra.

---

## 3. Trabajar en un módulo (el flujo más común)

**Importante:** un submódulo es un repositorio completo. Trabajas *dentro* de su
carpeta, commiteas y haces PR **en el repo del módulo**, no en el superproyecto.

### 3.1 Posicionarte en la rama de integración

Al clonar, los submódulos quedan en *detached HEAD* (apuntando al commit fijado
por el superproyecto). Antes de trabajar, sitúate en la rama de integración:

```bash
cd auth                       # entra al módulo
git checkout develop          # o master, según el módulo (ver tabla)
git pull origin develop
```

### 3.2 Crear tu rama de trabajo

```bash
git checkout -b feature/login-rate-limit develop
```

### 3.3 Desarrollar localmente

Trabaja y prueba como en cualquier repo. Para los microservicios Quarkus, desde
la carpeta del módulo:

```bash
./mvnw quarkus:dev      # hot reload
./mvnw test             # tests unitarios
./mvnw verify           # tests de integración
```

Para levantar el stack completo (BD, gateway, servicios) usa el compose del
superproyecto desde la raíz:

```bash
cd ..                   # raíz del superproyecto
cp .env.example .env    # primera vez: completa los secretos
docker compose up -d
```

### 3.4 Commitear y subir el PR del módulo

```bash
cd auth
git add -A
git commit -m "feat(auth): rate limiting en /login"
git push -u origin feature/login-rate-limit
```

Abre el Pull Request en GitHub **contra `develop`** del repositorio del módulo
(ej. `goslynn/edutrack-ms-auth`). Con el CLI de GitHub:

```bash
gh pr create --base develop --fill
```

Tras la revisión y el merge, la nueva versión del módulo vive en su `develop`.

### 3.5 Actualizar la referencia en el superproyecto

El superproyecto sigue apuntando al commit anterior hasta que **muevas el
puntero** explícitamente. Una vez mergeado el PR del módulo:

```bash
# desde la raíz del superproyecto
git submodule update --remote auth        # trae el HEAD de la rama registrada del módulo
git add auth
git commit -m "chore: bump auth -> rate limiting en /login"
git push                                  # o vía PR del superproyecto (ver §6)
```

> Alternativa manual: entra al módulo, haz `git checkout <commit-deseado>`, vuelve
> a la raíz y `git add <módulo> && git commit`. `git submodule update --remote`
> simplemente automatiza el paso al HEAD de la rama del `.gitmodules`.

---

## 4. Sincronizar tu copia (pull)

Cuando otra persona mueve un puntero de submódulo en el superproyecto, un `git
pull` normal **no** actualiza el contenido del módulo. Hazlo así:

```bash
git pull                                  # actualiza el superproyecto (punteros)
git submodule update --init --recursive   # ajusta cada módulo al commit referenciado
```

(Si configuraste `submodule.recurse true` en §1, `git pull` ya hace el segundo
paso automáticamente.)

Ojo: `git submodule update` deja los módulos en *detached HEAD* sobre el commit
fijado. Para seguir trabajando, vuelve a hacer `git checkout develop` dentro del
módulo (§3.1).

---

## 5. Agregar un nuevo submódulo

Requisito: el módulo ya debe existir como repositorio remoto independiente con su
contenido publicado.

```bash
# desde la raíz del superproyecto
git submodule add -b develop git@github.com:goslynn/edutrack-ms-<nombre>.git <carpeta>
```

- `-b develop` registra la rama de integración en `.gitmodules` (usa la rama que
  corresponda al módulo).
- `<carpeta>` debe coincidir con el **primer segmento del path** del servicio en
  el gateway y con su nombre de app en Fly.io (sin el prefijo `edutrack-`); ver
  `CLAUDE.md`.

Esto crea/actualiza `.gitmodules` y añade el gitlink al index. Confírmalo:

```bash
git status                # verás .gitmodules y la nueva carpeta como cambios
git commit -m "chore: agrega submódulo <nombre>"
git push                  # o vía PR (§6)
```

### Adoptar un directorio que ya existe localmente como repo

Si la carpeta ya está presente y es un repo git con su remoto configurado, el
mismo comando la **adopta sin re-clonar** (`Adding existing repo at '<carpeta>'`).

### Cambiar la rama registrada de un submódulo existente

```bash
git config -f .gitmodules submodule.<carpeta>.branch develop
git submodule update --remote <carpeta>
git add .gitmodules <carpeta>
git commit -m "chore: <carpeta> sigue ahora la rama develop"
```

### Eliminar un submódulo

```bash
git submodule deinit -f <carpeta>
git rm <carpeta>
rm -rf .git/modules/<carpeta>
git commit -m "chore: elimina submódulo <carpeta>"
```

---

## 6. Trabajar en el superproyecto y subir PRs

El superproyecto se versiona con la **misma estrategia git-flow**. Los cambios
propios del superproyecto son típicamente:

- mover punteros de submódulos (bumps de versión, §3.5),
- editar `doc/`, `openspec/`, `CLAUDE.md`, `docker-compose.yml`, `.env.example`.

Flujo:

```bash
git checkout -b feature/bump-auth-y-front develop
# ... cambios (p. ej. git submodule update --remote ... ; git add ...) ...
git commit -m "chore: bump auth y front al último develop"
git push -u origin feature/bump-auth-y-front
gh pr create --base develop --fill
```

Reglas para PRs del superproyecto:

- Un PR que mueve un puntero de submódulo **solo debe mergearse cuando el commit
  referenciado ya está publicado** en el remoto del módulo (en su rama de
  integración). Si referencias un commit no pusheado, el clon del superproyecto
  fallará al traer ese submódulo.
- Describe en el PR qué módulos se bumpean y a qué cambian (resumen de los PRs de
  módulo incluidos).
- No mezcles en un mismo PR cambios de contenido propio (docs/compose) con bumps
  de muchos módulos salvo que estén relacionados; mantén los PRs enfocados.

---

## 7. Checklist rápido

**Empezar a trabajar en un módulo**
1. `cd <módulo>` → `git checkout develop` → `git pull`
2. `git checkout -b feature/<algo> develop`
3. desarrollar + probar
4. `git push -u origin feature/<algo>` → PR contra `develop` del módulo
5. tras merge: en la raíz, `git submodule update --remote <módulo>` → commit → PR del superproyecto

**Ponerte al día**
- `git pull && git submodule update --init --recursive`

**Nunca**
- commitear a `main`/`master` o `develop` directo (siempre PR)
- versionar `.env`, `.certs/` o `*.pem`
- referenciar en el superproyecto un commit de módulo que no esté pusheado
