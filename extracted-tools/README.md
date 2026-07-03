# Extracted Open-Source Tools

These are generic, reusable packages we've extracted from Zaruni and open-sourced. They contain **zero Zaruni-specific business logic** and can be used in any Django project.

---

## Published Packages

### django-mpesa

M-PESA payment integration for Django applications.

**Features:**
- STK Push (Lipa na M-PESA Online)
- Callback handling with built-in idempotency
- Transaction status queries
- Webhook signature verification
- Test mode for development

**Installation:**
```bash
pip install django-mpesa
```

**Repository:** *Coming soon - extraction in progress*  
**Status:** ✅ Production-ready (used in Zaruni since 2025)

**Why we extracted it:**  
M-PESA integration is needed by many Kenyan apps, but existing libraries had gaps (poor idempotency, missing test support). We built what we needed and open-sourced it.

---

## In Development

### mainauth

Phone-based authentication system with OTP verification for Django.

**Features:**
- Phone number registration
- OTP generation and verification (HMAC-based, no plaintext storage)
- JWT token issuance
- Device verification for new logins
- Rate limiting built-in
- Multi-tenant support

**Status:** 🚧 In active development  
**Expected Release:** Q3 2026  
**Repository:** *Coming soon*

**Why we're extracting it:**  
Most African apps need phone-based auth, not email. This is a batteries-included solution with security best practices.

---

## Planned Extractions

### django-notify

Multi-channel notification system.

**Features:**
- Push notifications (FCM)
- SMS notifications (Twilio, Africa's Talking)
- Email notifications
- In-app notifications
- Notification templates
- Delivery tracking and retry logic

**Status:** 📋 Planned for extraction  
**Expected Start:** Q4 2026

---

## Extraction Philosophy

We extract components that meet these criteria:

1. **Zero business logic** — No Zaruni-specific logic, only generic functionality
2. **Production-tested** — Battle-tested in real usage, not experimental
3. **Genuinely reusable** — Useful in other projects without modification
4. **Well-documented** — README, examples, API docs
5. **Maintainable** — Clear code, tests, contribution guidelines

**What we DON'T extract:**
- Domain-specific business logic (escrow rules, matching algorithms)
- Zaruni-specific workflows or state machines
- Experimental or unfinished code

---

## Using These Tools

Each package has its own repository with:
- 📖 Full documentation and examples
- ✅ Test suite
- 📦 PyPI distribution (when published)
- 🤝 Contribution guidelines
- ⚖️ Separate license (usually MIT)

---

## Future Candidates

Components we're considering for extraction:

- **Generic escrow state machine** — Finite state machine for payment holds
- **Event bus pattern** — Async event emission and subscription
- **Idempotency helpers** — Request deduplication utilities
- **QR/OTP proof system** — Proof-of-completion with time-bound tokens

These are still coupled to Zaruni's domain logic but could be genericized with effort.

---

## Contributing

We're not accepting code contributions to this documentation repo, but:

- **Bug reports welcome** — If you find issues in extracted packages, open issues in their repos
- **Feature requests welcome** — Suggest features for extracted packages
- **Documentation improvements** — PRs accepted for typos or clarity

See individual package repositories for contribution guidelines.

---

**Last Updated:** July 2026
