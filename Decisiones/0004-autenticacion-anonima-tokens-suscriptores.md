# 0004 - Autenticación Anónima de Suscriptores mediante Tokens Opacos

* **Estado:** propuesta
* **Fecha:** 2026-11-14
* **Decisores (ASS):** Antonio Machuca, Enrique Aldama
* **Consultados (ASC):** Sergio González, Sara Guillen
* **Informados (ASJ):** David Sticea, Elena Romera

## Contexto y Planteamiento del Problema
Los lectores pueden suscribirse y darse de baja de las categorías de noticias de pago de forma anónima. El sistema nunca debe solicitar nombre, DNI ni correo electrónico, pero debe poder reconocer a un suscriptor en cada petición para decidir si tiene acceso a una sección, gestionar su baja y asociarle su suscripción de pago en Stripe (ADR-0003). Además, debe encajar con las decisiones previas del sistema: comunicación HTTP/REST *stateless* (ADR-0001.3), propagación de peticiones a través del `ApiGateway` (ADR-0002.1) y persistencia exclusivamente no personal en la base de datos PostgreSQL 19 de suscriptores.

## Requisitos Motivadores (Decision Drivers)
* **Requisito Clave:** RF2 - Suscripción y baja anónima (Los lectores pueden suscribirse y darse de baja de las diferentes categorías de noticias de forma anónima).

## Opciones Consideradas
* **Opción 1:** Tokens opacos (UUID v4 aleatorio) emitidos por el servidor al suscribirse.
* **Opción 2:** Usuario y clave autogenerados por el sistema.
* **Opción 3:** Huella digital del navegador (browser fingerprinting).

## Decisión Adoptada
Opción elegida: "**Opción 1: Tokens opacos (UUID v4 aleatorio) emitidos por el servidor al suscribirse**", porque:
* Cumple RF2 estrictamente: el alta no recibe ningún dato personal; el servidor genera un UUID v4 (con generador criptográficamente seguro) que actúa a la vez como identificador de suscriptor y como credencial de acceso.
* Es stateless y compatible con RD2: el cliente envía el token en cada petición (`Authorization: Bearer <token>`), el `ApiGateway` lo reenvía sin interpretarlo y el `ServicioSuscripciones` lo valida contra la base de suscriptores, sin sesiones en memoria de servidor.
* Mantiene el anonimato incluso ante una filtración: en `SuscriptoresDAO` solo se guarda el hash (SHA-256) del token, nunca el token en claro, junto con las secciones suscritas, el estado de la suscripción y los identificadores opacos de Stripe (ADR-0003). Ninguno de esos datos identifica a una persona.
* Funciona igual en PC y Tablet, ya que no depende de características del dispositivo.

### Flujo (base para el diagrama)
1. **Alta:** el lector elige categorías y modalidad; el frontend envía `POST /suscripciones` sin datos personales. `ServicioSuscripciones` genera el UUID v4, guarda su hash y crea la suscripción en estado pendiente de pago.
2. **Entrega del token:** el token en claro se devuelve **una sola vez** en la respuesta y el frontend lo guarda en el navegador (almacenamiento local) y lo muestra al lector para que pueda copiarlo y recuperarlo en otro dispositivo.
3. **Pago:** el token se asocia al `customer_id`/`subscription_id` de Stripe creados sin datos personales (ADR-0003).
4. **Acceso:** en cada petición a noticias de pago, el frontend envía el token; el `ApiGateway` lo reenvía junto con `X-Device-Type` al `NoticiasController`, que consulta `ServicioSuscripciones` (hash del token → estado y secciones permitidas).
5. **Baja:** `DELETE /suscripciones` con el token; el sistema cancela en Stripe y elimina el registro asociado.

### Consecuencias
* **Positivas:** Anonimato real por diseño: no existe ningún dato personal que proteger en la base de suscriptores; implementación sencilla; sin estado de sesión en el servidor; punto único de validación (`ServicioSuscripciones`).
* **Negativas:** Quien pierda el token pierde el acceso, y no hay recuperación posible por correo ni por datos personales (consecuencia inherente al anonimato); quien obtenga el token puede usar la suscripción (hay que servirlo siempre por HTTPS y mostrar al lector una advertencia al emitirlo); el almacenamiento local del navegador es vulnerable a XSS.
* **Neutras:** Los ASJ deben modelar el campo `tokenHash` en la entidad de suscriptor y un componente de validación en el backend. El `ApiGateway` no valida el token (mantiene solo sus dos funciones de ADR-0002.1), solo lo propaga.

