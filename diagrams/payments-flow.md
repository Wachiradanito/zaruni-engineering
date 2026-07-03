# Payments Flow Diagrams

> STK Push flow, escrow lifecycle, and wallet ledger.

---

## M-PESA STK Push — Full Payment Flow

```mermaid
sequenceDiagram
    actor User
    participant App as Mobile App
    participant API as Zaruni API
    participant DB as Database
    participant Safaricom

    User->>App: Tap "Pay with M-PESA"
    App->>API: POST /checkout {amount, phone}
    API->>DB: Create PaymentIntent (pending)
    API->>Safaricom: STK Push Request
    Safaricom-->>API: CheckoutRequestID
    API-->>App: "Check your phone"
    App-->>User: "Enter your M-PESA PIN"

    Note over User,Safaricom: User enters PIN on their phone

    Safaricom->>API: POST /payments/webhook/mpesa/
    Note over API,DB: Idempotent processing begins

    API->>DB: get_or_create(provider, event_id)
    Note over API,DB: Unique constraint prevents duplicates

    API->>DB: select_for_update(skip_locked=True)
    Note over API,DB: Row lock — only one worker proceeds

    API->>DB: Mark Intent confirmed
    API->>DB: Credit wallet (idempotency_key)
    API->>DB: Fund escrow (if applicable)
    API->>DB: Mark webhook processed

    API--)App: Push notification "Payment confirmed ✓"
    App-->>User: "Payment successful"
```

---

## Escrow Lifecycle

```mermaid
stateDiagram-v2
    [*] --> pending : create_escrow()

    pending --> funded : fund_escrow()\n[buyer pays]
    pending --> expired : expire_stale_escrows()\n[Celery Beat task]

    funded --> released : release_escrow()\n[both parties confirm]
    funded --> refunded : refund_escrow()\n[dispute resolved for buyer]
    funded --> disputed : mark_escrow_disputed()\n[party raises dispute]

    disputed --> released : resolve_dispute(winner=seller)
    disputed --> refunded : resolve_dispute(winner=buyer)

    released --> [*]
    refunded --> [*]
    expired --> [*]

    note right of funded
        select_for_update() on all transitions
        Idempotency keys on release + refund
        Terminal states have no outgoing transitions
    end note
```

---

## Wallet Ledger — Double-Entry Flow

```mermaid
flowchart LR
    subgraph Buyer["Buyer Wallet"]
        BW["balance: 1000"]
    end

    subgraph Escrow["Escrow (held by platform)"]
        EW["amount: 500\nstatus: funded"]
    end

    subgraph Seller["Seller Wallet"]
        SW["balance: 200"]
    end

    subgraph Ledger["LedgerEntry (append-only log)"]
        L1["debit: buyer -500\nidempotency_key: fund_escrow_XYZ"]
        L2["credit: seller +500\nidempotency_key: release_escrow_XYZ"]
    end

    BW -->|"fund_escrow()\nselect_for_update()"| EW
    EW -->|"release_escrow()\nidempotency check"| SW
    BW -.->|"recorded in"| L1
    SW -.->|"recorded in"| L2
```

---

## Payment Integration Architecture

```mermaid
classDiagram
    class PaymentIntegrationAdapter {
        <<interface>>
        +initiate(intent, payload) PaymentInitiationResult
        +verify_webhook(payload, headers) bool
        +process_webhook(payload) PaymentWebhookResult
        +get_status(intent) PaymentStatusResult
        +refund(intent, amount) dict
    }

    class MpesaIntegration {
        +integration_code = "mpesa"
        +initiate()
        +verify_webhook()
        +process_webhook()
        +get_status()
        +refund()
    }

    class StripeIntegration {
        +integration_code = "stripe"
        +initiate()
        +verify_webhook()
        +process_webhook()
        +get_status()
        +refund()
    }

    class MockIntegration {
        +integration_code = "mock"
        +Used in tests only
    }

    PaymentIntegrationAdapter <|-- MpesaIntegration
    PaymentIntegrationAdapter <|-- StripeIntegration
    PaymentIntegrationAdapter <|-- MockIntegration
```

Adding a new payment method = implementing this interface. Core payment flow unchanged.

---

*Source: [architecture/payments-architecture.md](payments-architecture.md) · [case-studies/2025-mpesa-idempotency.md](../case-studies/2025-mpesa-idempotency.md) · [case-studies/2025-escrow-design.md](../case-studies/2025-escrow-design.md)*
