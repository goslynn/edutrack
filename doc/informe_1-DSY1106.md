**INFORME TÉCNICO**

*Diseno de Arquitectura de Microservicios*

**Plataforma EduTrack \- Libro de Clases Digital**

Colegio Bernardo O Higgins \- Coquimbo

Asignatura: DSY1106 \- Desarrollo Fullstack III

Evaluacion Parcial N1 \- Diseno de Arquitecturas

*Version 2 | Abril 2026*

# **1\. Contexto y Necesidades del Cliente**

El Colegio Bernardo O Higgins de Coquimbo es una institucion educativa que atiende ensenanza basica y media. Actualmente, el establecimiento utiliza libros de clases fisicos complementados con soluciones digitales fragmentadas, lo que dificulta la consolidacion de informacion academica y el acceso oportuno a datos historicos.

La plataforma EduTrack busca digitalizar integralmente el libro de clases, permitiendo a docentes, administradores y apoderados acceder a un entorno seguro para registrar y consultar informacion academica. Los requerimientos funcionales identificados son:

* Autenticacion y gestion dinamica de usuarios con sistema de roles completamente configurable (roles precargados: docente, administrador, superusuario) y modelo de permisos extensible.

* Gestion de cursos con control de acceso granular por usuario: multiples docentes pueden tener permisos diferenciados de lectura y escritura sobre cada curso.

* Registro de alumnado y apoderados con notificaciones por correo electronico.

* Gestion de contenido academico organizado segun una jerarquia de arbol configurable globalmente (estructura por defecto: Semestre \> Asignatura \> Unidad \> Clase).

* Registro de evaluaciones (notas) y control de asistencia por clase, con contrato de API agnostico al mecanismo de captura.

* Anotaciones positivas y negativas por alumno con notificacion automatica a apoderados.

* Generacion de reportes academicos en multiples formatos: JSON, CSV y PDF renderizado.

Adicionalmente, se requiere que el sistema sea escalable para soportar crecimiento futuro, seguro para proteger datos sensibles de menores de edad, y sostenible en terminos operativos y ambientales.

# **2\. Seleccion de Patrones de Arquitectura**

La arquitectura propuesta se fundamenta en tres patrones principales, seleccionados en funcion de los requerimientos del colegio:

## **2.1 API Gateway**

Se implementa un API Gateway como punto de entrada unico para todas las solicitudes del frontend. Este patron centraliza la autenticacion y autorizacion mediante JWT, aplica rate limiting para proteger los servicios de sobrecarga, y simplifica la comunicacion entre el cliente React y los multiples microservicios del backend. El gateway propaga los claims del token (sub, roles\[\]) como headers internos hacia cada microservicio.

## **2.2 Service Registry (Descubrimiento de Servicios)**

Dado que los microservicios se despliegan en contenedores sobre Kubernetes en AWS, se utiliza el mecanismo de Service Discovery nativo de Kubernetes combinado con Consul como registry complementario.

## **2.3 Circuit Breaker**

Para garantizar la resiliencia del sistema, se implementa el patron Circuit Breaker mediante la libreria SmallRye Fault Tolerance integrada en Quarkus. Si el servicio de notificaciones falla temporalmente, el Circuit Breaker evita que las llamadas en cascada afecten a otros servicios criticos.

# **3\. Herramientas y Estrategias de Implementacion**

**Framework de backend: Quarkus.** Se selecciona Quarkus como framework principal por su diseno nativo para contenedores y cloud. Su tiempo de arranque sub-segundo y bajo consumo de memoria lo hacen ideal para microservicios desplegados en Kubernetes.

**Comunicacion hibrida: REST \+ eventos asincronos.** Las consultas directas se realizan mediante REST sincrono a traves del API Gateway. Para operaciones no criticas se utiliza RabbitMQ con comunicacion asincrona basada en eventos.

**Contenedorizacion y orquestacion: Docker \+ Kubernetes (EKS).** Cada microservicio se empaqueta en una imagen Docker con Quarkus JVM. Los contenedores se despliegan en Amazon EKS con auto-scaling.

**Base de datos: Shared DB con schemas separados en Amazon RDS .** Se adopta el patrón de base de datos compartida con schemas separados por microservicio. Cada microservicio posee credenciales de BD propias con acceso restringido exclusivamente a su schema, garantizando el aislamiento logico de forma tecnica. El Report Service cuenta con credenciales adicionales de solo lectura sobre todos los schemas, lo que le permite generar reportes consolidados cross-dominio sin infraestructura adicional. Las migraciones de schema se coordinan mediante Flyway/Liquibase. La instancia RDS se despliega con backups automaticos, replicas de lectura y alta disponibilidad.

**Frontend: React \+ TypeScript con Vite.** La interfaz se construye como SPA que consume los microservicios a traves del API Gateway, con diseno responsive.

**Almacenamiento de contenido: Amazon S3.** El material multimedia se almacena en S3 con URLs pre-firmadas, organizadas segun la jerarquia academica configurada.

