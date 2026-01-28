# 📘 E-TOUR System Documentation

## PART 9: PAYMENT & TRANSACTION HANDLING

---

# 💳 Why Payment is a Separate Concern

Payment processing is isolated from other modules because:

| Reason | Explanation |
|--------|-------------|
| **Regulatory compliance** | Payment data has legal requirements |
| **Security sensitivity** | PCI-DSS standards for card data |
| **External dependency** | Relies on Razorpay (external service) |
| **Different failure modes** | Network issues, gateway downtime |
| **Audit requirements** | Every transaction must be traceable |
| **Replaceability** | May switch from Razorpay to Stripe |

---

# 🏗️ Generic vs Gateway-Specific Logic

## Architecture Split:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      PAYMENT MODULE SEPARATION                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────┐     ┌─────────────────────────────┐
│  GENERIC PAYMENT            │     │  RAZORPAY-SPECIFIC          │
│  (Gateway-agnostic)         │     │  (Razorpay implementation)  │
├─────────────────────────────┤     ├─────────────────────────────┤
│ PaymentController           │     │ PaymentGatewayController    │
│   /api/payment/*            │     │   /payment-gateway/*        │
│                             │     │                             │
│ PaymentService interface    │     │ PaymentGatewayService interface
│ PaymentServiceImpl          │     │ RazorpayServiceImpl         │
│                             │     │                             │
│ Operations:                 │     │ Operations:                 │
│ • Get payment by ID         │     │ • Create Razorpay order     │
│ • Get receipt               │     │ • Confirm payment           │
│ • Record payment            │     │ • Handle webhook            │
└─────────────────────────────┘     └─────────────────────────────┘
```

## Why This Split Matters:

| Scenario | Benefit |
|----------|---------|
| **Adding Stripe** | Create `StripeServiceImpl`, no controller changes |
| **Generic receipt** | Works regardless of payment gateway |
| **Testing** | Mock gateway service, test generic logic |

---

# 🔄 Transaction Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      PAYMENT STATUS TRANSITIONS                             │
└─────────────────────────────────────────────────────────────────────────────┘

     ┌────────────┐
     │ NO RECORD  │  Initial state - no payment attempted
     └──────┬─────┘
            │
            │ User clicks "Pay Now"
            │ Backend creates Razorpay order
            │ PaymentMaster record created
            │
            ▼
     ┌────────────┐
     │  INITIATED │  Order created, waiting for user to pay
     └──────┬─────┘
            │
     ┌──────┴──────┐
     │             │
     │ User pays   │ User cancels/fails
     │ on gateway  │
     │             │
     ▼             ▼
┌────────────┐  ┌────────────┐
│  SUCCESS   │  │   FAILED   │
│            │  │            │
│ Booking    │  │ Can retry  │
│ confirmed  │  │            │
└────────────┘  └────────────┘
```

## Status Definitions:

| Status | Meaning | Next Actions |
|--------|---------|--------------|
| **INITIATED** | Order created, user on payment page | Wait for confirmation |
| **SUCCESS** | Payment received and verified | Generate receipt |
| **FAILED** | Payment failed or cancelled | Allow retry |

---

# 🔗 Booking–Payment Relationship

```
┌─────────────────┐         ┌─────────────────┐
│ booking_header  │         │ payment_master  │
├─────────────────┤         ├─────────────────┤
│ booking_id: 123 │◄────────│ booking_id: 123 │
│ status: PENDING │         │ status: INITIATED│
│ total: 25000    │         │ amount: 25000   │
└─────────────────┘         │ txn_ref: order_x│
                            └─────────────────┘
        │
        │ On payment SUCCESS
        ▼
┌─────────────────┐         ┌─────────────────┐
│ booking_header  │         │ payment_master  │
├─────────────────┤         ├─────────────────┤
│ booking_id: 123 │◄────────│ booking_id: 123 │
│ status:CONFIRMED│         │ status: SUCCESS │
│ total: 25000    │         │ txn_ref: pay_abc│
└─────────────────┘         └─────────────────┘
```

**Key Point**: Payment SUCCESS triggers booking status update.

---

# 🔔 Webhooks Concept

## The Problem Webhooks Solve:

```
SCENARIO WITHOUT WEBHOOKS:
1. User completes payment on Razorpay popup
2. User's browser crashes before confirm-payment API called
3. Backend never knows payment succeeded
4. Booking stays PENDING forever
5. Customer paid but booking not confirmed! 😱
```

## How Webhooks Fix This:

```
SCENARIO WITH WEBHOOKS:
1. User completes payment on Razorpay
2. Razorpay sends POST to /payment-gateway/webhook
3. Backend receives confirmation directly from Razorpay
4. Payment marked SUCCESS, booking confirmed
5. Even if browser crashed, payment is processed ✅
```

## Webhook Flow:

```
Razorpay Server                          E-TOUR Backend
      │                                        │
      │ POST /payment-gateway/webhook          │
      │ Body: { event, payload }               │
      │ Header: X-Razorpay-Signature           │
      ├───────────────────────────────────────▶│
      │                                        │ Verify signature
      │                                        │ Parse event type
      │                                        │ If "payment.captured":
      │                                        │   Update payment STATUS
      │                                        │   Confirm booking
      │◀───────────────────────────────────────┤
      │ Response: 200 OK                       │
      │                                        │
```

## Critical Webhook Rules:

| Rule | Reason |
|------|--------|
| **Always return 200** | Non-200 = Razorpay retries |
| **Verify signature** | Prevent fake webhooks |
| **Be idempotent** | Same webhook may arrive twice |
| **Process async** | Don't block response |

---

# 🔁 Idempotency: Safe Retries

## The Problem:

```
User clicks "Pay Now" twice quickly
OR
Network timeout, frontend retries
OR
Webhook arrives twice

RESULT WITHOUT IDEMPOTENCY:
- Two orders created
- Double charge possible
```

## How E-TOUR Handles This:

```java
// In RazorpayServiceImpl.createOrder():

// 1. Check if already paid
if (paymentRepository.existsByBooking_IdAndPaymentStatus(
        bookingId, "SUCCESS")) {
    throw new RuntimeException("Already paid");
}

// 2. Check for existing INITIATED order
Optional<PaymentMaster> existing = paymentRepository
    .findByBooking_IdAndPaymentStatus(bookingId, "INITIATED");

if (existing.isPresent()) {
    // Return existing order instead of creating new one
    return existing.get().getTransactionRef();
}

// 3. Only now create new order
```

## In confirmPayment():

```java
// Already SUCCESS? Skip update
if (payment.getPaymentStatus().equals("SUCCESS")) {
    return; // Already processed, safe to ignore
}

// Amount mismatch? Reject
if (expectedAmount != receivedAmount) {
    throw new RuntimeException("Amount mismatch");
}

// Safe to update
payment.setPaymentStatus("SUCCESS");
```

---

# 📝 Audit Trail Importance

## What Gets Recorded:

| Field | Purpose |
|-------|---------|
| `payment_id` | Unique identifier |
| `booking_id` | Links to booking |
| `payment_date` | When payment processed |
| `payment_amount` | How much was paid |
| `payment_status` | INITIATED/SUCCESS/FAILED |
| `transaction_ref` | Razorpay order_id or payment_id |
| `payment_mode` | Card, UPI, NetBanking |

## Why Audit Trail Matters:

| Scenario | How Audit Helps |
|----------|-----------------|
| **Refund request** | Verify original payment |
| **Dispute** | Show transaction proof |
| **Debugging** | Trace what happened |
| **Compliance** | Financial regulations |
| **Analytics** | Payment success rates |

---

# ⚖️ Balanced View: Payment as One Module

Payment is **critical** but it's **one of many** modules:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MODULE IMPORTANCE BALANCE                            │
└─────────────────────────────────────────────────────────────────────────────┘

│ Without Auth        │ → No secure users
│ Without Tours       │ → Nothing to book
│ Without Booking     │ → No reservations
│ Without Payment     │ → No revenue
│ Without Search      │ → Poor discovery
│ Without Passengers  │ → Incomplete booking

ALL modules are essential for complete functionality.
```

---

> **Continue to Part 10: Failure Scenarios, Scalability & Future →**
