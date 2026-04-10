# EventPass — Architecture Design Backlog

> **Proyecto:** EventPass — Event Ticketing Platform
> **Autor:** Estuardo Sabán
> **Repo GitHub:** `eventpass-architecture`
> **Fecha:** 2026-04-10
> **Objetivo:** Diseñar la arquitectura completa de software de una plataforma de venta de boletos para eventos. No se implementa código; se producen documentos de arquitectura de calidad profesional.

---

## Descripción del Proyecto

**EventPass** es una plataforma de ticketing para eventos en vivo (conciertos, conferencias, deportes, teatro). Conecta a **organizadores** que crean y gestionan eventos con **compradores** que descubren, compran boletos y asisten. Un equipo de **administradores** modera la plataforma, gestiona disputas y monitorea métricas.

### ¿Por qué Event Ticketing?

- **Múltiples roles de usuario:** Comprador, Organizador, Administrador.
- **Flujos síncronos y asíncronos:** Búsqueda de eventos (sync), procesamiento de pagos y generación de boletos (async), notificaciones (async).
- **Flujo financiero complejo:** Compra de boletos con Stripe, reembolsos, payouts a organizadores, comisiones de plataforma.
- **Integraciones externas:** Stripe (pagos), SendGrid (email), Twilio (SMS), Google Maps (ubicación de venues), Cloudflare (CDN).
- **Alta concurrencia:** Ventas flash con miles de usuarios simultáneos compitiendo por inventario limitado.
- **6+ bounded contexts** bien diferenciados.

---

## Bounded Contexts Identificados (7)

| # | Bounded Context | Tipo de Dominio | Descripción |
|---|----------------|-----------------|-------------|
| 1 | **Identity & Access** | Generic Subdomain | Registro, autenticación, autorización, gestión de roles (Buyer/Organizer/Admin) |
| 2 | **Event Management** | Core Domain | CRUD de eventos, configuración de venues, fechas, categorías, estado del evento |
| 3 | **Catalog & Discovery** | Supporting Subdomain | Búsqueda, filtrado, recomendaciones, listado público de eventos |
| 4 | **Ticketing** | Core Domain | Tipos de boleto, inventario, selección de asientos, reservas temporales, generación de QR |
| 5 | **Order & Checkout** | Core Domain | Carrito, checkout, ciclo de vida de la orden, historial de compras |
| 6 | **Payment** | Supporting Subdomain | Intents de pago, cobros, reembolsos, payouts a organizadores, comisiones |
| 7 | **Notification** | Generic Subdomain | Emails transaccionales, SMS, push notifications, recordatorios de eventos |

---

## Decisiones Arquitectónicas Clave (Preview)

- **Arquitectura recomendada:** Modular Monolith (equipo pequeño de 3-5 devs, velocidad inicial, migrable a microservicios).
- **Comparación:** Modular Monolith vs. Microservices.
- **Base de datos:** PostgreSQL con schema-per-module.
- **Message Bus:** RabbitMQ (volumen moderado, routing flexible).
- **Auth:** Auth0 (managed identity provider).
- **API externa:** REST (público) + gRPC potencial (interno futuro).
- **Cache:** Redis para inventario de boletos y sesiones.
- **Frontend:** Next.js (SSR + SPA híbrido).
- **CI/CD:** GitHub Actions + Docker + AWS ECS.
- **Observabilidad:** Grafana + Prometheus + Loki + Tempo.

---

## Estructura del Repositorio Esperada

```
eventpass-architecture/
├── README.md
├── proposals/
│   ├── README.md
│   ├── 01-high-level-architecture.md
│   ├── 02-bounded-contexts.md
│   ├── 03-service-module-decomposition.md
│   └── 04-data-flow-and-interactions.md
├── adrs/
│   ├── README.md
│   ├── ADR-001-deployment-model.md
│   ├── ADR-002-communication-style.md
│   ├── ADR-003-database-strategy.md
│   ├── ADR-004-event-bus.md
│   ├── ADR-005-authentication.md
│   ├── ADR-006-observability.md
│   ├── ADR-007-api-design.md
│   ├── ADR-008-caching-strategy.md
│   ├── ADR-009-deployment-cicd.md
│   └── ADR-010-frontend-architecture.md
├── diagrams/
│   ├── README.md
│   ├── 01-system-context.md
│   ├── 02-bounded-context-map.md
│   ├── 03-data-flow.md
│   └── 04-deployment.md
└── infrastructure/
    ├── README.md
    ├── 01-api-gateway.md
    ├── 02-cdn.md
    ├── 03-load-balancer.md
    ├── 04-cache.md
    ├── 05-message-queue.md
    ├── 06-primary-database.md
    ├── 07-object-storage.md
    ├── 08-auth-provider.md
    ├── 09-payment-provider.md
    ├── 10-notification-system.md
    ├── 11-observability-stack.md
    └── 12-cicd-pipeline.md
```

