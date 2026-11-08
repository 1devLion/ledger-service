# ledger-service

[English](README.md) | [Español](README.es.md)

> Backend de billetera digital en Java y Spring Boot: cuentas multimoneda, libro contable de doble entrada, transferencias idempotentes e ingresos externos.

**Estado:** en desarrollo. Fase de diseño: se están escribiendo los casos de uso y el modelo de datos es lo siguiente. Nada de lo que sigue está implementado todavía, salvo lo marcado en la hoja de ruta.

## Qué es

Un backend que se comporta como un servicio de cuentas bancarias. Los usuarios se registran, abren cuentas en ARS o USD y mueven dinero entre cuentas. El dinero también llega desde afuera a través de proveedores externos autenticados (por ejemplo, un banco o una red de cobranza en efectivo).

Lo difícil no es la cuenta, sino garantizar que el dinero **nunca se pierda ni se duplique**, aunque haya reintentos, caídas o pedidos concurrentes.

## Alcance de la versión 1

- Registrar usuarios y autenticarlos con JWT.
- Abrir cuentas (una por moneda por usuario), cada una con un alias y un número de cuenta externo de 22 dígitos.
- Recibir ingresos externos de proveedores, autenticados con clave de API y pedido firmado.
- Transferir dinero entre cuentas por alias o número de cuenta.
- Consultar saldo, historial de movimientos y detalle de transferencias.

**Fuera de alcance en la versión 1:** conversión de monedas, comisiones, transferencias programadas, integraciones bancarias reales, verificación de identidad (KYC) y reversiones por reclamo.

## Decisiones de diseño

- **El dinero nunca es un número de coma flotante.** `BigDecimal` en Java y `NUMERIC(19,2)` en PostgreSQL.
- **Libro contable de doble entrada.** Cada movimiento crea dos asientos inmutables (un débito y un crédito) que suman cero. El saldo de una cuenta siempre debe coincidir con la suma de sus asientos. Los errores se corrigen con asientos nuevos, nunca editando los anteriores.
- **Idempotencia.** Las transferencias usan una `Idempotency-Key` por usuario y los ingresos externos usan un id de operación por proveedor. Ambos se garantizan con restricciones `UNIQUE` en la base, de modo que los duplicados concurrentes se manejan de forma segura.
- **Concurrencia.** Las cuentas involucradas en una operación se bloquean siempre en el mismo orden (por id) para evitar bloqueos cruzados.
- **Cuentas de liquidación.** Los ingresos externos debitan una cuenta de liquidación por proveedor, así la suma de todos los asientos del sistema se mantiene en cero y un proveedor no puede acreditar más de lo que respalda.
- **Seguridad.** Contraseñas con hash BCrypt, límite de intentos de login por e-mail e IP, claves de API hasheadas y pedidos firmados (HMAC más timestamp) para los proveedores, y mensajes de error genéricos que no revelan si un e-mail existe.
- **Arquitectura (planificada).** Paquetes organizados por funcionalidad (`account`, `transfer`, `ledger`...), con capas dentro de cada uno (`api`, `application`, `domain`, `infrastructure`) e interfaces solo en los bordes donde ayudan a testear o a cambiar dependencias.
- **Eventos (planificado).** Patrón transactional outbox publicando a Kafka, con consumidores idempotentes.

## Stack tecnológico (planificado)

Java 21, Spring Boot, PostgreSQL, Flyway, Testcontainers, Docker Compose, GitHub Actions. Kafka en una fase posterior.

## Documentación

- [Casos de uso](docs/casos-de-uso.md) (en español; versión en inglés próximamente)

## Hoja de ruta

- [x] Casos de uso CU-01 a CU-04
- [ ] Casos de uso CU-05 a CU-09
- [ ] Modelo de datos
- [ ] Arranque del proyecto (Spring Boot, PostgreSQL, Flyway, Testcontainers)
- [ ] Usuarios y autenticación
- [ ] Cuentas
- [ ] Ingresos externos
- [ ] Transferencias y tests de concurrencia
- [ ] Transactional outbox y Kafka
- [ ] CI, observabilidad y prueba de carga
- [ ] Despliegue público

## Cómo empezar

Próximamente.

## Licencia

MIT