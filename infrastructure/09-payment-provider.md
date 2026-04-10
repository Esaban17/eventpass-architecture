# Payment Provider — Stripe

## 1. Responsibilities

- **Payment intent creation** — Create Stripe PaymentIntents for ticket purchases, returning a `client_secret` for the frontend's Stripe Elements integration. Supports USD, EUR, and GTQ currencies.
- **Webhook-based payment confirmation** — Receive `payment_intent.succeeded` and `payment_intent.payment_failed` webhooks from Stripe, triggering order confirmation or retry flows.
- **Refund processing** — Process full refunds for cancelled events and partial refunds for disputed orders via the Stripe Refunds API. Batch processing with exponential backoff for mass refund scenarios.
- **Organizer payouts via Stripe Connect** — Transfer funds to organizer Stripe accounts after event completion, deducting the platform commission. Uses Stripe Connect's destination charges model.
- **PCI compliance delegation** — Stripe.js and Stripe Elements handle card data collection on the frontend — card numbers never touch EventPass servers, maintaining PCI DSS SAQ-A compliance.

## 2. Why This Project Needs It

EventPass processes financial transactions on behalf of three parties: buyers (who pay), organizers (who receive payouts), and the platform (which takes a commission). This multi-party financial model requires a payment provider that supports: marketplace-style payouts (Stripe Connect), webhook-based asynchronous confirmation (critical for the event-driven architecture), and robust refund handling (mass refunds during event cancellations).

PCI compliance is non-negotiable for a platform handling credit card payments. By using Stripe Elements, EventPass avoids the burden of PCI DSS SAQ-D compliance (full assessment) and qualifies for SAQ-A (minimal self-assessment) — a significant reduction in compliance scope and cost for a startup.

## 3. Technology Choice

| Dimension | Stripe | PayPal | Adyen |
|-----------|--------|--------|-------|
| **Managed/Self-hosted** | Fully managed SaaS | Fully managed SaaS | Fully managed SaaS |
| **Operational complexity** | Low — excellent API docs, webhooks, SDKs | Low-moderate — multiple APIs (Classic, REST, Braintree) with inconsistent patterns | Low — unified API, but onboarding requires sales process |
| **Cost at your scale** | 2.9% + $0.30/transaction | 2.9% + $0.30/transaction | 2.6% + $0.10/transaction (lower but requires volume commitment) |
| **Key differentiating feature** | Stripe Connect for marketplace payouts, best-in-class developer experience, Elements for PCI compliance | PayPal wallet (300M+ users), buyer protection program | Global payment methods (iDEAL, Alipay, Boleto), enterprise-grade fraud detection |

**Decision: Stripe** — Stripe Connect is the decisive factor: it provides a native marketplace model where EventPass can collect payments from buyers and distribute funds to organizers with configurable commission splits. PayPal's marketplace equivalent (PayPal for Marketplaces) is less mature and has stricter onboarding requirements. Adyen offers lower per-transaction fees but requires a sales-driven onboarding process and minimum volume commitments unsuitable for an MVP.

## 4. Trade-offs

| Advantages | Disadvantages |
|------------|---------------|
| Stripe Connect handles multi-party payouts — platform commission calculated automatically | 2.9% + $0.30 per transaction — higher per-transaction cost than Adyen for high-volume scenarios |
| Stripe Elements handles PCI compliance — card data never touches EventPass servers | Vendor dependency — Stripe outage blocks all payment processing (mitigated by order retry logic) |
| Webhook-based architecture aligns with EventPass's async communication model (ADR-002) | Payout timing — Stripe holds funds for 2-7 days before transferring to organizer accounts |
| Excellent developer experience — comprehensive docs, test mode, CLI tooling | Limited payment methods in Latin America — Stripe supports cards and OXXO in Mexico but limited local methods in Guatemala |
| Idempotency keys prevent duplicate charges from retried requests | Stripe's dispute resolution process requires manual admin intervention |

## 5. Integration

- **Payment module:** The `infrastructure/stripe.adapter.ts` wraps Stripe's Node.js SDK behind an Anti-Corruption Layer. The `PaymentsFacade` exposes `createPaymentIntent()`, `processRefund()`, and `createPayout()` without leaking Stripe-specific types. See [ADR-002](../adrs/ADR-002-communication-style.md).
- **Webhook handling:** Stripe webhooks are received at `POST /api/webhooks/stripe`. The handler verifies the webhook signature using Stripe's signing secret, then publishes domain events (`PaymentSucceeded`, `PaymentFailed`) to the internal event bus.
- **Stripe Connect:** Organizers connect their Stripe accounts during the verification process. EventPass uses destination charges: the buyer pays EventPass, and EventPass transfers the organizer's share (minus platform fee) automatically.
- **Idempotency:** Every PaymentIntent creation includes an idempotency key (`order:{orderId}:attempt:{n}`) to prevent duplicate charges on network retries.
- **Frontend:** Next.js checkout page uses `@stripe/react-stripe-js` and Stripe Elements for card input. The `clientSecret` from the backend initializes the payment form.
- **Deployment diagram reference:** See [diagrams/04-deployment.md](../diagrams/04-deployment.md) — Stripe is an external service accessed from the Private VPC. Stripe webhooks enter via Cloudflare → ALB → ECS.
