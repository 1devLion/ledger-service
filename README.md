# ledger-service

[English](README.md) | [Español](README.es.md)

> Digital wallet backend in Java and Spring Boot: multi-currency accounts, double-entry ledger, idempotent transfers and external deposits.

**Status:** work in progress. Design phase: use cases are being written and the data model is next. Nothing below is implemented yet unless it is checked in the roadmap.

## What it is

A backend that behaves like a bank account service. Users register, open accounts in ARS or USD, and move money between accounts. Money also arrives from outside through authenticated external providers (a bank or a cash-payment network, for example).

The hard part is not the account, it is guaranteeing that money is **never lost or duplicated**, even with retries, crashes or concurrent requests.

## Scope of v1

- Register users and authenticate with JWT.
- Open accounts (one per currency per user), each with an alias and a 22-digit external account number.
- Receive external deposits from providers, authenticated with an API key and a signed request.
- Transfer money between accounts by alias or account number.
- Check balance, transaction history and transfer details.

**Out of scope for v1:** currency conversion, fees, scheduled transfers, real bank integrations, identity verification (KYC) and dispute reversals.

## Key design decisions

- **Money is never a floating point.** `BigDecimal` in Java and `NUMERIC(19,2)` in PostgreSQL.
- **Double-entry ledger.** Every movement creates two immutable entries (a debit and a credit) that sum to zero. An account balance must always equal the sum of its entries. Mistakes are fixed with new entries, never by editing old ones.
- **Idempotency.** Transfers use an `Idempotency-Key` per user and external deposits use an operation id per provider. Both are enforced by `UNIQUE` constraints in the database, so concurrent duplicates are handled safely.
- **Concurrency.** Accounts involved in an operation are locked in a consistent order (by id) to avoid deadlocks.
- **Settlement accounts.** External deposits debit a per-provider settlement account, so the sum of all entries in the system stays at zero and providers cannot credit more than they back.
- **Security.** Passwords hashed with BCrypt, login rate limiting by e-mail and IP, hashed API keys and signed requests (HMAC plus timestamp) for providers, and generic error messages that do not reveal whether an e-mail exists.
- **Architecture (planned).** Packages organized by feature (`account`, `transfer`, `ledger`...), with layers inside each one (`api`, `application`, `domain`, `infrastructure`) and interfaces only at the boundaries where they help testing or swapping dependencies.
- **Events (planned).** Transactional outbox pattern publishing to Kafka, with idempotent consumers.

## Tech stack (planned)

Java 21, Spring Boot, PostgreSQL, Flyway, Testcontainers, Docker Compose, GitHub Actions. Kafka in a later phase.

## Documentation

- [Use cases](docs/casos-de-uso.md) (Spanish; English version coming soon)

## Roadmap

- [x] Use cases CU-01 to CU-04
- [ ] Use cases CU-05 to CU-09
- [ ] Data model
- [ ] Project bootstrap (Spring Boot, PostgreSQL, Flyway, Testcontainers)
- [ ] Users and authentication
- [ ] Accounts
- [ ] External deposits
- [ ] Transfers and concurrency tests
- [ ] Transactional outbox and Kafka
- [ ] CI, observability and load test
- [ ] Public deployment

## Getting started

Coming soon.

## License

MIT