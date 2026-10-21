# ADR-02: Selecting SQLite for the MVP

- **Status:** Accepted for the MVP
- **Date:** 2026-10-17

## Context

The service is intended for autonomous installation inside a LAN. The expected MVP is one application instance with users, encrypted TOTP secrets, recovery codes, RBAC, and an audit log. An external database server must not be a required component of the offline installation.

SQLite and PostgreSQL were considered. Transactional TOTP verification, atomic prevention of code reuse, backups, and a possible later move to multiple instances are critical.

## Decision

Use **SQLite** as the MVP database. Enable transactions and an appropriate journaling mode, apply migrations, and back up the protected database file. Perform TOTP verification and recording of the used time step atomically. Keep the AES-256-GCM key outside the database and never write it to the audit log.

## Consequences

Benefits:

- minimal number of components and simple one-command deployment;
- fully local operation without a PostgreSQL server;
- convenient single-file backups and portability in an air-gapped environment;
- sufficient model for a single-node MVP.

Limitations:

- limited write concurrency and no built-in horizontal scaling;
- file access control, a backup policy, and restore verification are required;
- the database must not be placed on an unreliable shared network file;
- multiple instances will require PostgreSQL and a separate migration strategy.

## Alternative

**PostgreSQL** is better suited to high concurrency, multiple instances, centralized backups, and a future SIEM/audit pipeline, but it requires a separate service, volume, configuration, and maintenance. It therefore remains a scalable alternative rather than a baseline MVP dependency.
