# Service/Module Decomposition

## 1. Introduction

This document describes the concrete code organization for EventPass's Modular Monolith architecture. Each bounded context maps to a self-contained module within a single TypeScript/Node.js application. The structure enforces context boundaries through directory conventions, explicit public interfaces, and linting rules — not through network isolation.

---

## 2. Complete Directory Tree

```
eventpass/
├── src/
│   ├── modules/
│   │   ├── identity/                  → Identity & Access BC
│   │   │   ├── public/
│   │   │   │   ├── identity.contracts.ts    → DTOs, interfaces consumed by other modules
│   │   │   │   ├── identity.events.ts       → UserRegistered, OrganizerApproved, etc.
│   │   │   │   └── identity.facade.ts       → Public service interface
│   │   │   ├── internal/
│   │   │   │   ├── entities/
│   │   │   │   │   ├── user.entity.ts
│   │   │   │   │   ├── profile.entity.ts
│   │   │   │   │   └── organizer-verification.entity.ts
│   │   │   │   ├── services/
│   │   │   │   │   ├── auth.service.ts
│   │   │   │   │   ├── registration.service.ts
│   │   │   │   │   └── organizer-verification.service.ts
│   │   │   │   └── repositories/
│   │   │   │       ├── user.repository.ts
│   │   │   │       └── organizer-verification.repository.ts
│   │   │   ├── infrastructure/
│   │   │   │   ├── auth0.adapter.ts         → Auth0 integration
│   │   │   │   ├── identity.schema.sql      → PostgreSQL schema: identity.*
│   │   │   │   └── identity.migrations/
│   │   │   └── index.ts                     → Re-exports from public/ ONLY
│   │   │
│   │   ├── events/                    → Event Management BC
│   │   │   ├── public/
│   │   │   │   ├── events.contracts.ts
│   │   │   │   ├── events.events.ts         → EventCreated, EventPublished, etc.
│   │   │   │   └── events.facade.ts
│   │   │   ├── internal/
│   │   │   │   ├── entities/
│   │   │   │   │   ├── event.entity.ts
│   │   │   │   │   ├── venue.entity.ts
│   │   │   │   │   └── category.entity.ts
│   │   │   │   ├── services/
│   │   │   │   │   ├── event-lifecycle.service.ts
│   │   │   │   │   └── venue.service.ts
│   │   │   │   └── repositories/
│   │   │   │       ├── event.repository.ts
│   │   │   │       ├── venue.repository.ts
│   │   │   │       └── category.repository.ts
│   │   │   ├── infrastructure/
│   │   │   │   ├── events.schema.sql
│   │   │   │   └── events.migrations/
│   │   │   └── index.ts
│   │   │
│   │   ├── catalog/                   → Catalog & Discovery BC
│   │   │   ├── public/
│   │   │   │   ├── catalog.contracts.ts
│   │   │   │   └── catalog.facade.ts
│   │   │   ├── internal/
│   │   │   │   ├── entities/
│   │   │   │   │   ├── event-listing.entity.ts
│   │   │   │   │   ├── search-index.entity.ts
│   │   │   │   │   └── featured-event.entity.ts
│   │   │   │   ├── services/
│   │   │   │   │   ├── search.service.ts
│   │   │   │   │   └── listing-sync.service.ts
│   │   │   │   ├── repositories/
│   │   │   │   │   └── event-listing.repository.ts
│   │   │   │   └── acl/
│   │   │   │       ├── event-management.translator.ts
│   │   │   │       └── ticketing.translator.ts
│   │   │   ├── infrastructure/
│   │   │   │   ├── catalog.schema.sql
│   │   │   │   └── catalog.migrations/
│   │   │   └── index.ts
│   │   │
│   │   ├── ticketing/                 → Ticketing BC
│   │   │   ├── public/
│   │   │   │   ├── ticketing.contracts.ts
│   │   │   │   ├── ticketing.events.ts      → TicketReserved, TicketSold, etc.
│   │   │   │   └── ticketing.facade.ts
│   │   │   ├── internal/
│   │   │   │   ├── entities/
│   │   │   │   │   ├── ticket-type.entity.ts
│   │   │   │   │   ├── ticket.entity.ts
│   │   │   │   │   └── seat-reservation.entity.ts
│   │   │   │   ├── services/
│   │   │   │   │   ├── inventory.service.ts
│   │   │   │   │   ├── reservation.service.ts
│   │   │   │   │   └── qr-generator.service.ts
│   │   │   │   └── repositories/
│   │   │   │       ├── ticket-type.repository.ts
│   │   │   │       └── ticket.repository.ts
│   │   │   ├── infrastructure/
│   │   │   │   ├── ticketing.schema.sql
│   │   │   │   ├── ticketing.migrations/
│   │   │   │   └── redis-lock.adapter.ts    → Redis-based reservation TTL
│   │   │   └── index.ts
│   │   │
│   │   ├── orders/                    → Order & Checkout BC
│   │   │   ├── public/
│   │   │   │   ├── orders.contracts.ts
│   │   │   │   ├── orders.events.ts         → OrderCreated, OrderConfirmed, etc.
│   │   │   │   └── orders.facade.ts
│   │   │   ├── internal/
│   │   │   │   ├── entities/
│   │   │   │   │   ├── order.entity.ts
│   │   │   │   │   ├── order-item.entity.ts
│   │   │   │   │   └── cart.entity.ts
│   │   │   │   ├── services/
│   │   │   │   │   ├── checkout.service.ts
│   │   │   │   │   └── order-lifecycle.service.ts
│   │   │   │   └── repositories/
│   │   │   │       ├── order.repository.ts
│   │   │   │       └── cart.repository.ts
│   │   │   ├── infrastructure/
│   │   │   │   ├── orders.schema.sql
│   │   │   │   └── orders.migrations/
│   │   │   └── index.ts
│   │   │
│   │   ├── payments/                  → Payment BC
│   │   │   ├── public/
│   │   │   │   ├── payments.contracts.ts
│   │   │   │   ├── payments.events.ts       → PaymentSucceeded, PaymentFailed, etc.
│   │   │   │   └── payments.facade.ts
│   │   │   ├── internal/
│   │   │   │   ├── entities/
│   │   │   │   │   ├── payment-intent.entity.ts
│   │   │   │   │   ├── refund.entity.ts
│   │   │   │   │   └── organizer-payout.entity.ts
│   │   │   │   ├── services/
│   │   │   │   │   ├── payment-processing.service.ts
│   │   │   │   │   ├── refund.service.ts
│   │   │   │   │   └── payout.service.ts
│   │   │   │   └── repositories/
│   │   │   │       ├── payment-intent.repository.ts
│   │   │   │       └── refund.repository.ts
│   │   │   ├── infrastructure/
│   │   │   │   ├── stripe.adapter.ts        → Stripe API ACL
│   │   │   │   ├── stripe-webhook.handler.ts
│   │   │   │   ├── payments.schema.sql
│   │   │   │   └── payments.migrations/
│   │   │   └── index.ts
│   │   │
│   │   └── notifications/            → Notification BC
│   │       ├── public/
│   │       │   ├── notifications.contracts.ts
│   │       │   ├── notifications.events.ts  → NotificationSent, NotificationFailed
│   │       │   └── notifications.facade.ts
│   │       ├── internal/
│   │       │   ├── entities/
│   │       │   │   ├── notification-template.entity.ts
│   │       │   │   ├── notification-log.entity.ts
│   │       │   │   └── notification-preference.entity.ts
│   │       │   ├── services/
│   │       │   │   ├── notification-dispatcher.service.ts
│   │       │   │   └── template-renderer.service.ts
│   │       │   └── repositories/
│   │       │       ├── notification-template.repository.ts
│   │       │       └── notification-log.repository.ts
│   │       ├── infrastructure/
│   │       │   ├── sendgrid.adapter.ts      → SendGrid email ACL
│   │       │   ├── twilio.adapter.ts        → Twilio SMS ACL
│   │       │   ├── notifications.schema.sql
│   │       │   └── notifications.migrations/
│   │       └── index.ts
│   │
│   ├── shared/
│   │   ├── kernel/
│   │   │   ├── value-objects/
│   │   │   │   ├── money.vo.ts              → Currency + amount value object
│   │   │   │   ├── email.vo.ts              → Validated email value object
│   │   │   │   └── uuid.vo.ts              → Typed UUID wrapper
│   │   │   ├── base-entity.ts               → Abstract base entity with id, timestamps
│   │   │   ├── domain-event.ts              → Base domain event interface
│   │   │   └── result.ts                    → Result<T, E> for error handling
│   │   ├── events/
│   │   │   ├── event-bus.interface.ts        → IEventBus: publish, subscribe
│   │   │   ├── in-memory-event-bus.ts        → In-process implementation (default)
│   │   │   └── rabbitmq-event-bus.ts         → RabbitMQ implementation (future extraction)
│   │   └── infrastructure/
│   │       ├── database/
│   │       │   ├── connection.ts             → PostgreSQL connection pool (single instance)
│   │       │   └── transaction-manager.ts    → Cross-module transaction support
│   │       ├── cache/
│   │       │   └── redis-client.ts           → Redis connection and utilities
│   │       ├── logging/
│   │       │   └── logger.ts                 → Structured logging (Pino)
│   │       └── config/
│   │           └── env.ts                    → Typed environment configuration
│   │
│   ├── api/
│   │   ├── rest/
│   │   │   ├── identity.controller.ts        → /api/auth/*, /api/users/*
│   │   │   ├── events.controller.ts          → /api/events/*
│   │   │   ├── catalog.controller.ts         → /api/catalog/*, /api/search/*
│   │   │   ├── ticketing.controller.ts       → /api/events/:id/tickets/*
│   │   │   ├── orders.controller.ts          → /api/orders/*, /api/cart/*
│   │   │   ├── payments.controller.ts        → /api/payments/*, /api/webhooks/stripe
│   │   │   └── notifications.controller.ts   → /api/notifications/preferences/*
│   │   └── middleware/
│   │       ├── auth.middleware.ts             → JWT validation via Auth0
│   │       ├── rate-limiter.middleware.ts     → Redis-backed rate limiting
│   │       ├── error-handler.middleware.ts    → Global error handling
│   │       └── request-logger.middleware.ts   → Request/response logging
│   │
│   └── main.ts                               → Application bootstrap
│
├── docker-compose.yml                         → Local dev: PostgreSQL, Redis, RabbitMQ
├── Dockerfile                                 → Production build
├── package.json
├── tsconfig.json
└── .eslintrc.js                               → Boundary enforcement rules
```

