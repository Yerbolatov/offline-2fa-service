# Project Proposal

## 1. Project title

Development of a web service with local two-factor authentication for isolated corporate networks

**Repository:** [offline-2fa-service](https://github.com/Yerbolatov/offline-2fa-service)

## 2. Team and leader

| Role | Name | Division / Group |
|---|---|---|
| Student, author and developer | Yerbolatov Ali | SE-2401 |
| Supervisor | Myrzakhmetov Almas | School of Software Engineering |

## 3. Problem statement

Most two-factor authentication solutions (SMS codes, push notifications, cloud services from Google, Microsoft, and Duo) require an internet connection. Isolated corporate networks—such as those of government agencies, banks, industrial facilities, and defense and research organizations—are intentionally disconnected from the internet, so such tools cannot be used on them. As a result, organizations often rely solely on passwords, which are vulnerable to phishing, password reuse, and credential theft by insiders.

The problem is the lack of a simple self-deployable second authentication factor that works entirely within a closed network, is independent of external services, supports auditing, and integrates seamlessly with internal applications.

### Clarification of terms

- **Isolated network** — a network without Internet access, but with an internal local infrastructure: an internal network (LAN), an internal NTP server for time synchronization and an internal certificate authority (CA) for TLS certificates.

### Targeted users

- **System administrators and information security specialists** of isolated networks: deploy and manage the service.
- **Employees of such organizations**: use the second factor to access systems.
- **Internal application developers**: integrate the service via API or standard protocol.

## 4. Subject

Design and implement a self-developed web service that provides two-factor authentication completely within an isolated corporate network, without Internet access and without third-party cloud services.

## 5. Tasks

1. Analyze the existing methods of two-factor authentication (TOTP, HOTP, FIDO2/WebAuthn, hardware tokens) and solutions (Keycloak, privacyIDEA, FreeIPA, Duo), and assess their suitability for standalone environments.
2. Build a threat model for the isolated network (STRIDE) and define security requirements using OWASP ASVS as a benchmark.
3. Design the architecture: server part, database, API, user and administrator interfaces.
4. Implement authentication of the first factor (login and password, Argon2id hashing) and the second factor based on TOTP (RFC 6238).
5. Implement 2FA enrollment (QR code, secret transfer), backup recovery codes, and device reset by administrator.
6. Implement secure storage of TOTP secrets (AES-256-GCM encryption, the key stored separately from the database).
7. Implement protection against brute force (rate limiting, blocking) and against reuse of the code (a TOTP code is accepted only once), as well as tolerance for clock discrepancies.
8. Implement an administrator panel with a role model (RBAC) and an append-only audit log.
9. Provide an integration mechanism for internal applications: REST API and forward-auth for reverse proxies (for example, Nginx). OIDC support is considered as an additional option.
10. Prepare the system for offline deployment (Docker images or an offline installer, internal TLS certification center, absence of external CDNs and fonts).
11. Test the system (unit, integration, and security tests) and evaluate the performance.
12. Prepare documentation: administrator manual, user manual, and API description.

## 6. Expected results

- A running source code service deployed by a single team in an isolated environment.
- Threat model and security analysis document.
- API specification (OpenAPI) and demonstration integration with an example of an internal application.
- A set of tests and a test report, including security tests.
- Deployment documentation and the thesis text.

## 7. Project boundaries (scope)

### Included in the project (MVP)

- Login and password plus TOTP, backup codes.
- User and administrator interfaces, RBAC, audit log.
- Limiting request frequency and encrypting secrets.
- Integration via REST API and forward-auth.
- Offline deployment with internal CA and HTTPS.
- Tests and documentation.

### Secondary objectives

- FIDO2/WebAuthn hardware keys (work offline, but require HTTPS and correct origin).
- Integration with Active Directory.
- RADIUS server for VPN and network equipment.
- OIDC support.
- Export logs to syslog or SIEM.

### Not part of the project

- Cloud channels, SMS, e-mail, and push notifications.
- Biometric data.
- Mobile app (users use existing authentication apps such as FreeOTP, Aegis, or Google Authenticator that work offline).
- Certification according to national standards and regulatory requirements.
