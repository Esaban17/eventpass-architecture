# Data Flow and Interactions

## 1. Introduction

This document describes four end-to-end flows through EventPass, showing how bounded contexts interact via synchronous calls (facade invocations) and asynchronous events (internal event bus). Each flow includes a happy path, at least one failure path, and a Mermaid sequence diagram.

All flows assume the Modular Monolith architecture where modules communicate in-process. Protocol labels indicate the type of interaction:

- **SYNC** — Direct facade method call (in-process, sub-millisecond)
- **ASYNC** — Domain event published on the internal event bus
- **HTTP** — External API call (Stripe, SendGrid, Twilio, Auth0)

---

## 2. Flow 1: User Registration and Organizer Verification

### 2.1 Happy Path

1. **Buyer registers** — submits email and password to the Identity module.
2. Identity creates the User (status: ACTIVE, role: BUYER) and publishes `UserRegistered`.
3. Notification module consumes `UserRegistered` and sends a welcome email via SendGrid.
4. **Organizer registers** — submits email, password, and business details.
5. Identity creates the User (role: ORGANIZER) and an OrganizerVerification (status: PENDING). Publishes `UserRegistered`.
6. Admin reviews the organizer application in the admin panel.
7. Admin approves — Identity updates OrganizerVerification (status: VERIFIED) and publishes `OrganizerApproved`.
8. Event Management consumes `OrganizerApproved` and enables event creation for this organizer.
9. Notification consumes `OrganizerApproved` and sends approval email.

### 2.2 Failure Path: Email Verification Timeout

1. User registers but does not verify their email within 24 hours.
2. A scheduled job in the Identity module detects expired unverified accounts.
3. Identity resends the verification email by publishing a `VerificationEmailRequested` event.
4. Notification consumes the event and sends the email via SendGrid.
5. If the user still does not verify after 72 hours, the account is marked as SUSPENDED.

### 2.3 Sequence Diagram

```mermaid
sequenceDiagram
    actor Buyer
    participant Identity
    participant Notification
    participant SendGrid as 🔌 SendGrid

    Note over Buyer, SendGrid: Buyer Registration (Happy Path)
    Buyer->>Identity: POST /api/auth/register {email, password, role: BUYER}
    Identity->>Identity: Create User (ACTIVE, BUYER)
    Identity-->>Buyer: 201 {userId, token}
    Identity-)Notification: ASYNC: UserRegistered {userId, email, role}
    Notification->>SendGrid: HTTP: Send welcome email
    SendGrid-->>Notification: 202 Accepted

    Note over Buyer, SendGrid: Failure: Verification email expires
    Identity->>Identity: Cron: detect unverified > 24h
    Identity-)Notification: ASYNC: VerificationEmailRequested {userId, email}
    Notification->>SendGrid: HTTP: Resend verification email
```

```mermaid
sequenceDiagram
    actor Organizer
    actor Admin
    participant Identity
    participant EventMgmt as Event Management
    participant Notification

    Note over Organizer, Notification: Organizer Verification (Happy Path)
    Organizer->>Identity: POST /api/auth/register {email, password, role: ORGANIZER, businessName, taxId}
    Identity->>Identity: Create User + OrganizerVerification (PENDING)
    Identity-->>Organizer: 201 {userId, token}
    Identity-)Notification: ASYNC: UserRegistered {userId, email, role: ORGANIZER}

    Admin->>Identity: PATCH /api/admin/organizers/:id/approve
    Identity->>Identity: Update OrganizerVerification (VERIFIED)
    Identity-)EventMgmt: ASYNC: OrganizerApproved {userId, organizerId}
    Identity-)Notification: ASYNC: OrganizerApproved {userId, email}
    Notification->>Notification: Send approval email via SendGrid
```

---

## 3. Flow 2: Ticket Purchase (Core Transaction)

This is EventPass's most critical flow. It involves 5 bounded contexts and must handle concurrent buyers competing for limited inventory during flash sales.

### 3.1 Happy Path

