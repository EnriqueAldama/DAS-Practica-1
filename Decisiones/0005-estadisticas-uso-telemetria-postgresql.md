# 0005 - Captura y Almacenamiento de Estadísticas de Uso mediante Telemetría Propia en PostgreSQL 19

* **Estado:** propuesta
* **Fecha:** 2026-11-16
* **Decisores (ASS):** Antonio Machuca, Enrique Aldama
* **Consultados (ASC):** Sergio González, Sara Guillen
* **Informados (ASJ):** David Sticea, Elena Romera

## Contexto y Planteamiento del Problema
Los administradores y redactores necesitan consultar estadísticas de uso del periódico: noticias más leídas, tiempo de permanencia, tasas de rebote y conversiones a suscriptores. El sistema ya usa una arquitectura Cliente-Servidor REST (ADR-0001.3), un `ApiGateway` que distingue PC y Tablet mediante `X-Device-Type` (ADR-0002.1), suscripciones anónimas sin datos personales (ADR-0004) y dos bases de datos PostgreSQL 19 (una para administración/redacción y otra para suscriptores). Hay que decidir cómo recoger y guardar estas métricas sin añadir infraestructura innecesaria ni romper el anonimato.

## Requisitos Motivadores (Decision Drivers)
* **Requisito Clave:** RF6 - Estadísticas de uso (Los administradores y redactores pueden consultar las noticias más leídas, el tiempo de permanencia, las tasas de rebote y las conversiones a suscriptores).

## Opciones Consideradas
* **Opción 1:** Telemetría propia desde `FrontendWeb` al backend REST, guardando las métricas en la base PostgreSQL 19 de administración y redacción (`AdminRedaccionDAO`).
* **Opción 2:** Herramienta externa de analítica web (Google Analytics o Matomo SaaS) en `FrontendWeb`.
* **Opción 3:** Pipeline de eventos con broker de mensajería (Kafka) y base analítica separada (ClickHouse).

## Decisión Adoptada
Opción elegida: "**Opción 1: Telemetría propia en PostgreSQL 19 (`AdminRedaccionDAO`)**", porque:
* Cumple RF6 usando los componentes que ya existen (`EstadisticasController`, `ServicioEstadisticas` y `AdminRedaccionDAO`), sin añadir servidores ni bases de datos nuevas.
* Cubre las cuatro métricas de forma sencilla:
  1. **Noticias más leídas y rebote:** eventos de lectura enviados desde `FrontendWeb` con un ID de sesión temporal del navegador (sin cookies ni datos personales).
  2. **Tiempo de permanencia:** envío ligero con `Navigator.sendBeacon()` al cerrar o cambiar de pestaña.
  3. **Conversiones a suscriptores:** aviso interno de `ServicioSuscripciones` a `ServicioEstadisticas` cuando se confirma un alta (ADR-0003 y ADR-0004).
* Usa la cabecera `X-Device-Type` del `ApiGateway` para separar los datos entre PC y Tablet.
* Guarda todo en la base PostgreSQL 19 de administración y redacción, sin mezclar ni guardar `tokenHash` ni datos de suscriptores.

### Consecuencias
* **Positivas:** Control total de los datos en PostgreSQL 19; no le afectan los bloqueadores de anuncios; las conversiones solo se cuentan cuando el pago se confirma en el backend; coste mínimo.
* **Negativas:** Genera más escrituras en la base de administración y redacción cuando hay muchos lectores; obliga a programar la captura en `FrontendWeb` y el cálculo en `ServicioEstadisticas`.
* **Neutras:** Los ASJ deben añadir las operaciones de estadísticas y la relación `ServicioSuscripciones --> ServicioEstadisticas` en `Arquitecturas/`.

## Pros y Contras de las Opciones Consideradas

### Opción 1: Telemetría propia en PostgreSQL 19 (`AdminRedaccionDAO`)
* **A favor (+):** Aprovecha los componentes actuales (`EstadisticasController`, `ServicioEstadisticas`, `AdminRedaccionDAO`) sin infraestructura nueva.
* **A favor (+):** No falla por bloqueadores de rastreo y registra conversiones reales desde el backend manteniendo el anonimato.
* **En contra (-):** Añade escrituras a PostgreSQL 19 en picos de tráfico y obliga a mantener la lógica de agregación propia.

### Opción 2: Herramienta externa de analítica web (Google Analytics / Matomo SaaS)
* **A favor (+):** Trae ya hechos el cálculo de permanencia, rebote y paneles gráficos sin programar nada en el backend.
* **En contra (-):** Los bloqueadores del navegador cortan sus peticiones y falsean las cifras de lectura y conversión.
* **En contra (-):** Dificulta cuadrar las conversiones confirmadas por Stripe en el backend y envía datos de uso a terceros.

### Opción 3: Pipeline de eventos con Kafka y ClickHouse
* **A favor (+):** Soporta mucho volumen de eventos sin cargar la base PostgreSQL 19.
* **En contra (-):** Añade demasiada complejidad y dos sistemas nuevos innecesarios para el tamaño del periódico.
* **En contra (-):** Incumple la restricción de usar únicamente las dos bases de datos PostgreSQL 19 del sistema.

## Validación y Trazabilidad UML
* **Modelado ASJ:** David Sticea y Elena Romera deben reflejar en `Arquitecturas/`:
  1. En `EstadisticasController` (o `EstadisticasRestController`): `registrarEventoUso()` y `consultarEstadisticas()`.
  2. En `ServicioEstadisticas`: `procesarEvento()`, `registrarConversion()` y `obtenerMetricas()`.
  3. Dependencia `ServicioSuscripciones --> ServicioEstadisticas : notificar conversion`.
  4. Persistencia únicamente a través de `ServicioEstadisticas --> AdminRedaccionDAO`, sin acceso a `SuscriptoresDAO`.
* **Notas de Discusión ASS-ASC:**
  * *Pendiente de contraste con Sergio González y Sara Guillen (ASC):* Completar tras la revisión. Puntos a tratar: carga de escrituras en PostgreSQL 19 frente a agrupar eventos en memoria, y cómo definir la sesión temporal del rebote sin perder anonimato.

## Referencias
* MDN Web Docs, "Navigator: sendBeacon() method", [https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon)
* MDN Web Docs, "Page Visibility API (`visibilitychange`)", [https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API)
* PostgreSQL Global Development Group, "Materialized Views", [https://www.postgresql.org/docs/current/rules-materializedviews.html](https://www.postgresql.org/docs/current/rules-materializedviews.html)
