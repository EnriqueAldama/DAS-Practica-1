# 0002 - Estrategia de Segregación de Datos PostgreSQL 19

* **Estado:** aceptada
* **Fecha:** 2026-11-03
* **Decisores (ASS):** Antonio Machuca, Enrique Aldama
* **Consultados (ASC):** Sergio González, Sara Guillen
* **Informados (ASJ):** David Sticea, Elena Romera

## Contexto y Planteamiento del Problema
El sistema del periódico digital requiere almacenar datos editoriales y métricas de uso (noticias en cinco secciones, tiempo de permanencia, tasas de rebote) por un lado, y datos de lectores (tokens anónimos y suscripciones con Stripe) por otro. El Anexo I exige dos bases de datos PostgreSQL 19 independientes: una para administradores y redactores, y otra para suscriptores. Debemos decidir la estrategia de aislamiento y el modo de conexión entre los componentes del servidor y ambos almacenes de datos.

## Requisitos Motivadores (Decision Drivers)
* **Requisito(s) Clave:** RD5 - Segregación física obligatoria de datos
* **Restricción de Dominio:** RD2 - Preservación de anonimato en suscripciones

## Opciones Consideradas
* **Opción 1:** [Dos instancias independientes de PostgreSQL 19 en servidores y puertos separados (5432 y 5433) con DAOs desacoplados (`AdminRedaccionDAO` y `SuscriptoresDAO`)](https://microservices.io/patterns/data/database-per-service.html)
* **Opción 2:** [Una única instancia de servidor PostgreSQL 19 con dos bases de datos lógicas independientes (`bd_admin` y `bd_suscriptores`) en el puerto 5432 y control de acceso por roles (RBAC)](https://www.postgresql.org/docs/current/managing-databases.html)
* **Opción 3:** [Dos instancias independientes de PostgreSQL 19 con replicación lógica nativa de artículos desde la base editorial hacia la de suscriptores](https://www.postgresql.org/docs/current/logical-replication.html)

## Decisión Adoptada
Opción elegida: "**Dos instancias independientes de PostgreSQL 19 en servidores y puertos separados (5432 y 5433) con DAOs desacoplados (`AdminRedaccionDAO` y `SuscriptoresDAO`)**", porque:
* Garantiza el aislamiento de recursos del sistema operativo (CPU, memoria y disco). Las consultas analíticas de métricas editoriales no saturan la base de datos de suscriptores.
* Aplica el principio de mínimo privilegio en red. El pool de conexiones de suscriptores no tiene acceso de red al puerto de la base de datos editorial.
* Evita la posibilidad de realizar consultas `JOIN` directas entre los tokens de suscriptores y las lecturas de noticias, garantizando el anonimato exigido en RD2.

### Consecuencias
* **Positivas:** Aislamiento de fallos entre ambos entornos de datos. Si una instancia cae por sobrecarga o mantenimiento, la otra continúa en servicio.
* **Negativas:** Obliga a gestionar dos pools de conexiones independientes en el servidor. Impide realizar transacciones ACID nativas entre ambos almacenes.
* **Neutras:** Requiere mantener dos componentes de acceso a datos independientes en la capa de datos: `AdminRedaccionDAO` y `SuscriptoresDAO`.

## Pros y Contras de las Opciones Consideradas

### Dos instancias independientes de PostgreSQL 19 en servidores y puertos separados con DAOs desacoplados
* **A favor (+):** Cumple el requisito RD5 al separar físicamente los motores de base de datos en los puertos 5432 y 5433, eliminando un punto único de fallo para ambos subsistemas.
* **A favor (+):** Aislamiento de rendimiento. Permite configurar memoria (`shared_buffers`, `work_mem`) y límites de conexiones adaptados a cada perfil: lecturas concurrentes de tokens en suscriptores frente a escrituras y agregaciones analíticas en redacción.
* **A favor (+):** Principio de mínimo privilegio. Las credenciales de red del pool de suscriptores no tienen acceso físico ni permisos sobre el puerto 5432 de administración.
* **A favor (+):** Anonimato estricto (RD2). La separación de motores imposibilita sentencias SQL `JOIN` entre las lecturas de noticias y los tokens de suscriptores en la base de datos.
* **En contra (-):** Mayor coste operativo. Requiere desplegar, configurar y monitorizar dos procesos o contenedores de base de datos independientes.
* **En contra (-):** Correlación en capa de negocio. Cualquier estadística combinada entre noticias y suscripciones debe resolverse en el componente `ServicioEstadisticas`.

### Una única instancia de servidor PostgreSQL 19 con dos bases de datos lógicas independientes
* **A favor (+):** Reduce el consumo de memoria en el servidor al ejecutar un solo proceso gestor de base de datos.
* **A favor (+):** Simplifica la operativa de copias de seguridad al consolidar la administración en una única instancia.
* **En contra (-):** Una consulta analítica pesada de estadísticas editoriales consume CPU y memoria del mismo proceso que valida tokens de suscriptores.
* **En contra (-):** Un fallo o reinicio del motor PostgreSQL 19 interrumpe el acceso de administradores y suscriptores de forma simultánea.
* **En contra (-):** Riesgo de acceso no autorizado entre bases de datos lógicas si ocurre un error de configuración en los roles y permisos RBAC del motor.

### Dos instancias independientes de PostgreSQL 19 con replicación lógica nativa
* **A favor (+):** Permite que la base de suscriptores mantenga una copia local de las noticias de las cinco secciones para lecturas directas.
* **En contra (-):** Añade complejidad en la gestión de conflictos de réplica, slots de replicación y retrasos de sincronización de red.
* **En contra (-):** No aporta valor al subsistema de suscriptores, ya que los lectores solo necesitan validar su suscripción y no requieren una copia local de la base editorial.
* **En contra (-):** Incrementa el uso de almacenamiento en disco al duplicar el contenido informativo en ambos motores.

## Validación y Trazabilidad UML
* **Modelado ASJ:** El diagrama `Arquitecturas/0001-estilo-arquitectonico-monolito-semana1.puml` muestra la capa de datos con dos componentes: `AdminRedaccionDAO` conectado a la base `PostgreSQL 19 (Instancia Editorial)` en puerto 5432, y `SuscriptoresDAO` conectado a la base `PostgreSQL 19 (Instancia Suscriptores)` en puerto 5433.
* **Notas de Discusión ASS-ASC:**
  * *Intervención de Sergio González (ASC):* Sergio preguntó si la separación en dos instancias independientes exige transacciones distribuidas (2PC).
  * *Respuesta de Enrique Aldama (ASS):* Se aclaró que no existen operaciones transaccionales conjuntas. Los redactores publican noticias y consultan métricas en su propia base. Los lectores gestionan altas, bajas y pagos en la base de suscriptores.
  * *Intervención de Sara Guillen (ASC):* Sara validó que esta separación física es indispensable para salvaguardar el anonimato estricto (RD2) y aislar las credenciales de Stripe.

## Referencias
* Chris Richardson, "Pattern: Database per service", [https://microservices.io/patterns/data/database-per-service.html](https://microservices.io/patterns/data/database-per-service.html)
* PostgreSQL Global Development Group, "PostgreSQL Documentation: Chapter 23. Managing Databases", [https://www.postgresql.org/docs/current/managing-databases.html](https://www.postgresql.org/docs/current/managing-databases.html)
* PostgreSQL Global Development Group, "PostgreSQL Documentation: Chapter 31. Logical Replication", [https://www.postgresql.org/docs/current/logical-replication.html](https://www.postgresql.org/docs/current/logical-replication.html)
* Martin Fowler, "Patterns of Enterprise Application Architecture: Table Data Gateway", [https://martinfowler.com/eaaCatalog/tableDataGateway.html](https://martinfowler.com/eaaCatalog/tableDataGateway.html)
