---
status: "proposed"
date: 2026-11-03
---

# Procesamiento de notificaciones de constancias

## Context and Problem Statement

ConstanciaUV debe validar al estudiante, generar y almacenar su constancia y mantener el historial, además de notificarle cuando esté disponible. El correo depende de un proveedor externo; por RT2 y RT3, una interrupción de ese proveedor no debe impedir la generación ni el almacenamiento, y las notificaciones pendientes deben conservarse para reintentos. ¿Cómo organizar la comunicación para cumplir estos requisitos y prevenir solicitudes duplicadas por constancia (RT4), sin depender de que el proveedor garantice una entrega única?

## Decision Drivers

* Continuidad y durabilidad: aislar la generación y el almacenamiento de constancias de la disponibilidad del proveedor, conservando los pendientes para reintentos (RT2 y RT3).
* Control de duplicados y separación de responsabilidades: evitar solicitudes duplicadas para una constancia y limitar el acoplamiento entre la generación y el envío (RT4).

## Considered Options

* Alternativa A: comunicación directa con recuperación de fallos.
* Alternativa B: procesamiento asíncrono mediante eventos persistidos.

## Decision Outcome

Opción elegida: "Alternativa B: procesamiento asíncrono mediante eventos persistidos", porque separa el ciclo de generación del envío y permite que un componente independiente procese los pendientes cuando el proveedor esté disponible. El registro persistente desacopla la generación de la disponibilidad del correo (RT2 y RT3); una restricción de unicidad por constancia y el control idempotente del procesamiento permiten prevenir solicitudes duplicadas (RT4), sin atribuir esa garantía al proveedor.

### Consequences

* Beneficio: la generación y el almacenamiento pueden concluir aunque el proveedor no responda; el pendiente queda disponible para reintentos.
* Beneficio: el procesamiento y la recuperación de notificaciones tienen una responsabilidad separada del flujo de generación.
* Costo: se debe mantener el almacenamiento de eventos o pendientes, el estado de procesamiento y la coordinación del componente que los consume.
* Riesgo y compromiso: el procesamiento asíncrono puede retrasar la notificación y requiere gestionar reintentos, concurrencia y fallos. La unicidad local evita crear solicitudes duplicadas para una constancia, pero no garantiza por sí sola que el proveedor entregue el correo una sola vez.

### Confirmation

En una implementación futura, se comprobaría mediante pruebas que:

* una falla o indisponibilidad del proveedor no impida generar ni almacenar la constancia;
* cada constancia produzca como máximo un registro de notificación, incluso ante solicitudes repetidas o procesamiento concurrente;
* los pendientes permanezcan almacenados tras fallos y se reintenten cuando el proveedor vuelva a estar disponible;
* el componente de procesamiento no cree solicitudes duplicadas al recuperar o reintentar un evento;
* una revisión de diseño y código confirme que el envío se realiza fuera del flujo de generación y que la persistencia del pendiente no depende de la respuesta del proveedor.

Estas son comprobaciones propuestas; no se afirma que se hayan ejecutado.

## Pros and Cons of the Options

### Alternativa A: comunicación directa con recuperación de fallos

La aplicación intenta solicitar el envío después de almacenar la constancia y registra el pendiente si el proveedor no responde.

* Ventaja: en condiciones normales, la solicitud de envío comienza inmediatamente después de generar la constancia.
* Ventaja: el flujo de envío puede ser más sencillo de seguir al estar cerca de la operación que origina la notificación.
* Desventaja: la aplicación que genera constancias también debe gestionar la comunicación, los fallos y los reintentos, aumentando su acoplamiento con el proveedor.
* Desventaja: la disponibilidad o latencia del proveedor puede afectar el tiempo de respuesta del flujo de generación, aunque no deba impedir que la constancia se almacene (RT2).
* Desventaja: requiere persistir los pendientes y controlar duplicados de todas formas; el intento directo no resuelve por sí solo RT3 ni RT4.

### Alternativa B: procesamiento asíncrono mediante eventos persistidos

Después de almacenar la constancia, la aplicación registra un pendiente persistente que procesa posteriormente un componente independiente.

* Ventaja: la generación y el almacenamiento no necesitan esperar al proveedor y pueden continuar durante una interrupción (RT2).
* Ventaja: los pendientes persistidos pueden conservarse y reintentarse cuando el proveedor vuelva a estar disponible (RT3).
* Ventaja: una clave única por constancia y un procesamiento idempotente permiten prevenir solicitudes de notificación duplicadas dentro del sistema (RT4).
* Ventaja: separa la responsabilidad de generar constancias de la de comunicarse con el proveedor externo.
* Desventaja: añade un componente de procesamiento y exige operar y supervisar la cola o el almacén de pendientes.
* Desventaja: el envío puede ocurrir con demora y es necesario manejar reintentos, concurrencia y estados de procesamiento.
* Desventaja: el control local de duplicados no garantiza una entrega única por parte del proveedor; esa limitación debe considerarse en la política de reintentos.

## More Information

Se registra la revisión de esta propuesta solicitada para la Parte 4. La revisión está pendiente y no implica aprobación institucional; el estado permanece `proposed`.