# **4\. Diagrama de Arquitectura de Microservicios**

La arquitectura se compone de los siguientes microservicios, comunicados a traves del API Gateway y RabbitMQ. Todos comparten una unica instancia Amazon RDS con schemas separados; el Report Service posee ademas acceso de solo lectura a los schemas de los demas servicios.

| Microservicio | Responsabilidad | Comunicacion |
| ----- | ----- | :---: |
| **Auth Service** | Autenticacion JWT, gestion dinamica de roles, modelo de permisos Unix-style (flags numericos por resource\_uuid \+ role\_uuid) | REST (sincrono) |
| **Course Service** | CRUD de cursos, control de acceso granular por usuario-curso | REST (sincrono) |
| **Student Service** | Registro de alumnos y apoderados | REST \+ Eventos |
| **Content Service** | Gestion de contenido con jerarquia de arbol configurable, archivos en S3 | REST (sincrono) |
| **Assessment Service** | Registro de notas y evaluaciones | REST \+ Eventos |
| **Attendance Service** | Control de asistencia con contrato de API agnostico al mecanismo de captura | REST (sincrono) |
| **Annotation Service** | Anotaciones positivas y negativas | REST \+ Eventos |
| **Notification Service** | Notificaciones con contrato generico extensible (Strategy Pattern); v1: EMAIL\_HTML | Eventos (async) |
| **Report Service** | Definicion y generacion de reportes (JSON/CSV/PDF), acceso read-only cross-schema a RDS | REST \+ Eventos |

# **5\. Seguridad, Privacidad y Sostenibilidad**

## **5.1 Seguridad**

La seguridad se implementa en multiples capas:

* El API Gateway valida tokens JWT (RS256) en cada solicitud y propaga claims al microservicio destino.

* El Auth Service implementa un modelo de permisos Unix-style: flags numericos (r=4, w=2, x=1) por par (role\_uuid, resource\_uuid). Los roles son completamente dinamicos y gestionables via API.

* El aislamiento de datos entre microservicios se garantiza tecnicamente mediante credenciales de BD distintas por schema en RDS. El Report Service posee credenciales adicionales de solo lectura sobre los demas schemas.

* Las comunicaciones se cifran mediante TLS/HTTPS en transito; los datos en reposo con AES-256 en RDS y S3.

* Se aplican politicas de rate limiting, validacion y sanitizacion de inputs para prevenir inyeccion y DDoS.

## **5.2 Privacidad**

Al tratarse de datos de menores de edad, la plataforma cumple con la Ley 19.628 de Proteccion de Datos Personales de Chile. Se implementa el principio de minimo privilegio: cada docente solo visualiza los cursos con acceso explicitamente asignado. Los datos de apoderados se almacenan de forma segregada y solo el Notification Service accede a ellos. Se mantienen logs de auditoria inmutables para trazabilidad de accesos.

## **5.3 Sostenibilidad**

La arquitectura contribuye a la sostenibilidad en tres dimensiones. Primero, ambiental: Quarkus reduce el consumo de memoria y CPU, y el auto-scaling minimiza recursos en baja demanda. Segundo, economica: el modelo de pago por uso de AWS, la eficiencia de Quarkus y la simplificacion operativa de una unica instancia RDS reducen los costos. Tercero, tecnica: el desacoplamiento de microservicios permite actualizar componentes individuales sin afectar al sistema completo.

# **6\. Evaluacion General del Diseno**

El diseno propuesto responde integralmente a los requerimientos del Colegio Bernardo O Higgins. La adopcion del patron Shared DB con schemas separados simplifica la operacion y resuelve de forma directa el acceso cross-dominio del Report Service, con un trade-off de coordinacion de migraciones aceptable para el contexto del proyecto.

El modelo de permisos Unix-style en el Auth Service provee una base generica y extensible para el control de acceso, complementado por la gestion granular por usuario-curso en el Course Service. El contrato del Attendance Service, disenado como una API flexible agnostica al origen de los datos, permite incorporar futuras integraciones como clientes externos sin modificar el core del servicio.

El Notification Service aplica el patron Strategy internamente con un contrato generico que permite incorporar nuevos canales sin modificar la interfaz de entrada ni requerir redeploy.

En conclusion, la arquitectura propuesta equilibra robustez tecnica, eficiencia operativa y responsabilidad etica, proporcionando una solucion moderna y sostenible para la digitalizacion del libro de clases.

# **7\. Referencias**

Quarkus Project. (2024). Quarkus: Supersonic Subatomic Java. https://quarkus.io/

Richardson, C. (2018). Microservices Patterns. Manning Publications.

Newman, S. (2021). Building Microservices (2nd ed.). O Reilly Media.

Amazon Web Services. (2024). Amazon EKS Documentation. https://docs.aws.amazon.com/eks/

Congreso Nacional de Chile. (1999). Ley 19.628 sobre Proteccion de la Vida Privada.

Nygard, M. T. (2018). Release It\! Design and Deploy Production-Ready Software (2nd ed.). Pragmatic Bookshelf.