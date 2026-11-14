# 0004 - Gestión de Pagos con Stripe Billing, Webhooks y Reintentos

* **Estado:** aceptada
* **Fecha:** 2026-11-14
* **Decisores (ASS):** Antonio Machuca, Enrique Aldama
* **Consultados (ASC):** Sergio González, Sara Guillen
* **Informados (ASJ):** David Sticea, Elena Romera

## Contexto y Planteamiento del Problema
El periódico digital ofrece tres modalidades de suscripción de pago: trimestral (70 €), semestral (130 €) y anual (190 €). Los cobros deben realizarse a través de una pasarela externa, que según el catálogo de requisitos debe ser Stripe, y cuando un pago falla el sistema debe permitir reintentarlo hasta tres veces. En la arquitectura vigente (ADR-0001.3 y ADR-0002), el módulo de facturación del backend es el único que se comunica con Stripe.

La pregunta arquitectónica a resolver es: ¿quién es responsable de planificar los cobros recurrentes, detectar los fallos y ejecutar los tres reintentos: nuestro propio backend (cron, cola de mensajes) o la propia pasarela, notificando el resultado al sistema de forma asíncrona?

## Requisitos Motivadores (Decision Drivers)
* **Requisito(s) Clave:** RF5 - Reintentos de pago (Cuando falla un pago, el sistema debe permitir reintentarlo hasta tres veces).
* RF4 - Cobro externalizado (El sistema debe realizar los cobros mediante una pasarela externa).
* RD1 - Uso de Stripe (La pasarela de pago utilizada debe ser Stripe).
* RF3 - Modalidades de suscripción (Suscripciones trimestrales, semestrales y anuales con sus importes).
* RD2 - Comunicación HTTP/REST: la notificación de resultados de pago debe encajar en el canal HTTP/REST ya definido.
* RF2 / RD3 - Anonimato y segregación de datos: los datos de tarjeta no deben almacenarse en la base de datos de suscriptores ni asociarse a datos personales del lector (ver ADR-0005).

## Opciones Consideradas
* **Opción 1:** Stripe Billing (suscripciones recurrentes y reglas de reintento en Stripe) con notificación de resultados mediante Webhooks.
* **Opción 2:** Cron Job local en el backend que planifica los cobros y reintenta los pagos fallidos invocando la API de Stripe.
* **Opción 3:** Cola de mensajes (RabbitMQ) en el backend para encolar cobros y reintentos con espera diferida.

## Decisión Adoptada
Opción elegida: "**Opción 1: Stripe Billing con Webhooks**", porque:
* Cumple RF4 y RD1 de forma directa: la renovación periódica de cada modalidad la planifica Stripe a partir de los tres importes configurados en la pasarela, de modo que el backend no calcula ni dispara cobros.
* Cumple RF5: la política de reintentos (máximo 3 tras un fallo) la configura y ejecuta Stripe, y el sistema solo reacciona a los eventos que recibe.
* Encaja con el estilo Cliente-Servidor REST (RD2): Stripe notifica los resultados mediante HTTP a un endpoint del backend, sin añadir infraestructura nueva (ni planificador ni broker).
* Mantiene la privacidad: los datos de tarjeta los gestiona exclusivamente Stripe, y el sistema solo almacena identificadores opacos de Stripe vinculados al token anónimo del suscriptor.

### Consecuencias
* **Positivas:** Se elimina del backend la lógica de planificación de cobros y de temporización de reintentos; el estado del cobro tiene una única fuente de verdad (Stripe); menos componentes que modelar y desplegar; cobros y reintentos continúan aunque el backend esté caído temporalmente.
* **Negativas:** Dependencia fuerte de Stripe; el endpoint de notificaciones es una superficie de entrada pública que debe autenticar a Stripe y tolerar eventos duplicados o desordenados; el estado local es eventualmente consistente con el de Stripe.
* **Neutras:** Los ASJ deben modelar la recepción de notificaciones de Stripe como una dependencia en sentido inverso, de Stripe hacia el backend, y reflejar el estado de pago de la suscripción. Esta recepción no discrimina dispositivo (RF1), por lo que no necesita pasar por el API Gateway.

