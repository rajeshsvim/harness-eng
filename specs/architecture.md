# Architecture: Customer Login

Derived from [customer-login.md](./customer-login.md). Implementation detail lives in [customer-login-design.md](./customer-login-design.md). Technology choices are not yet made, so components are logical.

## 1. System Context

```mermaid
flowchart LR
    C([Customer]) -->|Customer ID + password<br/>HTTPS| UI[Login UI]
    UI -->|POST /api/v1/auth/login| AUTH[Auth Service]
    AUTH -->|session cookie + redirect| UI
    UI -->|on success| DASH[Dashboard]
    AUTH --- DB[(Credential Store)]
    AUTH --- SESS[(Session Store)]
    AUTH --- AUD[(Audit Log)]
```

## 2. Components and Requirements

```mermaid
flowchart TB
    subgraph Client
        UI[Login UI<br/>FR-1, FR-2 validation]
    end
    subgraph Edge
        GW[Gateway / WAF<br/>TLS, per-IP rate limit]
    end
    subgraph Auth Service
        V[Input Validation<br/>FR-2]
        RL[Rate Limiter<br/>NFR security]
        VER[Credential Verifier<br/>FR-3, FR-4, FR-7]
        LK[Lockout Manager<br/>FR-5, FR-6, FR-8]
        SM[Session Manager<br/>FR-3, FR-9]
        AU[Audit Writer<br/>FR-10]
    end
    CS[(Credential Store)]
    SS[(Session Store)]
    AL[(Audit Log)]

    UI --> GW --> V --> RL --> VER
    VER --> CS
    VER --> LK --> CS
    VER --> SM --> SS
    VER --> AU
    LK --> AU
    AU --> AL
```

## 3. Login Flow

```mermaid
flowchart TD
    A[Submit Customer ID + password] --> B{Both fields present?}
    B -- No --> B1[400 VALIDATION_ERROR]
    B -- Yes --> C{Rate limited?}
    C -- Yes --> C1[429 TOO_MANY_REQUESTS]
    C -- No --> D{Account locked?}
    D -- Yes --> D1[423 ACCOUNT_LOCKED]
    D -- No --> E{Active account and<br/>password matches?}
    E -- No --> F[Increment failed attempts]
    F --> G{Attempts >= 5?}
    G -- Yes --> G1[Lock account for 30 min]
    G -- No --> H
    G1 --> H[401 INVALID_CREDENTIALS<br/>generic message]
    E -- Yes --> I[Reset failed attempts]
    I --> J[Create session]
    J --> K[200 + redirect to dashboard]
    B1 & C1 & D1 & H & K --> L[(Write audit record)]
```

## 4. Account Lockout States

```mermaid
stateDiagram-v2
    [*] --> Active
    Active --> Active: failed attempt (count < 5)
    Active --> Locked: 5th consecutive failure
    Locked --> Active: lock period (30 min) expires
    Active --> Active: successful login (count reset)
    Active --> Disabled: account closed or disabled
    Disabled --> [*]
```

Thresholds (5 attempts, 30 min) are placeholders from the spec.

## 5. Session Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Authenticated: successful login (new session ID)
    Authenticated --> Authenticated: activity (idle timer reset)
    Authenticated --> Expired: idle > 10 min
    Authenticated --> Ended: logout
    Expired --> [*]: redirect to login
    Ended --> [*]
```