---

## 3. Module Ownership and Public API

Each module follows the same internal structure: `public/`, `internal/`, `infrastructure/`, and an `index.ts` barrel file.

| Module | Bounded Context | Owns | Exposes (Public API) |
|--------|----------------|------|---------------------|
| `identity/` | Identity & Access | Users, Profiles, Organizer Verification | `IdentityFacade`: getUserById, validateToken, getOrganizerStatus |
| `events/` | Event Management | Events, Venues, Categories | `EventsFacade`: createEvent, publishEvent, cancelEvent, getEventById |
| `catalog/` | Catalog & Discovery | Event Listings, Search Indexes, Featured Events | `CatalogFacade`: searchEvents, getEventListing, getFeaturedEvents |
| `ticketing/` | Ticketing | Ticket Types, Tickets, Seat Reservations | `TicketingFacade`: reserveTickets, releaseReservation, validateTicket, getAvailability |
| `orders/` | Order & Checkout | Orders, Order Items, Carts | `OrdersFacade`: createOrder, confirmOrder, cancelOrder, getOrderHistory |
| `payments/` | Payment | Payment Intents, Refunds, Organizer Payouts | `PaymentsFacade`: createPaymentIntent, processRefund, createPayout |
| `notifications/` | Notification | Templates, Logs, Preferences | `NotificationsFacade`: updatePreferences, getNotificationHistory |