## Pros y Contras de las Opciones Consideradas

### Opción 1: Tokens opacos (UUID v4)
* **A favor (+):** Preserva el anonimato estricto de RF2 y permite uso multidispositivo (PC y Tablet): el UUID v4 aleatorio no contiene metadatos personales y en `SuscriptoresDAO` solo se guarda su hash SHA-256.
* **A favor (+):** Flujo mínimo y *stateless*: evita endpoints de inicio de sesión o gestión de contraseñas; `ApiGateway` se limita a propagar la cabecera `Authorization: Bearer` hacia `ServicioSuscripciones`.
* **En contra (-):** Cada petición a noticias de pago exige una consulta de validación a `SuscriptoresDAO` (al no ser un token autofirmado) y la credencial queda expuesta a robo por XSS si se almacena en el navegador.

### Opción 2: Usuario y clave autogenerados
* **A favor (+):** Modelo de inicio de sesión familiar para el lector al trasladar su acceso entre PC y Tablet.
* **En contra (-):** Duplica la complejidad frente a un único token: obliga a añadir un endpoint de *login* en `SuscripcionesController`, gestionar *salts* de contraseña e igualmente emitir un token o credencial para las peticiones siguientes.
* **En contra (-):** No aporta recuperación real: al estar prohibido recoger correo electrónico por RF2, olvidar la contraseña autogenerada bloquea el acceso igual que perder un token.

### Opción 3: Huella digital del navegador
* **A favor (+):** Experiencia sin fricción para el lector, que no necesita copiar, guardar ni introducir ninguna credencial manual en el mismo navegador.
* **En contra (-):** Impide usar la misma suscripción entre PC y Tablet y resulta frágil ante actualizaciones del navegador o extensiones de privacidad.
* **En contra (-):** Entra en conflicto con el principio de privacidad de RF2 al recolectar atributos del dispositivo que actúan como rastreo pasivo.

## Validación y Trazabilidad UML
* **Modelado ASJ:** David Sticea y Elena Romera deben reflejar en `Arquitecturas/`, ampliando el diagrama de la semana 2:
  1. `SuscripcionesController` con las operaciones de alta (que emite el token) y baja.
  2. `ServicioSuscripciones` con las operaciones `generarToken()` y `validarToken()`, y una clase auxiliar `TokenSuscriptor` (UUID v4 y hash SHA-256).
  3. En `SuscriptoresDAO`, la entidad `Suscriptor` con `tokenHash`, `secciones`, `estado` y los identificadores opacos de Stripe, **sin** atributos de nombre, DNI ni correo.
  4. Una anotación (`note`) en el `ApiGateway` indicando que propaga la cabecera `Authorization: Bearer <token>` sin interpretarla, junto con `X-Device-Type`.
  5. Las peticiones del frontend al `ApiGateway` indicando `HTTP/REST + token` y la dependencia `NoticiasController --> ServicioSuscripciones` para comprobar el acceso a las secciones de pago.
* **Notas de Discusión ASS-ASC:**
  * *Pendiente de contraste con Sergio González y Sara Guillen (ASC):* Debe completarse tras la sesión de revisión con las objeciones planteadas y la respuesta técnica de los ASS. Puntos previstos a discutir: recuperación de acceso por pérdida del token y riesgo de robo del token en el navegador.

## Referencias
* OWASP, "Session Management Cheat Sheet", [https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
* IETF, "RFC 9562 - Universally Unique IDentifiers (UUIDs)", [https://www.rfc-editor.org/rfc/rfc9562](https://www.rfc-editor.org/rfc/rfc9562)
* IETF, "RFC 6750 - The OAuth 2.0 Authorization Framework: Bearer Token Usage", [https://www.rfc-editor.org/rfc/rfc6750](https://www.rfc-editor.org/rfc/rfc6750)
* Electronic Frontier Foundation, "Cover Your Tracks (browser fingerprinting)", [https://coveryourtracks.eff.org/](https://coveryourtracks.eff.org/)