---

## TAREAS (Backlog para Claude Code)

Ejecutar estas tareas en orden. Cada tarea produce uno o más archivos Markdown. Todos los diagramas deben ser Mermaid embebido en Markdown. Todos los documentos deben ser consistentes entre sí (misma terminología, mismas decisiones, mismos nombres de contextos y servicios).

---

### FASE 0 — Setup del Repositorio

#### Tarea 0.1: Crear estructura de carpetas
- Crear todas las carpetas: `proposals/`, `adrs/`, `diagrams/`, `infrastructure/`.
- Crear archivos README.md vacíos en cada carpeta (se llenarán al final).
- NO crear el README.md raíz todavía.

---

### FASE 1 — Proposals (`/proposals`) — 4 puntos

#### Tarea 1.1: `proposals/01-high-level-architecture.md`
Comparar **Modular Monolith** vs. **Microservices** aplicados específicamente a EventPass.

**Requisitos obligatorios:**
- Narrativa de cada enfoque explicando cómo se aplicaría al proyecto EventPass (no descripciones genéricas).
- Tabla comparativa con EXACTAMENTE estas dimensiones: Deployment complexity, Team size fit, Horizontal scalability, Fault isolation, Development speed (initial), Development speed (long term), Infrastructure cost, Operational complexity.
- Recomendación clara: **Modular Monolith**, justificada con: equipo de 3-5 devs, fase de producto temprana (MVP), dominio aún en descubrimiento, posibilidad de migrar módulos a servicios independientes si la escala lo requiere.
- El tono debe ser de un arquitecto que toma una decisión real, no un ensayo académico.

#### Tarea 1.2: `proposals/02-bounded-contexts.md`
Definir los **7 bounded contexts** con DDD. Para CADA contexto incluir:

| Campo | Requerimiento |
|-------|--------------|
| Name | Nombre corto y significativo del dominio |
| Purpose | 1-2 oraciones sobre la capacidad de negocio que posee |
| Core Entities | Mínimo 3 entidades con campos clave (nombre, tipo, constraints) |
| Domain Events Published | Eventos en pasado (ej: `TicketReserved`, `OrderPlaced`) |
| Domain Events Consumed | Eventos de otros contextos a los que reacciona |
| Upstream Contexts | De quién depende |
| Downstream Contexts | Quién depende de él |
| Integration Pattern | OHS/PL, ACL, CF, SK, o Partnership |

**Contextos a definir:**

1. **Identity & Access** (Generic Subdomain)
   - Entities: User (id: UUID, email: string UNIQUE, passwordHash: string, role: enum[BUYER,ORGANIZER,ADMIN], status: enum[ACTIVE,SUSPENDED], createdAt: timestamp), Profile (id: UUID, userId: FK, displayName: string, avatarUrl: string?), OrganizerVerification (id: UUID, userId: FK, businessName: string, taxId: string, status: enum[PENDING,VERIFIED,REJECTED])
   - Events Published: UserRegistered, UserVerified, OrganizerApproved, OrganizerRejected
   - Events Consumed: (ninguno — es upstream puro)
   - Downstream: todos los demás contextos
   - Integration: OHS/PL (Open Host Service con Published Language)

2. **Event Management** (Core Domain)
   - Entities: Event (id: UUID, organizerId: FK, title: string NOT NULL, description: text, categoryId: FK, venueId: FK, startsAt: timestamp, endsAt: timestamp, status: enum[DRAFT,PUBLISHED,CANCELLED,COMPLETED], maxCapacity: int > 0), Venue (id: UUID, name: string, address: string, city: string, country: string, latitude: decimal, longitude: decimal, capacity: int), Category (id: UUID, name: string UNIQUE, slug: string UNIQUE)
   - Events Published: EventCreated, EventPublished, EventCancelled, EventCompleted
   - Events Consumed: OrganizerApproved (de Identity)
   - Upstream: Identity & Access
   - Downstream: Catalog & Discovery, Ticketing
   - Integration: Partnership (con Ticketing), OHS/PL (hacia Catalog)

3. **Catalog & Discovery** (Supporting Subdomain)
   - Entities: EventListing (id: UUID, eventId: FK, title: string, category: string, city: string, startsAt: timestamp, minPrice: decimal, maxPrice: decimal, availableTickets: int, thumbnailUrl: string), SearchIndex (eventId: FK, keywords: text[], location: point, dateRange: tsrange), FeaturedEvent (id: UUID, eventListingId: FK, priority: int, startsAt: timestamp, endsAt: timestamp)
   - Events Published: (ninguno — es read model)
   - Events Consumed: EventPublished, EventCancelled, TicketInventoryUpdated
   - Upstream: Event Management, Ticketing
   - Downstream: (ninguno — sirve al frontend directamente)
   - Integration: ACL (Anti-Corruption Layer contra Event Management y Ticketing)

