# offline-2fa-service

Self-deployed web service of local two-factor authentication for isolated (air-gapped) corporate networks.
Student's diploma project from group SE-2401. Yerbolatov Ali.

## Problem

Cloud-based MFA services, SMS, and push notifications require external communication channels.
They can not be used in closed networks of state, banking,
industrial, defense and research organizations. Therefore, such
organizations may be forced to rely only on a password that
is prone to phishing, reuse, and credential theft.

The project solves this problem with the help of a local service that works
inside the corporate network and does not depend on the Internet or third-party cloud
providers. It is assumed that there is an internal NTP server and an internal
certification Authority (CA).

## Targeted Users

- **System administrators and information security specialists** —
deploy the service, manage users and control the audit.
- **Employees of the organization** — are authenticated using
password and the second factor.
- **Internal application developers** — connect applications via REST
API or endpoint forward-auth.

## Objective and expected outcome

The purpose of the project is to design and implement the independently
deployable two-factor authentication service suitable for
completely isolated corporate network.

Expected Result:

- a working local service with password authentication and TOTP;
- TOTP connection via QR code and backup recovery codes;
- administrative panel with RBAC and audit log;
- REST API and integration with reverse proxy via forward-auth;
- safe keeping of secrets and protection against brute force and reuse of codes;
- deployment with HTTPS and internal CA without external CDNs and services;
- automatic tests, API specification and operation documentation.

## Project scope:

### Included in MVP

The service provides local authentication by login, password and TOTP without
access to the Internet, SMS, push notifications and cloud MFA providers.
Intended closed network infrastructure includes internal NTP and CA.

- username and password authentication
- TOTP connection via QR code;
- verification of a one-time code and protection against reuse;
- backup recovery codes;
- user and administrator roles;
- audit log;
- REST API and endpoint forward-auth;
- HTTPS with certificates of the internal certification center;
- autonomous deployment without external CDNs and services.

## More Destinations

WebAuthn/FIDO2, OIDC, LDAP/Active Directory, RADIUS and log export to SIEM
are considered as areas for further development and are not
mandatory parts of the current MVP.

The project does not include SMS, e-mail and push notifications through external services,
biometrics and a separate mobile application.

## Current status

At this stage, an analysis of the subject area and project documentation have been prepared:

- proposal and project boundaries;
- literature review and analysis of sources;
- comparison of existing systems and technologies;
- specification of requirements with acceptance criteria;
- architectural diagrams and ADR;
- an interactive HTML prototype of user and administrative scenarios.
- verification and evaluation plan
- project management plan and risks

The implementation of the server part and automated tests are on the following
stages of development. Instructions for installation, start-up and testing will be
added to this README simultaneously with the appearance of the application being launched.

## Selected technologies

- **Backend:** Python and FastAPI
- **MVP storage:** SQLite
- **Integration:** REST API and forward-auth for reverse proxy;
- **Second MVP factor:** TOTP according to RFC 6238;
- **Password hashing:** Argon2id;
- **Encryption of TOTP secrets:** AES-256-GCM with a separate key;
- **Deployment:** stand-alone installation in an isolated network;
- **Diagrams:** draw.io.

## YouTrack & Documentation

- [Project in YouTrack](https://offline2fa.youtrack.cloud/)

## Documentation

- [Project Proposal and Project Scope](docs/proposal.md)
- [Requirements specification](docs/requirements/requirements.md)
- [Verification and Evaluation Plan](docs/verification/plan.md)
- [Project Management Plan and Risks](docs/management/plan-and-risks.md)
- [Bibliography](docs/references.md)
- [Source analysis table](docs/literature-review/sources-table.md)
- [Critical literature review](docs/literature-review/review.md)
- [Comparison of existing systems](docs/literature-review/systems-comparisons.md)
- [Technology Comparison](docs/literature-review/technologies.md)
- [Architecture and diagrams](docs/architecture/architecture.md)
- [Architectural Solutions (ADR)](docs/architecture/adr/)
- [Open HTML prototype in browser](https://yerbolatov.github.io/offline-2fa-service/prototype/serviceprototype.html)
- [Prototype source file](docs/prototype/serviceprototype.html)

## Repository Structure

```text
config/       configuration examples
diagrams/     architecture diagram sources and exports
docs/         project and technical documentation
src/          backend and frontend source code
tests/        automated tests
```

## Installation, launch and testing

The running version of the app has not been published yet. After adding
source code in the repository here will be provided:

1. requirements for the environment;
2. installing dependencies from local or preloaded packages;
3. Configuration setting according to the example file;
4. launch of the service;
5. launch of unit and integration tests.

## Team

| Role | Participant | Responsibility |
|---|---|---|
| Student, author and developer | Yerbolatov Ali, SE-2401 | Analysis, design, implementation and testing |
| Supervisor | Almas Myrzakhmetov, School of Software Engineering | Supervisor |