### Public Interface Rules

1. **Other modules ONLY import from `module/public/` or `module/index.ts`** — never from `module/internal/` or `module/infrastructure/`.
2. **Each `index.ts` re-exports only what is in `public/`** — contracts (DTOs/interfaces), events (domain event types), and the facade (public service interface).
3. **The facade is the single entry point** for synchronous communication between modules. If the Orders module needs to check ticket availability, it calls `TicketingFacade.getAvailability()`, not `TicketRepository.findAvailable()`.
4. **Domain events are the mechanism for asynchronous communication.** When the Payment module processes a successful payment, it publishes `PaymentSucceeded` on the event bus. The Orders module subscribes to this event — it does not poll the Payment module.

---

## 4. Boundary Enforcement

### 4.1 Directory Structure

The `public/` and `internal/` separation is a convention, but conventions are enforced through tooling:

```
✅ import { TicketingFacade } from '@modules/ticketing';           // → index.ts → public/
✅ import { TicketReserved } from '@modules/ticketing';            // → index.ts → public/events
❌ import { TicketRepository } from '@modules/ticketing/internal'; // BLOCKED by lint
❌ import { RedisLock } from '@modules/ticketing/infrastructure';  // BLOCKED by lint
```

### 4.2 ESLint Boundary Rules

The `.eslintrc.js` includes rules that prevent cross-module boundary violations:

```javascript
// Simplified ESLint boundary configuration
{
  rules: {
    'no-restricted-imports': ['error', {
      patterns: [
        {
          group: ['@modules/*/internal/*'],
          message: 'Cannot import internal module code. Use the public facade.'
        },
        {
          group: ['@modules/*/infrastructure/*'],
          message: 'Cannot import infrastructure adapters. Use the public facade.'
        }
      ]
    }]
  }
}
```

### 4.3 Communication Patterns

| Pattern | When | Example |
|---------|------|---------|
| **Facade call** (synchronous) | Module A needs data from Module B in the same request cycle | `OrdersController` calls `TicketingFacade.reserveTickets()` during checkout |
| **Event bus** (asynchronous) | Side-effect that does not need to complete in the same request | `PaymentModule` publishes `PaymentSucceeded` → `OrderModule` subscribes and confirms the order |
| **Shared kernel** | Common value objects used across modules | `Money` value object (amount + currency) used by Ticketing, Orders, and Payment |