4. **Ticketing** (Core Domain)
   - Entities: TicketType (id: UUID, eventId: FK, name: string, price: decimal >= 0, currency: enum[USD,EUR,GTQ], totalQuantity: int > 0, availableQuantity: int >= 0, salesStart: timestamp, salesEnd: timestamp), Ticket (id: UUID, ticketTypeId: FK, orderId: FK?, status: enum[AVAILABLE,RESERVED,SOLD,CANCELLED,USED], qrCode: string UNIQUE, reservedUntil: timestamp?), SeatReservation (id: UUID, ticketId: FK, seatLabel: string, expiresAt: timestamp)
   - Events Published: TicketReserved, TicketReservationExpired, TicketSold, TicketCancelled, TicketInventoryUpdated, TicketValidated
   - Events Consumed: EventPublished, EventCancelled, OrderConfirmed, OrderCancelled
   - Upstream: Event Management, Order & Checkout
   - Downstream: Catalog & Discovery
   - Integration: Partnership (con Event Management y Order & Checkout)

5. **Order & Checkout** (Core Domain)
   - Entities: Order (id: UUID, buyerId: FK, status: enum[PENDING,CONFIRMED,CANCELLED,REFUNDED], totalAmount: decimal, currency: enum[USD,EUR,GTQ], createdAt: timestamp, confirmedAt: timestamp?), OrderItem (id: UUID, orderId: FK, ticketTypeId: FK, quantity: int > 0, unitPrice: decimal, subtotal: decimal), Cart (id: UUID, buyerId: FK, expiresAt: timestamp, items: OrderItem[])
   - Events Published: OrderCreated, OrderConfirmed, OrderCancelled, RefundRequested
   - Events Consumed: TicketReserved, TicketReservationExpired, PaymentSucceeded, PaymentFailed
   - Upstream: Identity & Access, Ticketing, Payment
   - Downstream: Ticketing, Payment, Notification
   - Integration: Partnership (con Ticketing y Payment)

6. **Payment** (Supporting Subdomain)
   - Entities: PaymentIntent (id: UUID, orderId: FK, amount: decimal, currency: enum[USD,EUR,GTQ], stripePaymentIntentId: string, status: enum[CREATED,PROCESSING,SUCCEEDED,FAILED,REFUNDED], createdAt: timestamp), Refund (id: UUID, paymentIntentId: FK, amount: decimal, reason: string, stripeRefundId: string, status: enum[PENDING,COMPLETED,FAILED]), OrganizerPayout (id: UUID, organizerId: FK, eventId: FK, amount: decimal, platformFee: decimal, stripeTransferId: string, status: enum[PENDING,COMPLETED,FAILED])
   - Events Published: PaymentSucceeded, PaymentFailed, RefundCompleted, PayoutCompleted
   - Events Consumed: OrderCreated, RefundRequested, EventCompleted
   - Upstream: Order & Checkout
   - Downstream: Notification
   - Integration: ACL (contra Stripe API externa)

7. **Notification** (Generic Subdomain)
   - Entities: NotificationTemplate (id: UUID, type: enum[EMAIL,SMS,PUSH], eventType: string, subject: string?, bodyTemplate: text), NotificationLog (id: UUID, recipientId: FK, channel: enum[EMAIL,SMS,PUSH], status: enum[QUEUED,SENT,DELIVERED,FAILED], sentAt: timestamp?, error: string?), NotificationPreference (id: UUID, userId: FK, channel: enum[EMAIL,SMS,PUSH], enabled: boolean)
   - Events Published: NotificationSent, NotificationFailed
   - Events Consumed: UserRegistered, OrderConfirmed, PaymentFailed, EventCancelled, TicketValidated, RefundCompleted
   - Upstream: todos los contextos que emiten eventos
   - Downstream: (ninguno — es terminal)
   - Integration: ACL (contra SendGrid, Twilio)

**Al final del documento**, incluir un **Context Map** en diagrama Mermaid que muestre todos los contextos con sus relaciones y patrones de integración etiquetados.

#### Tarea 1.3: `proposals/03-service-module-decomposition.md`
Mostrar la organización concreta del código para el Modular Monolith recomendado:

- **Directory tree** del codebase completo mostrando la estructura de módulos.
- Para cada módulo: qué posee, qué expone (API pública del módulo), y a qué bounded context mapea.
- Explicar cómo se refuerzan las fronteras de contexto: no hay imports cruzados entre módulos, cada módulo expone solo interfaces públicas, comunicación entre módulos vía eventos internos o interfaces definidas.
- Estructura sugerida (TypeScript/Node.js):

