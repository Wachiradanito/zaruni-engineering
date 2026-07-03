# Authentication System

Zaruni's auth is built for Kenya: phone-first, not email. Users register with a phone number, verify via OTP, and receive JWT tokens for API access.

---

## Design Decisions

**Phone-first, not email**  
Most users in Kenya don't have or check email regularly. Phone numbers are the universal identifier — the same number used for M-PESA, calls, and SMS.

**OTP for verification**  
SMS OTP is the most friction-free verification method. It works on any phone (smartphone or feature phone), doesn't require app store accounts or Google sign-in, and is familiar to users from mobile banking.

**JWT for sessions**  
JWTs (via `djangorestframework-simplejwt`) avoid session storage in the database or Redis. The token itself carries the claims. No lookup required on every authenticated request.

**Device tracking for fraud prevention**  
New devices require additional verification (OTP + device fingerprint). If a user's credentials are compromised, the attacker's device will trigger verification even with correct password/OTP.

---

## Registration Flow

```
User                    API                      Database
 │                       │                           │
 │ POST /auth/register   │                           │
 │ {phone, name}         │                           │
 │──────────────────────►│                           │
 │                       │  [phone normalization]    │
 │                       │  [check if exists]        │
 │                       │                           │
 │                       │  User.objects.create()    │
 │                       │──────────────────────────►│
 │                       │                           │
 │                       │  PhoneOTP.create_otp()    │
 │                       │  [send SMS]               │
 │                       │◄──────────────────────────│
 │  "OTP sent"           │                           │
 │◄──────────────────────│                           │
 │                       │                           │
 │ POST /auth/verify-otp │                           │
 │ {phone, otp}          │                           │
 │──────────────────────►│                           │
 │                       │  PhoneOTP.verify()        │
 │                       │  [HMAC comparison]        │
 │                       │◄──────────────────────────│
 │                       │  User.phone_verified=True │
 │                       │  DeviceVerification.create│
 │                       │  JWT tokens generated     │
 │  {access, refresh}    │                           │
 │◄──────────────────────│                           │
```

---

## OTP Security

OTPs are never stored in plaintext. When an OTP is generated:

1. A random 6-digit code is created
2. An HMAC-SHA256 hash is computed: `hmac(secret_key, otp + phone + timestamp)`
3. Only the hash is stored in `PhoneOTP.otp_hash`
4. The raw OTP is sent via SMS
5. On verification, the same hash is computed from the user's input and compared

This means a database leak does not reveal OTPs. An attacker would need both the database and the Django secret key to brute-force OTPs offline.

**Rate limiting:** 5 OTP requests per hour per phone number. Enforced at the view level with Django cache.

**Expiry:** OTPs expire after 10 minutes.

---

## JWT Token Lifecycle

After successful OTP verification (or login), the user receives two tokens:

- **Access token** — short-lived (15 minutes), used on every API request
- **Refresh token** — long-lived (7 days), used to obtain new access tokens

When the access token expires, the client sends the refresh token to `/auth/token/refresh/` and receives a new access token. This avoids asking the user to log in again every 15 minutes.

**Token claims include:**
- `user_id` — primary key
- `phone` — for audit logging
- `device_id` — for device verification
- `exp` — expiration timestamp

---

## Device Verification

When a user logs in from a new device:

1. Device fingerprint computed (user agent, IP, device model)
2. `DeviceVerification` checked for this user + fingerprint
3. If not found: **device is unverified** — require additional OTP before issuing tokens
4. If found: **device is verified** — issue tokens immediately

This adds friction only when it matters (new device = potential account takeover). It doesn't affect regular use.

---

## Google OAuth (Optional)

For users who prefer Google sign-in:

1. Client initiates OAuth flow with Google
2. User authorizes in Google's UI
3. Client sends the authorization code to Zaruni's `/auth/google/`
4. Zaruni exchanges code for access token with Google
5. Zaruni fetches user info from Google (`/userinfo`)
6. If phone matches existing user → log them in
7. If no match → create user with `auth_provider = 'google'` and issue tokens

The phone number is still required (requested during OAuth scope) because it's needed for M-PESA payments.

---

## Permission Tiers

Users have a `verification_tier` field that unlocks capabilities:

| Tier | Unlocks |
|------|---------|
| **Basic** | Browse, chat, apply for jobs |
| **Enhanced** | Post jobs, create shop, withdraw up to KES 5,000/day |
| **Full** | Unlimited withdrawals, delivery services, verification badge |

Tier advancement requires identity verification (ID upload + liveness check). This is handled by the `id_verification/` module.

---

## Session Management

**Access token expiry** — 15 minutes  
**Refresh token expiry** — 7 days  
**Logout** — client discards tokens (no server-side session to invalidate)

For admin-forced logout (e.g., account compromised), we maintain a `TokenBlacklist` — checked on every authenticated request if the token is blacklisted. This adds a database lookup but is only used in emergency situations.

---

*← Back to [README](../README.md)*
