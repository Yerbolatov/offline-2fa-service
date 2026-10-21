# Verification and Evaluation Plan

## 1. Purpose

This plan defines how the achievement of the project’s objective will be verified: the service must provide local two-factor authentication within an isolated corporate network without relying on the Internet or cloud providers.

The test is conducted in accordance with the requirements specified in the [requirements specification](../requirements/requirements.md) prepared for task [DIP-26](https://offline2fa.youtrack.cloud/issue/DIP-26).
Requirements with a priority of `MUST` are considered mandatory for MVP acceptance.
Requirements with a priority of `SHOULD` and `COULD` are verified if the corresponding feature has been implemented by the time of the demonstration.

## 2. Types of Testing

| Type of Test | Purpose | What Is Tested | Result |
|---|---|---|---|
| Modular Tests | Test individual functions without external dependencies | TOTP and Argon2id generation and verification, AES-256-GCM, recovery code hashing, rate limiting, RBAC, time handling | Automatic report with results and coverage of critical modules |
| Integration Tests | Verify Component Interaction | API, SQLite, sessions, auditing, encryption key, transactions, forward authentication | a set of API tests with response codes and database status checks |
| E2E Tests | Test user and administrative scenarios in their entirety | enrollment, two-factor authentication, recovery, device reset, logout, administrative operations | a reproducible scenario from the HTTP/UI request to the result |
| Security Tests | Test STRIDE threat mitigation measures | brute force, replay, RBAC bypass, spoofed headers, secret leaks, TLS, validation, and CSRF | report detailing the negative test scenarios run and defects found |
| Load Testing | Test latency, stability, and capacity | login, TOTP, forward authentication, rate limiting, service startup, and memory consumption | p50/p95/p99 metrics, RPS, error rate, and resource usage |

Each test must use only synthetic accounts, test keys, and sample data. Real passwords, personal data, production certificates, and keys must not be included in the test suite.

## 3. Test Environment and Measurement Conditions

### 3.1. Base Test Bench

The main test bench for MVP acceptance:

- one test virtual machine or physical computer;
- at least 2 vCPUs, 2 GB of RAM, and a local SSD;
- one service instance and SQLite;
- an internal test CA for HTTPS;
- an internal NTP server or a managed test clock;
- no external DNS and blocked outbound traffic;
- Fixed versions of the runtime and dependencies from the offline bundle.

Machine specifications, OS versions, Python, databases, rate-limiting configuration, dataset size, and load profile are recorded along with the results of each run.

### 3.2. Measurement Guidelines

1. Before the load test, the service is warmed up and its readiness is verified.
2. For each scenario, measurements are taken for at least 10 minutes after the warm-up.
3. Latency is measured on the client side and separately on the server side to avoid mixing processing time with network latency.
4. For latency, at least the p50, p95, p99, and error rate are published.
5. Each key test is repeated at least three times; each run and the final result are recorded in the report.
6. If the configuration or code is changed, the results are considered a new run.
7. Only a result that meets the requirement threshold is considered successful; an average value that masks individual violations is not.

## 4. Functional and Integration Test Scenarios

| Test ID | Scenario | Expected Result | Requirements |
|---|---|---|---|
| F-01 | Create a user and log in with the correct password | Login permitted; the password is not returned and is not logged | FR-001, FR-002 |
| F-02 | Enter an incorrect password for an existing but unknown username | Identical external failure without revealing the user’s existence | API-005, SEC-007 |
| F-03 | Set up TOTP via QR code or `otpauth` URI | The authenticator receives the data; a valid code passes verification | FR-003 |
| F-04 | Check codes within allowed and disallowed time windows | Only codes within the configured drift window are accepted | FR-004, NFR-003 |
| F-05 | Send a single TOTP code twice simultaneously | No more than one successful response; the second is rejected | FR-005 |
| F-06 | Create and use a recovery code | The code is accepted only once and then becomes invalid | FR-006, SEC-003 |
| F-07 | Perform a TOTP admin reset | The old secret is not accepted; the operation and target are logged in the audit | FR-009, SEC-006 |
| F-08 | Verify the `user` and `admin` roles | `user` receives a 403 error on the admin endpoint; `admin` gains access after logging in | FR-010, SEC-008 |
| F-09 | Verify session and logout/revocation | After logout or expiration, the secure endpoint rejects the request | SEC-005 |
| F-10 | Access a REST endpoint without credentials or with credentials for a different role | The request is rejected; secrets are not included in the response or log | API-001, API-002 |
| F-11 | Perform a reverse-proxy scenario for forward authentication | Only full password and TOTP authentication yield a positive result | API-003 |
| F-12 | Pass on fake identity headers | Upstream receives only normalized headers from the service | API-004 |

## 5. Security Tests

Security tests address the threats listed in the DIP-26 requirements:

| Threat | Test | Passing Criteria | Requirements |
|---|---|---|---|
| T-01 Spoofing | password without TOTP, expired session, fake identity | access denied | FR-003–FR-005, API-003, SEC-005 |
| T-02 Credential theft | inspection of databases, responses, and logs | no plaintext password, TOTP secret, recovery code, token, or encryption key | FR-002, SEC-002, SEC-003, SEC-007 |
| T-03 Brute force | series of incorrect password/TOTP/recovery requests | a temporary lockout is applied after the threshold is reached; the error message does not provide details | SEC-004, API-005 |
| T-04 Replay | replay of a previously accepted TOTP step or recovery code | repeat requests are rejected; a concurrent test allows for a maximum of one success | FR-005, FR-006 |
| T-05 Tampering | modification of role, user, ciphertext, and audit record | unauthorized modifications are rejected; corrupted ciphertext is not decrypted | SEC-002, SEC-006, SEC-008 |
| T-06 Repudiation | login, enrollment, reset, role change, and lockout | the event contains actor/target, timestamp, type, result, and request ID | SEC-006 |
| T-07 Information Disclosure | errors, health endpoint, forged headers, log scanning | no distinction between unknown login, stack trace, and secrets | API-004, API-005, SEC-007, SEC-009, SEC-012 |
| T-08 Denial of Service | high volume of requests and oversized input | the service remains available for legitimate requests; invalid sizes receive a 413/422 response | SEC-004, SEC-009, PERF-003 |

Additionally, HTTPS connections with an internal CA certificate are checked, as well as the absence of
external requests and protection via the `Secure`, `HttpOnly`, and appropriate
`SameSite` cookie flags (SEC-001, SEC-005, NFR-001).

## 6. Metrics and Acceptance Thresholds

| Metric | Measurement Method | Conditions | Target Result | Related Requirements |
|---|---|---|---|---|
| Full login latency | 20 RPS, password + TOTP, 10 minutes after warm-up | 2 vCPUs, 2 GB RAM, SQLite, local SSD | p95 ≤ 500 ms, error rate ≤ 1% | PERF-001 |
| Forward-auth latency | 50 RPS with active sessions, 10 minutes | same test setup, without external proxy latency | p95 ≤ 100 ms, error rate ≤ 1% | PERF-002 |
| Brute-Force Lockout | 100 RPS of invalid credentials with one valid request | fixed lockout and timeout thresholds specified in the configuration | malicious requests are blocked; a valid request incurs no more than a 5-second additional delay | SEC-004, PERF-003 |
| Rejection of Repeated TOTP | two consecutive and two parallel requests with the same code/step | one account, one active session | the first request succeeds, all repeats are rejected; maximum of one success | FR-005 |
| Rejection of duplicate recovery code | Two requests with the same recovery code | Code has been issued but not yet used | First request successful, second rejected | FR-006, SEC-003 |
| Permissible clock drift | Test clocks are set on either side of the current step | The window is defined by configuration, e.g., ±1 step | Codes within the window are accepted; those outside the window are rejected | FR-004, NFR-003 |
| External network traffic | packet capture during installation, startup, and login | blocked egress, internal NTP and CA | 0 DNS/HTTP requests to external domains | NFR-001 |
| Startup time | 5 consecutive container/service launches | healthy databases and external key | readiness status becomes “healthy” within 30 seconds at most during each launch | PERF-004 |
| Memory | RSS after 5 minutes of idle time and after the smoke test | without a reverse proxy and separate database logging | no more than 512 MB, no OOM | PERF-005 |

The actual time window value, lockout threshold, and lockout duration must be specified in the configuration and report. The example `±1` step is a test scenario, not a requirement to secretly change project settings.

## 7. Basic Solutions for Comparison

### 7. Password-Only Login

The first basic solution is the same service without two-factor authentication, using only a username and password. The following are compared using identical data:

- login success rate and login delay;
- number of successful logins in scenarios involving a stolen password;
- resistance to brute-force attacks;
- presence of replay protection and audit capabilities;
- network dependencies and ease of deployment.

The password-based baseline is expected to be simpler and may have less latency, but it will not provide protection against the use of a stolen password. The comparison is used to demonstrate the practical value of the second factor, not to argue that higher latency is an advantage in and of itself.

### 7.2. privacyIDEA (Additional Baseline)

If a privacyIDEA test installation is available, it is used as an external baseline for functional comparisons of TOTP and administrative management. The comparison is conducted on the same isolated network and using the same scenarios: enrollment, login, one-time code, lockout, audit, deployment time, and resources.

The results from privacyIDEA do not replace a review of your own service’s requirements: they show how the MVP compares to the existing solution. If privacyIDEA cannot be installed on an offline test bench, this is noted as a limitation, and password-only login remains the mandatory baseline.

## 8. How will the achievement of the project’s goal be demonstrated?

The goal is considered achieved if all of the following conditions are met simultaneously:

1. All `MUST` requirements from DIP-26 have an automated test or a reproducible acceptance test scenario.
2. Enrollment, full two-factor login, recovery, admin reset, and forward authentication are performed on an isolated network with no external egress.
3. Security tests do not reveal plaintext credentials, RBAC bypasses, reuse of accepted codes, leaks of secrets in logs, or spoofing of identity headers.
4. The metrics from Section 6 meet the established thresholds, or each deviation is explicitly explained in the report and is related to an unmet `SHOULD` priority.
5. The comparison with the password-based baseline shows that TOTP adds a verifiable second factor without relying on external services.
6. The test suite configuration, versions, test execution commands, raw results, errors, and known limitations are recorded in the test report.

For each test, the ID, date, commit, configuration, input data, expected result, actual result, and `PASS`/`FAIL` status are stored. Test data must be reproducible and must not contain any secrets.

## 9. Trace Matrix

| Verification Group | DIP-26 Requirements |
|---|---|
| Passwords, TOTP, recovery, and replay | FR-001–FR-006, FR-009, SEC-002–SEC-005 |
| Users, RBAC, and auditing | FR-008–FR-010, SEC-006, SEC-008 |
| REST API and forward authentication | API-001–API-005 |
| HTTPS, validation, and log protection | SEC-001, SEC-007, SEC-009, SEC-010 |
| Offline and reproducible deployment | NFR-001–NFR-004, NFR-006 |
| Backup, post-restart recovery, and health | NFR-005, NFR-008, SEC-012 |
| Login, forward authentication, and fault tolerance | PERF-001–PERF-005 |
| Future Features | API-006–API-007, SEC-011, SEC-012, NFR-007, NFR-009, NFR-010 |

## 10. Evaluation Limitations and Risks

- Performance results apply to the specified single-node test environment and do not demonstrate the characteristics of a multi-instance deployment.
- SQLite and a local SSD may produce different results on server hardware.
- The test internal CA and NTP simulate a closed network infrastructure but do not replace verification of an organization’s specific infrastructure.
- Verification of WebAuthn, OIDC, LDAP/AD, RADIUS, and SIEM is not included in the mandatory MVP assessment.
- The unavailability of privacyIDEA does not prevent the assessment of your own service if the password baseline and all mandatory scenarios have been completed.