```
eventpass/
├── src/
│   ├── modules/
│   │   ├── identity/          → Identity & Access BC
│   │   │   ├── public/        → Interfaces públicas (DTOs, contracts)
│   │   │   ├── internal/      → Lógica interna (services, repos, entities)
│   │   │   ├── infrastructure/ → Adaptadores (Auth0, DB)
│   │   │   └── index.ts       → Re-exports públicos únicamente
│   │   ├── events/            → Event Management BC
│   │   ├── catalog/           → Catalog & Discovery BC
│   │   ├── ticketing/         → Ticketing BC
│   │   ├── orders/            → Order & Checkout BC
│   │   ├── payments/          → Payment BC
│   │   └── notifications/     → Notification BC
│   ├── shared/
│   │   ├── kernel/            → Value objects compartidos, tipos base
│   │   ├── events/            → Event bus interno (in-process)
│   │   └── infrastructure/    → DB connection, logging, config
│   ├── api/
│   │   ├── rest/              → Controllers REST por módulo
│   │   └── middleware/        → Auth, rate limiting, error handling
│   └── main.ts
├── docker-compose.yml
├── Dockerfile
└── package.json
```

#### Tarea 1.4: `proposals/04-data-flow-and-interactions.md`
Describir mínimo **4 flujos end-to-end** (el assignment pide 3, hacemos 4 para máxima nota):

**Flujo 1: Registro de usuario y verificación de organizador**
- Buyer se registra → email de verificación → activa cuenta.
- Organizer se registra → verificación de email → solicita aprobación como organizador → Admin aprueba → OrganizerApproved event.
- Happy path + failure: email de verificación expira y se reenvía.
- Diagrama de secuencia Mermaid.

**Flujo 2: Compra de boletos (transacción core)**
- Buyer busca evento → selecciona tickets → se crea reserva temporal (10 min TTL) → checkout → se crea orden → payment intent con Stripe → Stripe webhook confirma pago → orden confirmada → tickets marcados SOLD → email de confirmación con QR.
- Happy path + failure: pago rechazado → reserva liberada → inventario restaurado.
- Diagrama de secuencia Mermaid (dividido en 2 partes si > 6 participantes).

**Flujo 3: Procesamiento de pago fallido y retry**
- Payment intent falla → PaymentFailed event → Order queda en PENDING → se notifica al buyer → buyer reintenta → nuevo payment intent → éxito o cancelación definitiva.
- Happy path + failure: 3 reintentos fallidos → orden cancelada automáticamente.
- Diagrama de secuencia Mermaid.

**Flujo 4: Cancelación de evento y reembolso masivo**
- Organizer cancela evento → EventCancelled event → todas las órdenes activas se marcan para refund → se procesan reembolsos en batch → RefundCompleted events → notificaciones a cada buyer.
- Happy path + failure: reembolso falla en Stripe → se reintenta con backoff exponencial → si falla 3 veces, se escala a admin.
- Diagrama de secuencia Mermaid.

**Requisitos de cada diagrama:**
- Máximo 6 participantes por diagrama. Si se excede, dividir en partes.
- Flechas etiquetadas con protocolo y payload: `POST /orders {buyerId, items}`.
- Mostrar request y response para llamadas síncronas.
- Mostrar publish/consume para pasos asíncronos.
- Incluir al menos un failure path por flujo.

#### Tarea 1.5: `proposals/README.md`
Crear un índice de la carpeta proposals con una tabla que enlace a cada documento y una descripción de una línea.

---

### FASE 2 — ADRs (`/adrs`) — 4 puntos

Cada ADR DEBE seguir EXACTAMENTE este formato:

```markdown
# ADR-XXX: [Decision Title]

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-04-10 |
| Deciders | Estuardo Sabán |

## Context
[Problema específico en EventPass. NO descripciones genéricas.]

## Options Considered

### Option 1: [Name]
| Pros | Cons |
|------|------|
| ... | ... |

### Option 2: [Name]
| Pros | Cons |
|------|------|
| ... | ... |

## Decision
[Cuál opción y por qué encaja específicamente en EventPass.]

## Consequences
[Qué habilita, qué previene, qué riesgos y cómo se mitigan.]

## Related ADRs
[Links a ADRs relacionados.]
```

**REGLAS CRÍTICAS:**
- Cada ADR debe tener mínimo 2 opciones genuinas con pros/cons honestos y específicos al proyecto.
- La justificación DEBE referenciar constraints concretos de EventPass: equipo de 3-5 devs, carga esperada (~5K usuarios concurrentes en flash sales, ~500 en operación normal), presupuesto de startup, dominio de ticketing.
- Las consecuencias deben reconocer riesgos reales y cómo se mitigan. NO solo listar beneficios.
- Todos los ADRs deben ser consistentes entre sí. No pueden contradecirse.

