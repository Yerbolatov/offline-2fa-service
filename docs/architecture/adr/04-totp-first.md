# ADR-04: TOTP as the MVP's Primary Second Factor

- **Status:** Accepted for the MVP
- **Date:** 2026-10-17

## Context

The project must operate in an isolated network with no SMS, push, external identity provider, or project mobile application. Users can use existing offline authenticator apps. TOTP, QR enrollment, recovery codes, clock-skew tolerance, and code-reuse prevention are required; WebAuthn is an additional goal.

TOTP and WebAuthn were considered. TOTP is standardized by RFC 6238 and does not require a persistent network connection. WebAuthn provides stronger phishing resistance, but requires HTTPS, a correct origin/RP ID, a compatible browser, and an authenticator.

## Decision

Choose **TOTP according to RFC 6238** as the MVP's primary second factor. Implement:

- enrollment through a QR code;
- encrypted AES-256-GCM secret storage with an external key;
- internal NTP and an explicitly configurable, limited time window;
- atomic storage of the used time step;
- rate limiting and brute-force lockout;
- one-time recovery codes and administrative reset;
- RFC 6238 vector tests and negative security scenarios.

Prepare WebAuthn as an extension after TOTP and HTTPS/origin configuration have stabilized.

## Consequences

Benefits:

- works without Internet access or an external provider;
- users do not need a new mobile application;
- enrollment, recovery, and operation in an isolated network are simpler;
- fits the MVP scope and the project's existing documentation well.

Limitations:

- TOTP uses a shared secret and is not fully phishing-resistant;
- clock drift and device loss require a recovery policy;
- recovery codes are sensitive credentials and must be stored as hashes/one-time records;
- WebAuthn remains an important direction for stronger phishing protection.

## Alternative

**WebAuthn/FIDO2** provides origin-bound public-key credentials and better phishing protection, but increases requirements for the internal CA, browser/device support, RP ID, and recovery. It is therefore selected as an additional factor rather than a mandatory dependency of the first version.
