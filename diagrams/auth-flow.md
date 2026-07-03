# Authentication Flow Diagrams

> Registration, login, JWT lifecycle, and device verification.

---

## Registration & OTP Verification

```mermaid
sequenceDiagram
    actor User
    participant App as Mobile App
    participant API as Zaruni API
    participant DB as Database
    participant SMS as SMS Provider

    User->>App: Enter phone number
    App->>API: POST /auth/register {phone, name}

    API->>DB: Check phone not already registered
    API->>DB: User.objects.create(phone, tier=bronze)
    API->>DB: PhoneOTP.create_otp()\n[HMAC hash stored, raw OTP returned]
    API->>SMS: Send OTP to phone number
    SMS-->>User: SMS with 6-digit code
    API-->>App: "OTP sent"

    User->>App: Enter OTP code
    App->>API: POST /auth/verify-otp {phone, otp}

    API->>DB: PhoneOTP.verify()\n[HMAC comparison — no plaintext lookup]

    alt OTP valid
        API->>DB: user.is_phone_verified = True
        API->>DB: DeviceVerification.create(device_fingerprint)
        API-->>App: {access_token, refresh_token}
        App-->>User: Logged in ✓
    else OTP invalid or expired
        API-->>App: 400 Invalid OTP
        App-->>User: "Incorrect code, try again"
    end
```

---

## Login Flow (Returning User)

```mermaid
sequenceDiagram
    actor User
    participant App as Mobile App
    participant API as Zaruni API
    participant DB as Database

    User->>App: Enter phone number
    App->>API: POST /auth/request-otp {phone}
    API->>DB: PhoneOTP.create_otp()
    API--)User: SMS with OTP

    User->>App: Enter OTP
    App->>API: POST /auth/verify-otp {phone, otp}
    API->>DB: Verify OTP (HMAC)

    API->>DB: Check DeviceVerification(user, device_fingerprint)

    alt Known device
        API-->>App: {access_token, refresh_token}
        App-->>User: Logged in ✓
    else New / unrecognised device
        API-->>App: 200 device_verification_required
        App-->>User: "Verify this new device"
        Note over User,DB: Additional OTP required for new device
        API->>DB: DeviceVerification.create() on success
        API-->>App: {access_token, refresh_token}
    end
```

---

## JWT Token Lifecycle

```mermaid
stateDiagram-v2
    [*] --> HasTokens : Successful OTP verification

    state HasTokens {
        AccessToken : Access Token\n(15 min lifetime)
        RefreshToken : Refresh Token\n(7 day lifetime)
    }

    HasTokens --> MakingRequest : API call with access token

    MakingRequest --> Success : Token valid → 200 OK
    MakingRequest --> RefreshNeeded : Token expired → 401

    RefreshNeeded --> HasTokens : POST /auth/token/refresh\n{refresh_token} → new access token
    RefreshNeeded --> LoggedOut : Refresh token also expired\nUser must log in again

    Success --> MakingRequest : Continue making requests

    LoggedOut --> [*] : User re-authenticates
```

---

## Verification Tier Progression

```mermaid
flowchart TD
    REG["📱 Register with phone\nOTP verified"]

    BRONZE["🥉 Bronze\nDefault tier"]
    SILVER["🥈 Silver\nLight verification"]
    GOLD["🥇 Gold\nFull ID verification"]

    REG --> BRONZE

    BRONZE -->|"Complete profile\n+ profile photo"| SILVER
    SILVER -->|"Government ID\n+ liveness check"| GOLD

    subgraph BronzeFeatures["Bronze unlocks"]
        B1["Browse & search"]
        B2["Send messages"]
        B3["Buy products"]
        B4["Apply for jobs"]
        B5["Receive payments"]
        B6["Withdraw (daily limit)"]
    end

    subgraph SilverFeatures["Silver adds"]
        S1["Create & manage shop"]
        S2["Post job listings"]
        S3["Provide services"]
        S4["Higher withdrawal limit"]
        S5["Reputation score visible"]
    end

    subgraph GoldFeatures["Gold adds"]
        G1["Delivery services (rider)"]
        G2["Unlimited withdrawals"]
        G3["Verified badge on profile"]
        G4["Priority in job matching"]
        G5["Reputation score boost +150"]
    end

    BRONZE --- BronzeFeatures
    SILVER --- SilverFeatures
    GOLD --- GoldFeatures

    style BRONZE fill:#cd7f32,color:#fff
    style SILVER fill:#c0c0c0,color:#000
    style GOLD fill:#ffd700,color:#000
```

---

## OTP Security Model

```mermaid
flowchart LR
    subgraph Generation["OTP Generation"]
        RAW["Random 6-digit OTP"]
        HMAC["HMAC-SHA256\nhmac(secret, otp+phone+ts)"]
        HASH["Hash stored in DB\nRaw OTP sent via SMS"]
        RAW --> HMAC --> HASH
    end

    subgraph Verification["OTP Verification"]
        INPUT["User enters OTP"]
        RECOMPUTE["Recompute HMAC\nfrom user input"]
        COMPARE["Compare hashes\n(constant-time)"]
        INPUT --> RECOMPUTE --> COMPARE
    end

    HASH -.->|"DB leak reveals only hashes\nnot raw OTPs"| COMPARE

    style HASH fill:#f0fff4,stroke:#22c55e
    style COMPARE fill:#f0fff4,stroke:#22c55e
```

No plaintext OTPs ever written to the database. An attacker with database access cannot recover valid OTPs.

---

*Source: [architecture/auth-architecture.md](auth-architecture.md) · [case-studies/2025-verification-tiers.md](../case-studies/2025-verification-tiers.md)*