#### Tarea 2.1: `adrs/ADR-001-deployment-model.md`
- **Tema:** Modular Monolith vs. Microservices
- **Decisión:** Modular Monolith
- **Opciones:** (1) Modular Monolith, (2) Microservices
- **Justificación:** Equipo de 3-5 devs, fase MVP, dominio aún en descubrimiento, menor overhead operacional. Los módulos pueden extraerse a servicios cuando métricas lo justifiquen.
- **Consecuencias:** Habilita desarrollo rápido; previene escalamiento independiente de módulos; riesgo de acoplamiento accidental mitigado con boundaries estrictos y linting.
- **Related:** ADR-002, ADR-003, ADR-009

#### Tarea 2.2: `adrs/ADR-002-communication-style.md`
- **Tema:** Sync (HTTP REST) vs. Async (Events) — cuándo usar cada uno
- **Opciones:** (1) Primariamente síncrono con REST, eventos solo para notificaciones, (2) Event-driven con REST solo para queries, (3) Híbrido: REST para commands/queries, eventos para side-effects y cross-module communication
- **Decisión:** Opción 3 — Híbrido
- **Ejemplos concretos:** REST sync para: browse events, view ticket availability, create order. Async para: payment confirmation (webhook), ticket QR generation, email/SMS notifications, catalog index update, analytics ingestion.
- **Related:** ADR-001, ADR-004, ADR-007

#### Tarea 2.3: `adrs/ADR-003-database-strategy.md`
- **Tema:** Shared DB vs. DB-per-service vs. Shared DB con schema-per-module
- **Opciones:** (1) Single shared database y schema, (2) Schema-per-module en un PostgreSQL compartido, (3) Database-per-service
- **Decisión:** Opción 2 — Schema-per-module (un PostgreSQL, cada módulo tiene su schema: `identity.users`, `ticketing.tickets`, etc.)
- **Justificación:** Mantiene isolation lógica sin overhead de múltiples servidores DB. Cross-module queries solo vía APIs públicas del módulo, nunca joins directos entre schemas.
- **Related:** ADR-001, ADR-004

#### Tarea 2.4: `adrs/ADR-004-event-bus.md`
- **Tema:** RabbitMQ vs. Kafka vs. AWS SQS/SNS
- **Opciones:** (1) RabbitMQ, (2) Apache Kafka, (3) AWS SQS + SNS
- **Decisión:** RabbitMQ
- **Justificación:** Volumen moderado (~1K events/min en pico de flash sale), routing flexible con exchanges, menor complejidad que Kafka para el equipo, soporte nativo de dead-letter queues para manejo de errores, menor costo en fase temprana. En el monolito modular, el event bus interno es in-process; RabbitMQ se usa para background jobs y comunicación que necesita durabilidad.
- **Related:** ADR-001, ADR-002, ADR-003

#### Tarea 2.5: `adrs/ADR-005-authentication.md`
- **Tema:** Auth0 vs. Keycloak (self-hosted) vs. Custom JWT implementation
- **Opciones:** (1) Auth0 (managed), (2) Keycloak (self-hosted), (3) Custom con bcrypt + JWT
- **Decisión:** Auth0
- **Justificación:** Equipo pequeño no debe invertir tiempo en auth infrastructure. Auth0 maneja MFA, social login, brute force protection. Free tier cubre hasta 7K MAU (suficiente para MVP). Roles RBAC: BUYER, ORGANIZER, ADMIN configurados en Auth0 y propagados via JWT claims.
- **Related:** ADR-001, ADR-007

#### Tarea 2.6: `adrs/ADR-006-observability.md`
- **Tema:** Grafana stack (Prometheus + Loki + Tempo) vs. Datadog vs. ELK Stack
- **Opciones:** (1) Grafana + Prometheus + Loki + Tempo, (2) Datadog, (3) ELK (Elasticsearch + Logstash + Kibana)
- **Decisión:** Grafana stack
- **Justificación:** Open source, sin costo por volumen, equipo tiene experiencia con Prometheus. SLOs a monitorear: p99 latency < 200ms para búsquedas, p99 < 500ms para checkout, disponibilidad > 99.9%.
- **Related:** ADR-009

#### Tarea 2.7: `adrs/ADR-007-api-design.md`
- **Tema:** REST vs. GraphQL vs. gRPC — para APIs externas e internas
- **Opciones:** (1) REST para todo, (2) GraphQL para frontend + REST para webhooks, (3) REST externo + gRPC interno
- **Decisión:** REST para todo (fase actual del monolito modular)
- **Justificación:** En un monolito modular, no hay comunicación inter-servicio por red (todo es in-process). REST es suficiente para la API pública. GraphQL añade complejidad innecesaria para un frontend con flujos bien definidos. gRPC se reserva como opción futura si se extraen módulos a servicios.
- **Related:** ADR-001, ADR-002, ADR-005

