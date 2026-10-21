# ADR-03: REST API and Forward-Auth as Primary Integrations

- **Status:** Accepted
- **Date:** 2026-10-17

## Context

Internal applications must connect to the MFA service without an external cloud. A REST API and a forward-auth endpoint for a reverse proxy are required; OIDC is considered an optional capability. Two scenarios must be covered: programmatic management/verification and protection of existing web applications through Nginx or a similar proxy.

## Decision

Implement in the MVP:

1. **REST API** for user management, TOTP enrollment, authentication checks, recovery codes, and administrative operations;
2. a **forward-auth endpoint** for the reverse proxy that returns only the verification result and safe identity headers after successful authentication.

Design the API with TLS, rate limiting, RBAC, consistent errors, TOTP replay protection, and strict trust rules for forwarded headers. Do not make OIDC mandatory in the first version, but do not block adding a local OIDC provider.

## Consequences

Benefits:

- REST fits internal applications and administrative automation;
- forward-auth avoids embedding MFA code in every web application;
- both interfaces can operate entirely inside the LAN;
- the MVP surface is smaller than that of a full federation/SSO provider.

Limitations:

- REST clients must manage sessions and errors correctly;
- forward-auth is secure only when proxy headers are properly cleared and validated;
- without OIDC there is no single standard token/claims flow for all clients;
- OIDC will be needed if the project moves to federation and many independent applications.

## Alternatives

- **OIDC:** standard for SSO and identity claims, but it requires an issuer, discovery, key rotation, client registration, redirect URIs, and a more complex trust model.
- **REST only:** simpler, but it does not cover the reverse-proxy scenario without a special client in every application.
- **Forward-auth only:** convenient for web applications, but insufficient for enrollment, administration, and non-HTTP client integration.
