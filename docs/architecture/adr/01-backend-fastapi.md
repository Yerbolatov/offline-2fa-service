# ADR-01: Selecting FastAPI for the Backend

- **Status:** Accepted
- **Date:** 2026-10-17

## Context

A small self-hosted backend is needed for an isolated network. The MVP must provide a REST API, a forward-auth endpoint, OpenAPI documentation, RBAC, auditing, SQLite support, and cryptographic operations. External cloud dependencies are not allowed, and deployment must be possible in Docker or an offline installer.

FastAPI, Django, and Go were considered. The comparison is provided in [technologies.md](../../literature-review/technologies.md).

## Decision

Choose **FastAPI** as the MVP server platform. Use explicit Pydantic schemas for input data and an automatically generated OpenAPI schema. Implement cryptography, secret storage, rate limiting, and authorization in separate tested modules; do not treat FastAPI itself as a substitute for security controls.

## Consequences

Benefits:

- fast path to a typed REST API and data validation;
- small application footprint and convenient offline/container deployment;
- Python ecosystem support for Argon2id, TOTP, AES-GCM, QR, and SQLite;
- simpler implementation of the research MVP and its tests.

Risks and mitigations:

- MFA/RBAC security is not provided automatically: add separate services, dependency checks, auditing, and security tests;
- incorrect use of async can block the event loop: run blocking operations in a synchronous layer or thread pool;
- pin the Python runtime and dependencies in a lock file and ship them in an offline image/wheel cache.

## Alternatives

- **Django:** a mature monolith with an ORM and admin, but it adds more components and configuration for a focused MFA API.
- **Go:** convenient for a single binary and high concurrency, but it requires more custom code for the ORM, admin workflow, schemas, and cryptographic integrations.
