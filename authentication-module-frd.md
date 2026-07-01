# Functional Requirements Document (FRD)
## Authentication Module — Corporate Banking Platform

**Document Version:** 1.0.0
**Classification:** Internal — Confidential
**Status:** Draft for Review
**Date:** 2026-06-30
**Parent Document:** Authentication Module PRD v1.0.0

---

# Table of Contents

1. Introduction
2. Scope & Boundaries
3. Actors & Roles
4. Functional Area 1: Login & Credential Verification
5. Functional Area 2: Multi-Factor Authentication (MFA)
6. Functional Area 3: Session Management
7. Functional Area 4: Device Management
8. Functional Area 5: Password Management
9. Functional Area 6: Token Management
10. Functional Area 7: Administrative Operations
11. Business Rules
12. Input Validation Rules
13. Error Catalog
14. Notification Triggers
15. Audit Requirements
16. Configuration Parameters

---

---

# 1. Introduction

## 1.1 Purpose

This Functional Requirements Document (FRD) provides the detailed, implementation-level specification of every function, behavior, rule, and constraint within the Authentication Module of the Corporate Banking Platform.

Where the PRD answers **what** the system must do and **why**, this FRD answers **how** the system must behave in every scenario, including normal flows, alternate flows, edge cases, and failure conditions.

This document is the primary reference for:
- Software Engineers implementing the module
- QA Engineers writing test cases
- Security Engineers performing threat modeling and code review
- Product Managers reviewing scope during sprint planning

## 1.2 Reference Documents

| Document | Version | Location |
|---|---|---|
| Authentication Module PRD | 1.0.0 | `docs/authentication-module-prd.md` |
| NIST SP 800-63B (Digital Identity Guidelines) | 3rd Edition | External |
| RFC 6749 (OAuth 2.0) | — | IETF |
| RFC 7519 (JWT) | — | IETF |
| RFC 7636 (PKCE) | — | IETF |
| RFC 7662 (Token Introspection) | — | IETF |
| RFC 7009 (Token Revocation) | — | IETF |
| W3C WebAuthn Level 2 | — | W3C |
| RFC 6238 (TOTP) | — | IETF |

## 1.3 Conventions Used

- **SHALL** — mandatory requirement. Must be implemented exactly as stated.
- **SHOULD** — recommended. Strong preference; deviation requires documented justification.
- **MAY** — optional. Permitted but not required.
- `FR-XXX-NNN` — Functional Requirement identifier (area code + sequence number).
- `BR-NNN` — Business Rule identifier.
- `ERR-NNN` — Error code identifier.

---

---

# 2. Scope & Boundaries

## 2.1 In Scope

The following functional areas are fully specified in this document:

| Functional Area | Description |
|---|---|
| Login & Credential Verification | All login flows including lockout handling |
| Multi-Factor Authentication | Enrollment, verification, recovery for all factor types |
| Session Management | Full session lifecycle, expiry, and revocation |
| Device Management | Device registration, trust, and revocation |
| Password Management | Policy enforcement, change, reset, expiry |
| Token Management | Issuance, rotation, revocation, introspection |
| Administrative Operations | Force logout, account unlock, admin password reset |

## 2.2 Out of Scope

The following are explicitly excluded from this document:

| Excluded Area | Owner |
|---|---|
| User registration / identity creation | Identity Bounded Context |
| Role and permission assignment | Authorization / Role Management Context |
| User profile data (name, email, phone) | Identity Bounded Context |
| Corporate onboarding | Corporate Management Context |
| Notification content templates | Notification Context |
| Fraud scoring algorithm | Fraud / Risk Context |
| Payment authorization | Payment Context |
| API gateway configuration | Platform Infrastructure |

---

---

# 3. Actors & Roles

## 3.1 Actor Definitions

### Corporate User
An employee of a corporate customer who accesses the Corporate Portal. May have various business roles (maker, checker, approver) that are determined by the Authorization Context, not the Authentication Module. From the Authentication Module's perspective, a Corporate User is simply a principal with an `identity_id`.

### Bank Administrator
An employee of the bank who accesses the Bank Administration Portal. This actor requires a higher security posture: MFA is mandatory for all Bank Administrators regardless of corporate policy configuration.

### Mobile Approver
A Corporate User accessing the platform exclusively via the Mobile Approval native application. Authentication flows are identical to Corporate User but the client application context differs (different `client_id`, different device fingerprinting approach).

### API Client (Machine)
A system or application accessing the Public API using OAuth2 Client Credentials. This actor has no human user behind it; credentials are a `client_id` and `client_secret`. MFA does not apply. Session management is based on token expiry rather than user sessions.

### System Actor (Internal)
The Authentication Module itself, operating autonomously. Examples: the background session expiry sweeper, the outbox relay process, the lockout auto-unlock process.

### Fraud/Risk System
An external system within the platform that may send inbound commands to the Authentication Module (e.g., request Force Logout for a suspected compromised identity). Authentication Module exposes an internal-only API for this purpose, accessible only via mTLS from authorized service accounts.

## 3.2 Actor Capabilities Matrix

| Capability | Corporate User | Bank Admin | Mobile Approver | API Client | System Actor | Fraud/Risk |
|---|---|---|---|---|---|---|
| Login (password) | ✓ | ✓ | ✓ | — | — | — |
| Login (client credentials) | — | — | — | ✓ | — | — |
| MFA Enrollment | ✓ | ✓ (mandatory) | ✓ | — | — | — |
| MFA Verification | ✓ | ✓ | ✓ | — | — | — |
| Session List (own) | ✓ | ✓ | ✓ | — | — | — |
| Session Revoke (own) | ✓ | ✓ | ✓ | — | — | — |
| Force Logout (any identity) | — | ✓ | — | — | — | ✓ (via API) |
| Device Register | ✓ | ✓ | ✓ | — | — | — |
| Device Trust | ✓ | ✓ | ✓ | — | — | — |
| Password Change | ✓ | ✓ | ✓ | — | — | — |
| Password Reset (self) | ✓ | ✓ | ✓ | — | — | — |
| Admin Password Reset (any) | — | ✓ | — | — | — | — |
| Account Unlock | — | ✓ | — | — | ✓ (auto) | — |
| Token Refresh | ✓ | ✓ | ✓ | ✓ | — | — |
| Token Introspect | — | — | — | ✓ (RS) | — | — |

*RS = Resource Server*

---

---

# 4. Functional Area 1: Login & Credential Verification

## 4.1 Standard Login Flow

### FR-LOGIN-001: Login Endpoint Acceptance
The system SHALL accept login requests at `POST /api/v1/auth/login` with the following inputs:

| Field | Type | Required | Validation |
|---|---|---|---|
| `identifier` | string | Yes | 3–256 characters; trimmed of leading/trailing whitespace |
| `password` | string | Yes | 1–128 characters (max enforced to prevent DoS on hash computation) |
| `client_id` | string | Yes | Must match a registered OAuth2 client |
| `device_fingerprint` | object | No | See Device Fingerprint specification (Section 4.6) |

### FR-LOGIN-002: Identifier Resolution
The system SHALL resolve the submitted `identifier` to an internal `identity_id` using the following lookup order:

1. Exact match on `username` field in the Identity Context
2. (Phase 2) Exact match on `email` field in the Identity Context

If no identity is found, the system SHALL proceed through the same timing-consistent path as a failed credential check (see FR-LOGIN-008) before returning a response. The system SHALL NOT return an early response that reveals whether the identity exists.

### FR-LOGIN-003: Identity Status Check
Before verifying credentials, the system SHALL verify with the Identity Context that the resolved identity is in `ACTIVE` status.

Acceptable identity statuses for login:
- `ACTIVE` → proceed with credential verification

Statuses that SHALL block login with a generic error (same response as invalid credentials):
- `INACTIVE`
- `SUSPENDED`
- `PENDING_ACTIVATION`
- `DELETED`

**Rationale:** Revealing that an identity exists but is suspended allows enumeration of valid identities.

### FR-LOGIN-004: Credential Lockout Pre-Check
Before performing password hash verification, the system SHALL check the credential's `lockout_status`:

- If `LOCKED_TEMPORARY` AND `locked_until > NOW()`: return lockout response (ERR-LOGIN-003) immediately. Do NOT perform hash verification.
- If `LOCKED_TEMPORARY` AND `locked_until <= NOW()`: automatically unlock the credential (reset `lockout_status` to `UNLOCKED`, reset `failed_attempt_count` to 0, set `locked_until` to NULL), then proceed with verification.
- If `LOCKED_PERMANENT`: return lockout response (ERR-LOGIN-003) immediately. Do NOT perform hash verification.
- If `UNLOCKED`: proceed with verification.

### FR-LOGIN-005: Password Verification
The system SHALL verify the submitted password against the stored credential using the algorithm specified in the `password_algorithm` field:

- **Argon2id** (default): Use stored parameters (time cost, memory cost, parallelism) from `password_version`. Verify using constant-time comparison.
- **bcrypt** (legacy, migration only): Verify using bcrypt.CheckPasswordHash with constant-time comparison.

The system SHALL NOT log, store, or transmit the submitted plaintext password at any point.

### FR-LOGIN-006: Successful Credential Verification — Decision Tree

After successful password verification, the system SHALL evaluate the following in order:

```
1. Is force_password_change = TRUE?
   → YES: Create a limited "force change" session token (not a full access token).
          Return response with authentication_result = PASSWORD_CHANGE_REQUIRED.
          The force change session token is only valid for the /auth/password/change endpoint.

2. Is password_expires_at < NOW()?
   → YES: Same as step 1 — force password change.

3. Is MFA required? (See BR-003 for MFA requirement evaluation)
   → YES: Create temporary MFA session state in Redis. Return MFA_REQUIRED response.

4. MFA not required:
   → Proceed to session creation (FR-SESSION-001).
```

### FR-LOGIN-007: Failed Credential Verification

