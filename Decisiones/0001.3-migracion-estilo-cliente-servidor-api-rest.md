# 0003 - Migración a Estilo Arquitectónico Cliente-Servidor con API REST

* **Estado:** aceptada (supera a ADR-0001)
* **Fecha:** 2026-11-06
* **Decisores (ASS):** Antonio Machuca, Enrique Aldama
* **Consultados (ASC):** Sergio González, Sara Guillen
* **Informados (ASJ):** David Sticea, Elena Romera

## Contexto y Planteamiento del Problema
En una primera aproximación al problema de la práctica se adoptó un estilo monolítico en capas (ADR-0001) para validar el modelo de dominio. Sin embargo, el Anexo I impone la restricción estricta de que todas las interacciones entre los clientes web y el servidor deben realizarse mediante protocolos HTTP/REST. Asimismo, el periódico digital atiende a lectores en PC y Tablet, ofrece suscripciones anónimas con cobros en Stripe y recopila métricas de lectura para redactores. 
Mantener un monolito con renderizado de vistas en el servidor contradice el requisito REST, sobrecarga la CPU del backend y complica aislar las dos bases de datos físicas independientes (PostgreSQL 19). Por ello, dentro del plazo límite del 10 de noviembre fijado en la práctica, se precisa revocar el estilo monolítico y seleccionar la arquitectura definitiva del sistema.

## Requisitos Motivadores (Decision Drivers)
* **Requisito Clave:** RD7 - Comunicación mediante HTTP/REST (Toda petición entre el front-end web y el back-end se realiza mediante el protocolo HTTP/REST, quedando excluido cualquier otro protocolo).
* **Restricción de Rendimiento e Integración:** Permitir que los clientes PC y Tablet consuman recursos ligeros en JSON, descargando al servidor de renderizar HTML y facilitando la integración con las dos bases de datos segregadas en la ADR-0002.

## Opciones Consideradas
* **Opción 1:** Arquitectura Cliente-Servidor desacoplada con Frontend Web responsivo y Backend API REST sin estado (Stateless).
* **Opción 2:** Arquitectura Orientada a Servicios (SOA) con Enterprise Service Bus (ESB) y servicios basados en contratos SOAP/XML.
* **Opción 3:** Arquitectura basada en Microfrontends y Microservicios REST independientes.

## Decisión Adoptada
Opción elegida: "**Arquitectura Cliente-Servidor desacoplada con Frontend Web responsivo y Backend API REST sin estado (Stateless)**", porque:
* Cumple estrictamente el requisito RD7 al canalizar todas las peticiones entre los clientes (PC y Tablet) y el servidor a través de endpoints HTTP/REST que devuelven recursos en formato estándar JSON.
* Descarga de trabajo al servidor: el navegador del lector asume la maquetación y la responsividad para PC y Tablet, eliminando el coste de CPU del motor de plantillas en el backend.
* Respeta la segregación física de datos de la ADR-0002: los controladores REST del backend dirigen las peticiones de noticias y métricas a la base de datos de administración, y los flujos de suscripción anónima y Stripe a la base de datos de suscriptores.
### Consecuencias
* **Positivas:** Separación clara de responsabilidades; el backend es sin estado (*stateless*), lo que facilita su escalado; el diseño visual se puede actualizar sin tocar el código del servidor; integración limpia de los avisos de Stripe mediante endpoints REST.
* **Negativas:** Exige definir formalmente los contratos de datos (DTOs y modelos JSON) para las 5 secciones de noticias, suscripciones y métricas; requiere configurar políticas de seguridad CORS para peticiones web.
* **Neutras:** Obliga a los ASJ a reorganizar el diagrama UML dividiendo el sistema en dos bloques principales: `FrontendWeb` y `BackendServidor`.

## Pros y Contras de las Opciones Consideradas

### Opción 1: Arquitectura Cliente-Servidor desacoplada con Frontend Web responsivo y Backend API REST Stateless
* **A favor (+):** Cumplimiento total del requisito RD7 y del Anexo I al usar peticiones HTTP/REST en JSON para las noticias y suscripciones.
* **A favor (+):** Descarga al servidor de procesar plantillas HTML, dejando que PC y Tablet se adapten en el navegador mediante diseño responsivo.
* **En contra (-):** Mayor latencia en la primera carga en el navegador y necesidad de configurar filtros CORS en el backend.

### Opción 2: Arquitectura Orientada a Servicios (SOA) con Enterprise Service Bus (ESB) y servicios SOAP/XML
* **A favor (+):** Gran robustez corporativa tradicional.
* **En contra (-):** Viola directamente el requisito RD7 al imponer protocolos SOAP/XML en lugar de HTTP/REST.
* **En contra (-):** Sobrecarga de computación y complejidad innecesaria para transferir noticias en tablets.

### Opción 3: Arquitectura basada en Microfrontends y Microservicios REST
* **A favor (+):** Aislamiento total de despliegue entre el módulo de noticias, el de Stripe y el de estadísticas.
* **En contra (-):** Complejidad desproporcionada para el tamaño y las 3 semanas de la práctica.

## Validación y Trazabilidad UML

* **Modelado ASJ:** Se solicita a David Sticea y Elena Romera (ASJ) la elaboración de un nuevo diagrama en `Arquitecturas/0002-estilo-cliente-servidor-rest-semana2.puml` que sustituya la vista monolítica. El diagrama deberá modelar el paquete `FrontendWeb` (con componentes adaptativos para PC y Tablet), el protocolo de conexión HTTP/REST (JSON), y en el backend los controladores REST (`NoticiasRestController`, `SuscripcionesRestController`, `FacturacionStripeRestController`, `EstadisticasRestController`) comunicándose con los servicios y las dos pasarelas DAO independientes definidas en la ADR-0002.
* **Notas de Discusión ASS-ASC:**
  * *Intervención crítica de Sergio González (ASC):* Sergio argumentó formalmente que mantener la decisión 0001 (Monolito con plantillas HTML en servidor) constituía un riesgo crítico de evaluación al violar el requisito de dominio RD7 ("Toda petición entre front-end web y back-end se realiza mediante HTTP/REST"). Además, advirtió que concentrar en un monolito la discriminación de clientes, el cálculo de métricas y la conexión a las dos bases de datos físicas PostgreSQL 19 generaría una arquitectura rígida y vulnerable. Por ello, Sergio propuso formalmente revocar la ADR-0001 antes de la fecha límite del 10 de noviembre y sustituirla por una arquitectura Cliente-Servidor desacoplada con API REST.
  * *Intervención de Sara Guillen (ASC):* Sara respaldó plenamente el planteamiento de Sergio, añadiendo que desacoplar el frontend mediante REST simplifica de forma determinante la gestión de las tres modalidades de pago (trimestral, semestral y anual) y la recepción de webhooks de la pasarela Stripe sin interferir con la navegación de los lectores.
  * *Resolución de los ASS (Antonio Machuca y Enrique Aldama):* Los ASS aceptaron unánimemente la objeción planteada por los ASC. Se acordó actualizar el estado de la ADR-0001 a `superada por ADR-0003` y aprobar la presente decisión como la arquitectura base definitiva del periódico digital.

## Referencias
* Roy T. Fielding, "Architectural Styles and the Design of Network-based Software Architectures (REST)", [https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)
* Martin Fowler, "Richardson Maturity Model", [https://martinfowler.com/articles/richardsonMaturityModel.html](https://martinfowler.com/articles/richardsonMaturityModel.html)
* Documentación oficial DAS, "IS-P1-2026.pdf: Tarea 2 - Cambio de estilo arquitectónico antes del 10 de Noviembre".