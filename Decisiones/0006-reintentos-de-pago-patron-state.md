# 0006 - Reintentos de Pago Fallido mediante Patrón State

* **Estado:** propuesta
* **Fecha:** 2026-11-17
* **Decisores (ASS):** Antonio Machuca, Enrique Aldama
* **Consultados (ASC):** Sergio González, Sara Guillen
* **Informados (ASJ):** David Sticea, Elena Romera

## Contexto y Planteamiento del Problema
Con el cobro externalizado en Stripe Billing (ADR-0003), a veces un pago periódico falla y el sistema debe permitir reintentarlo hasta tres veces. La suscripción pasa entonces por varias situaciones (al corriente, con pago fallido y reintento en curso, suspendida) y el backend debe saber en cuál está para decidir qué se permite hacer. La pregunta es quién ejecuta los reintentos y cómo representa el backend la situación de pago de cada suscripción, sin guardar datos de tarjeta (ADR-0004).

## Requisitos Motivadores (Decision Drivers)
* **Requisito Clave:** RF5 - Reintentos de pago (Cuando falla un pago, el sistema debe permitir reintentarlo hasta tres veces).
* Restricciones del sistema: cobro con Stripe y notificación mediante webhooks (ADR-0003), anonimato de las suscripciones (ADR-0004) y ausencia de infraestructura nueva.

## Opciones Consideradas
* **Opción 1:** Reintentos gestionados por Stripe y modelado de la situación de pago en el backend con el patrón `State`.
* **Opción 2:** Reintentos controlados por `ServicioFacturacion` con un contador y el patrón `Strategy` para la política de reintento.
* **Opción 3:** Cola de mensajes (RabbitMQ) con espera diferida para encolar los reintentos.

## Decisión Adoptada
Opción elegida: "**Opción 1: Reintentos gestionados por Stripe y patrón `State` en el backend**", porque:
* Cumple RF5: Stripe aplica la política de hasta tres reintentos tras un fallo y el backend solo reacciona a los eventos que recibe, sin temporizadores propios.
* **Representa el ciclo de pago mediante `State`:** la suscripción (`Suscripcion`) delega su comportamiento en un estado (`EstadoPago`) con variantes como `AlCorriente`, `PagoFallido` (reintento en curso) y `Suspendida`. Cada evento de Stripe provoca una transición, y las reglas de qué se puede hacer en cada situación (por ejemplo, acceso a contenido de pago) quedan en cada estado y no en condicionales dispersos.
* Reutiliza el `Observer` definido en ADR-0003: `ServicioSuscripciones` recibe los resultados de pago y los traslada al estado de la suscripción.
* No añade infraestructura: no hacen falta planificador ni broker.

### Consecuencias
* **Positivas:** Se elimina del backend la lógica de temporización y de conteo de intentos; las transiciones permitidas están explícitas y son fáciles de revisar; añadir un estado nuevo no obliga a tocar los existentes.
* **Negativas:** Dependencia de la política de reintentos de Stripe; el estado local puede ir con retraso respecto al de Stripe si falla un webhook; hay más clases que modelar (una por estado).
* **Neutras:** Los ASJ deben modelar la interfaz `EstadoPago` y sus implementaciones, y mostrar que las transiciones se disparan desde las notificaciones de Stripe.

## Pros y Contras de las Opciones Consideradas

### Opción 1: Reintentos gestionados por Stripe y patrón `State`
* **A favor (+):** Stripe ejecuta los hasta 3 reintentos y evita programar contadores de intentos ni bloqueos contra cobros duplicados en `ServicioFacturacion`.
* **A favor (+):** El patrón `State` deja claro cuándo una suscripción está al corriente, en reintento o suspendida, sin condicionales repartidos por `ServicioSuscripciones`.
* **En contra (-):** El intervalo entre reintentos depende de lo que ofrezca Stripe y el backend no lo controla directamente.
* **En contra (-):** Si el endpoint de webhooks falla, el estado local queda desfasado del cobro real.

### Opción 2: Reintentos en `ServicioFacturacion` con `Strategy`
* **A favor (+):** Control total de los intervalos entre los 3 reintentos, y la política se puede cambiar sin tocar el resto (`Strategy`).
* **A favor (+):** Actualización inmediata del estado al recibir la respuesta de la API.
* **En contra (-):** Obliga a guardar el contador de intentos y la fecha del siguiente reintento en `SuscriptoresDAO`, y a evitar cobros dobles con varias instancias del backend.
* **En contra (-):** Reimplementa lógica que Stripe ya ofrece y aumenta el riesgo de inconsistencias si el servidor cae a mitad de un reintento.

### Opción 3: Cola de mensajes (RabbitMQ) con espera diferida
* **A favor (+):** Desacopla los reintentos de las peticiones HTTP y conserva los pendientes si el servicio de facturación se reinicia.
* **En contra (-):** Introduce un broker desproporcionado para el alcance del sistema (mismo criterio de contención de ADR-0001.3).
* **En contra (-):** Solo programa el envío; sigue haciendo falta contar intentos y decidir cuándo se agotan.

## Validación y Trazabilidad UML
* **Modelado ASJ:** David Sticea y Elena Romera deben reflejar esta decisión en `Arquitecturas/`:
  1. Definir la interfaz `EstadoPago` (`<<interface>>`) con las clases `AlCorriente`, `PagoFallido` y `Suspendida`, y la relación con `Suscripcion` (`Context`).
  2. Mostrar que `ServicioSuscripciones`, como observador del `WebhookController` (ADR-0003), provoca las transiciones de estado.
  3. Incluir el estado de pago de la suscripción en el almacén de suscriptores, sin datos de tarjeta.
* **Notas de Discusión ASS-ASC:**
  * Pendiente de contraste con Sergio González y Sara Guillen (ASC): completar tras la sesión de revisión. Puntos previstos: dependencia de los reintentos de Stripe frente a un control propio, y coherencia del estado local ante eventos duplicados o desordenados.

## Referencias
* Stripe, "Automatic retries (Smart Retries) for failed payments", [https://docs.stripe.com/billing/revenue-recovery/smart-retries](https://docs.stripe.com/billing/revenue-recovery/smart-retries)
* Erich Gamma et al., *Design Patterns: Elements of Reusable Object-Oriented Software* (Patrones `State`, `Strategy` y `Observer`).
* Refactoring Guru, "State Pattern", [https://refactoring.guru/design-patterns/state](https://refactoring.guru/design-patterns/state)
* Refactoring Guru, "Strategy Pattern", [https://refactoring.guru/design-patterns/strategy](https://refactoring.guru/design-patterns/strategy)