### 4.4 Database Schema Isolation

Each module owns a PostgreSQL schema. Modules cannot read or write to other modules' schemas directly.

```sql
-- Schema-per-module layout
CREATE SCHEMA identity;    -- User, Profile, OrganizerVerification
CREATE SCHEMA events;      -- Event, Venue, Category
CREATE SCHEMA catalog;     -- EventListing, SearchIndex, FeaturedEvent
CREATE SCHEMA ticketing;   -- TicketType, Ticket, SeatReservation
CREATE SCHEMA orders;      -- Order, OrderItem, Cart
CREATE SCHEMA payments;    -- PaymentIntent, Refund, OrganizerPayout
CREATE SCHEMA notifications; -- NotificationTemplate, NotificationLog, NotificationPreference
```

Cross-schema data access is explicitly prohibited at the application level. If the Orders module needs user information, it calls `IdentityFacade.getUserById()`, which queries the `identity` schema — the Orders module never runs `SELECT * FROM identity.users` directly.

---

## 5. Module-to-Bounded Context Mapping

```
┌─────────────────────────────────────────────────────────────────┐
│                        eventpass (monolith)                      │
│                                                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐   │
│  │ identity │ │  events  │ │ catalog  │ │    ticketing     │   │
│  │          │ │          │ │          │ │                  │   │
│  │ Identity │ │  Event   │ │ Catalog  │ │   Ticketing BC   │   │
│  │ & Access │ │  Mgmt BC │ │ & Disc.  │ │                  │   │
│  │    BC    │ │          │ │    BC    │ │                  │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘   │
│                                                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐                │
│  │  orders  │ │ payments │ │  notifications   │                │
│  │          │ │          │ │                  │                │
│  │ Order &  │ │ Payment  │ │  Notification    │                │
│  │ Checkout │ │    BC    │ │      BC          │                │
│  │    BC    │ │          │ │                  │                │
│  └──────────┘ └──────────┘ └──────────────────┘                │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                     shared/                               │   │
│  │  kernel/  (value objects, base types)                     │   │
│  │  events/  (event bus interface + implementations)         │   │
│  │  infrastructure/  (DB, cache, logging, config)            │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      api/                                 │   │
│  │  rest/  (controllers per module)                          │   │
│  │  middleware/  (auth, rate-limit, error-handler)            │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Shared Kernel

The `shared/` directory contains code that is genuinely shared across multiple modules. It is intentionally kept minimal to avoid creating a coupling magnet.

### What belongs in shared:

| Component | Purpose | Used By |
|-----------|---------|---------|
| `Money` value object | Encapsulates amount + currency with arithmetic operations | Ticketing, Orders, Payment |
| `Email` value object | Validated email format | Identity, Notification |
| `UUID` value object | Typed UUID wrapper | All modules |
| `BaseEntity` | Abstract entity with `id`, `createdAt`, `updatedAt` | All modules |
| `DomainEvent` | Base event interface with `eventId`, `occurredAt`, `aggregateId` | All modules |
| `Result<T, E>` | Functional error handling without exceptions | All modules |
| `IEventBus` | Event bus contract (publish/subscribe) | All modules |
| Database connection | Single PostgreSQL connection pool | All modules |
| Redis client | Single Redis connection | Ticketing (locks), API (rate limiting), Catalog (cache) |
| Logger | Structured logging (Pino) | All modules |

### What does NOT belong in shared:

- Business logic specific to any bounded context
- DTOs or contracts that only two modules exchange (these go in the publishing module's `public/`)
- Repository implementations (each module owns its own)
- External service adapters (these are module-specific infrastructure)

---

## 7. Future Extraction Path

When a module needs to be extracted into an independent service, the process is:

1. **Deploy the module as a separate application** — copy the module's `internal/`, `infrastructure/`, and `public/` directories into a new repository. Point its database connection to a new PostgreSQL instance and run its schema migrations.

2. **Replace facade calls with HTTP/gRPC** — where other modules previously called `TicketingFacade.reserveTickets()` in-process, they now make an HTTP request to `POST /api/tickets/reserve`. The facade interface stays the same; only the implementation changes from direct function call to HTTP client.

3. **Replace in-process events with RabbitMQ** — the extracted module subscribes to events on RabbitMQ instead of the in-memory event bus. The `shared/events/rabbitmq-event-bus.ts` already exists as an alternative implementation. Switching from `InMemoryEventBus` to `RabbitMQEventBus` is a configuration change.

4. **Update the monolith** — remove the extracted module's `internal/` code from the monolith. Replace its facade with an HTTP client adapter that implements the same interface. The rest of the monolith's code does not change.

This extraction can happen one module at a time, with no big-bang migration.