When password verification fails, the system SHALL:

1. Increment `failed_attempt_count` by 1 atomically.
2. Set `last_failed_at` to `NOW()`.
3. Evaluate lockout policy (FR-LOGIN-009).
4. Persist the `LoginAttempt` record with `outcome = FAILED_CREDENTIALS`.
5. Publish `LoginFailed` domain event via outbox.
6. Return ERR-LOGIN-001 (generic invalid credentials — same response regardless of reason).

**Timing consistency:** The system SHALL ensure that the response time for a non-existent identity matches the response time for an existing identity with an incorrect password. This prevents timing-based user enumeration. Implementation: always perform a dummy Argon2id computation when the identity is not found.

### FR-LOGIN-008: Rate Limiting Enforcement

Rate limiting SHALL be evaluated before any database operations:

1. Check per-IP rate limit in Redis (10 attempts / 5 minutes).
2. Check per-identity-hash + IP rate limit in Redis (5 attempts / 15 minutes). The identity key is a SHA-256 hash of the submitted identifier.
3. If either limit is exceeded: return ERR-RATE-001 immediately. Do NOT proceed with lookup or verification.

### FR-LOGIN-009: Lockout Policy Evaluation

After a failed credential verification, the system SHALL evaluate lockout using the following logic:

```
If failed_attempt_count >= lockout_threshold (default: 5)
  AND all failures occurred within lockout_window (default: 15 minutes):

  Determine lockout tier from lockout history:
    - 0 prior lockouts → LOCKED_TEMPORARY, duration = 30 minutes
    - 1 prior lockout  → LOCKED_TEMPORARY, duration = 2 hours
    - 2+ prior lockouts → LOCKED_PERMANENT

  Set lockout_status, locked_at = NOW(), locked_until (if temporary).
  Emit AccountLocked event.
  Send notification to user (via event → Notification Context).
```

All thresholds and durations are configurable via system configuration (see Section 16).

**Prior Lockout Tracking:**

The system SHALL track the number of prior lockouts using a `lockout_history_count` column on the `credentials` table:

- `lockout_history_count` is initialized to `0` on credential creation.
- `lockout_history_count` is incremented by 1 each time a new lockout is triggered (regardless of tier).
- `lockout_history_count` SHALL NOT be reset when a temporary lockout expires or is manually unlocked — it is a cumulative lifetime counter.
- The tier determination uses `lockout_history_count` BEFORE incrementing:

| `lockout_history_count` (before increment) | Tier Applied |
|---|---|
| 0 | LOCKED_TEMPORARY, 30 minutes |
| 1 | LOCKED_TEMPORARY, 2 hours |
| >= 2 | LOCKED_PERMANENT |

### FR-LOGIN-010: MFA Session State

When MFA is required, the system SHALL:

1. Generate a cryptographically random `mfa_session_token` (32 bytes, URL-safe Base64 encoded).
2. Store the following in Redis with TTL = `mfa_session_window` (default: 600 seconds):
   ```
   key:   mfa_session:{token}
   value: {
     identity_id,
     credential_verified_at,
     available_factor_ids,
     failed_mfa_attempts: 0,
     client_id,
     device_fingerprint
   }
   ```
3. Return `mfa_session_token` and list of `available_factors` to the client.
4. The MFA session token SHALL NOT be a JWT. It is an opaque random value.

### FR-LOGIN-011: MFA Session Expiry

The `mfa_session_token` is valid for exactly `mfa_session_window` seconds (default: 10 minutes) from issuance.

If the MFA session token has expired when the user submits a code:
- The system SHALL return ERR-MFA-004.
- The user MUST restart the login flow from the beginning (re-submit credentials).
- The expired MFA session state SHALL be deleted from Redis.

### FR-LOGIN-012: Concurrent Session Policy Enforcement

On successful authentication (after all factors verified), before creating a new session, the system SHALL enforce the concurrent session policy configured for the `client_id`:

| Policy | Behavior |
|---|---|
| `ALLOW_ALL` | Create new session without any action on existing sessions |
| `LIMIT_N` | Count active sessions for identity + client_id. If count >= N, revoke the oldest session (by `created_at`). Then create new session. |
| `SINGLE` | Revoke all existing active sessions for identity + client_id. Then create new session. |

Default policy: `ALLOW_ALL` for API clients; `LIMIT_N` (N=5) for web portal clients.

---

## 4.2 MFA Verification Flow

### FR-MFA-VERIFY-001: MFA Code Submission

The system SHALL accept MFA verification at `POST /api/v1/auth/mfa/verify` with:

| Field | Type | Required | Validation |
|---|---|---|---|
| `mfa_session_token` | string | Yes | Non-empty; looked up in Redis |
| `factor_type` | string | Yes | Must be one of: `TOTP`, `SMS_OTP`, `FIDO2`, `RECOVERY_CODE` |
| `code` | string | Conditional | Required for TOTP, SMS_OTP, RECOVERY_CODE |
| `credential` | object | Conditional | Required for FIDO2 (WebAuthn assertion response) |

### FR-MFA-VERIFY-002: TOTP Verification Logic

1. Retrieve `mfa_session_token` state from Redis. If not found or expired → ERR-MFA-004.
2. Load the active TOTP factor for the `identity_id`.
3. Decrypt the TOTP secret.
4. Compute the expected TOTP code for:
   - Current 30-second time window: `floor(NOW() / 30)`
   - Previous window: `floor(NOW() / 30) - 1`
   - Next window: `floor(NOW() / 30) + 1`
5. Compare submitted `code` against all three windows using constant-time comparison.
6. Check that the submitted code has not been used in the current time window (replay prevention). Store `last_used_window` on the factor; reject if `current_window == last_used_window` AND the code matches.
7. If valid: update `last_used_at`, `last_used_window`; proceed to session creation.
8. If invalid: increment `failed_mfa_attempts` in Redis session state. If >= 5 → trigger account lockout (same policy as credential lockout). Return ERR-MFA-001.

### FR-MFA-VERIFY-003: SMS OTP Verification Logic

1. Retrieve `mfa_session_token` state from Redis.
2. Look up the active `mfa_pending_otp` record for the identity with `purpose = MFA_VERIFICATION` and `is_used = FALSE` and `expires_at > NOW()`.
3. If no valid OTP record found → ERR-MFA-005 (OTP expired or not requested).
4. Increment `attempt_count` on the OTP record. If `attempt_count > 3` → invalidate OTP (set `is_used = TRUE`), return ERR-MFA-006 (too many OTP attempts; user must request new OTP).
5. Compare submitted `code` against `otp_hash` using constant-time comparison.
6. If valid: mark OTP as used (`is_used = TRUE`, `used_at = NOW()`); proceed to session creation.
7. If invalid: return ERR-MFA-001.

### FR-MFA-VERIFY-004: FIDO2 Assertion Verification Logic

1. Retrieve `mfa_session_token` state, which must include the `challenge` generated during the MFA challenge phase.
2. Parse the WebAuthn assertion response:
   - `clientDataJSON` (Base64URL decoded)
   - `authenticatorData` (Base64URL decoded)
   - `signature`
   - `credentialId`
3. Validate `clientDataJSON`:
   - `type` MUST be `"webauthn.get"`
   - `challenge` MUST match the stored challenge (Base64URL encoded)
   - `origin` MUST match the configured `rpOrigin`
4. Validate `authenticatorData`:
   - `rpIdHash` MUST match SHA-256 of the configured `rpId`
   - User Presence (UP) flag MUST be set (bit 0)
   - User Verification (UV) flag: checked according to `userVerification` policy
5. Look up the FIDO2 factor by `credentialId` for the `identity_id`.
6. Verify signature using the stored `fido2_public_key` (COSE-encoded, ES256 or RS256).
7. Validate the signature counter: `new_counter > stored_counter` (prevents cloned authenticator replay). If `new_counter <= stored_counter` → revoke factor, emit security event, return ERR-MFA-007.
8. Update `fido2_counter` to `new_counter`, `last_used_at = NOW()`.
9. Proceed to session creation.

### FR-MFA-VERIFY-005: Recovery Code Verification Logic

1. Retrieve `mfa_session_token` state from Redis.
2. Load all unused recovery codes for the identity's active MFA factors.
3. For each stored `code_hash`, compare SHA-256 of the submitted recovery code using constant-time comparison.
4. If a match is found:
   - Mark the matched recovery code as used (`is_used = TRUE`, `used_at = NOW()`).
   - Emit `MFARecoveryCodeUsed` event.
   - Count remaining unused recovery codes. If count < 3: include `low_recovery_codes_warning: true` in the response.
   - Proceed to session creation with AAL1 (recovery code does not elevate to AAL2).
5. If no match → ERR-MFA-001.

### FR-MFA-VERIFY-006: FIDO2 Challenge Generation

When the MFA challenge is issued for a FIDO2 factor, the system SHALL:

1. Generate a cryptographically random challenge: minimum 16 bytes, Base64URL encoded.
2. Store the challenge in the MFA session state in Redis.
3. Return `public_key_credential_request_options` to the client:
   ```json
   {
     "challenge": "<base64url-random>",
     "rpId": "bank.com",
     "allowCredentials": [
       {
         "type": "public-key",
         "id": "<base64url-credential-id>"
       }
     ],
     "userVerification": "preferred",
     "timeout": 60000
   }
   ```

---

## 4.3 Post-Authentication Actions

### FR-POST-AUTH-001: Login Attempt Record Creation

Regardless of outcome, the system SHALL persist a `LoginAttempt` record AFTER the authentication decision is made. The record SHALL include:

| Field | Value |
|---|---|
| `identity_id` | Resolved identity_id or NULL if not found |
| `identifier_used` | SHA-256 hash of the raw submitted identifier |
| `outcome` | `SUCCEEDED`, `FAILED_CREDENTIALS`, `FAILED_MFA`, `FAILED_LOCKED`, `FAILED_RATE_LIMITED`, `FAILED_IDENTITY_INACTIVE` |
| `failure_reason` | Specific sub-reason (e.g., `WRONG_PASSWORD`, `OTP_EXPIRED`) |
| `ip_address` | Client IP from `X-Forwarded-For` (first hop, validated) or `REMOTE_ADDR` |
| `user_agent` | `User-Agent` header value, truncated to 512 characters |
| `device_fingerprint` | Computed fingerprint hash (or NULL if not provided) |
| `attempted_at` | `NOW()` |
| `session_id` | Created session_id (if outcome = `SUCCEEDED`) |
| `mfa_factor_type` | Factor type used for MFA (if applicable) |

### FR-POST-AUTH-002: Successful Login Response

On successful authentication, the system SHALL return:

```json
{
  "authentication_result": "COMPLETED",
  "access_token": "<JWT>",
  "token_type": "Bearer",
  "expires_in": <access_token_ttl_seconds>,
  "refresh_token": "<opaque_token>",
  "session_id": "<session_id>",
  "aal": "<AAL1|AAL2|AAL3>",
  "force_password_change": false
}
```

The `refresh_token` SHALL be delivered:
- In the JSON body for mobile/API clients
- Additionally as an `HttpOnly; Secure; SameSite=Strict` cookie named `__Secure-refresh_token` for web browser clients (detected by `client_id` type or explicit `delivery_method` parameter)

---

---

# 5. Functional Area 2: Multi-Factor Authentication (MFA)

## 5.1 MFA Factor Enrollment

### FR-MFA-ENROLL-001: Enrollment Authorization

MFA enrollment SHALL only be available to authenticated users (valid Access Token required), with the following exception: if MFA enrollment is mandatory and the user has no active factor, enrollment is accessible via the forced-enrollment flow using a restricted session token.

### FR-MFA-ENROLL-002: TOTP Enrollment — Initiation

`POST /api/v1/auth/mfa/totp/enroll`

1. Generate a cryptographically random TOTP secret: 20 bytes (160 bits), encoded as Base32.
2. Generate an `enrollment_id` (UUID).
3. Store the following in a `mfa_factors` record with `status = PENDING`:
   - `identity_id` from Access Token claims
   - `factor_type = TOTP`
   - `totp_secret_encrypted` = AES-256-GCM encryption of the Base32 secret; key from Vault
   - `totp_algorithm = SHA1`, `totp_digits = 6`, `totp_period = 30`
4. Construct the `otpauth://` URI:
   ```
   otpauth://totp/{issuer}:{display_name}?secret={base32_secret}&issuer={issuer}&algorithm=SHA1&digits=6&period=30
   ```
   Where `issuer` is the configured platform name and `display_name` is the user's display name from Identity Context.
5. Generate a QR code PNG from the URI, encoded as `data:image/png;base64,{...}`.
6. Return `enrollment_id`, `otpauth_uri`, `qr_code_data_url`, and `expires_in` (enrollment window: 300 seconds).

**Pending enrollment cleanup:** If the user navigates away, the PENDING factor record is cleaned up by a background sweeper after the enrollment window expires.

### FR-MFA-ENROLL-003: TOTP Enrollment — Confirmation

`POST /api/v1/auth/mfa/totp/confirm`

1. Look up the `enrollment_id` in `mfa_factors` where `status = PENDING` and `identity_id` matches Access Token.
2. If not found or enrollment window has expired → ERR-MFA-ENROLL-001.
3. Decrypt the stored TOTP secret.
4. Compute TOTP codes for current and adjacent windows (±1).
5. Validate the submitted `code` against the computed codes.
6. If invalid → ERR-MFA-001. If this is the 3rd consecutive failure, delete the PENDING factor and return ERR-MFA-ENROLL-002.
7. If valid:
   - Update factor `status = ACTIVE`, `enrolled_at = NOW()`.
   - Generate 10 recovery codes:
     - Each code: 8 characters, alphanumeric (A-Z, 2-9), split as `XXXX-XXXX` for display.
     - Store SHA-256 hash of each code in `mfa_recovery_codes`.
     - The raw codes are returned to the user ONCE and never stored in plaintext.
   - Emit `MFAEnabled` domain event.
8. Return `factor_id`, `status = ACTIVE`, and the array of 10 raw recovery codes.

**The recovery codes must be displayed to the user with a strong advisory to save them before closing the screen. After this response, raw codes cannot be retrieved again.**

### FR-MFA-ENROLL-004: FIDO2 Enrollment — Begin

`POST /api/v1/auth/mfa/fido2/register/begin`

1. Generate a cryptographically random registration challenge: 32 bytes, Base64URL encoded.
2. Generate an `enrollment_id` (UUID).
3. Store the challenge and enrollment metadata in Redis with TTL = 300 seconds.
4. Retrieve the authenticated user's identity_id and display name (from Identity Context ACL).
5. Assemble and return `public_key_credential_creation_options`:

```json
{
  "enrollment_id": "<uuid>",
  "public_key_credential_creation_options": {
    "challenge": "<base64url-32-bytes>",
    "rp": {
      "name": "<configured_rp_name>",
      "id": "<configured_rp_id>"
    },
    "user": {
      "id": "<base64url-identity_id>",
      "name": "<username>",
      "displayName": "<display_name>"
    },
    "pubKeyCredParams": [
      { "type": "public-key", "alg": -7 },
      { "type": "public-key", "alg": -257 }
    ],
    "excludeCredentials": [
      { "type": "public-key", "id": "<existing_credential_id>" }
    ],
    "authenticatorSelection": {
      "residentKey": "preferred",
      "userVerification": "preferred"
    },
    "attestation": "indirect",
    "timeout": 60000
  }
}
```

`excludeCredentials` SHALL include all existing FIDO2 credential IDs for the identity, to prevent re-registration of an already-registered key.

### FR-MFA-ENROLL-005: FIDO2 Enrollment — Complete

`POST /api/v1/auth/mfa/fido2/register/complete`

1. Retrieve enrollment challenge from Redis by `enrollment_id`. If expired → ERR-MFA-ENROLL-001.
2. Parse `attestationObject` and `clientDataJSON` from the request.
3. Validate `clientDataJSON`:
   - `type` MUST be `"webauthn.create"`
   - `challenge` MUST match the stored challenge
   - `origin` MUST match configured `rpOrigin`
4. Validate `attestationObject`:
   - Parse CBOR-encoded attestation object
   - Verify `rpIdHash` = SHA-256 of configured `rpId`
   - User Presence (UP) flag MUST be set
   - Extract `credentialId`, COSE-encoded `credentialPublicKey`, `signCount`, and `aaguid`
5. Verify `credentialId` is not already registered for ANY identity (global uniqueness check).
6. If attestation format is `"indirect"` or `"packed"`: validate attestation statement (verify certificate chain). Log attestation result.
7. Create `mfa_factors` record:
   - `factor_type = FIDO2`
   - `status = ACTIVE`
   - `fido2_credential_id` = raw credential ID bytes
   - `fido2_public_key` = COSE-encoded public key bytes
   - `fido2_aaguid` = parsed AAGUID
   - `fido2_counter = signCount` from attestation
   - `display_name` = submitted `device_name` parameter
8. Emit `MFAEnabled` event.
9. Return `factor_id`, `status = ACTIVE`.

### FR-MFA-ENROLL-006: OTP MFA — No Explicit Enrollment

SMS OTP does not require explicit enrollment by the user. The user's phone number is sourced from the Identity Context. If no phone number is registered in the Identity Context, SMS OTP is unavailable as a factor.

The system SHALL query the Identity Context at MFA challenge time to determine whether SMS OTP is available for the user. The Identity Context returns a masked phone number (e.g., `+62-***-****-7890`) for display.

### FR-MFA-ENROLL-007: OTP Delivery

When SMS OTP is selected as the MFA factor during login:

1. Generate a 6-digit cryptographically random numeric code (not using `Math.random()` or equivalent non-CSPRNG sources).
2. Compute SHA-256 hash of the code.
3. Store in `mfa_pending_otps`:
   - `identity_id`
   - `otp_hash` = SHA-256 of raw code
   - `purpose = MFA_VERIFICATION`
   - `expires_at = NOW() + 300 seconds`
   - `attempt_count = 0`
4. Publish `OTPRequested` internal event → Notification Context delivers the raw OTP code via SMS.
5. Rate limit: if 3 OTP send requests have been made in the last 10 minutes for this identity, return ERR-MFA-008 (too many OTP requests). Do not generate a new OTP.

---

## 5.2 MFA Factor Lifecycle Management

### FR-MFA-LIFECYCLE-001: List MFA Factors

`GET /api/v1/auth/mfa/factors`

Returns all MFA factors for the authenticated identity, regardless of status. Response omits all sensitive data (secrets, private keys). Returns: `factor_id`, `factor_type`, `status`, `display_name`, `enrolled_at`, `last_used_at`.

### FR-MFA-LIFECYCLE-002: Disable MFA Factor

`PATCH /api/v1/auth/mfa/factors/{factor_id}`  with `{ "status": "DISABLED" }`

Conditions:
- Factor must belong to the authenticated identity.
- If this is the only active factor AND corporate MFA policy is mandatory → ERR-MFA-010 (cannot disable last mandatory factor).
- If this is the only active factor AND corporate MFA policy is optional → warn but allow.
- Emit `MFADisabled` event.

### FR-MFA-LIFECYCLE-003: Revoke (Remove) MFA Factor

`DELETE /api/v1/auth/mfa/factors/{factor_id}`

The system SHALL require step-up authentication:
- If the user has another active MFA factor: require re-verification of that factor first.
- If no other active MFA factor: require current password verification.

On successful authorization:
- Set factor `status = REVOKED`.
- Delete all associated recovery codes.
- If this removes all active MFA factors: emit `MFADisabled` event for the identity.
- Emit `MFAFactorRevoked` event.