## Pros y Contras de las Opciones Consideradas

### Opción 1: Stripe Billing con Webhooks
* **A favor (+):** Los tres importes y sus periodicidades quedan definidos en Stripe, sin duplicar esa lógica en el backend del periódico.
* **A favor (+):** Los 3 reintentos de RF5 son una configuración de Stripe, no código propio que probar y mantener.
* **A favor (+):** El backend solo necesita un punto de entrada REST adicional, coherente con el estilo adoptado en ADR-0001.3.
* **A favor (+):** La tarjeta del lector nunca llega a nuestro sistema, reduciendo el alcance de seguridad y preservando el anonimato.
* **En contra (-):** Obliga a autenticar las notificaciones recibidas y a tratarlas de forma idempotente.
* **En contra (-):** Cambiar la política de reintentos requiere modificar la configuración en Stripe, no solo código propio.

### Opción 2: Cron Job local
* **A favor (+):** Control total del calendario de cobros y de la política de reintentos dentro del código propio.
* **En contra (-):** El backend debe guardar fechas de renovación y contadores de intentos, y revisar periódicamente las suscripciones para localizar cobros pendientes, lo que replica funcionalidad que Stripe ya ofrece.
* **En contra (-):** Introduce estado temporal y un proceso planificado que contradicen el diseño stateless del backend (ADR-0001.3); si el proceso falla o se duplica, hay cobros omitidos o dobles.
* **En contra (-):** Obliga a guardar un método de pago reutilizable y a iniciar los cobros desde el servidor, aumentando la responsabilidad del sistema sobre datos de pago.

### Opción 3: Cola de mensajes (RabbitMQ)
* **A favor (+):** Reintentos con espera diferida y desacoplados del flujo HTTP, con buena tolerancia a fallos.
* **En contra (-):** Añade un broker de mensajería que no existe en la arquitectura, desproporcionado para un caso de estudio de 6 requisitos funcionales (mismo argumento que descartó los microservicios orientados a eventos en ADR-0001.3).
* **En contra (-):** Sigue siendo necesario planificar las renovaciones periódicas y la lógica de reintentos, solo que desplazadas a los consumidores de la cola.

## Validación y Trazabilidad UML
* **Modelado ASJ:** David Sticea y Elena Romera deben reflejar esta decisión en `REPO/Arquitecturas/`, ampliando el diagrama de la semana 2 con:
  1. El componente de facturación del backend, que se comunica con la pasarela Stripe en ambos sentidos: peticiones del backend a Stripe y notificaciones de Stripe al backend.
  2. El estado de pago de la suscripción dentro del almacén de suscriptores.
* **Notas de Discusión ASS-ASC:**
  * Pendiente de contraste con Sergio González y Sara Guillen (ASC): debe completarse tras la sesión de revisión con las objeciones planteadas y la respuesta técnica de los ASS. Puntos previstos a discutir: sobreingeniería de los webhooks frente al cron, y gestión de eventos duplicados o desordenados.

## Referencias
* Stripe, "Billing: Subscriptions overview", [https://docs.stripe.com/billing/subscriptions/overview](https://docs.stripe.com/billing/subscriptions/overview)
* Stripe, "Automatic retries (Smart Retries) for failed payments", [https://docs.stripe.com/billing/revenue-recovery/smart-retries](https://docs.stripe.com/billing/revenue-recovery/smart-retries)
* Stripe, "Receive events in your webhook endpoint", [https://docs.stripe.com/webhooks](https://docs.stripe.com/webhooks)
* Chris Richardson, "Pattern: Idempotent Consumer", [https://microservices.io/patterns/communication-style/idempotent-consumer.html](https://microservices.io/patterns/communication-style/idempotent-consumer.html)
* Documentación oficial DAS, enunciado de la práctica (Requisitos de cobro externalizado, Stripe y reintentos de pago).
