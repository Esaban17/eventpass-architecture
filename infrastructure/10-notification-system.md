# Notification System — SendGrid (Email) + Twilio (SMS)

## 1. Responsibilities

- **Transactional email delivery** — Send order confirmation emails with QR code attachments, refund notifications, organizer approval/rejection emails, and event reminder emails via SendGrid's transactional API.
- **SMS notifications** — Send ticket purchase confirmations, event reminders (24h and 2h before event), and critical alerts (event cancellation) via Twilio's Programmable SMS API.
- **Dynamic email templates** — Use SendGrid's dynamic template engine with Handlebars syntax for personalized emails: buyer name, event details, ticket quantities, QR code images, and refund amounts.
- **Delivery tracking** — Track email delivery status (delivered, bounced, opened) and SMS delivery status (delivered, failed, undelivered) for operational monitoring and troubleshooting.
- **User preference management** — Respect per-user notification preferences (email on/off, SMS on/off) stored in the Notification module's `NotificationPreference` entity.

## 2. Why This Project Needs It

EventPass must send transactional communications at key moments in the buyer journey: purchase confirmation (with QR code — the buyer's "ticket"), payment failure alerts (so buyers can retry), and event cancellation notices (with refund details). These are not marketing emails — they are critical business communications that buyers expect immediately.

SMS is particularly important for event reminders: a buyer who purchased tickets 3 weeks ago may forget the event date. A reminder SMS 24h and 2h before the event reduces no-shows and improves the experience for organizers. Additionally, SMS provides a fallback channel when email delivery fails (full inbox, spam filters).

## 3. Technology Choice

| Dimension | SendGrid + Twilio | AWS SES + SNS | Mailgun + Vonage |
|-----------|-------------------|---------------|------------------|
| **Managed/Self-hosted** | Fully managed SaaS (both) | Fully managed (AWS) | Fully managed SaaS (both) |
| **Operational complexity** | Low — API-based, no infrastructure | Low — native AWS, IAM integration | Low — API-based, similar to SendGrid |
| **Cost at our scale** | SendGrid: Free up to 100 emails/day; $19.95/month for 50K. Twilio: $0.0079/SMS | SES: $0.10/1K emails. SNS: $0.00645/SMS | Mailgun: $35/month for 50K. Vonage: $0.0068/SMS |
| **Key differentiating feature** | SendGrid: Dynamic templates + delivery analytics. Twilio: Global SMS reach + phone verification | SES: Highest deliverability with AWS. SNS: Native AWS pub/sub | Mailgun: Advanced email routing. Vonage: WhatsApp integration |

**Decision: SendGrid (email) + Twilio (SMS)** — SendGrid's dynamic template engine allows the marketing/design team to update email layouts without code changes — templates are managed in SendGrid's dashboard, and the application only provides the data variables. Twilio provides the broadest SMS coverage in Latin America (Guatemala, Mexico, Colombia), which is critical for EventPass's target market. AWS SES has better pricing but no visual template editor; Mailgun + Vonage is comparable but Vonage's Latin American SMS coverage is less mature than Twilio's.

## 4. Trade-offs

| Advantages | Disadvantages |
|------------|---------------|
| SendGrid's visual template editor — design team can update email layouts without developer involvement | Two vendor dependencies — both SendGrid and Twilio outages affect notifications |
| Twilio's global SMS coverage — delivers to Guatemala, Mexico, and other LATAM countries reliably | SMS costs add up with scale — at 10K SMS/month, Twilio costs ~$79/month |
| Delivery analytics — open rates, bounce rates, click tracking for email; delivery receipts for SMS | Email may land in spam folders — requires proper DKIM/SPF/DMARC configuration |
| Both APIs are idempotent — safe to retry on network errors without duplicate sends | Regulatory compliance for SMS — must handle opt-in/opt-out and comply with local telecom regulations |
| Generous free tiers for MVP — SendGrid 100 emails/day, Twilio $15 trial credit | Template management split between SendGrid (email) and application code (SMS) — no unified template system |

## 5. Integration

- **Notification module:** The `infrastructure/sendgrid.adapter.ts` and `infrastructure/twilio.adapter.ts` wrap both providers behind ACLs. The `NotificationDispatcher` service selects the appropriate adapter based on the user's notification preferences and the notification channel.
- **Domain event consumption:** The Notification module subscribes to events on the internal event bus (and RabbitMQ for durable delivery):
  - `UserRegistered` → welcome email
  - `OrderConfirmed` → confirmation email (with QR) + SMS
  - `PaymentFailed` → "retry payment" email
  - `EventCancelled` → cancellation email + SMS
  - `RefundCompleted` → refund confirmation email
  - `TicketValidated` → "enjoy the event" push/SMS
- **SendGrid configuration:** Dynamic templates with variables: `{{buyerName}}`, `{{eventTitle}}`, `{{orderTotal}}`, `{{qrCodeUrl}}`, `{{refundAmount}}`. Templates versioned in SendGrid's dashboard.
- **Twilio configuration:** Programmable SMS with alphanumeric sender ID "EventPass" where supported; phone number fallback for countries requiring numeric sender.
- **Monitoring:** Email delivery rates and SMS delivery status exported to Prometheus. Alerts on bounce rate > 5% or SMS failure rate > 2%. See [11-observability-stack.md](11-observability-stack.md).
- **Deployment diagram reference:** See [diagrams/04-deployment.md](../diagrams/04-deployment.md) — SendGrid and Twilio are external services accessed from the Private VPC.
