# ADR-010: Frontend Architecture

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-04-10 |
| Deciders | Estuardo Sabán |

## Context

EventPass's frontend serves three user types with distinct needs:

- **Buyers:** Browse events (SEO-critical — organic search is a primary acquisition channel), search and filter, view event details, purchase tickets through a checkout flow, view order history and QR codes.
- **Organizers:** Create and manage events, configure ticket types and pricing, view sales dashboards and analytics, manage payouts.
- **Admins:** Moderate platform content, approve organizer verifications, manage disputes, monitor system metrics.

The frontend must balance **SEO** (event catalog pages must be indexable by search engines for discoverability), **performance** (sub-3-second load on mobile/3G), and **interactivity** (real-time ticket availability during flash sales, dynamic checkout flow).

The backend exposes a REST API (ADR-007) consumed by the frontend via Server Components (SSR) and client-side fetch (React Query for mutations and real-time updates).

## Options Considered

### Option 1: Next.js App Router (SSR + Client Components)

React meta-framework with server-side rendering, static generation, and client-side interactivity in a single application.

| Pros | Cons |
|------|------|
| SSR for SEO-critical pages — event listings and detail pages rendered on the server, indexable by search engines | Learning curve for App Router's server/client component mental model — team must understand the boundary |
| React Server Components (RSC) — fetch data on the server without client-side JavaScript, reducing bundle size | Deployment complexity — requires a Node.js server (not purely static), though Vercel/ECS handle this |
| Built-in image optimization, code splitting, and font loading | Tighter coupling to Vercel's ecosystem for optimal performance (though self-hosting on ECS is fully supported) |
| Streaming SSR — progressive page loading, critical for event detail pages with images and ticket availability | Server Components cannot use React hooks or browser APIs — requires careful component architecture |
| File-system routing — pages map naturally to EventPass's URL structure | Over-abstraction risk — RSC + client components + middleware can create complex data flow |

### Option 2: React SPA with Vite

Client-side React application bundled with Vite, served as static files.

| Pros | Cons |
|------|------|
| Simplest development model — everything runs in the browser, no server/client boundary | No SSR — event pages are not indexable by search engines without additional tooling (prerendering) |
| Fast development iteration — Vite's HMR is near-instant | All data fetching happens client-side — longer Time to Interactive, especially on mobile/3G |
| Lightweight deployment — static files served from CDN (Cloudflare) | SEO requires adding a prerendering service (e.g., Prerender.io) — additional infrastructure and cost |
| No Node.js server to manage | Initial JavaScript bundle must include React + all route code (code splitting helps but doesn't eliminate the issue) |
| Full control over rendering — no framework conventions to follow | No streaming — the entire page waits for all data before rendering |

### Option 3: Astro with React Islands

Content-first static site generator with interactive React components hydrated on demand.

| Pros | Cons |
|------|------|
| Excellent performance — pages are static HTML by default, JavaScript only for interactive islands | Not designed for highly interactive applications — checkout flow and organizer dashboard require many islands |
| Built-in SSG for event catalog pages — pre-rendered HTML is SEO-optimal | Island architecture adds complexity for shared state (e.g., cart state across components) |
| Zero JavaScript by default — only interactive components ship JS | React island hydration can cause layout shifts if not carefully managed |
| Multi-framework support — could use Svelte for some islands, React for others | Less mature ecosystem for complex SPA patterns — routing, auth, state management |
| Excellent for content-heavy pages (event descriptions, blog, help center) | Dynamic pages (search results, real-time ticket availability) require server-side rendering or client-side fetch |

## Decision

**Next.js App Router** (deployed as a containerized Node.js application on ECS alongside the backend, or on Vercel for optimized delivery).

EventPass requires both SEO (event catalog, event detail pages) and rich interactivity (checkout flow, organizer dashboard, real-time ticket availability). Next.js App Router uniquely provides both through React Server Components (SSR for SEO) and Client Components (interactivity for checkout and dashboards).

### Page Architecture

| Page | Rendering Strategy | Why |
|------|-------------------|-----|
| Event catalog (`/events`) | Server Component (SSR) | SEO-critical; data fetched from Catalog API on the server |
| Event detail (`/events/:id`) | Server Component (SSR) + Client island for ticket selection | SEO for event info; interactive ticket picker needs client state |
| Search results (`/search`) | Server Component (SSR) with streaming | SEO for search pages; streaming shows results progressively |
| Checkout (`/checkout`) | Client Component | Fully interactive; Stripe Elements requires client-side JavaScript |
| Order history (`/orders`) | Server Component (SSR) | No SEO needed, but SSR reduces client bundle; authenticated route |
| Organizer dashboard (`/dashboard/*`) | Client Component | Highly interactive; charts, forms, real-time data |
| Admin panel (`/admin/*`) | Client Component | Internal tool; no SEO needed; rich interactivity |

### State Management

- **Server state:** React Server Components fetch from the REST API directly — no client-side cache needed for initial page load.
- **Client mutations:** React Query (TanStack Query) for data fetching on the client, optimistic updates, and cache invalidation after mutations (e.g., after placing an order, invalidate the cart query).
- **Client UI state:** Zustand for lightweight client-side state (cart items, UI toggles, form state). No global state management library needed — RSC handles most data flow.

### API Consumption Pattern

```
Server Component → fetch('http://localhost:3000/v1/catalog/events')
                   (server-to-server, no CORS, no auth header needed for public data)

Client Component → useQuery('/v1/orders', { headers: { Authorization: `Bearer ${token}` } })
                   (client-to-server, needs CORS, JWT token from Auth0)
```

## Consequences

**Enables:**
- SEO for event discovery — event catalog and detail pages are server-rendered HTML, indexable by Google, Bing, and social media crawlers (Open Graph meta tags).
- Fast initial load — Server Components send HTML immediately; client JavaScript hydrates only interactive parts (ticket picker, checkout form).
- Progressive loading — streaming SSR shows event details progressively while ticket availability loads.
- Code organization — file-system routing maps naturally to EventPass's URL structure (`/events`, `/checkout`, `/dashboard`, `/admin`).

**Prevents:**
- Purely static deployment — Next.js requires a Node.js server for SSR, which adds an ECS task or Vercel deployment. Cannot be served purely from a CDN.
- Simple mental model — the server/client component boundary requires developers to understand which components run where.

**Risks and Mitigations:**
- **Risk:** Server Component latency if the backend API is slow — SSR waits for data before sending HTML. **Mitigation:** Use streaming SSR with Suspense boundaries; show skeleton UI while slow data loads; cache popular pages in Redis (ADR-008).
- **Risk:** Hydration mismatch — server-rendered HTML does not match client-rendered output, causing UI glitches. **Mitigation:** Follow Next.js best practices for server/client component boundaries; test SSR output in CI.
- **Risk:** Vendor coupling to Next.js — migration away would require significant refactoring. **Mitigation:** Business logic lives in the backend API; the frontend is a presentation layer. Replacing Next.js with another framework means rewriting pages and components, not business logic.

## Related ADRs

- [ADR-007: API Design](ADR-007-api-design.md) — REST API consumed by Server Components (SSR) and React Query (client)
- [ADR-008: Caching Strategy](ADR-008-caching-strategy.md) — Redis caches API responses to speed up SSR data fetching
