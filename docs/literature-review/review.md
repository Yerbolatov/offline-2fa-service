# Critical Literature Review

## 1. Scope of the review

The project considers a two-factor authentication service for an isolated corporate network. Unlike a cloud MFA platform, it must operate inside the LAN, rely on internal NTP and CA infrastructure, avoid sending secrets to external providers, and support REST and forward-auth integration. The literature is therefore assessed not only for cryptographic strength, but also for operational autonomy, recovery procedures, auditing, and deployment complexity.

## 2. Main approaches

### 2.1. OTP: HOTP and TOTP

HOTP generates a one-time password from a shared secret and a counter; RFC 4226 defines the HMAC algorithm, code format, and counter desynchronization rules ([R09](../references.md#r09)). TOTP follows the same principle but replaces the counter with a time interval ([R10](../references.md#r10)). This fits a standalone network well: the code is generated on the user's device, and the server does not need an SMS gateway, push provider, or external API.

The limitation of OTP is that the server and client share a long-term secret. If this secret is compromised, valid codes can be generated, and TOTP itself does not prevent code theft via phishing or malware. In practice, it is also necessary to account for time desynchronization, code reuse, and brute-force attacks. Therefore, the project is not limited to implementing the RFC formula: the secret is stored in encrypted form, the time window is limited, verification is subject to a rate limit, and the adopted time step is set atomically. These latest measures are design requirements derived from the NIST requirements for throttling and authenticator management ([R12](../references.md#r12)) and the OWASP ASVS testing requirements ([R13](../references.md#r13)).

### 2.2. FIDO2 and WebAuthn

WebAuthn uses public-key cryptography: the server stores the public key, and the authenticator signs a challenge bound to the relying party origin. This avoids storing a shared OTP secret on the server and is more resistant to phishing. This is defined in the WebAuthn Level 2 specification ([R14](../references.md#r14)), while formal analyses of FIDO2, CTAP 2.1, and WebAuthn 2 show that protocol security properties can be verified cryptographically ([R02](../references.md#r02)).

However, WebAuthn requires HTTPS, a valid RP ID/origin, a compatible browser, and an authenticator. For a private network, this means setting up an internal CA in advance and ensuring device support; restoring access in the event of key loss is more complicated than issuing a backup code. Research on Remote WebAuthn also shows that using FIDO2 on less accessible devices requires additional trusted intermediary architecture ([R03](../references.md#r03)). Therefore, WebAuthn is intended as an extension, while TOTP remains the MVP mechanism.

### 2.3. Hardware tokens and push/cloud MFA

Hardware OTP tokens reduce reliance on smartphones and can operate without an internet connection, but they require the procurement, issuance, replacement, and secure lifecycle management of the devices. WebAuthn-based security keys provide stronger protection against phishing, but present a similar challenge regarding device recovery and management. Research on phishing-resistant network authentication confirms the viability of security keys, but focuses on a specialized captive portal scenario rather than a universal local service ([R06](../references.md#r06)).

Push-MFA and cloud platforms simplify centralized management but conflict with the requirement for a fully isolated network: an external channel is needed for push delivery, license verification, or a cloud gateway. Therefore, the project eliminates external dependencies from the main scenario.

### 2.4. Federated authentication

OAuth 2.0 separates authorization from client access to resources, while OpenID Connect adds a layer of identity and claims ([R11](../references.md#r11)). Federation is convenient when there are already multiple applications and a single Identity Provider, but it increases the number of keys, endpoints, trusted clients, and operational procedures. For the MVP, the project is opting for local REST and forward-auth; OIDC remains a compatible path for future expansion.

## 3. What we know about threats and best practices

Scientific studies show that MFA cannot be evaluated solely by the presence of a second factor. A review of MFA systems highlights the cost of complexity and infrastructure overhead in distributed and blockchain-oriented approaches ([R01](../references.md#r01)). A study of OTP protocols in a microservice architecture shows possible component separation, yet also demonstrates that blockchain dependence is unnecessary for small autonomous deployments ([R05](../references.md#r05)).

For industrial environments, the workflow matters: usable security includes understandable user actions, administration, and authorization, not just mathematical strength ([R08](../references.md#r08)). In this project, that is reflected in separate 2FA enrollment flows, backup codes, administrator reset procedures, role-based access control, and audit logging.

The practical requirements for implementation are formalized in NIST SP 800-63B-4: authenticators must have a controlled lifecycle, brute-force protection must be limited, and recovery must not circumvent the trust model ([R12](../references.md#r12)). OWASP ASVS translates these principles into verifiable requirements for authentication, sessions, cryptography, APIs, and access control ([R13](../references.md#r13)). Therefore, the project supports Argon2id for passwords, AES-256-GCM for TOTP secrets, separate storage of encryption keys, secure sessions, rate limiting, RBAC, and immutable auditing.
WebAuthn analysis also reveals that secure integration depends on checking protocol formats and parameters, not merely on successful validation in the happy path ([R04](../references.md#r04)). This affects the future integration tests: challenge generation, origin and RP ID validation, authenticator counters, and error handling must be tested, not just the success case.

## 4. Critical evaluation and limitations

1. **TOTP is practical, but not phishing-resistant.** It is better than a password alone and requires no external service, but the shared secret and code entry remain vulnerable to compromise and live phishing. The project mitigates this by limiting code acceptance time, applying throttling and lockout, and planning WebAuthn as an additional factor.
2. **WebAuthn is stronger against phishing, but harder for offline operation.** It requires HTTPS, a valid origin, a compatible authenticator, and a recovery procedure. Therefore, it does not replace TOTP in the baseline deployment.
3. **Centralized IdPs reduce duplication but increase the attack surface.** Keycloak, FreeIPA, and similar platforms are useful in larger infrastructures, but their realm, client, LDAP, and key-management models are excessive for a single local service. A custom solution should be simple and self-contained rather than a partial reimplementation of a full IdP.
4. **Cloud solutions do not match the project threat model well.** They provide mature management and reporting, but they violate the requirement for no external connections and may create an implicit vendor dependency.
5. **Complexity and usability are part of security.** If deployment is too complicated, it increases the risk of incorrect TLS settings, wrong time synchronization, weak secret handling, and administrative bypass of MFA. The MVP therefore stays limited to a single local service, internal CA/NTP, a minimal API, and explicit diagnostic errors.

## 5. Requirements adopted by the project

- RFC 6238-compliant TOTP implementation and RFC test vectors;
- internal NTP and a documented valid time window;
- encryption of TOTP secrets with a key stored outside the database;
- atomic prevention of OTP reuse within a time step;
- rate limiting, blocking, and uniform error messages;
- backup codes with one-time use and administrator reset;
- Argon2id password hashing, HTTPS, and internal CA;
- RBAC and audit logging;
- REST and forward-auth integration as the MVP, with OIDC and WebAuthn as extension paths;
- security baselines aligned with OWASP ASVS and NIST SP 800-63B.

In summary, the project does not aim to reproduce a full identity platform. It focuses on a TOTP-first local service for LAN environments, with WebAuthn and OIDC treated as future extensions when the operating context and deployment model justify them.
