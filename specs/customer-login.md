# Spec: Customer Login (Customer ID + Password)

- **Status:** Draft
- **Feature:** Authentication
- **Owner:** TBD

## 1. User Story

As a **customer of XYZ Bank**, I want to **log in using my Customer ID and password**, so that I can **securely access my accounts and banking services**.

## 2. Goals / Non-Goals

**Goals**
- Authenticate a registered customer with Customer ID + password.
- Establish an authenticated session on success.
- Protect against brute-force and credential-guessing attacks.

**Non-Goals (this spec)**
- Registration / onboarding
- Forgot Customer ID / password reset
- Multi-factor authentication (see Open Questions)
- Social / biometric login

## 3. Actors & Preconditions

- **Actor:** Registered retail customer with an active account.
- **Preconditions:** Customer has a Customer ID and password; the login page is reachable over HTTPS.

## 4. Functional Requirements

| ID | Requirement |
|----|-------------|
| FR-1 | The login form shall have a Customer ID field, a password field (masked), and a Login button. |
| FR-2 | Both fields are mandatory; Login is rejected client- and server-side if either is empty. |
| FR-3 | On valid credentials, the system creates a session and redirects the customer to the dashboard. |
| FR-4 | On invalid credentials, the system shows a single generic error ("Invalid Customer ID or password") that does not reveal which field was wrong. |
| FR-5 | After N consecutive failed attempts (N = 5, configurable), the account is temporarily locked for M minutes (M = 30, configurable). |
| FR-6 | A locked account shall be told it is locked, regardless of whether the password entered is correct. |
| FR-7 | Disabled/closed accounts cannot log in and receive the generic error. |
| FR-8 | A successful login resets the failed-attempt counter. |
| FR-9 | The session expires after a period of inactivity (default 10 min, configurable) and on explicit logout. |
| FR-10 | Every login attempt (success/failure/lockout) is written to an audit log. |

## 5. Non-Functional Requirements

- **Security**
  - All traffic over TLS; passwords never logged or returned in responses.
  - Passwords stored only as salted hashes using a modern adaptive algorithm (e.g. Argon2id/bcrypt).
  - Session tokens are random, HttpOnly, Secure, SameSite cookies; rotated on login.
  - Login responses have uniform timing/shape to limit user enumeration.
  - Rate limiting per IP and per Customer ID.
  - CSRF protection on the login endpoint.
- **Performance:** p95 login response < 1s under expected load.
- **Accessibility:** WCAG 2.1 AA; form is keyboard-navigable and screen-reader labelled.
- **Availability:** Aligned with overall platform SLA (TBD).

## 6. Acceptance Criteria (Given / When / Then)

**AC-1 Successful login**
- Given a registered, active customer
- When they submit a correct Customer ID and password
- Then a session is created and they land on the dashboard

**AC-2 Wrong password**
- Given a registered customer
- When they submit a correct Customer ID and wrong password
- Then they see "Invalid Customer ID or password" and the failed-attempt counter increments

**AC-3 Unknown Customer ID**
- When a non-existent Customer ID is submitted
- Then the same generic error as AC-2 is shown

**AC-4 Empty fields**
- When either field is empty
- Then validation errors are shown and no authentication request is processed

**AC-5 Account lockout**
- Given 5 consecutive failed attempts
- When the customer attempts again (even with the correct password)
- Then the account is reported locked until the lock period expires

**AC-6 Counter reset**
- Given 3 failed attempts followed by a successful login
- Then the failed-attempt counter is reset to 0

**AC-7 Session timeout**
- Given an authenticated session idle beyond the timeout
- When the customer takes any action
- Then they are redirected to login

**AC-8 Audit**
- For each of AC-1, AC-2, AC-5, an audit record exists with timestamp, Customer ID, outcome, and source IP (no password data).

## 7. API Contract (proposed)

`POST /api/v1/auth/login`

Request
```json
{ "customerId": "string", "password": "string" }
```

Responses
| Status | Meaning | Body |
|--------|---------|------|
| 200 | Authenticated | `{ "redirectTo": "/dashboard" }` + session cookie |
| 400 | Missing/malformed fields | `{ "error": "VALIDATION_ERROR", "fields": [...] }` |
| 401 | Invalid credentials | `{ "error": "INVALID_CREDENTIALS" }` |
| 423 | Account locked | `{ "error": "ACCOUNT_LOCKED", "retryAfterSeconds": n }` |
| 429 | Rate limited | `{ "error": "TOO_MANY_REQUESTS" }` |

## 8. Test Plan

- **Unit:** credential verification, lockout counter logic, input validation, hashing.
- **Integration:** login endpoint against all status codes above; session cookie attributes; audit log writes.
- **E2E:** AC-1 through AC-7 via UI.
- **Security:** brute force/rate limit, user enumeration (response + timing), CSRF, session fixation, SQL injection/XSS on inputs.
- **Accessibility:** automated scan + manual keyboard/screen-reader pass.

## 9. Open Questions

1. Is MFA/OTP required for login (common for banking regulation)?
2. Customer ID format and length? Password policy (length, complexity)?
3. Exact lockout thresholds and unlock method (timed vs. support/reset)?
4. Session timeout value and concurrent-session policy?
5. Applicable regulatory requirements (e.g. PSD2/SCA, RBI, FFIEC) and audit-retention period?
6. Platform scope: web only, or also mobile apps?
7. Tech stack for this repo (needed to define implementation tasks).

## 10. Implementation Tasks (to be refined once stack is chosen)

- [ ] Login UI form + client-side validation
- [ ] `POST /api/v1/auth/login` endpoint
- [ ] Credential store + password hashing
- [ ] Failed-attempt tracking and lockout
- [ ] Session management + timeout
- [ ] Rate limiting and CSRF protection
- [ ] Audit logging
- [ ] Automated tests per Section 8
