# Requirements Specification

## 1. Purpose and Scope

This document defines the test requirements for a local two-factor authentication web service for an isolated corporate network. The requirements are based on the [project proposal](../proposal.md), the [technology ADR](../architecture/adr/01-backend-fastapi.md), and the [literature review](../literature-review/review.md).

The MVP includes login and password, TOTP, backup codes, RBAC, auditing, REST API, forward authentication, and standalone deployment with HTTPS. WebAuthn, OIDC, LDAP/AD, RADIUS, and SIEM export are considered optional features and are not required for the first release.

### 1.1. Priorities

- **MUST** — a mandatory requirement for the MVP; failure to meet this requirement blocks acceptance.
- **SHOULD** — an important requirement; may be deferred only with justification.
- **COULD** — an optional feature; does not block acceptance of the MVP.

### 1.2. Testability

An acceptance criterion is specified for each requirement. Verification is performed through unit, integration, or security testing; a manual acceptance test scenario; or a measurement in a test environment. Real passwords, keys, personal data, or production secrets must not be used in tests.

### 1.3. MVP Boundaries

The mandatory MVP includes only requirements with a **MUST** priority. **SHOULD** and **COULD** requirements outline important areas for development and are tested only if time permits and the appropriate implementation is available. Failure to meet these requirements does not, in and of itself, block the project’s first demonstration.

## 2. Threat Context

The model below is the minimal STRIDE-oriented decomposition for the MVP.
Threats are used in the requirements as references to the specific risks being controlled.

| ID | Threat | Example | Required Measures |
|---|---|---|---|
| T-01 | Spoofing | A stolen password or TOTP code is used to log in | Two-factor authentication, one-time codes, rate limiting, secure sessions |
| T-02 | Credential theft | password, TOTP secret, or recovery code read from the database/logs | Argon2id, AES-256-GCM, external key, secret masking |
| T-03 | Brute force | brute-forcing a password, TOTP, or recovery code | throttling, blocking, consistent errors |
| T-04 | Replay | reuse of a previously accepted TOTP or recovery code | atomic logging of the used step/code |
| T-05 | Tampering | Modification of a user, role, or audit entry | RBAC, transactions, append-only audit log, integrity checks |
| T-06 | Repudiation | A user or administrator denies performing an action | Audit entries including actor, time, operation type, and result |
| T-07 | Information Disclosure | Disclosure of a user’s existence or internal configuration | Consistent error messages, minimal responses, HTTPS, prohibition of secrets in logs |
| T-08 | Denial of service | A large number of requests blocks access or the API | rate limiting, size limits, timeouts, observability |

## 3. Functional Requirements

| ID | Priority | Requirement | Acceptance Criteria | Threats/Implications |
|---|---|---|---|---|
| FR-001 | MUST | The system must allow login using a unique username and password. | The integration test creates a user, verifies a successful login with the correct credentials, and verifies a failed login with an incorrect password. | T-01, T-03 |
| FR-002 | MUST | Passwords must be stored only as Argon2id hashes with a unique salt; the original password must not be recoverable via the API or through an audit. | A database schema check finds no plaintext; two identical test passwords have different hashes; the API and logs do not contain the original value. | T-02 |
| FR-003 | MUST | The user must be able to set up TOTP using a secret and a QR code in a format compatible with RFC 6238. | A test authenticator reads the QR/otpauth URI and successfully passes the RFC 6238 test vectors. | T-01 |
| FR-004 | MUST | The system must validate TOTP with a limited time desynchronization window. | Only allowed time steps are accepted for the configured window; codes outside the window are rejected; the window value is displayed in the configuration. | T-01, T-04 |
| FR-005 | MUST | The same TOTP code/time step for a user must be accepted no more than once. | Two concurrent requests with the same code result in no more than one successful response; a repeat request is rejected; validation is performed on a transactional basis. | T-04 |
| FR-006 | MUST | The system must support backup recovery codes. | During enrollment, a specified number of codes are generated; each code is accepted once, marked as used after use, and subsequently rejected. | T-01, T-02, T-04 |
| FR-007 | SHOULD | The user must be able to end active sessions and change their password after successful authentication. | The integration test terminates the session, after which the secure endpoint returns a failure; changing the password invalidates the previous password credential. | T-01, T-02 |
| FR-008 | SHOULD | An administrator must be able to create, lock, unlock, and deactivate a user. | For each operation, there is a success test and a permission-based failure test; a deactivated user cannot log in even with valid credentials. | T-05, T-06 |
| FR-009 | MUST | The administrator must be able to reset a user’s TOTP device, with mandatory confirmation of the operation. | After the reset, the previous TOTP secret is no longer accepted; re-enrollment is possible, and the operation is logged in the audit trail with the actor and target. | T-02, T-05, T-06 |
| FR-010 | MUST | The system must support at least the `user` and `admin` roles. | A user with the `user` role receives an HTTP 403 response on admin endpoints; `admin` users gain access only after authentication. | T-05 |
| FR-011 | SHOULD | The system must provide an administrative and user web interface or equivalent API scenarios as described in the documentation. | The acceptance test scenario performs enrollment, login, recovery, and reset without manually modifying the database. | FR-003, FR-006, FR-009 |
| FR-012 | SHOULD | The user must be able to view the 2FA status, the enrollment date, and whether there are any active recovery codes, without revealing their values. | The API/UI displays the status and the number of available codes, but does not return the TOTP secret or previously generated codes. | T-02, T-07 |

