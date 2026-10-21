# Comparison of Technologies and Architectural Solutions

## Context of the Choice

- The service must operate entirely within an isolated network without any cloud-based channels;
- The MVP includes a username and password, TOTP, backup codes, RBAC, auditing, and HTTPS with an internal CA;
- The integration must support REST API and forward-auth for reverse proxy;
- Deployment must be simple and reproducible in Docker or via an offline installer;
- TOTP secrets must be encrypted with a key stored separately from the database;
- OIDC, LDAP/AD, and WebAuthn are optional features, not mandatory dependencies for the MVP.

A “high/medium/low” rating indicates suitability specifically for this project, not the technology's overall popularity.

## 1. Server Platform

We will consider FastAPI, Django, and Go with an HTTP framework (such as Gin or Chi).

| Criterion | FastAPI | Django | Go |
|---|---|---|---|
| REST API implementation | High: typing, OpenAPI, and validation are part of the typical FastAPI stack ([docs](https://fastapi.tiangolo.com/)) | High via Django REST Framework, but it adds a separate layer ([DRF](https://www.django-rest-framework.org/)) | High, but routes, schemas, and OpenAPI are typically handled by additional libraries |
| Security and Maturity | High when using proven libraries; responsibility for MFA/RBAC remains with the project | High: mature middleware, ORM, CSRF, and auth components ([security](https://docs.djangoproject.com/en/stable/topics/security/)) | High at the language and standard library level, but more infrastructure code needs to be written from scratch ([net/http](https://pkg.go.dev/net/http)) |
| MVP Simplicity | High: a small async/sync service and explicit schemas | Medium: built-in admin/auth features are useful, but a Django project is usually more extensive than necessary | Medium: a single binary is convenient, but developing CRUD, migrations, and admin functions takes longer |
| Offline Deployment | High: dependencies can be fixed in the wheel cache and container | High, but the package is heavier and requires more configuration | Very high after building: a single static binary reduces runtime dependencies |
| Project Compatibility | High: Easy to implement REST, forward authentication, OpenAPI, and cryptographic services | Medium: A robust monolithic stack is overkill for a narrow-scope MFA service | Medium: Good for a high-traffic gateway, but not necessary for the expected scale |

**Choice: FastAPI.** It provides a quick path to a typed REST API, OpenAPI, and input validation, while keeping the application size small. Django remains a reasonable choice for a large admin panel, while Go is suitable for a separate, high-traffic gateway; however, both options increase the cost of the MVP. Asynchronous operations should not be used automatically: database and cryptographic operations must be performed using proven synchronous or external mechanisms to avoid blocking the event loop.

## 2. Data Storage

PostgreSQL and SQLite are compared. Both DBMSs run locally and do not require an internet connection; the choice depends on the size of the installation, write contention, and operating system requirements.

| Criterion | PostgreSQL | SQLite |
|---|---|---|
| Offline operation | High: The server is hosted on the LAN and does not require external services ([docs](https://www.postgresql.org/docs/)) | Very high: Built-in file-based database without a separate server ([about](https://www.sqlite.org/about.html)) |
| Concurrent Requests and Auditing | High: MVCC and a separate server are better suited for concurrent reads and the log | Medium: WAL improves read performance, but write operations remain a limited resource ([WAL](https://www.sqlite.org/wal.html)) |
| Deployment | Medium: Requires a separate container, volume, backup, and migration | Very high: A single file and a minimum of services |
| Backup Reliability | High: built-in backup/replication tools | Medium/high for a single node with proper online backup ([backup](https://www.sqlite.org/backup.html)) |
| Scalability | High for multiple service instances | Low for horizontal scaling and a shared network file |
| MVP Compliance | Medium: Overkill for a single small installation, but better for high-load scenarios | High: Minimizes dependencies and is suitable for a standalone single-node deployment |

**Choice: SQLite for MVP.** The required scenario is a single standalone service on the internal network; SQLite reduces the deployment footprint and does not require a separate database server. The database must reside on a local, secure volume and use WAL/transactions, regular backups, and migrations. PostgreSQL should remain a supported option for multiple instances, high concurrency, centralized backup, or future audit export. The encryption key for TOTP secrets is not stored in SQLite.

## 3. Integration Method

REST APIs, OIDC, and forward authentication is discussed. These are not entirely mutually exclusive: REST and forward authentication are part of the MVP, while OIDC can be added later.

| Criterion | REST API | OIDC | Forward-auth |
|---|---|---|---|
| Primary Scenario | Direct call by an internal application | Federated login/SSO and claims | Reverse proxy request validation before upstream |
| Offline Operation | High: internal HTTP endpoint | High when using a local IdP, but requires an issuer, keys, and client configuration | Very high: proxy and auth service operate within the LAN |
| Client complexity | Medium: the application handles sessions and errors on its own | High: discovery, redirect URI, state/nonce, token validation | Low for an application running behind Nginx, but high for correctly configuring trusted headers |
| Security | Depends on mTLS/API key, scopes, and rate limiting | Strong token model, but more keys and trust relationships ([OIDC](https://openid.net/specs/openid-connect-core-1_0.html)) | Minimal authorization protocol, but trust in the proxy and cleaning of forwarded headers are critical |
| MVP coverage | Yes | Additional goal | Yes |
| Compatibility | Internal Applications and Automation | Many applications that require a standard IdP | Nginx and other reverse proxies |

**Choice: REST API and forward-auth in the MVP; OIDC as an extension.** REST is needed for user management, 2FA setup, and application integration. Forward-auth addresses the practical scenario of securing internal web applications without embedding MFA into every application. OIDC is not being ruled out: it will become the next layer for federation, but it cannot be made a mandatory dependency for the first offline version due to its larger configuration and cryptographic attack surface.

## 4. Second Factor

TOTP and WebAuthn are compared.

| Criterion | TOTP | WebAuthn |
|---|---|---|
| Offline User Operation | High: The authenticator app generates a code without an internet connection | Possible within a LAN, but requires HTTPS, a valid origin, and a compatible authenticator |
| Phishing Protection | Medium: The entered code can be intercepted | High: The credential is tied to the origin and uses a challenge-response mechanism |
| Implementation complexity | Low/medium: RFC 6238, QR enrollment, and time window | High: challenge, RP ID, attestation/assertion, browser/device compatibility |
| Recovery | Backup one-time codes and administrative reset | Need a backup authenticator or recovery flow |
| MVP Compliance | High | Medium |

**Choice: TOTP for MVP; WebAuthn as an extension.** TOTP meets the goal of a simple, standalone service. Security requirements: encryption of secrets, time window, rate limiting, and prohibition of retries. WebAuthn should be added after TOTP has stabilized, while preserving the ability to enhance phishing resistance without changing the underlying deployment.

## Summary of Decisions

| Area | MVP Solution | Deferred/Scalable Alternative |
|---|---|---|
| Backend | FastAPI | Django for a large admin platform or Go for a separate gateway |
| Database | SQLite | PostgreSQL for multi-instance and high concurrency |
| Integration | REST + forward-auth | OIDC for federation |
| Second factor | TOTP | WebAuthn for a phishing-resistant scenario |