### FR-MFA-LIFECYCLE-004: Regenerate Recovery Codes

`POST /api/v1/auth/mfa/factors/{factor_id}/recovery-codes/regenerate`

The system SHALL require step-up MFA verification before regenerating recovery codes.

On success:
- Mark all existing recovery codes for the factor as `is_used = TRUE`, `used_at = NOW()`.
- Generate 10 new recovery codes (same algorithm as FR-MFA-ENROLL-003 step 7).
- Return the 10 raw codes once.
- Emit `MFARecoveryCodesRegenerated` event.

---

---

# 6. Functional Area 3: Session Management

## 6.1 Session Creation

### FR-SESSION-001: Session Creation on Successful Authentication

Immediately after all authentication factors are verified, the system SHALL create a session:

1. Generate `session_id`: UUID v4 (128-bit random).
2. Create `authentication_sessions` record:
   - `identity_id` from credential lookup
   - `device_id` from device registration/lookup result (Section 7)
   - `status = ACTIVE`
   - `aal` = determined by factors used:
     - Password only → `AAL1`
     - Password + OTP/TOTP → `AAL2`
     - Password + FIDO2 (UV required) → `AAL3`
     - Recovery code used → `AAL1` (recovery does not elevate)
   - `mfa_factors_used` = JSON array of factor IDs used
   - `created_at = NOW()`
   - `last_activity_at = NOW()`
   - `idle_timeout_secs` = configured policy for the `client_id`
   - `absolute_expires_at = NOW() + absolute_session_duration` (configured policy)
   - `creation_ip` = client IP
   - `creation_user_agent` = User-Agent header
3. Cache session in Redis (TTL = `absolute_expires_at - NOW()`) for hot-path lookup.
4. Emit `SessionCreated` event.

### FR-SESSION-002: Access Token Issuance

After session creation, the system SHALL issue an Access Token (JWT):

Claims construction:
```
{
  "iss": <configured_issuer_url>,
  "sub": <identity_id>,
  "aud": [<client_id>, <additional_audiences_from_config>],
  "iat": <now_unix_timestamp>,
  "exp": <now + access_token_ttl>,
  "jti": <UUID v4 — unique token ID>,
  "session_id": <session_id>,
  "aal": <AAL1|AAL2|AAL3>,
  "device_id": <device_id>,
  "corporate_id": <corporate_id from Identity Context, if applicable>,
  "scope": <space-separated OAuth2 scopes granted to client_id>
}
```

Signing:
- Sign with the current active RSA private key using RS256.
- Include `kid` (key ID) in the JWT header matching the key published in JWKS.

### FR-SESSION-003: Refresh Token Issuance

Simultaneously with Access Token issuance:

1. Create `refresh_token_families` record (status = `ACTIVE`).
2. Generate raw refresh token: 32 cryptographically random bytes, Base64URL encoded (43 characters output).
3. Compute `token_hash = SHA-256(raw_token)`.
4. Create `refresh_tokens` record:
   - `token_hash`
   - `status = ACTIVE`
   - `client_id`
   - `issued_at = NOW()`
   - `expires_at = NOW() + refresh_token_ttl` (policy-based; default: 24h web, 7d mobile)
   - `issuer_ip`
5. Return `raw_token` to the client (never stored again).
6. Emit `RefreshTokenIssued` event.

---

## 6.2 Session Expiry

### FR-SESSION-004: Lazy Expiry Check

The system SHALL evaluate session validity on every token refresh request:

```
If session.status != ACTIVE → reject (ERR-TOKEN-002)
If session.last_activity_at + idle_timeout_secs < NOW() → mark session EXPIRED; reject (ERR-SESSION-002)
If session.absolute_expires_at < NOW() → mark session EXPIRED; reject (ERR-SESSION-002)
```

### FR-SESSION-005: Eager Expiry Sweep

A background process SHALL run every 5 minutes and:

1. Query `authentication_sessions` where `status = ACTIVE` AND (`last_activity_at + idle_timeout_secs < NOW()` OR `absolute_expires_at < NOW()`).
2. For each expired session:
   - Set `status = EXPIRED`.
   - Revoke all associated `refresh_tokens` (set `status = EXPIRED`).
   - Emit `SessionRevoked` with `revoke_reason = SESSION_EXPIRED_IDLE` or `SESSION_EXPIRED_ABSOLUTE`.
   - Remove from Redis cache.

### FR-SESSION-006: Session Activity Update

On every successful token refresh:
- Update `session.last_activity_at = NOW()`.
- Update the Redis session cache TTL.

This resets the idle timeout clock. The absolute expiry is NOT reset.

---

## 6.3 Session Revocation

### FR-SESSION-007: User-Initiated Logout

`POST /api/v1/auth/logout`

1. Validate the Access Token from the `Authorization` header. Extract `session_id` and `jti`.
2. Extract the raw `refresh_token` from the request body (or `__Secure-refresh_token` cookie).
3. Compute `token_hash = SHA-256(raw_refresh_token)`.
4. Load the `refresh_token_families` record for this `token_hash`. Verify `session_id` matches.
5. Set session `status = REVOKED`, `revoked_at = NOW()`, `revoke_reason = USER_LOGOUT`.
6. Set all `refresh_tokens` in the family to `status = REVOKED`.
7. Add the Access Token's `jti` to the Redis revocation list with TTL = `exp - NOW()`.
8. Clear the session from Redis cache.
9. If delivered via cookie: set `__Secure-refresh_token` cookie with `Max-Age=0` (delete cookie).
10. Emit `SessionRevoked` event.
11. Return 204 No Content.

### FR-SESSION-008: Revoke Specific Session

`DELETE /api/v1/auth/sessions/{session_id}`

The authenticated user (Access Token) can revoke any of their own sessions EXCEPT the session associated with the current Access Token.

1. Load the session by `session_id`. Verify `identity_id` matches the Access Token's `sub`.
2. If `session_id` matches the current `session_id` in the Access Token → ERR-SESSION-003 (use /logout to end current session).
3. Set session `status = REVOKED`, `revoke_reason = USER_REVOKED_OTHER_SESSION`.
4. Revoke all refresh tokens in all families for this session.
5. Emit `SessionRevoked` event.
6. Return 204.

### FR-SESSION-009: Force Logout (Admin or Fraud System)

`DELETE /api/v1/auth/sessions` (with `identity_id` in request body)

Requires `scope: admin:sessions` in the Access Token.

1. Load all `ACTIVE` sessions for the target `identity_id`.
2. For each active session:
   - Set `status = REVOKED`, `revoke_reason = ADMIN_FORCE_LOGOUT`.
   - Revoke all refresh tokens in all families for the session.
   - Add any known Access Token JTIs to the Redis revocation list. (JTIs are tracked in the session cache; if the cache has expired, the AT will expire naturally within its TTL.)
3. Clear all Redis session cache entries for the identity.
4. Emit `SessionRevoked` (per session) and `AdminForceLogout` (one aggregate event).
5. Return 204.

### FR-SESSION-010: Session List

`GET /api/v1/auth/sessions`