## 4. API and Integration Requirements

| ID | Priority | Requirement | Acceptance Criteria | Threats/Relationships |
|---|---|---|---|---|
| API-001 | MUST | The system must provide a REST API for user management, enrollment, login, 2FA verification, and administration. | The OpenAPI document contains endpoints for each MVP scenario; contract tests verify response codes and JSON schemas. | FR-001…FR-011 |
| API-002 | MUST | Secured REST endpoints must require a valid session or service credential with minimal permissions. | Requests without credentials or with credentials for a different role are rejected; credentials do not appear in the response body or in logs. | T-01, T-05, T-07 |
| API-003 | MUST | The forward-auth endpoint must return a positive result only after fully verifying the first and second factors. | The secure proxy scenario allows only valid sessions to pass through; passwords without TOTP and expired sessions are rejected. | T-01 |
| API-004 | MUST | Forward-auth must pass only normalized identity headers to the upstream, and any incoming spoofed headers must be removed or overwritten. | The integration test sends a spoofed header and verifies that the upstream receives only the identity set by the service. | T-05, T-07 |
| API-005 | MUST | The API must use a uniform error format without distinguishing between “username does not exist” and “incorrect password.” | For an unknown username and an incorrect password, the same external status/message is returned; details are retained only in the secure audit log. | T-03, T-07 |
| API-006 | SHOULD | The system should support versioning of the public API. | The version is included in the URL or header, and an incompatible request returns a documented error. | Operation |
| API-007 | COULD | The system may implement a local OIDC provider. | A separate set of OIDC contract tests runs without an external IdP; the absence of this feature does not block the MVP. | Extension |

## 5. Security Requirements

