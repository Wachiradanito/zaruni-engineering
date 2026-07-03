# Designing the Verification Tier System

**Published:** July 2026  
**Module:** Accounts, Identity  
**Difficulty:** Medium — product design as much as engineering

---

## The Problem

Zaruni is a platform where strangers transact with strangers. A gig worker picks up a job from someone they've never met. A buyer sends money to a seller they found online. A homeowner lets a repair person into their house.

For these transactions to happen, both parties need to trust the platform's word that the other person is who they say they are. That trust is earned in stages — we can't demand a full identity check upfront without killing registration conversion, but we can't let completely unverified users access high-risk features like large withdrawals or delivery services.

The problem: **how do you build a progressive trust system that doesn't punish new users while protecting the platform from fraud?**

---

## Why It Mattered

Two failure modes were unacceptable:

**Failure mode 1: Require verification upfront**  
If registration requires an ID upload and liveness check before doing anything, conversion rates drop sharply. Most users leave before completing. This was empirically confirmed during early testing.

**Failure mode 2: No verification at all**  
Without verification, fraudsters create disposable accounts, complete transactions, withdraw everything, and disappear. Platforms that allow unverified large withdrawals get systematically exploited.

The tension: **friction costs conversions; lack of friction enables fraud.** The system needs to resolve this tension explicitly, not accidentally.

---

## The Investigation

We mapped out what actions on Zaruni carry what level of risk, then grouped them:

**Low risk (anyone can do these):**
- Browse listings
- Send messages
- Apply for jobs
- View prices and reviews

**Medium risk (some verification needed):**
- Post jobs or sell products
- Accept service bookings
- Withdraw small amounts

**High risk (full verification needed):**
- Large withdrawals
- Provide delivery services
- High-value transactions

The insight was that **most users never need high-risk features**, and the users who do need them are willing to verify to get access. So we weren't punishing most users with friction — we were only adding friction for the users whose actions warranted it.

---

## The Solution

Three tiers, each unlocking a progressively larger set of capabilities:

### Tier 1: Bronze (Default)

Every account starts here. Awarded on successful phone OTP verification during registration.

**What it unlocks:**
- Browse, search, message
- Apply for jobs, post service requests
- Purchase products (buying, not selling)
- Receive payments into wallet
- Withdraw up to a daily limit

**Why these restrictions:**  
A phone number proves access to a SIM — it's weak identity but better than nothing. It filters out bots and disposable email accounts, but doesn't verify the person behind the phone.

### Tier 2: Silver

Awarded after completing profile + light identity verification (profile photo + basic personal details). Reviewed and approved by platform or automated checks.

**What it unlocks:**
- Everything in Bronze
- Create and manage a shop (sell products)
- Post job listings
- Provide services
- Higher daily withdrawal limit
- Reputation score visible to others

**Why here:**  
Selling and job posting are medium-risk — a fraudulent seller can scam buyers, a job poster can waste workers' time. Requiring a photo and some personal details creates accountability without the friction of a full ID check.

### Tier 3: Gold

Awarded after full identity verification: government-issued ID (National ID or Passport) + liveness check (match photo to ID). Reviewed by staff if automated checks are inconclusive.

**What it unlocks:**
- Everything in Silver
- Delivery services (rider account)
- Unlimited withdrawals
- Verified badge visible on profile
- Priority in job matching
- Reputation score boost

**Why here:**  
Delivery services involve picking up and handling goods in person — higher real-world risk. Unlimited withdrawals remove the last fraud backstop — only appropriate for fully verified users.

---

## The Technical Implementation

### User Model

The tier is a simple field on the `User` model:

```python
class User(models.Model):
    verification_tier = models.CharField(
        max_length=10,
        choices=[
            ('bronze', 'Bronze'),
            ('silver', 'Silver'),
            ('gold', 'Gold'),
        ],
        default='bronze',
    )
```

### Capability Checks

Features gate on tier at the view layer, not scattered through business logic:

```python
class WithdrawalView(APIView):
    def post(self, request):
        tier = request.user.verification_tier
        limit = get_withdrawal_limit_for_tier(tier)
        
        if requested_amount > limit:
            raise PermissionDenied(
                f"Withdrawals above {limit} require Silver verification."
            )
```

The tier check is declarative — the view knows what it needs, it doesn't contain the verification logic.

### Reputation Integration

Verification tier feeds directly into reputation score. A Bronze user starts with a baseline. Silver adds a bonus. Gold adds a larger bonus. This means verified users naturally surface higher in search and job matching — an incentive to verify without making it mandatory.

```python
tier_bonuses = {
    'bronze': 50,
    'silver': 150,
    'gold': 300,
}

reputation_score = base_score + tier_bonuses.get(user.verification_tier, 0)
```

### Tier Advancement Flow

When a user submits verification documents:

1. `id_verification` module receives submission
2. Automated checks run (face match, document validity)
3. On auto-pass: `User.verification_tier` updated, event emitted
4. On inconclusive: queued for staff review
5. On approval: tier updated, user notified
6. On rejection: user told what to resubmit

The `accounts` module doesn't handle document review — it listens to events from `id_verification` and updates the tier field when told to.

### Tier Change Events

When a tier changes, the event bus notifies downstream modules:

```python
# signal fires on tier change
emit_event('accounts.verification_tier.changed', {
    'user_id': str(user.id),
    'old_tier': old_tier,
    'new_tier': new_tier,
})
```

Subscribers: wallet (update limits), notifications (tell the user), reputation (recompute score).

---

## The Outcome

The tiered system solved both failure modes:

**Conversion:** Bronze users can do everything they need to get started — browse, message, buy. No upfront friction. The verification prompt only appears when they try to access a feature that requires it.

**Fraud prevention:** Sellers, job posters, and delivery workers all require at least Silver. Large withdrawals require Gold. Fraudsters can't silently extract money through a fresh account — the withdrawal limits stop them while giving legitimate users a clear path to higher limits.

**Incentive alignment:** Higher tiers unlock better features *and* reputation boosts. Verification isn't just a gate — it's a benefit.

The hardest part wasn't the implementation — the tiers themselves are a field on a model. The hard part was agreeing on which capabilities belong at which tier. Getting that mapping right required multiple iterations based on early fraud incidents and user interviews. The engineering was easy; the product design took time.

---

*← Back to [case studies](../README.md#-case-studies)*