Returns all `ACTIVE` sessions for the authenticated user. Response fields per session:
- `session_id`
- `device_name` (from linked `registered_devices.display_name`)
- `created_at`
- `last_activity_at`
- `absolute_expires_at`
- `ip_address` (creation IP)
- `aal`
- `is_current` (true if `session_id` matches the current Access Token's `session_id` claim)

---

---

# 7. Functional Area 4: Device Management

## 7.1 Device Fingerprinting

### FR-DEVICE-001: Fingerprint Computation

Device fingerprint computation SHALL occur server-side using client-supplied attributes from the login request body and server-observed request attributes.

**Fingerprint components (canonical order for hash computation):**

| Component | Source | Notes |
|---|---|---|
| `user_agent` | `User-Agent` header | Truncated to 512 chars |
| `accept_language` | `Accept-Language` header | First 64 chars |
| `screen_resolution` | Request body (client-supplied) | Format: `{width}x{height}` or `null` |
| `timezone` | Request body (client-supplied) | IANA timezone string or offset |
| `platform` | Request body (client-supplied) | e.g., `Win32`, `MacIntel`, `iPhone` |
| `color_depth` | Request body (client-supplied) | Integer bits |
| `language` | Request body (client-supplied) | BCP 47 language tag |
| `ip_subnet` | Derived from client IP | First 24 bits (e.g., `203.0.113.0/24`) |

**Computation:**
1. Canonicalize each component: trim whitespace, lowercase where applicable, replace NULL with empty string.
2. Concatenate in canonical order with `|` separator.
3. Compute SHA-256. Encode as lowercase hex.
4. Result is `fingerprint_hash` (64-character hex string).
5. Store `fingerprint_version = 1` alongside the hash.

### FR-DEVICE-002: Device Lookup and Registration (Automatic)

On each successful credential verification (before session creation):

1. Compute `fingerprint_hash` from login request.
2. Look up `registered_devices` where `identity_id = ?` AND `fingerprint_hash = ?` AND `trust_status != REVOKED`.
3. **If found (existing device):**
   - Update `last_seen_at = NOW()`.
   - Proceed with the found `device_id`.
4. **If not found (new device):**
   - Create new `registered_devices` record with `trust_status = REGISTERED`.
   - Set `display_name` = inferred name from User-Agent (e.g., `Chrome on Windows`).
   - Set `registration_ip` = client IP.
   - Emit `DeviceRegistered` (with `was_automatic = true`).
   - Store `device_id` for session association.

### FR-DEVICE-003: Device Name Inference

When automatically registering a device, the system SHALL infer a human-readable `display_name` by parsing the `User-Agent` string:

| UA Pattern | Display Name |
|---|---|
| Contains `Mobile` + `Safari` | `Mobile Safari on iOS` |
| Contains `Chrome` + `Android` | `Chrome on Android` |
| Contains `Chrome` (not Android) | `Chrome on {OS}` |
| Contains `Firefox` | `Firefox on {OS}` |
| Contains `Safari` (not Chrome) | `Safari on macOS` |
| Contains `PostmanRuntime` | `Postman API Client` |
| Other | `Unknown Browser` |

OS detection from UA: `Windows NT` → `Windows`; `Mac OS X` → `macOS`; `Linux` → `Linux`.

---

## 7.2 Device Trust

### FR-DEVICE-004: Mark Device as Trusted

`PATCH /api/v1/auth/devices/{device_id}` with `{ "trusted": true }`

1. Load device by `device_id`. Verify `identity_id` matches the authenticated user.
2. Verify device `trust_status != REVOKED`.
3. Set `trust_status = TRUSTED`.
4. Set `trust_expires_at = NOW() + device_trust_duration` (default: 30 days; configurable per corporate policy).
5. Emit `DeviceTrusted` event.
6. Return updated device object.

### FR-DEVICE-005: Trust Expiry

On each login, when loading the associated device:
- If `trust_status = TRUSTED` AND `trust_expires_at < NOW()`:
  - Set `trust_status = REGISTERED` (downgrade; not revoked).
  - Clear `trust_expires_at`.

**Trusted device behavior:** When an identity has an active session on a trusted device, the system MAY apply `step_up_mfa_only` policy: MFA is required only for sensitive operations (defined by the downstream service via scope claims), not for every login. This is a corporate-policy-configurable option.

### FR-DEVICE-006: Device Revocation

`DELETE /api/v1/auth/devices/{device_id}`

1. Load device. Verify `identity_id` matches.
2. Set `trust_status = REVOKED`, `revoked_at = NOW()`.
3. Find all ACTIVE sessions associated with this `device_id`.
4. For each session: revoke session and all associated refresh tokens (same as FR-SESSION-007 steps 5–8).
5. Emit `DeviceRevoked`.
6. Emit `SessionRevoked` for each revoked session.
7. Return 204.

### FR-DEVICE-007: Device List

`GET /api/v1/auth/devices`

Returns all non-revoked devices for the authenticated user (trust_status IN `REGISTERED`, `TRUSTED`). Includes `is_current` = true for the device associated with the current session's `device_id`.

---

---

# 8. Functional Area 5: Password Management

## 8.1 Password Policy Enforcement

### FR-PWD-001: Policy Evaluation Engine

The system SHALL apply the following checks in order when validating a proposed new password:

| Check | Rule | Error |
|---|---|---|
| Maximum length | `len(password) <= 128` | ERR-PWD-001 |
| Minimum length | `len(password) >= policy.min_length` (default: 12) | ERR-PWD-002 |
| Uppercase requirement | At least 1 uppercase letter (A–Z) | ERR-PWD-003 |
| Lowercase requirement | At least 1 lowercase letter (a–z) | ERR-PWD-004 |
| Digit requirement | At least 1 digit (0–9) | ERR-PWD-005 |
| Special character | At least 1 special character from: `!@#$%^&*()-_=+[]{}|;:',.<>?/~` | ERR-PWD-006 |
| Username inclusion | Password SHALL NOT contain the username (case-insensitive substring match) | ERR-PWD-007 |
| Common password | Password SHALL NOT match any entry in the common password blocklist | ERR-PWD-008 |
| Password history | Password hash SHALL NOT match any of the last N password hashes (default N=12) | ERR-PWD-009 |

All validation errors from a single submission SHALL be collected and returned together in the response (not one at a time), so the user can correct all issues in a single attempt.

### FR-PWD-002: Common Password Blocklist

The system SHALL maintain a configurable blocklist of commonly used passwords.

- Minimum blocklist size: 100,000 entries.
- Recommended source: NIST-recommended list, HaveIBeenPwned top passwords list.
- Comparison: normalize submitted password (lowercase, strip non-alphanumeric) before blocklist lookup. Store blocklist entries in normalized form.
- Blocklist is stored in a memory-mapped structure (e.g., Redis SET or Bloom filter) for O(1) lookup.
- Blocklist MUST be updatable without service restart (hot-reload via configuration).

### FR-PWD-003: Password History Check

When checking history:
1. Load the last `policy.history_depth` (default: 12) entries from `password_history` for the credential, ordered by `created_at DESC`.
2. For each history entry: verify submitted password against the stored hash using the `algorithm` of the history entry.
3. If any match is found → ERR-PWD-009.

### FR-PWD-004: Password Hashing

When storing a new password:

1. Generate a 16-byte random salt (Argon2id embeds salt, but store explicitly for clarity).
2. Compute: `hash = Argon2id(password, salt, t=3, m=65536, p=4, tagLen=32)`.
3. Store the full encoded hash string (includes algorithm, version, salt, and hash in the standard `$argon2id$v=19$...` format).
4. Set `password_algorithm = argon2id`, `password_version = 1`.
5. Set `password_created_at = NOW()`.
6. Set `password_expires_at = NOW() + policy.max_password_age_days` (if expiry is configured).

---

## 8.2 Password Change (Authenticated)

### FR-PWD-005: Change Password Flow

`POST /api/v1/auth/password/change`

Required: valid Access Token.

1. Extract `identity_id` from Access Token.
2. Load credential. Check lockout status (if locked → ERR-LOGIN-003).
3. Verify `current_password` against the stored hash.
4. If verification fails:
   - Increment `failed_attempt_count`.
   - Return ERR-PWD-010 (current password incorrect).
5. If verification passes:
   - Validate `new_password` through FR-PWD-001 policy engine.
   - If validation fails → return all ERR-PWD-* errors.
   - Hash `new_password` (FR-PWD-004).
   - Add current password hash + salt + algorithm to `password_history`.
   - Trim `password_history` to `policy.history_depth` entries (delete oldest excess).
   - Update `credentials`: new hash, salt, algorithm, `password_created_at`, `password_expires_at`, reset `failed_attempt_count = 0`, `force_password_change = FALSE`.
   - Revoke all sessions for this identity EXCEPT the current session (identified by `session_id` in Access Token claims).
   - Revoke all refresh token families for revoked sessions.
   - Emit `PasswordChanged`.
   - Return 204.

---

## 8.3 Forgot Password / Reset Flow

### FR-PWD-006: Forgot Password Initiation

`POST /api/v1/auth/password/forgot`

No authentication required. Unauthenticated endpoint.

1. Apply rate limiting: 3 requests per 60 minutes per identifier hash, and 5 per 60 minutes per IP.
2. Normalize and hash the submitted `identifier`.
3. Query Identity Context for the identity. **Always return 200 with a generic message regardless of outcome.** All subsequent steps are conditional and transparent to the caller.
4. If identity is found AND is in `ACTIVE` status:
   a. Check for existing unused, unexpired reset token for this identity. If one exists and was created < 5 minutes ago → do not generate a new one (prevents token flooding). Return 200 silently.
   b. Generate reset token: 32 cryptographically random bytes, URL-safe Base64 encoded (43 characters).
   c. Compute `token_hash = SHA-256(raw_token)`.
   d. Insert into `password_reset_tokens`:
      - `identity_id`, `token_hash`, `is_used = FALSE`
      - `expires_at = NOW() + 3600 seconds` (60 minutes)
      - `requested_ip`
   e. Emit `PasswordResetRequested` event (payload contains `reset_token_id`, NOT the raw token).
5. Return 200: `{ "message": "If that account exists, a password reset link has been sent." }`

### FR-PWD-007: Reset Password — Token Validation

`POST /api/v1/auth/password/reset`

No authentication required.

1. Apply rate limiting: 10 requests per 60 minutes per IP (prevents token brute-force).
2. Compute `token_hash = SHA-256(submitted_reset_token)`.
3. Look up `password_reset_tokens` by `token_hash`:
   - If not found → ERR-PWD-011 (invalid or expired token).
   - If `is_used = TRUE` → ERR-PWD-012 (token already used).
   - If `expires_at < NOW()` → ERR-PWD-013 (token expired).
4. Validate `new_password` through FR-PWD-001 policy engine.
5. If validation fails → return all ERR-PWD-* errors.
6. Hash `new_password`.
7. Update `credentials`: new hash, reset `failed_attempt_count = 0`, `lockout_status = UNLOCKED`, `force_password_change = FALSE`.
8. Add old password to `password_history`.
9. Mark reset token: `is_used = TRUE`, `used_at = NOW()`.
10. Revoke ALL active sessions for the identity.
11. Revoke all refresh token families for revoked sessions.
12. Emit `PasswordResetCompleted`.
13. Return 204.

---

## 8.4 Password Expiry Management

### FR-PWD-008: Password Expiry Warning

A background process SHALL run daily and:
1. Query `credentials` where `password_expires_at` is within the following warning thresholds from `NOW()`:
   - 14 days, 7 days, 3 days, 1 day
2. For each credential at a warning threshold: emit `PasswordExpiryWarning` event (→ Notification Context sends email/SMS).
3. Track last warning sent to avoid duplicate warnings on same day. Use a flag per threshold in the credential record or Redis.

### FR-PWD-009: Expired Password Handling at Login

When FR-LOGIN-006 step 2 detects `password_expires_at < NOW()`:
1. Issue a restricted `forced_change_session_token` (not a full JWT; opaque token stored in Redis with TTL = 10 minutes).
2. Return:
   ```json
   {
     "authentication_result": "PASSWORD_CHANGE_REQUIRED",
     "forced_change_session_token": "<token>",
     "expires_in": 600
   }
   ```
3. The `forced_change_session_token` is accepted ONLY at `POST /api/v1/auth/password/change-forced`.
4. `POST /api/v1/auth/password/change-forced`:
   - Accepts `forced_change_session_token` + `new_password`.
   - Validates `new_password` against policy.
   - Updates password hash.
   - Creates a full session and issues Access Token + Refresh Token.
   - Returns standard `COMPLETED` authentication response.

---

---

# 9. Functional Area 6: Token Management

## 9.1 Access Token Lifecycle

### FR-TOKEN-001: Token Issuance

Access Token issuance occurs in FR-SESSION-002. Additional rules:

- The `jti` claim MUST be a UUID v4, unique across all tokens ever issued (stored in Redis revocation list only if revoked; not stored proactively).
- The `aud` claim MUST include the `client_id` of the requesting client and any additional audiences configured for that client.
- The Access Token SHALL NOT contain: user passwords, MFA secrets, personal contact information, role/permission lists.
- Token signature MUST use the currently active signing key. The `kid` header MUST match the key's ID in JWKS.

### FR-TOKEN-002: JWKS Endpoint

`GET /.well-known/jwks.json`

No authentication required. Public endpoint.

The system SHALL return all signing keys that are currently valid (i.e., keys for which tokens could still be active). This includes:
- The current active signing key.
- Any previous signing keys where `revoked_at + max_access_token_ttl > NOW()`.

Response is cached with `Cache-Control: public, max-age=3600`. The endpoint SHALL respond within 100ms (served from in-memory cache; keys loaded from Vault at startup and on rotation).

### FR-TOKEN-003: Access Token Revocation

`POST /api/v1/auth/token/revoke` (RFC 7009)

1. Parse `token` and `token_type_hint` from the request body (`application/x-www-form-urlencoded` format per RFC 7009).
2. Authenticate the client via `Authorization: Basic` header.
3. If `token_type_hint = access_token` or unspecified:
   - Decode the JWT (without verifying expiry) to extract `jti` and `exp`.
   - If `exp > NOW()` (token is still active):
     - Add `jti` to Redis revocation list with TTL = `exp - NOW()`.
4. If `token_type_hint = refresh_token`:
   - Compute `token_hash = SHA-256(token)`.
   - Find the `refresh_tokens` record by hash.
   - Set `status = REVOKED`, `revoked_at = NOW()`.
5. Always return 200 regardless of whether the token was found or already expired. (Per RFC 7009 §2.2: the server SHALL respond with HTTP status code 200 if the token has been revoked successfully or if the client submitted an invalid token.)

### FR-TOKEN-004: Token Introspection

`POST /api/v1/auth/token/introspect` (RFC 7662)

Client authentication: `Authorization: Basic {base64(client_id:client_secret)}`. Only registered Resource Server clients may call this endpoint (separate client type from user-facing clients).

1. Authenticate the calling client. If authentication fails → 401.
2. Parse `token` from request body.
3. Decode JWT header to extract `kid`. If `kid` is not in the current JWKS → `{ "active": false }`.
4. Verify JWT signature using the public key for `kid`.
5. Check `exp` claim: if `exp < NOW()` → `{ "active": false }`.
6. Check `jti` against Redis revocation list: if found → `{ "active": false }`.
7. Check `session_id`: load session from Redis/DB. If session status != `ACTIVE` → `{ "active": false }`.
8. Check idle timeout: if `session.last_activity_at + idle_timeout_secs < NOW()` → mark expired → `{ "active": false }`.
9. If all checks pass → return full active introspection response with all claims.

**Response time SLA:** P95 < 50ms. Introspection is on the critical path of every authenticated API call.

---

## 9.2 Refresh Token Lifecycle

### FR-TOKEN-005: Token Refresh

`POST /api/v1/auth/token/refresh`

1. Parse `refresh_token` from request body.
2. Parse `client_id` from request body or `Authorization: Basic` header.
3. Compute `token_hash = SHA-256(refresh_token)`.
4. Look up `refresh_tokens` by `token_hash`.

**Case A — Token not found:**
- Return ERR-TOKEN-001 (invalid refresh token).

**Case B — Token status is `USED`:**
- **Reuse detection triggered.**
- Load `refresh_token_families` for this token's `family_id`.
- Set family `status = COMPROMISED`.
- Set all tokens in the family to `status = REVOKED`.
- Load the associated session and revoke it (FR-SESSION-007 steps 5–8).
- Emit `RefreshTokenFamilyCompromised` event (critical priority).
- Return ERR-TOKEN-003 (token reuse detected; all sessions revoked).

**Case C — Token status is `REVOKED` or `EXPIRED`:**
- Return ERR-TOKEN-002 (token revoked or expired).

**Case D — Token status is `ACTIVE`:**
1. Verify `expires_at > NOW()`. If expired → set status `EXPIRED`; return ERR-TOKEN-002.
2. Verify `client_id` matches the token's stored `client_id`.
3. Load associated `refresh_token_families`. Verify family `status = ACTIVE`.
4. Load associated session. Verify session `status = ACTIVE`.
5. Evaluate session idle timeout and absolute timeout (FR-SESSION-004). If session expired → ERR-SESSION-002.
6. **Rotation:**
   a. Mark current token `status = USED`, `used_at = NOW()`.
   b. Generate new raw refresh token (32 bytes random, Base64URL).
   c. Compute new `token_hash`.
   d. Create new `refresh_tokens` record in the same `family_id`.
   e. Set new token `expires_at = NOW() + refresh_token_ttl`.
7. Update session `last_activity_at = NOW()`.
8. Issue new Access Token (FR-TOKEN-001).
9. Emit `RefreshTokenRotated` event.
10. Return new Access Token + new Refresh Token.

**Grace window (reuse false positive mitigation):**
If a `USED` token is submitted AND `used_at > NOW() - 5 seconds`, AND the family status is still `ACTIVE`:
- Return the SAME new refresh token (the one that was issued during the most recent rotation for this family).
- Do NOT trigger reuse detection in this narrow window.
- Log the occurrence with warning severity.

---

---

# 10. Functional Area 7: Administrative Operations

## 10.1 Account Management

### FR-ADMIN-001: Unlock Account

`POST /api/v1/admin/auth/accounts/{identity_id}/unlock`

Requires: `scope: admin:accounts` in Access Token.

1. Load credential for `identity_id`.
2. If `lockout_status = UNLOCKED` → return 409 (account is not locked).
3. Set `lockout_status = UNLOCKED`, `failed_attempt_count = 0`, `locked_at = NULL`, `locked_until = NULL`.
4. Emit `AccountUnlocked` event (includes `unlocked_by` = admin's identity_id).
5. Return 204.

### FR-ADMIN-002: Force Password Reset

`POST /api/v1/admin/auth/accounts/{identity_id}/force-password-reset`

Requires: `scope: admin:accounts`.

1. Load credential for `identity_id`.
2. Set `force_password_change = TRUE`.
3. Emit `AdminInitiatedPasswordReset` event.
4. Return 204.

The next time the user logs in, they will be redirected to the forced password change flow (FR-PWD-009).

### FR-ADMIN-003: View Account Security Status

`GET /api/v1/admin/auth/accounts/{identity_id}/status`

Requires: `scope: admin:accounts`.

Returns:
```json
{
  "identity_id": "...",
  "lockout_status": "UNLOCKED|LOCKED_TEMPORARY|LOCKED_PERMANENT",
  "locked_until": "...",
  "failed_attempt_count": 3,
  "last_failed_at": "...",
  "force_password_change": false,
  "password_expires_at": "...",
  "active_session_count": 2,
  "active_mfa_factor_count": 1,
  "last_login_at": "...",
  "last_login_ip": "..."
}
```

### FR-ADMIN-004: Admin MFA Factor Revocation

`DELETE /api/v1/admin/auth/accounts/{identity_id}/mfa/factors/{factor_id}`

Requires: `scope: admin:mfa`. Used when a user reports a compromised device.

1. Load factor. Verify it belongs to `identity_id`.
2. Set `status = REVOKED`.
3. Delete all recovery codes for the factor.
4. Emit `MFAFactorRevoked` (with `revoked_by = admin identity_id`).
5. Emit `MFADisabled` if no active factors remain.
6. Return 204.

### FR-ADMIN-005: Admin FIDO2 Challenge Bypass

For cases where a user has lost all MFA devices and recovery codes:

`POST /api/v1/admin/auth/accounts/{identity_id}/mfa/admin-bypass`

Requires: `scope: admin:mfa`. Rate-limited: 3 requests per hour per admin.

1. Generate a single-use bypass code (UUID, 32 bytes entropy).
2. Store in Redis with TTL = 30 minutes.
3. Emit `MFAAdminBypassGenerated` event (high severity audit).
4. Return bypass code to admin (admin communicates it to user via secure out-of-band channel).

The bypass code is accepted at `POST /api/v1/auth/mfa/bypass` in place of a normal MFA code. It produces a session with `aal = AAL1` and immediately flags `force_mfa_enrollment = TRUE` on the session (user is forced to re-enroll MFA before accessing protected resources).

---

---

# 11. Business Rules

## BR-001: Password Maximum Length Enforcement

Passwords submitted to any endpoint SHALL be truncated (server-side) to 128 characters before any processing. This prevents Denial of Service via hash computation of extremely long passwords.

**Rationale:** Argon2id computation time scales with input length. A 10,000-character password could take orders of magnitude longer to hash.

## BR-002: Identifier Normalization

All submitted `identifier` values SHALL be:
1. Trimmed of leading and trailing whitespace.
2. Converted to lowercase.
3. Not normalized further (no Unicode normalization, no accent removal) — exact match after trim + lowercase.

## BR-003: MFA Requirement Evaluation

MFA is required for a login attempt if ANY of the following conditions are true:

| Condition | Source |
|---|---|
| Corporate policy requires MFA for all users of this corporate | Corporate Context policy cache |
| The `client_id` configuration requires MFA (e.g., Bank Admin Portal) | Authentication Module config |
| The user has voluntarily enrolled MFA and it is active | Checked against `mfa_factors` |
| The risk score for this login attempt is >= 31 (medium) | Risk evaluation engine |
| The device is unrecognized (new device fingerprint) | Device lookup result |

If MFA is required and the user has NO active MFA factor:
- If the MFA requirement is due to corporate/client policy (mandatory): redirect user to MFA enrollment (cannot bypass).
- If the MFA requirement is due to voluntary enrollment (user has MFA): this case is logically impossible — if enrolled, a factor exists.
- If the MFA requirement is due to risk score: proceed without MFA but flag session with `elevated_risk = true`; downstream services can choose to reject or require step-up.

## BR-004: Single-Use Token Guarantee

All single-use tokens (reset tokens, OTP codes, FIDO2 challenges, MFA session tokens, recovery codes) SHALL be invalidated immediately upon first use, regardless of any error occurring after the use. If a downstream step fails after the token is consumed, the token remains consumed. The user must request a new token.

**Exception:** The Refresh Token grace window (FR-TOKEN-005) is the only deliberate deviation from this rule, limited to a 5-second window for network error resilience.

## BR-005: Account Lock Does Not Expire Credentials

Locking an account does not reset or extend the password expiry. Both mechanisms operate independently.

## BR-006: Session Revocation Propagates to All Tokens

When a session is revoked (for any reason), ALL refresh token families and ALL refresh tokens associated with that session SHALL be revoked. There is no partial revocation.

## BR-007: Access Token Revocation Is Best-Effort

Due to the stateless nature of JWT Access Tokens, revocation is implemented via a Redis blacklist. This means:
- Resource servers using local JWT verification may not see the revocation until the next JWKS cache refresh.
- Resource servers using token introspection will see the revocation immediately.

For high-security downstream services (e.g., payment approval), token introspection SHALL be used, not local verification.

## BR-008: No Authentication Module Awareness of Authorization

The Authentication Module SHALL NOT store, evaluate, or return roles, permissions, or product entitlements. Access Tokens SHALL NOT contain role/permission claims. Downstream services perform authorization using `identity_id` and `corporate_id` from the Access Token to query their own permission stores.

## BR-009: Administrative Actions Always Audit

Every administrative action (force logout, account unlock, admin password reset, MFA bypass) SHALL emit a domain event AND generate a `LoginAttempt`-equivalent audit record, regardless of whether the action succeeds or fails.

## BR-010: Self-Service Cannot Delete Own Account Lock Permanently

A user can only self-service recover from a `LOCKED_TEMPORARY` lockout by waiting for the lockout timer to expire. A `LOCKED_PERMANENT` lockout can ONLY be lifted by a Bank Administrator.

## BR-011: Force Password Change Cannot Be Bypassed

If `force_password_change = TRUE` on a credential, the forced change session token issued at login is valid ONLY for the password change endpoint. Attempting to use it on any other endpoint SHALL return 403 Forbidden.

## BR-012: Device Revocation is Irreversible

Once a device's `trust_status` is set to `REVOKED`, it cannot be re-registered using the same fingerprint. The user must use a new device or the same physical device will generate a new fingerprint (which will be registered as a new device). This is intentional — a revoked device is considered compromised.

---

---

# 12. Input Validation Rules

## 12.1 Global Input Rules

| Rule | Details |
|---|---|
| Content-Type | All POST/PATCH/PUT requests MUST include `Content-Type: application/json`. Requests without this header SHALL be rejected with 415 Unsupported Media Type. |
| Request body size | Maximum 64KB. Requests exceeding this SHALL be rejected with 413 Payload Too Large. |
| Character encoding | All inputs MUST be valid UTF-8. Invalid UTF-8 sequences SHALL be rejected with 400. |
| Null bytes | Null bytes (0x00) in any string input SHALL be rejected with 400. |
| SQL/NoSQL injection | Not applicable — all database queries use parameterized statements. This is an implementation constraint, not a validation rule. |

## 12.2 Field-Level Validation

### identifier (login)
- Type: string
- Min length: 1 character (after trim)
- Max length: 256 characters (after trim)
- Allowed characters: printable UTF-8 (no control characters)
- Trim: leading and trailing whitespace

### password (login/change/reset)
- Type: string
- Min length: 1 character (must be non-empty to attempt verification)
- Max length: 128 characters (server truncates at 128 before processing)
- Allowed characters: any printable UTF-8

### new_password (change/reset — applies BR-001 + FR-PWD-001)
- Type: string
- Min length: policy.min_length (default: 12)
- Max length: 128
- Character requirements: per FR-PWD-001

### mfa_code (TOTP / SMS OTP)
- Type: string
- Pattern: 6 digits only (`^[0-9]{6}$`)
- Must not be empty

### recovery_code
- Type: string
- Pattern: `^[A-Z2-9]{4}-[A-Z2-9]{4}$` (with hyphen) or `^[A-Z2-9]{8}$` (without)
- Case-insensitive (normalize to uppercase before comparison)

### client_id
- Type: string
- Max length: 128 characters
- Allowed characters: alphanumeric, hyphen, underscore
- Must match a registered OAuth2 client

### device_fingerprint fields
- All fields: optional
- user_agent: max 512 characters
- screen_resolution: pattern `^[0-9]{1,5}x[0-9]{1,5}$` or null
- timezone: must be a valid IANA timezone name or UTC offset (`±HH:MM`)
- All other string fields: max 128 characters, printable ASCII only

---

---

# 13. Error Catalog

All errors follow RFC 7807 format with a machine-readable `type` URI, `title`, `status`, and `detail`.

## 13.1 Login Errors

| Code | HTTP Status | Type URI Suffix | Detail |
|---|---|---|---|
| ERR-LOGIN-001 | 401 | `/errors/invalid-credentials` | The provided credentials are incorrect. |
| ERR-LOGIN-002 | 401 | `/errors/identity-inactive` | The account is not available. (Same as ERR-LOGIN-001 — response must be indistinguishable) |
| ERR-LOGIN-003 | 423 | `/errors/account-locked` | Account is locked. Includes `locked_until` for temporary locks. |
| ERR-LOGIN-004 | 401 | `/errors/mfa-session-invalid` | MFA session token is invalid or expired. |
| ERR-LOGIN-005 | 403 | `/errors/password-change-required` | Password change is required before access is granted. |

## 13.2 MFA Errors

| Code | HTTP Status | Type URI Suffix | Detail |
|---|---|---|---|
| ERR-MFA-001 | 401 | `/errors/invalid-mfa-code` | The submitted MFA code is incorrect. |
| ERR-MFA-002 | 401 | `/errors/mfa-code-already-used` | This MFA code has already been used. |
| ERR-MFA-003 | 409 | `/errors/mfa-factor-not-found` | The specified MFA factor does not exist or is not active. |
| ERR-MFA-004 | 401 | `/errors/mfa-session-expired` | MFA session has expired. Please restart the login process. |
| ERR-MFA-005 | 422 | `/errors/otp-not-requested` | No OTP has been requested or the OTP has expired. |
| ERR-MFA-006 | 429 | `/errors/otp-max-attempts` | Too many incorrect OTP attempts. Please request a new OTP. |
| ERR-MFA-007 | 401 | `/errors/fido2-counter-invalid` | Authenticator counter is invalid. Device may be cloned. |
| ERR-MFA-008 | 429 | `/errors/otp-rate-limited` | Too many OTP requests. Please wait before requesting another. Includes `retry_after`. |
| ERR-MFA-009 | 409 | `/errors/factor-already-enrolled` | An MFA factor of this type is already active. |
| ERR-MFA-010 | 409 | `/errors/last-mandatory-factor` | Cannot disable the last active MFA factor when MFA is mandatory. |

## 13.3 Enrollment Errors

| Code | HTTP Status | Type URI Suffix | Detail |
|---|---|---|---|
| ERR-MFA-ENROLL-001 | 410 | `/errors/enrollment-expired` | The enrollment session has expired. Please restart enrollment. |
| ERR-MFA-ENROLL-002 | 429 | `/errors/enrollment-max-attempts` | Too many incorrect confirmation codes. Please restart enrollment. |

## 13.4 Token Errors

| Code | HTTP Status | Type URI Suffix | Detail |
|---|---|---|---|
| ERR-TOKEN-001 | 401 | `/errors/invalid-refresh-token` | The refresh token is invalid. |
| ERR-TOKEN-002 | 401 | `/errors/refresh-token-expired` | The refresh token has expired or been revoked. |
| ERR-TOKEN-003 | 401 | `/errors/token-reuse-detected` | A security anomaly was detected. All sessions have been revoked. |
| ERR-TOKEN-004 | 401 | `/errors/access-token-revoked` | The access token has been revoked. |
| ERR-TOKEN-005 | 400 | `/errors/invalid-token-format` | The submitted token format is invalid. |

## 13.5 Session Errors

| Code | HTTP Status | Type URI Suffix | Detail |
|---|---|---|---|
| ERR-SESSION-001 | 404 | `/errors/session-not-found` | The specified session does not exist. |
| ERR-SESSION-002 | 401 | `/errors/session-expired` | The session has expired. Please log in again. |
| ERR-SESSION-003 | 422 | `/errors/cannot-revoke-current-session` | Use POST /auth/logout to end the current session. |

## 13.6 Password Errors

| Code | HTTP Status | Type URI Suffix | Detail |
|---|---|---|---|
| ERR-PWD-001 | 422 | `/errors/password-too-long` | Password must not exceed 128 characters. |
| ERR-PWD-002 | 422 | `/errors/password-too-short` | Password must be at least {min_length} characters. |
| ERR-PWD-003 | 422 | `/errors/password-no-uppercase` | Password must contain at least one uppercase letter. |
| ERR-PWD-004 | 422 | `/errors/password-no-lowercase` | Password must contain at least one lowercase letter. |
| ERR-PWD-005 | 422 | `/errors/password-no-digit` | Password must contain at least one digit. |
| ERR-PWD-006 | 422 | `/errors/password-no-special` | Password must contain at least one special character. |
| ERR-PWD-007 | 422 | `/errors/password-contains-username` | Password must not contain your username. |
| ERR-PWD-008 | 422 | `/errors/password-too-common` | This password is too common. Please choose a more unique password. |
| ERR-PWD-009 | 422 | `/errors/password-previously-used` | This password has been used recently. Please choose a different password. |
| ERR-PWD-010 | 401 | `/errors/current-password-incorrect` | The current password is incorrect. |
| ERR-PWD-011 | 400 | `/errors/invalid-reset-token` | The password reset token is invalid or has expired. |
| ERR-PWD-012 | 410 | `/errors/reset-token-used` | This password reset token has already been used. |
| ERR-PWD-013 | 410 | `/errors/reset-token-expired` | This password reset token has expired. Please request a new one. |

## 13.7 Rate Limiting Errors

| Code | HTTP Status | Type URI Suffix | Detail |
|---|---|---|---|
| ERR-RATE-001 | 429 | `/errors/rate-limited` | Too many requests. Includes `retry_after` (seconds). |

## 13.8 System Errors

| Code | HTTP Status | Type URI Suffix | Detail |
|---|---|---|---|
| ERR-SYS-001 | 503 | `/errors/service-unavailable` | The authentication service is temporarily unavailable. |
| ERR-SYS-002 | 503 | `/errors/dependency-unavailable` | A required upstream service is unavailable. |
| ERR-SYS-003 | 500 | `/errors/internal-error` | An internal error occurred. Includes `trace_id` for support. |

---

---

# 14. Notification Triggers

The Authentication Module emits events that the Notification Context subscribes to. The following table specifies the event, the notification trigger, and the expected channel.

| Event | Trigger Condition | Notification Channel | Priority |
|---|---|---|---|
| `AccountLocked` | Any lockout | Email + SMS (if available) | High |
| `DeviceRegistered` (was_automatic = true) | New unrecognized device | Email | High |
| `PasswordResetRequested` | User initiated forgot password | Email | High |
| `PasswordResetCompleted` | Reset completed | Email | Medium |
| `PasswordChanged` | User changed password | Email | Medium |
| `PasswordExpiryWarning` | 14, 7, 3, 1 day before expiry | Email | Low |
| `MFAEnabled` | Factor enrolled | Email | Medium |
| `MFADisabled` | Factor disabled/revoked | Email | High |
| `MFARecoveryCodeUsed` | Recovery code consumed | Email | High |
| `DeviceRevoked` | User or admin revoked device | Email | High |
| `RefreshTokenFamilyCompromised` | Token reuse detected | Email + SMS | Critical |
| `AdminForceLogout` | Admin force-logged out identity | Email | High |
| `MFAAdminBypassGenerated` | Admin issued MFA bypass | Email | Critical |
| `SessionRevoked` (reason = SECURITY_EVENT) | Security-triggered revocation | Email + SMS | Critical |

**Note:** The Notification Context is responsible for: template selection, localization, channel preference lookup (from Identity Context), delivery, and retry. Authentication Module only emits the event trigger.

---

---

# 15. Audit Requirements

## 15.1 Audit Coverage

The following events SHALL be written to the immutable audit trail. The audit trail is implemented via:
1. `login_attempts` table (primary store for login-specific events)
2. `outbox_events` table → Kafka → Audit Context (all domain events)

| Event | Audit Data |
|---|---|
| Every login attempt (success/failure) | identity_id, outcome, reason, ip, device, timestamp |
| Every MFA verification attempt | identity_id, factor_type, outcome, ip, timestamp |
| Every session creation | session_id, identity_id, device_id, aal, ip, timestamp |
| Every session revocation | session_id, revoke_reason, revoked_by, timestamp |
| Every token refresh | family_id, new_token_id, ip, timestamp |
| Every token family compromise | family_id, reused_token_id, suspicious_ip, timestamp |
| Every password change | identity_id, change_type, changed_by, timestamp |
| Every MFA enrollment/revocation | identity_id, factor_type, action, performed_by, timestamp |
| Every device action | device_id, action, performed_by, ip, timestamp |
| Every admin action | target_identity_id, action, admin_identity_id, timestamp |
| Every account lockout/unlock | identity_id, lockout_type, actor, timestamp |

## 15.2 Audit Record Immutability

- Audit records in the `login_attempts` table are INSERT-only. No UPDATE or DELETE operations are permitted by any application role.
- Database roles used by the application SHALL NOT have UPDATE or DELETE privileges on `login_attempts`.
- All audit domain events published to Kafka are written to a topic with retention policy = `delete` with `retention.ms = 365 days * 86400000` and `cleanup.policy = delete` (no compaction, no deletion before retention).

## 15.3 Audit Trail Integrity

The Audit Context SHALL implement hash chaining on received events:
- Each audit record includes a `previous_record_hash` field = SHA-256 of the previous record's content.
- Tampering with any record breaks the hash chain from that point forward.
- The Authentication Module is NOT responsible for hash chaining — that is the Audit Context's responsibility. Authentication Module publishes events with complete payload.

---

---

# 16. Configuration Parameters

All configuration parameters SHALL be externalized (not hardcoded). Configuration is loaded at startup from a configuration service or environment variables. The following parameters MUST be configurable:

## 16.1 Credential & Lockout Configuration

| Parameter | Default | Description |
|---|---|---|
| `auth.lockout.threshold` | 5 | Failed attempts before lockout |
| `auth.lockout.window_minutes` | 15 | Time window for failed attempt counting |
| `auth.lockout.tier1_duration_minutes` | 30 | Duration for first temporary lockout |
| `auth.lockout.tier2_duration_minutes` | 120 | Duration for second temporary lockout |
| `auth.argon2id.time_cost` | 3 | Argon2id time cost parameter |
| `auth.argon2id.memory_kb` | 65536 | Argon2id memory cost (64MB) |
| `auth.argon2id.parallelism` | 4 | Argon2id parallelism |

## 16.2 Session Configuration

| Parameter | Default | Description |
|---|---|---|
| `auth.session.idle_timeout_secs.web` | 900 | 15-minute idle timeout for web |
| `auth.session.idle_timeout_secs.mobile` | 1800 | 30-minute idle timeout for mobile |
| `auth.session.absolute_duration_hours.web` | 8 | 8-hour absolute session for web |
| `auth.session.absolute_duration_hours.mobile` | 24 | 24-hour absolute session for mobile |
| `auth.session.concurrent_policy.default` | `LIMIT_N` | Default concurrent session policy |
| `auth.session.concurrent_max.default` | 5 | Maximum concurrent sessions |
| `auth.mfa.session_window_secs` | 600 | MFA challenge session window |

## 16.3 Token Configuration

| Parameter | Default | Description |
|---|---|---|
| `auth.token.access_token_ttl_secs` | 900 | 15-minute access token lifetime |
| `auth.token.refresh_token_ttl_secs.web` | 86400 | 24-hour refresh token for web |
| `auth.token.refresh_token_ttl_secs.mobile` | 604800 | 7-day refresh token for mobile |
| `auth.token.issuer` | `https://auth.bank.com` | JWT `iss` claim value |
| `auth.token.rotation_grace_window_secs` | 5 | Refresh token rotation grace window |

## 16.4 Password Policy Configuration

| Parameter | Default | Description |
|---|---|---|
| `auth.password.min_length` | 12 | Minimum password length |
| `auth.password.max_length` | 128 | Maximum password length |
| `auth.password.history_depth` | 12 | Password history depth |
| `auth.password.max_age_days.admin` | 90 | Bank admin password max age |
| `auth.password.max_age_days.corporate` | 180 | Corporate user password max age |
| `auth.password.expiry_warning_days` | `[14,7,3,1]` | Warning days before expiry |

## 16.5 MFA Configuration

| Parameter | Default | Description |
|---|---|---|
| `auth.mfa.otp_ttl_secs` | 300 | OTP validity window |
| `auth.mfa.otp_max_attempts` | 3 | Max OTP verification attempts |
| `auth.mfa.otp_resend_rate_limit` | 3 | OTP resend limit per window |
| `auth.mfa.otp_resend_window_mins` | 10 | OTP resend rate limit window |
| `auth.mfa.totp_window` | 1 | TOTP time window tolerance (±N periods) |
| `auth.mfa.enrollment_window_secs` | 300 | TOTP enrollment confirmation window |
| `auth.mfa.recovery_code_count` | 10 | Number of recovery codes generated |
| `auth.mfa.recovery_code_low_threshold` | 3 | Threshold for low recovery code warning |

## 16.6 Device Configuration

| Parameter | Default | Description |
|---|---|---|
| `auth.device.trust_duration_days` | 30 | Default trusted device duration |
| `auth.device.fingerprint_version` | 1 | Current fingerprint algorithm version |

## 16.7 FIDO2 Configuration

| Parameter | Default | Description |
|---|---|---|
| `auth.fido2.rp_id` | `bank.com` | Relying Party ID (domain) |
| `auth.fido2.rp_name` | `Corporate Bank` | Relying Party display name |
| `auth.fido2.rp_origin` | `https://portal.bank.com` | Expected origin |
| `auth.fido2.user_verification` | `preferred` | `required`, `preferred`, or `discouraged` |
| `auth.fido2.attestation` | `indirect` | Attestation conveyance preference |
| `auth.fido2.challenge_ttl_secs` | 300 | FIDO2 challenge validity window |

---

---

**Document Control**

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0.0 | 2026-06-30 | Platform Architecture Team | Initial draft, derived from Authentication Module PRD v1.0.0 |

**Review Required From:** Engineering Lead (Backend), QA Lead, Security Engineer, API Design Team

**Approval Required From:** Chief Architect, Engineering Manager — Authentication Platform

**Traceability:** Every `FR-*` requirement in this document maps to one or more `FR-*` items in the PRD. QA engineers should cross-reference test cases against this document's `FR-*` identifiers.