1. **Buyer browses events** — Catalog module serves search results from its denormalized read model.
2. **Buyer selects tickets** — Frontend sends ticket type and quantity to the Ticketing module.
3. **Ticketing creates a reservation** — Locks the requested tickets with a 10-minute TTL using `SELECT ... FOR UPDATE SKIP LOCKED`. Sets ticket status to RESERVED. Publishes `TicketReserved`.
4. **Buyer proceeds to checkout** — Frontend creates an order via the Orders module.
5. **Orders creates the order** (status: PENDING) with order items. Publishes `OrderCreated`.
6. **Payment module creates a Stripe PaymentIntent** — Consumes `OrderCreated`, calls Stripe API to create a payment intent, and returns the `client_secret` to the frontend.
7. **Buyer completes payment in the frontend** using Stripe Elements.
8. **Stripe sends webhook** — `payment_intent.succeeded` webhook hits the Payment module.
9. **Payment publishes `PaymentSucceeded`** with orderId and paymentIntentId.
10. **Orders confirms the order** — Consumes `PaymentSucceeded`, updates order status to CONFIRMED. Publishes `OrderConfirmed`.
11. **Ticketing marks tickets as SOLD** — Consumes `OrderConfirmed`, updates ticket status, generates QR codes. Publishes `TicketSold`.
12. **Notification sends confirmation email** — Consumes `OrderConfirmed`, sends email with order details and QR codes via SendGrid.

### 3.2 Failure Path: Payment Rejected

1. Steps 1-6 happen as in the happy path.
2. Stripe declines the payment — webhook `payment_intent.payment_failed` arrives.
3. Payment module publishes `PaymentFailed` with orderId and failure reason.
4. Orders module consumes `PaymentFailed` — keeps order in PENDING status (buyer may retry).
5. Ticketing module does NOT release the reservation yet (10-min TTL is still active).
6. Notification sends a "payment failed, please try again" email.
7. If the buyer does not retry within 10 minutes, the reservation TTL expires.
8. Ticketing detects expiration, updates ticket status to AVAILABLE, publishes `TicketReservationExpired`.
9. Orders consumes `TicketReservationExpired` and cancels the order. Publishes `OrderCancelled`.
10. Catalog consumes `TicketInventoryUpdated` and updates available ticket counts in the listing.

### 3.3 Sequence Diagrams

**Part 1: Reservation and Checkout**

```mermaid
sequenceDiagram
    actor Buyer
    participant Catalog
    participant Ticketing
    participant Orders

    Note over Buyer, Orders: Reservation and Order Creation
    Buyer->>Catalog: GET /api/catalog/events?city=Guatemala
    Catalog-->>Buyer: 200 {events: [{id, title, minPrice, availableTickets}]}
    Buyer->>Ticketing: POST /api/events/:id/tickets/reserve {ticketTypeId, qty: 2}
    Ticketing->>Ticketing: SELECT ... FOR UPDATE SKIP LOCKED
    Ticketing->>Ticketing: Set tickets RESERVED (TTL: 10 min)
    Ticketing-->>Buyer: 200 {reservationId, expiresAt}
    Ticketing-)Orders: ASYNC: TicketReserved {reservationId, ticketTypeId, qty, unitPrice}
    Buyer->>Orders: POST /api/orders {reservationId, ticketTypeId, qty}
    Orders->>Orders: Create Order (PENDING) + OrderItems
    Orders-->>Buyer: 201 {orderId, totalAmount}
    Orders-)Orders: ASYNC: OrderCreated {orderId, buyerId, totalAmount, currency}
```

**Part 2: Payment and Confirmation**

```mermaid
sequenceDiagram
    actor Buyer
    participant Payment
    participant Stripe as 🔌 Stripe
    participant Orders
    participant Ticketing
    participant Notification

    Note over Buyer, Notification: Payment Processing
    Payment->>Stripe: HTTP: Create PaymentIntent {amount, currency}
    Stripe-->>Payment: 200 {paymentIntentId, clientSecret}
    Payment-->>Buyer: 200 {clientSecret}
    Buyer->>Stripe: Stripe.js confirmPayment({clientSecret})
    Stripe-->>Buyer: Payment confirmed
    Stripe->>Payment: HTTP: Webhook payment_intent.succeeded
    Payment->>Payment: Update PaymentIntent (SUCCEEDED)
    Payment-)Orders: ASYNC: PaymentSucceeded {orderId, paymentIntentId}
    Orders->>Orders: Update Order (CONFIRMED)
    Orders-)Ticketing: ASYNC: OrderConfirmed {orderId, items}
    Ticketing->>Ticketing: Set tickets SOLD, generate QR codes
    Orders-)Notification: ASYNC: OrderConfirmed {orderId, buyerEmail, items}
    Notification->>Notification: Send confirmation + QR via SendGrid
```

**Part 3: Payment Failure and Reservation Expiry**