| ID | Priority | Requirement | Acceptance Criteria | Threats/Correlations |
|---|---|---|---|---|
| SEC-001 | MUST | All external HTTP connections must use HTTPS with a certificate from an internal CA; production HTTP must be disabled or redirected in a secure manner. | The security test rejects plaintext production endpoints; TLS connections are validated using a certificate from a test internal CA. | T-02, T-07 |
| SEC-002 | MUST | TOTP secrets must be stored encrypted using AES-256-GCM; the encryption key must be stored outside the database. | When reading the database, no plaintext secret is present; decryption without the external key is impossible; modifying the ciphertext causes an authentication error. | T-02, T-05 |
| SEC-003 | MUST | Recovery codes must be stored as one-way hashes and must not be revealed again after creation. | A database audit does not find any plaintext codes; the API only returns the value in the enrollment response; a repeat request does not return it. | T-02, T-04 |
| SEC-004 | MUST | Login attempts, TOTP, and recovery attempts must be limited per user and per request source; a temporary lockout must be applied once the threshold is exceeded. | A load security test exceeds the configured threshold and triggers a lockout; after the timeout, a valid login is possible again. | T-03, T-08 |
| SEC-005 | MUST | Sessions must have a random identifier, an expiration time, and logout/revocation and secure cookie flags (`Secure`, `HttpOnly`, an appropriate `SameSite` value). | An automated browser/API test verifies entropy/uniqueness, expiration time, revocation, and all cookie flags in HTTPS mode. | T-01, T-02 |
| SEC-006 | MUST | All administrative operations and significant authentication events must be recorded in an append-only audit log. | For login success/failure, enrollment, reset, role change, and lockout, entries containing the event type, actor/target, timestamp, result, and request ID were found; a regular user cannot modify or delete an entry. | T-05, T-06 |
| SEC-007 | MUST | Passwords, TOTP secrets, recovery codes, session tokens, and encryption keys must not be logged in audit or application logs. | Negative tests using unique canary secrets scan all test logs and find no matching values. | T-02, T-07 |
| SEC-008 | MUST | Administrative endpoints must implement RBAC and the principle of least privilege. | A matrix test checks all roles and endpoints; any access outside the permitted matrix returns a 403/404 response without disclosing data. | T-05 |
| SEC-009 | MUST | Input data must undergo server-side validation, and the body size, username, password, and code must be subject to limits. | Fuzz/property tests do not return a 500 error or a stack trace when invalid data is provided; exceeding the limit results in a 413 or 422 error. | T-07, T-08 |
| SEC-010 | SHOULD | The system should use CSRF protection for cookie-based, state-changing browser requests or stateless, header-based authentication with a documented justification. | A security test using a spoofed origin without a CSRF token fails; API-only endpoints require explicit credentials. | T-01, T-05 |
| SEC-011 | SHOULD | The rotation of the external encryption key must be documented and supported by an administrative procedure that does not disclose secrets. | On the test database, re-encryption is performed using both the old and new keys; after rotation, TOTP continues to function, and the old key is not used. | T-02 |
| SEC-012 | SHOULD | The service should provide health/readiness checks without disclosing configuration or secrets. | The public health response contains only the status; the readiness response indicates that the database or key is unavailable, and no confidential values are included. | T-07, T-08 |

## 6. Non-Functional Requirements

| ID | Priority | Requirement | Acceptance Criteria | Threats/Relationships |
|---|---|---|---|---|
| NFR-001 | MUST | The service must run on an isolated network without DNS/HTTP requests to external domains. | Installation, launch, and login are performed on a test network with blocked egress; the network capture contains no external requests. | T-08, autonomy |
| NFR-002 | MUST | All runtime dependencies must be included in the Docker image or offline package bundle with fixed versions. | On a machine without access to a package registry, the service is built and run from a pre-prepared bundle; the checksum of the dependencies matches the manifest. | Self-containment |
| NFR-003 | MUST | The server time must be synchronized with the internal NTP or must be monitored to detect desynchronization. | If the clock deviates beyond the configured threshold, readiness/diagnostic reports an error; external NTP is not used. | FR-004, T-01 |
| NFR-004 | MUST | Configuration must be provided via environment variables or a template file with placeholder values; secrets must not be stored in the repository. | Secret scanning does not detect credentials; `.env.example` is run after being populated; actual `.env` files, keys, and certificates are ignored by Git. | T-02 |
| NFR-005 | SHOULD | The service must provide backup and restore capabilities for user data and audit records. | On the test environment, a backup is created; the original volume is deleted; and the restore operation restores users, settings, and audit records without plaintext secrets. | T-05, T-06 |
| NFR-006 | MUST | The MVP deployment must be reproducible by a single documented command or sequence of commands without relying on an external service. | A clean test VM/LAN starts the service according to the instructions; after startup, a login smoke test and a health check are performed. | Autonomy |
| NFR-007 | SHOULD | All public endpoints must be documented with their purpose, authentication scheme, request/response schema, and error messages. | OpenAPI and the documentation contain 100% of MVP endpoints; a contract review finds no undocumented public routes. | Operations |
| NFR-008 | SHOULD | The system should shut down and restart correctly without losing any committed transactions. | The kill/restart test does not lose any created users, recovery codes, or audit events; any uncommitted transactions are rolled back. | T-04, T-05 |
| NFR-009 | SHOULD | The web interface should be suitable for keyboard navigation and display clear error messages without revealing security details. | The manual checklist verifies keyboard-only enrollment, login, and recovery; the UI does not display stack traces, secrets, or whether a user exists. | T-07, usable security |
| NFR-010 | COULD | The project may support PostgreSQL for multi-instance deployment. | A separate integration profile runs on PostgreSQL and undergoes migrations and basic auth tests; the absence of this profile does not block the SQLite MVP. | Scalability |

