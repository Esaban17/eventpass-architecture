# Diagrams

Architecture diagrams for EventPass. All diagrams use Mermaid notation (rendered natively by GitHub) with a maximum of 10 nodes per diagram. Each diagram includes an explanatory section below it.

## Documents

| # | Document | Type | Description |
|---|----------|------|-------------|
| 1 | [System Context](01-system-context.md) | C4 Level 1 | EventPass as a black box with 3 actors and 6 external systems |
| 2 | [Bounded Context Map](02-bounded-context-map.md) | DDD Context Map | 7 bounded contexts grouped by domain type with integration patterns |
| 3 | [Data Flow](03-data-flow.md) | Sequence Diagrams | 3 flows: ticket purchase (happy + failure), event cancellation & mass refund |
| 4 | [Deployment](04-deployment.md) | Infrastructure | 4 network zones with ECS, RDS, ElastiCache, Amazon MQ, and observability stack |

## Diagram Notation

All diagrams use [Mermaid](https://mermaid.js.org/) syntax embedded in Markdown code blocks. GitHub renders these automatically — no external tools needed.

- **Graph diagrams** (`graph TB`): Used for system context, context map, and deployment topology.
- **Sequence diagrams** (`sequenceDiagram`): Used for data flow interactions between components.
- **Arrow labels**: Include protocol (SYNC/ASYNC/HTTP) and payload description.
- **Maximum 10 nodes**: Complex diagrams are split into multiple smaller diagrams for readability.