```mermaid
sequenceDiagram
    participant Stripe as 🔌 Stripe
    participant Payment
    participant Orders
    participant Ticketing
    participant Notification
    participant Catalog

    Note over Stripe, Catalog: Payment Failure Path
    Stripe->>Payment: HTTP: Webhook payment_intent.payment_failed
    Payment->>Payment: Update PaymentIntent (FAILED)
    Payment-)Orders: ASYNC: PaymentFailed {orderId, reason}
    Payment-)Notification: ASYNC: PaymentFailed {buyerEmail, reason}
    Notification->>Notification: Send "payment failed" email

    Note over Stripe, Catalog: Reservation Expires (10 min TTL)
    Ticketing->>Ticketing: Cron: detect expired reservations
    Ticketing->>Ticketing: Set tickets AVAILABLE
    Ticketing-)Orders: ASYNC: TicketReservationExpired {reservationId}
    Ticketing-)Catalog: ASYNC: TicketInventoryUpdated {eventId, available}
    Orders->>Orders: Update Order (CANCELLED)
    Orders-)Orders: ASYNC: OrderCancelled {orderId}
```

---

## 4. Flow 3: Failed Payment Processing and Retry

This flow handles the case where a buyer's payment fails and they attempt to retry before the reservation expires.

### 4.1 Happy Path (Retry Succeeds)

1. Initial payment fails (as described in Flow 2 failure path).
2. Orders module keeps the order in PENDING status.
3. Buyer clicks "Retry Payment" in the frontend.
4. Payment module creates a new Stripe PaymentIntent for the same order.
5. Buyer completes payment successfully.
6. Flow continues from step 8 of Flow 2 happy path.

### 4.2 Failure Path: 3 Retries Exhausted

1. Payment fails on first attempt. Buyer is notified.
2. Buyer retries — second PaymentIntent created. Payment fails again.
3. Buyer retries — third PaymentIntent created. Payment fails a third time.
4. Payment module publishes `PaymentFailed` with `attemptsExhausted: true`.
5. Orders module auto-cancels the order. Publishes `OrderCancelled`.
6. Ticketing releases the reservation immediately (does not wait for TTL). Publishes `TicketCancelled`.
7. Notification sends "order cancelled after 3 failed payment attempts" email.

### 4.3 Sequence Diagram

```mermaid
sequenceDiagram
    actor Buyer
    participant Payment
    participant Stripe as 🔌 Stripe
    participant Orders
    participant Ticketing
    participant Notification

    Note over Buyer, Notification: Retry Attempt 1 (fails)
    Buyer->>Payment: POST /api/payments/retry {orderId}
    Payment->>Stripe: HTTP: Create PaymentIntent (attempt 2)
    Stripe-->>Payment: 200 {clientSecret}
    Buyer->>Stripe: confirmPayment (attempt 2)
    Stripe->>Payment: Webhook: payment_failed
    Payment-)Notification: ASYNC: PaymentFailed {attempt: 2}

    Note over Buyer, Notification: Retry Attempt 2 (fails — max reached)
    Buyer->>Payment: POST /api/payments/retry {orderId}
    Payment->>Stripe: HTTP: Create PaymentIntent (attempt 3)
    Stripe-->>Payment: 200 {clientSecret}
    Buyer->>Stripe: confirmPayment (attempt 3)
    Stripe->>Payment: Webhook: payment_failed
    Payment-)Orders: ASYNC: PaymentFailed {orderId, attemptsExhausted: true}
    Orders->>Orders: Update Order (CANCELLED)
    Orders-)Ticketing: ASYNC: OrderCancelled {orderId}
    Ticketing->>Ticketing: Release tickets → AVAILABLE
    Ticketing-)Ticketing: ASYNC: TicketCancelled {ticketIds}
    Orders-)Notification: ASYNC: OrderCancelled {orderId, reason: "payment_attempts_exhausted"}
    Notification->>Notification: Send cancellation email
```

---

## 5. Flow 4: Event Cancellation and Mass Refund

This flow handles the most operationally complex scenario: an organizer cancels an event and all existing orders must be refunded.

### 5.1 Happy Path

1. **Organizer requests cancellation** — calls Event Management module to cancel the event.
2. Event Management validates that the event can be cancelled (status is PUBLISHED, not already CANCELLED or COMPLETED). Updates event status to CANCELLED. Publishes `EventCancelled`.
3. **Orders module consumes `EventCancelled`** — queries all orders for this event with status CONFIRMED. For each order, publishes `RefundRequested`.
4. **Payment module processes refunds in batch** — For each `RefundRequested`, creates a Stripe Refund via the API. Publishes `RefundCompleted` for each successful refund.
5. **Ticketing consumes `EventCancelled`** — marks all tickets for this event as CANCELLED. Publishes `TicketCancelled` for each ticket.
6. **Notification consumes `RefundCompleted`** — sends refund confirmation email to each buyer.
7. **Catalog consumes `EventCancelled`** — removes the event listing from the search index.