## 7. Performance and Capacity

The thresholds listed below are acceptance criteria for a single MVP instance on a test machine with at least 2 vCPUs, 2 GB of RAM, and a local SSD. The report must specify the machine’s specifications, the size of the dataset, and the workload profile. Server response time is measured without client-side network latency.

| ID | Priority | Requirement | Acceptance Criteria |
|---|---|---|---|
| PERF-001 | SHOULD | A typical successful login (password + TOTP) should be processed with a p95 of no more than 500 ms at a target load of 20 requests per second. | Load test lasting at least 10 minutes: p95 ≤ 500 ms, error rate ≤ 1%, at 20 RPS. |
| PERF-002 | SHOULD | A forward-auth check of an active session should have a p95 of no more than 100 ms at 50 requests per second. | Load test lasting at least 10 minutes: p95 ≤ 100 ms, error rate ≤ 1%, at 50 RPS. |
| PERF-003 | SHOULD | Rate limiting should ensure that legitimate requests remain accessible during a concurrent brute-force attack. | The test sends at least 100 RPS of invalid credentials and 1 valid request; the valid request experiences no more than 5 seconds of additional delay, and the service does not run out of memory. |
| PERF-004 | SHOULD | The service startup time in an offline environment must not exceed 30 seconds after the container is launched, provided the database and key are functioning properly. | Five consecutive launches: each readiness state must become “healthy” within ≤ 30 seconds. |
| PERF-005 | SHOULD | Memory consumption by an idle instance must not exceed 512 MB, excluding the reverse proxy and the database. | Measurement taken after 5 minutes of idle time and after a smoke test; RSS ≤ 512 MB without OOM. |

## 8. Traceability Matrix

| Proposal Goal/Objective | Requirements |
|---|---|
| Username, Password, and TOTP | FR-001–FR-005, SEC-002, PERF-001 |
| QR enrollment, recovery, and reset | FR-003, FR-006, FR-009, SEC-003 |
| Protection of Secrets and Brute-Force Attacks | FR-002, SEC-002–SEC-005, SEC-007, SEC-009 |
| RBAC and Immutable Auditing | FR-008–FR-010, SEC-006, SEC-008, NFR-005 |
| REST and forward-auth | API-001–API-005, NFR-007, PERF-002 |
| Isolated Deployment | SEC-001, NFR-001–NFR-006, PERF-004 |
| Testing and Performance | NFR-008–NFR-010, PERF-001–PERF-005 |
| Future OIDC/WebAuthn/LDAP | API-007, NFR-010, and ADR-003/ADR-004 |

## 9. MVP Acceptance Criteria

An MVP is considered to meet the specifications if:

1. All requirements with a **MUST** priority have an implemented test or a reproducible acceptance test scenario;
2. The functional scenarios for enrollment, login, TOTP renewal, recovery, lockout, admin reset, and forward authentication are run on an isolated test network;
3. Security tests do not reveal plaintext credentials, RBAC bypasses, replay of accepted codes, leaks of secrets in logs, or acceptance of spoofed identity headers;
4. The mandatory requirements for standalone deployment are met without an external Internet connection;
5. The **SHOULD** and **COULD** requirements, which have been implemented by the time of the demonstration, are additionally verified against their acceptance criteria;
6. Test results, the test bench configuration, and known deviations are documented in a separate test report.