#### Tarea 2.8: `adrs/ADR-008-caching-strategy.md`
- **Tema:** Redis vs. Memcached vs. Application-level cache
- **Opciones:** (1) Redis, (2) Memcached, (3) In-memory cache (Node.js LRU)
- **Decisión:** Redis
- **Datos a cachear:** Inventario de tickets (TTL 5s para consistencia eventual en flash sales), sesiones de usuario, listados de eventos populares (TTL 60s), resultados de búsqueda frecuentes (TTL 30s). **Invalidación:** TTL-based para reads, write-through para inventario de tickets.
- **Justificación:** Soporte para estructuras de datos avanzadas (sorted sets para rankings, pub/sub para invalidación, atomic decrement para inventario). Memcached solo soporta key-value simple.
- **Related:** ADR-003, ADR-007

#### Tarea 2.9: `adrs/ADR-009-deployment-cicd.md`
- **Tema:** Containerización, orquestación y pipeline de CI/CD
- **Opciones:** (1) Docker + AWS ECS Fargate + GitHub Actions, (2) Docker + Kubernetes (EKS), (3) Serverless (AWS Lambda)
- **Decisión:** Docker + AWS ECS Fargate + GitHub Actions
- **Justificación:** Fargate elimina gestión de servidores, GitHub Actions se integra nativamente con el repo. Kubernetes es overkill para un monolito modular con equipo pequeño. Lambda no es adecuado para un monolito con conexiones persistentes a DB.
- **Pipeline:** lint → test → build → push image to ECR → deploy to staging → smoke tests → deploy to production. Rollback: ECS permite blue-green deployment nativo.
- **Related:** ADR-001, ADR-006

#### Tarea 2.10: `adrs/ADR-010-frontend-architecture.md`
- **Tema:** Next.js (SSR/hybrid) vs. React SPA (Vite) vs. Astro (content-first)
- **Opciones:** (1) Next.js App Router (SSR + client), (2) React SPA con Vite, (3) Astro con React islands
- **Decisión:** Next.js App Router
- **Justificación:** SEO crítico para páginas de eventos (descubrimiento orgánico). SSR para catálogo y páginas de evento, client-side para checkout y dashboard de organizador. State management: React Server Components + Zustand para estado del cliente. API consumption: Server Components hacen fetch directo al backend; cliente usa React Query para mutaciones.
- **Related:** ADR-007, ADR-008

#### Tarea 2.11: `adrs/README.md`
Crear tabla resumen de todos los ADRs con columnas: #, Título, Estado, Resumen de una línea.

---

### FASE 3 — Diagramas (`/diagrams`) — 3 puntos

**REGLAS:**
- Todos los diagramas en **Mermaid** embebido en Markdown.
- Máximo **10 nodos** por diagrama. Si se excede, dividir.
- Cada diagrama DEBE tener un **párrafo explicativo** debajo.
- Flechas con dirección y etiqueta significativa.
- Consistente con proposals y ADRs (mismos nombres, mismas tecnologías).

#### Tarea 3.1: `diagrams/01-system-context.md` (C4 Level 1)
- EventPass como una sola caja central.
- Actores humanos: Buyer, Organizer, Admin.
- Sistemas externos: Stripe, SendGrid, Twilio, Google Maps, Auth0, Cloudflare CDN.
- Flechas etiquetadas con dirección y naturaleza: "purchases tickets via HTTPS", "receives payment webhooks from", "sends transactional emails via API", etc.
- NO mostrar componentes internos.
- Párrafo explicativo.

#### Tarea 3.2: `diagrams/02-bounded-context-map.md` (DDD Context Map)
- Los 7 bounded contexts agrupados por tipo de dominio: Core (Event Management, Ticketing, Order & Checkout), Supporting (Catalog & Discovery, Payment), Generic (Identity & Access, Notification).
- Flechas upstream → downstream con patrón de integración etiquetado (OHS/PL, ACL, Partnership, CF, SK).
- Leyenda.
- Párrafo explicativo.

#### Tarea 3.3: `diagrams/03-data-flow.md` (Sequence Diagrams)
Mínimo **3 sequence diagrams** (de los 4 flujos de proposals):

1. **Ticket Purchase — Happy Path** (max 6 participants: Buyer, API Gateway, Orders Module, Ticketing Module, Payment Module, Stripe)
2. **Ticket Purchase — Payment Failure** (mismos participants, mostrando el path de fallo)
3. **Event Cancellation & Mass Refund** (Organizer, API Gateway, Events Module, Orders Module, Payment Module, Notification Module)

