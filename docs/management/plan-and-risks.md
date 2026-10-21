# Project Management Plan and Risk Register

## 1. Purpose

This document describes the development phases of the local two-factor authentication service, as well as the allocation of responsibilities, dependencies, and project risks.
This plan is linked to the project tasks in [YouTrack](https://offline2fa.youtrack.cloud/issues/DIP) and to the requirements in [DIP-26](https://offline2fa.youtrack.cloud/issue/DIP-26).

## 2. Responsibilities

| Participant | Responsibility | Result |
| ---|---|---|
| Student — Ali Yerbolatov | Domain analysis, design, development, testing, documentation, Git and YouTrack management | Source code, documentation, tests, demo, and report |
| Supervisor — Almas Myrzakhmetov | Review of key decisions, feedback on documentation and prototype, monitoring readiness for project phases | Comments and confirmation of significant results |

The student is responsible for completing tasks and updating statuses. The supervisor reviews key results at checkpoints but does not replace the daily work on the board.

## 3. Phases and Checkpoints

The dates will be added after consultation with the supervisor and
the official announcement of the defense dates. Until then, the stages are completed
sequentially, and the transition to the next stage occurs after the
completion criteria for the previous stage have been met.

| Phase | Procedure | Content | Result and Completion Criteria |
|---| ---:|---|---|
| Analytical and Design Preparation | 1 | Complete the README, proposal, literature review, comparisons, requirements, diagrams, prototype, and verification plan | Materials are available in the repository, and tasks include links to the results |
| Preparing the MVP Framework | 2 | Create the FastAPI structure, configuration, health check, SQLite schema, and offline launch profile | The service launches locally, the configuration uses placeholders, and the first smoke test passes |
| First factor and TOTP enrollment | 3 | Implement users, Argon2id, sessions, QR/otpauth enrollment, and TOTP verification | Tests FR-001–FR-004 and the basic login scenario pass |
| Recovery, replay, and protection against brute-force attacks | 4 | Add recovery codes, one-time code usage, lockout/rate limiting, and secure storage of secrets | Pass FR-005–FR-006, SEC-002–SEC-005, and negative security tests |
| Administration and Integration | 5 | Implement RBAC, device reset, audit log, REST API, and forward authentication | Passes FR-008–FR-010, API-001–API-005, and administrator scenarios |
| Offline Deployment | 6 | Prepare Docker/offline bundle, internal CA, NTP, backup, and instructions | Deployment in a closed test environment occurs without external requests; NFR-001–NFR-006 are performed |
| Comprehensive Testing and Evaluation | 7 | Perform unit, integration, E2E, security, and load tests; compare with the password-only baseline | A report is completed with p95, error rate, replay/lockout/drift results, and known limitations |
| Preparing the Final Version | 8 | Fix critical defects, update documentation, and prepare a demonstration | All mandatory requirements have a “PASS” status or a justified exception; the release candidate version is ready |
| Final Presentation | 9 | Present the solution and defend the results | The repository, change log, test report, and use case have been presented |

## 5. Risk Register

Scale: Probability and impact are rated as low, medium, or high.
Risk priority is determined by a combination of these ratings.

| ID | Risk | Probability | Impact | Mitigation Measures | Response Plan |
|---|---|---|---|---|---|
| R-01 | Offline deployment requires packages, images, or certificates that are not available on the private network | High | High | Lock down versions; prepare an offline bundle, checksum, and test environment without egress in advance | Simplify the profile to a single instance and SQLite; migrate optional integrations |
| R-02 | Time error or an unavailable internal NTP server causes TOTP to fail | Medium | High | Add NTP diagnostics, document the drift window, use a master clock in tests | Block “readiness” with a clear error message and implement a test bench restriction |
| R-03 | Improper storage of TOTP secrets or encryption keys | Medium | Critical | AES-256-GCM, external key source, secret scanning, negative tests, and cryptography review | Halt the release, remove test secrets, fix the design, and repeat security tests |
| R-04 | Replay of TOTP or recovery code due to a race condition | Medium | High | Unique time step, atomic transaction, parallel integration test | Pause the login demonstration and fix the transaction boundary |
| R-05 | The MVP scope is becoming too broad due to OIDC, WebAuthn, LDAP, and RADIUS | High | High | Do not include extensions in the MVP; define the scope using MUST/SHOULD/COULD and ADR | Move the extension to the backlog; retain REST + forward-auth and TOTP |
| R-06 | Argon2id load or rate limiting increases input latency | Medium | Medium | Measure p95 on the target machine; configure cost and limits separately; do not compromise security without justification | Identify the trade-off, optimize the infrastructure, and repeat the PERF test |
| R-07 | Insufficient protection of administrative operations and audits | Medium | High | RBAC, append-only model, actor/target/request ID, security tests for 403 and tampering | Prohibit admin release until the issue is fixed and the log is verified |
| R-08 | Dependency on selected FastAPI, TOTP, cryptography, or SQLite libraries | Medium | Medium | Pin versions, use supported libraries, and isolate them with adapters | Replace the library only via ADR and rerun regression and security tests |
| R-09 | Insufficient time for testing and fixing defects | High | High | Freeze the MVP, implement MUST requirements first, and set aside a separate time slot for security and E2E testing | Reduce SHOULD/COULD requirements, retain mandatory scenarios, and clearly define limitations |
| R-10 | Data or audit loss due to SQLite failure or incorrect backup | Medium | High | Verify backup/restore, graceful shutdown, and recovery after a restart | Restore a test snapshot and document RPO/RTO for the MVP |
| R-11 | Changes to requirements made by a manager or committee at a late stage | Medium | Medium | Use checkpoints and document decisions in ADR/YouTrack | Assess the impact, update the backlog, and do not change the MVP without analyzing the timeline |
| R-12 | Internal CA or TLS configuration errors | Medium | High | Test CA, hostname/chain verification, trust setup instructions | Switch the test environment to a pre-validated certificate and document the procedure |

## 6. Dependencies

| Dependency | Purpose | Risk if unavailable | How it is managed |
| ---|---|---|---|
| Internal CA | HTTPS and trusted connections on a private network | Inability to properly validate TLS and browser/API scenarios | Test CA and certificates are stored outside of Git; the issuance procedure is described in the deployment guide |
| Internal NTP | Correct TOTP generation and verification | False failures or code acceptance outside the expected window | Time diagnostics, configured drift window, and test clocks |
| FastAPI and Python runtime | Backend and REST API | API changes or dependency incompatibilities | Fixed versions and offline package bundles |
| SQLite | Single-node MVP storage | Transaction loss or concurrency limitations | WAL/transactions, backup/restore, separate PostgreSQL as a future extension |
| TOTP library | Compatibility with RFC 6238 and otpauth | Code errors or QR code enrollment | RFC 6238 test vectors and integration tests |
| Cryptography Library | AES-256-GCM and Argon2id | Insecure or incompatible secret storage | Use a supported library; do not implement cryptography yourself |
| Reverse proxy | Forward-auth integration with internal applications | Unable to verify the application’s access scenario | Test Nginx configuration and API-003/API-004 contract tests |
| Offline registry/package storage | Installation without an internet connection | Non-reproducible deployment | Pre-build the bundle and verify the checksum on a clean machine |
| YouTrack and GitHub | Traceability of issues, reviews, and history | Cannot confirm progress or link the result to the issue | Include the issue ID and a link to the result in every PR/commit |

## 7. Next Steps After the First Milestone

1. Create separate tasks for the backend framework, database schema, TOTP enrollment, recovery/replay, RBAC/audit, forward authentication, offline deployment, and testing.
2. Prepare a minimal offline bundle and verify that it runs on a clean machine without external network access.
3. Implement the `MUST` requirements first, then move on to the `SHOULD` requirements.
4. Maintain a log of metrics and defects so that the source code, configuration, and commits are available by the evaluation phase.