### 5.2 Failure Path: Stripe Refund Fails

1. Steps 1-3 happen as in the happy path.
2. Payment module calls Stripe to process a refund — Stripe returns an error (e.g., network timeout, insufficient balance).
3. Payment module retries with **exponential backoff**: 1s, 2s, 4s (3 attempts total).
4. If all 3 retries fail, Payment publishes `RefundFailed` with the orderId and error details.
5. Notification consumes `RefundFailed` and alerts the admin team via email.
6. The failed refund is logged in the Payment module's `Refund` entity (status: FAILED) for manual resolution.
7. Admin resolves the issue manually (e.g., retries from the admin panel or contacts Stripe support).

### 5.3 Sequence Diagrams

**Part 1: Cancellation and Batch Refund Initiation**

```mermaid
sequenceDiagram
    actor Organizer
    participant EventMgmt as Event Management
    participant Orders
    participant Ticketing
    participant Catalog
    participant Notification

    Note over Organizer, Notification: Event Cancellation
    Organizer->>EventMgmt: PATCH /api/events/:id/cancel
    EventMgmt->>EventMgmt: Validate (status = PUBLISHED)
    EventMgmt->>EventMgmt: Update status → CANCELLED
    EventMgmt-->>Organizer: 200 {eventId, status: CANCELLED}
    EventMgmt-)Orders: ASYNC: EventCancelled {eventId}
    EventMgmt-)Ticketing: ASYNC: EventCancelled {eventId}
    EventMgmt-)Catalog: ASYNC: EventCancelled {eventId}

    Orders->>Orders: Query CONFIRMED orders for eventId
    loop For each order (batch)
        Orders-)Orders: ASYNC: RefundRequested {orderId, amount}
    end

    Ticketing->>Ticketing: Mark all event tickets CANCELLED
    Catalog->>Catalog: Remove event listing from index
```

**Part 2: Refund Processing with Retry**

```mermaid
sequenceDiagram
    participant Payment
    participant Stripe as 🔌 Stripe
    participant Notification

    Note over Payment, Notification: Batch Refund Processing
    loop For each RefundRequested
        Payment->>Stripe: HTTP: POST /refunds {paymentIntentId, amount}
        alt Stripe returns success
            Stripe-->>Payment: 200 {refundId}
            Payment->>Payment: Update Refund (COMPLETED)
            Payment-)Notification: ASYNC: RefundCompleted {orderId, amount, buyerEmail}
            Notification->>Notification: Send refund confirmation email
        else Stripe returns error
            Stripe-->>Payment: 500 Error
            Payment->>Payment: Retry with backoff (1s, 2s, 4s)
            alt Retry succeeds
                Payment-)Notification: ASYNC: RefundCompleted {orderId}
            else 3 retries exhausted
                Payment->>Payment: Update Refund (FAILED)
                Payment-)Notification: ASYNC: RefundFailed {orderId, error}
                Notification->>Notification: Alert admin team
            end
        end
    end
```

---

## 6. Summary: Event Flow Matrix

This matrix shows which domain events connect which contexts across all four flows.

| Domain Event | Published By | Consumed By | Flow(s) |
|-------------|-------------|-------------|---------|
| `UserRegistered` | Identity | Notification | 1 |
| `OrganizerApproved` | Identity | Event Management, Notification | 1 |
| `VerificationEmailRequested` | Identity | Notification | 1 (failure) |
| `TicketReserved` | Ticketing | Orders | 2 |
| `TicketReservationExpired` | Ticketing | Orders, Catalog | 2 (failure) |
| `TicketSold` | Ticketing | — | 2 |
| `TicketCancelled` | Ticketing | — | 3, 4 |
| `TicketInventoryUpdated` | Ticketing | Catalog | 2 (failure) |
| `OrderCreated` | Orders | Payment | 2 |
| `OrderConfirmed` | Orders | Ticketing, Notification | 2 |
| `OrderCancelled` | Orders | Ticketing, Notification | 2 (failure), 3 |
| `RefundRequested` | Orders | Payment | 4 |
| `PaymentSucceeded` | Payment | Orders | 2 |
| `PaymentFailed` | Payment | Orders, Notification | 2 (failure), 3 |
| `RefundCompleted` | Payment | Notification | 4 |
| `RefundFailed` | Payment | Notification | 4 (failure) |
| `EventCancelled` | Event Management | Orders, Ticketing, Catalog, Notification | 4 |
