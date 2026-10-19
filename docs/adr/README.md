# Architecture Decision Records

| ADR | Decision | Status |
| --- | --- | --- |
| 001 | Spring Boot 4.1 + Spring AI 2.0 instead of Boot 3.5 | Proposed |
| 002 | Hexagonal architecture for the wallet core, feature modules in one deployable | Planned |
| 003 | Optimistic locking + retry for concurrent balance updates | Planned |
| 004 | Transactional outbox instead of publishing to Kafka inside the transaction | Planned |
| 005 | Ollama local models; provider swappable through Spring AI | Planned |
| 006 | LLM uses server-bound tools, never generates SQL | Planned |

Template: Status (Proposed / Accepted / Superseded) → Context → Options considered → Decision → Consequences. Max one page.

The Architect agent drafts ADRs as `Proposed`; the maintainer decides and sets `Accepted`.