- Cada flecha etiquetada con protocolo y payload.
- Request y response para llamadas síncronas.
- Publish/consume para mensajes asíncronos.

#### Tarea 3.4: `diagrams/04-deployment.md` (Deployment / Infrastructure)
- Zonas de red: Public Internet, VPC pública, VPC privada, Managed Services.
- Componentes: Cloudflare CDN → ALB → ECS Fargate (EventPass Monolith) → PostgreSQL, Redis, RabbitMQ.
- Sistemas externos: Stripe, SendGrid, Twilio, Auth0.
- Observabilidad: Prometheus, Loki, Tempo → Grafana.
- S3 para assets estáticos y tickets QR.
- Etiquetas de tecnología en cada componente.
- Párrafo explicativo de decisiones de topología.

#### Tarea 3.5: `diagrams/README.md`
Índice con tabla de diagramas y descripción de una línea cada uno.

---

### FASE 4 — Infrastructure Components (`/infrastructure`) — 2 puntos

Cada archivo DEBE contener las **5 secciones obligatorias:**

1. **Responsibilities** — 3-5 bullets de lo que hace este componente en EventPass (NO definición genérica).
2. **Why This Project Needs It** — Específico al dominio de ticketing.
3. **Technology Choice** — Tabla comparativa de tu elección vs. 2 alternativas con dimensiones: Managed/Self-hosted, Operational complexity, Cost at your scale, Key differentiating feature.
4. **Trade-offs** — Tabla de 2 columnas: Advantages vs. Disadvantages (honestos).
5. **Integration** — Cómo se conecta con otros componentes; referencia al diagrama de deployment.

#### Tarea 4.1: `infrastructure/01-api-gateway.md`
- **Elección:** Kong Gateway (OSS, self-hosted en ECS)
- **Alternativas:** AWS API Gateway, Traefik
- **Responsabilidades en EventPass:** Rate limiting en endpoints de compra para flash sales, routing, request validation, JWT verification (delegada a Auth0), CORS.

#### Tarea 4.2: `infrastructure/02-cdn.md`
- **Elección:** Cloudflare
- **Alternativas:** AWS CloudFront, Fastly
- **Responsabilidades en EventPass:** Servir assets del frontend (Next.js static), cachear imágenes de eventos, DDoS protection durante flash sales.

#### Tarea 4.3: `infrastructure/03-load-balancer.md`
- **Elección:** AWS Application Load Balancer (ALB)
- **Alternativas:** Nginx (self-managed), HAProxy
- **Responsabilidades en EventPass:** Distribución de tráfico a instancias ECS, health checks, SSL termination.

#### Tarea 4.4: `infrastructure/04-cache.md`
- **Elección:** Redis (AWS ElastiCache)
- **Alternativas:** Memcached, KeyDB
- **Responsabilidades en EventPass:** Cache de inventario de tickets, sesiones, listados populares. Atomic operations para decrement de inventario en flash sales.

#### Tarea 4.5: `infrastructure/05-message-queue.md`
- **Elección:** RabbitMQ (Amazon MQ)
- **Alternativas:** Apache Kafka, AWS SQS+SNS
- **Responsabilidades en EventPass:** Procesamiento async de pagos, generación de tickets QR, envío de notificaciones, actualización del índice de búsqueda.

#### Tarea 4.6: `infrastructure/06-primary-database.md`
- **Elección:** PostgreSQL (AWS RDS)
- **Alternativas:** MySQL (RDS), MongoDB Atlas
- **Responsabilidades en EventPass:** Almacenamiento principal de todos los módulos con schema isolation. ACID para operaciones de inventario y órdenes.

#### Tarea 4.7: `infrastructure/07-object-storage.md`
- **Elección:** AWS S3
- **Alternativas:** Google Cloud Storage, MinIO (self-hosted)
- **Responsabilidades en EventPass:** Imágenes de eventos, QR codes de tickets, assets del frontend, backups de DB.

#### Tarea 4.8: `infrastructure/08-auth-provider.md`
- **Elección:** Auth0
- **Alternativas:** Keycloak, AWS Cognito
- **Responsabilidades en EventPass:** Login/register, MFA, social login, JWT issuance, RBAC (BUYER/ORGANIZER/ADMIN).

#### Tarea 4.9: `infrastructure/09-payment-provider.md`
- **Elección:** Stripe
- **Alternativas:** PayPal, Adyen
- **Responsabilidades en EventPass:** Payment intents, webhooks de confirmación, refunds, Connect para payouts a organizadores, PCI compliance delegada.

