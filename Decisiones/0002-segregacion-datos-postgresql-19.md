# 0002 - Estrategia de Segregación de Datos PostgreSQL 19

* **Estado:** aceptada
* **Fecha:** 2026-11-03
* **Decisores (ASS):** Antonio Machuca, Enrique Aldama
* **Consultados (ASC):** Sergio Gutiez, Sara Guillen
* **Informados (ASJ):** David Sticea, Elena Romera

## Contexto y Planteamiento del Problema
La especificación del periódico digital impone la existencia de dos bases de datos PostgreSQL 19 en el back-end: una destinada a los administradores y redactores (gestión editorial de las cinco secciones y almacenamiento de métricas de lectura y permanencia) y otra reservada exclusivamente para los suscriptores y la verificación de sus derechos de lectura anónima. La pregunta arquitectónica radica en cómo estructurar el aislamiento de estos almacenes de datos y su integración técnica con los componentes del servidor garantizando la máxima seguridad, confidencialidad y rendimiento.

## Requisitos Motivadores (Decision Drivers)
* **Requisito(s) Clave:** RD5 - Segregación física obligatoria de datos
* **Restricción de Privacidad y Seguridad:** Impedir que accesos o brechas en el subsistema público/suscriptores comprometan la base de datos de administración y redacción, y viceversa.
* **Optimización de Velocidad:** Evitar que una gran cantidad de entradas en la base de datos de suscriptores afecten a al rendimiento de la base de datos de administración y redacción.

## Opciones Consideradas
* **Opción 1:** Segregación física en dos instancias de PostgreSQL 19 independientes con Gateways/DAOs desacoplados por subsistema.
* **Opción 2:** Base de datos PostgreSQL 19 única lógica segregada por esquemas (`schema_admin` y `schema_suscriptores`) y control de acceso basado en roles (RBAC).
* **Opción 3:** Uso de la replicación lógica unidireccional de PostgreSQL para actualizar cambios (publicación/suscripción nativa) entre la base editorial y la de lectores.

## Decisión Adoptada
Opción elegida: "**Segregación física en dos instancias de PostgreSQL 19 independientes con Gateways/DAOs desacoplados por subsistema**", porque:
* Cumple estrictamente el mandato de aislamiento físico de la especificación oficial, garantizando que el subsistema de redacción y métricas (noticias de las 5 secciones, tasas de rebote y tiempos de lectura) resida en un servidor/puerto de base de datos totalmente diferenciado del subsistema de suscriptores anónimos.
* Reduce la superficie de ataque: un compromiso de seguridad en la base pública de suscriptores jamás expondrá las credenciales, artículos embargados o paneles analíticos de los administradores.
* Facilita políticas de afinamiento (*tuning*), copias de seguridad e indexación diferenciadas: la base de lectores prioriza transacciones concurrentes de lectura y consultas por tokens de suscripción, mientras que la editorial prioriza operaciones transaccionales de escritura de noticias y consultas analíticas agregadas.

### Consecuencias
* **Positivas:** Aislamiento total de fallos de infraestructura entre el entorno editorial y el de lectores; imposibilidad física de ejecutar *queries cross-database* accidentales; máxima conformidad con las restricciones de dominio de la práctica.
* **Negativas:** Obliga a mantener dos *pools* de conexiones independientes en el servidor; prohíbe realizar operaciones transaccionales ACID distribuidas nativas entre ambas bases de datos sin recurrir a patrones de coordinación a nivel de aplicación (ej. Saga o eventos).
* **Neutras:** Requiere que los ASJ modelen dos paquetes y jerarquías de persistencia claramente segregados (`AdminPersistenceGateway` y `SubscriberPersistenceGateway`).

## Pros y Contras de las Opciones Consideradas

### Segregación física en dos instancias de PostgreSQL 19 independientes con Gateways/DAOs desacoplados por subsistema
* **A favor (+):** Cumplimiento riguroso del Anexo I; aislamiento total de rendimiento (las consultas intensivas de métricas editoriales como tasa de rebote o tiempo de permanencia no degradan las comprobaciones de suscripción de los lectores).
* **A favor (+):** Preservación absoluta del principio de mínimo privilegio y privacidad: las credenciales de conexión del pool de suscriptores no tienen permisos de red ni acceso al motor editorial.
* **En contra (-):** Mayor coste y complejidad de configuración de infraestructura, requiriendo el aprovisionamiento y mantenimiento de dos servidores PostgreSQL 19 en puertos/contenedores separados.
* **En contra (-):** Imposibilidad de realizar sentencias `JOIN` directas entre las noticias leídas y los tokens anónimos de suscriptores; cualquier correlación estadística debe procesarse en la capa de servicios.

### Base de datos PostgreSQL 19 única lógica segregada por esquemas (`schema_admin` y `schema_suscriptores`) y control de acceso RBAC
* **A favor (+):** Menor consumo de recursos de computación y simplificación del aprovisionamiento al gestionar una única instancia de servidor PostgreSQL 19.
* **A favor (+):** Facilita la administración de copias de seguridad en un único archivo de volcado (*dump*).
* **En contra (-):** Incumple la directriz del Anexo I que exige expresamente dos bases de datos separadas para administradores y suscriptores.
* **En contra (-):** Riesgo de fuga de información o salto de permisos si falla la configuración RBAC a nivel de esquema, compartiendo CPU, memoria y el mismo proceso del motor gestor.

### Replicación lógica unidireccional de PostgreSQL 19 (publicación/suscripción nativa) entre la base editorial y la de lectores
* **A favor (+):** Permite sincronizar noticias publicadas en las 5 secciones desde la base de redacción hacia la base de suscriptores de forma automática mediante el motor nativo de replicación lógica de PostgreSQL 19.
* **En contra (-):** Introduce una notable complejidad operativa en la gestión de conflictos de réplica, slots de replicación y desincronización en caso de caídas transitorias de red.
* **En contra (-):** Difumina la propiedad del dato y no resuelve el aislamiento bidireccional necesario para las suscripciones anónimas y los cobros de Stripe.

## Validación y Trazabilidad UML
* **Modelado ASJ:** El diagrama de paquetes `Arquitecturas/0001-estilo-arquitectonico-monolito-semana1.puml` refleja explícitamente dos subsistemas de acceso a datos independientes: `AdminDataGateway` conectado al conector `PostgreSQL 19 - Admin/Redacción` y `SubscriberDataGateway` conectado al conector `PostgreSQL 19 - Suscriptores`.
* **Notas de Discusión ASS-ASC:**
  * *Intervención de Sergio Gutiez (ASC):* Sergio preguntó si la segregación física obligaría a utilizar transacciones distribuidas.
  * *Respuesta de Enrique Aldama (ASS):* Se aclaró que no, ya que no se comparten entidades entre redactores y suscriptores. Los redactores solo escriben noticias y leen métricas, mientras que los suscriptores solo validan tokens de acceso y ciclos de pago contra su propia base, por lo que no se requieren transacciones distribuidas.
  * *Intervención de Sara Guillen (ASC):* Sara validó que esta separación física es indispensable para salvaguardar el anonimato estricto y aislar las credenciales de Stripe.
