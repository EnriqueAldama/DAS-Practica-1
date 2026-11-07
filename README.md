# Sistema de Periodico Digital - Practica 1

## Informacion General
* **Asignatura:** Diseno y Arquitectura del Software (DAS)
* **Curso Academico:** 2026-2027
* **Institucion:** Universidad Rey Juan Carlos (URJC)

## Equipo de Trabajo y Asignacion de Roles
El equipo aplica la separacion de roles establecida en la normativa de la asignatura:

| Integrante | Rol Asignado | Responsabilidades Principales |
| :--- | :--- | :--- |
| **Antonio Machuca** | Arquitecto Software Senior (ASS) | Formulacion y captura de decisiones arquitectonicas en formato MADR. |
| **Enrique Aldama** | Arquitecto Software Senior (ASS) | Formulacion y captura de decisiones arquitectonicas en formato MADR. |
| **Sergio Gonzalez** | Arquitecto Software Cognitivo (ASC) | Cuestionamiento critico, deteccion de fricciones y propuesta de alternativas. |
| **Sara Guillen** | Arquitecto Software Cognitivo (ASC) | Cuestionamiento critico, analisis de requisitos y validacion de arquitectura. |
| **David Sticea** | Arquitecto Software Junior (ASJ) | Modelado y documentacion UML en PlantUML (diagramas de clases y paquetes). |
| **Elena Romera** | Arquitecto Software Junior (ASJ) | Modelado y documentacion UML en PlantUML (diagramas de clases y paquetes). |

## Estructura del Repositorio
El repositorio organiza los elementos del diseno en tres carpetas:

* `Requisitos/`: Contiene los requisitos funcionales y de dominio del sistema.
* `Decisiones/`: Contiene los registros de decision arquitectonica (ADR) en formato MADR estandar.
* `Arquitecturas/`: Contiene los modelos UML en formato PlantUML (.puml) y sus exportaciones graficas (.svg, .png).

## Entregables de la Semana 1

### 1. Catalogo de Requisitos
* **Ubicacion:** `Requisitos/REQUISITOS.xlsx`
* **Descripcion:**
  * **12 Requisitos Funcionales (RF1 a RF12):** 
  * **9 Requisitos de Dominio (RD1 a RD9):** 

### 2. Registro de Decisiones Arquitectonicas (MADR)
* **ADR-0001: Seleccion del Estilo Arquitectonico Global**
  * *Requisito Clave:* RF1 (Discriminacion del cliente) y RD9 (Clientes soportados PC y Tablet).
  * *Decision Inicial:* Monolito en capas MVC con renderizado en servidor para validar el dominio en el arranque.
  * *Estado:* Superada por ADR-0003 tras el debate tecnico con los ASC.
* **ADR-0002: Estrategia de Segregacion de Datos PostgreSQL 19**
  * *Requisito Clave:* RD5 (Segregacion fisica obligatoria de datos) y RD2 (Preservacion de anonimato en suscripciones).
  * *Decision:* Dos instancias independientes de PostgreSQL 19 en servidores y puertos separados (5432 y 5433) conectadas mediante DAOs desacoplados (`AdminRedaccionDAO` y `SuscriptoresDAO`).
  * *Estado:* Aceptada.
* **ADR-0003: Migracion a Estilo Arquitectonico Cliente-Servidor con API REST**
  * *Requisito Clave:* RD7 (Comunicacion mediante HTTP/REST).
  * *Decision:* Desacoplamiento de Frontend Web responsivo y Backend API REST sin estado, resolviendo las fricciones de rendimiento y cumplimiento normativo del monolito.
  * *Estado:* Aceptada (supera a ADR-0001).

### 3. Modelado Arquitectonico UML
* **Diagrama de Clases:**