#### Tarea 4.10: `infrastructure/10-notification-system.md`
- **Elección:** SendGrid (email) + Twilio (SMS)
- **Alternativas:** AWS SES + SNS, Mailgun + Vonage
- **Responsabilidades en EventPass:** Emails transaccionales (confirmación de compra, e-tickets), SMS para recordatorios de eventos, templates dinámicos.

#### Tarea 4.11: `infrastructure/11-observability-stack.md`
- **Elección:** Grafana + Prometheus + Loki + Tempo
- **Alternativas:** Datadog, ELK Stack
- **Responsabilidades en EventPass:** Métricas de p99 latency, logs estructurados, traces distribuidos (si se migra a servicios), dashboards de ventas en tiempo real, alertas de SLO.

#### Tarea 4.12: `infrastructure/12-cicd-pipeline.md`
- **Elección:** GitHub Actions + Docker + AWS ECR + ECS
- **Alternativas:** GitLab CI, CircleCI
- **Responsabilidades en EventPass:** Build, test, deploy automatizado. Blue-green deployments. Rollback automático si health check falla.

#### Tarea 4.13: `infrastructure/README.md`
Tabla resumen de los 12 componentes con columnas: #, Componente, Tecnología, Descripción de una línea.

---

### FASE 5 — README raíz y READMEs de subcarpetas

#### Tarea 5.1: `README.md` (raíz del repositorio)
Debe incluir:
1. **Nombre y descripción** (2-3 oraciones): EventPass es una plataforma de ticketing para eventos en vivo que conecta organizadores con compradores. Maneja el ciclo completo desde la publicación de eventos hasta la entrega de boletos digitales con código QR.
2. **Enfoques comparados** — Una oración por cada enfoque con link a la propuesta.
3. **Enfoque recomendado** — Una oración con link a la justificación completa.
4. **Tabla de navegación** — Link a CADA carpeta y CADA documento con descripción de una línea.

---

### FASE 6 — Verificación Final

#### Tarea 6.1: Consistencia cruzada
Revisar que:
- Los nombres de bounded contexts sean IDÉNTICOS en proposals, adrs, diagrams e infrastructure.
- Las tecnologías elegidas en ADRs coincidan con las de infrastructure.
- Los diagramas reflejen las decisiones documentadas en proposals y ADRs.
- Los domain events mencionados en bounded contexts aparezcan en los data flows.
- Todos los links internos entre documentos funcionen.
- No haya archivos vacíos ni placeholders.

#### Tarea 6.2: Checklist de rúbrica 100%

**Proposals (4 pts):**
- [ ] 4 documentos completos con profundidad
- [ ] 2 enfoques genuinamente comparados con razonamiento específico al proyecto
- [ ] 6+ bounded contexts bien definidos con campos tipados en entidades
- [ ] 3+ flujos con happy path + failure path
- [ ] Eventos de dominio en pasado, descriptivos, conectan contextos sin acoplamiento

**ADRs (4 pts):**
- [ ] 10 ADRs (5 requeridos + 5 extra) — supera el mínimo
- [ ] ≥2 opciones reales por decisión con pros/cons honestos
- [ ] Justificación referencia constraints específicos (equipo, carga, dominio)
- [ ] Consecuencias reconocen riesgos y mitigación
- [ ] Consistencia interna entre todos los ADRs

**Diagrams (3 pts):**
- [ ] 4 diagramas presentes con explicaciones escritas
- [ ] ≤10 nodos por diagrama, flechas etiquetadas con dirección
- [ ] Consistente con proposals y ADRs
- [ ] Flujos largos divididos en múltiples diagramas

**Infrastructure (2 pts):**
- [ ] 12 componentes (supera el mínimo de 8)
- [ ] Las 5 secciones completas en cada componente
- [ ] Tecnología justificada para la escala del proyecto
- [ ] Trade-offs honestos
- [ ] Integración conectada al diagrama de deployment

**Repo & README (2 pts):**
- [ ] Tabla de navegación completa con todos los links
- [ ] Naming consistente
- [ ] Cada carpeta tiene README
- [ ] Cero placeholders
- [ ] Calidad profesional

---

## Notas para Claude Code

- **Idioma de los documentos:** Inglés (es un proyecto de maestría en Software Architecture).
- **Tono:** Profesional, como documentación interna de un equipo de ingeniería real.
- **Mermaid:** Usar siempre bloques de código ```mermaid para diagramas.
- **Consistencia es la prioridad #1.** Cada decisión debe ser rastreable entre documentos.
- **No usar descripciones genéricas.** Todo debe ser específico a EventPass y su dominio de ticketing.
- **Entidades deben tener campos tipados** (nombre: tipo, constraints).
- **Domain events en pasado:** `OrderPlaced`, NO `PlaceOrder`.
- **Ejecutar las fases en orden** porque cada fase depende de las decisiones de la anterior.
