# Wallet Copilot — Technical Design

Read together with [PRD.md](PRD.md). Decisions are recorded in [adr/](adr/).

## 1. Stack

| Area | Choice |
| --- | --- |
| Language | Java 21 language level (runs on JDK 21 or 25) |
| Framework | Spring Boot 4.1.x, Spring Framework 7 (see ADR-001) |
| AI | Spring AI 2.0.x, Ollama locally |
| Models | Chat: `qwen2.5:7b` (good tool calling; use `llama3.2:3b` if RAM < 16 GB). Embeddings: `nomic-embed-text` (768 dimensions) |
| Database | PostgreSQL 17 + pgvector (`pgvector/pgvector:pg17`) |
| Migrations | Flyway |
| Messaging | Apache Kafka 3.9 (KRaft, single node locally) |
| Tests | JUnit 5, AssertJ, Mockito, Testcontainers, `@ServiceConnection` |
| Build / CI | Maven Wrapper, GitHub Actions |

Exact patch versions are pinned in `pom.xml`.

## 2. Modules

One deployable, organised as modules by feature. `wallet` uses strict hexagonal layering; the other modules use the same idea with fewer layers.

```
com.walletcopilot
├── wallet        core: wallets, transactions, transfers (hexagonal)
│   ├── domain                  Wallet, Money, WalletId, Transaction, Transfer — pure Java
│   ├── application/port/in     use case interfaces
│   ├── application/port/out    LoadWalletPort, SaveWalletPort, TransactionQueryPort, ...
│   ├── application/service     use case implementations, @Transactional boundary
│   ├── adapter/in/web          REST controllers, DTOs
│   └── adapter/out/persistence JPA entities, repositories, mappers
├── shared
│   ├── outbox                  outbox table, publisher, event envelope
│   ├── idempotency             Idempotency-Key handling
│   └── web                     ProblemDetail handler
├── assistant     spending assistant: ChatClient, tools, chat memory
├── support       RAG: ingestion, vector store, support chat
├── fraud         Kafka consumer, rules, LLM explanation
└── mcp           MCP server tools (read-only)
```

Rule: modules talk to `wallet` only through its inbound ports (use cases or query ports), never through its JPA repositories.

## 3. Domain model

```
Wallet (aggregate root)
  id: WalletId (UUID)       ownerId: String        currency: Currency
  balance: Money            status: ACTIVE | FROZEN | CLOSED
  version: long             createdAt: Instant
  deposit(Money)            pay(Money, Merchant)    debit/credit for transfers
  invariants: balance >= 0, same currency, only ACTIVE moves money

Money (value object, record)
  amount: BigDecimal (scale 2, HALF_EVEN)   currency: Currency
  add, subtract, isGreaterThan — reject mixed currencies

Transaction (entity, immutable)
  id, walletId, type: DEPOSIT | PAYMENT | TRANSFER_IN | TRANSFER_OUT
  amount, balanceAfter, merchantName?, category?, transferId?, createdAt

Transfer
  id, idempotencyKey, sourceWalletId, targetWalletId, amount
  status: COMPLETED | FAILED, failureReason?, createdAt
```

Categories (enum): `GROCERIES, RESTAURANTS, TRANSPORT, SHOPPING, BILLS, ENTERTAINMENT, TRAVEL, HEALTH, OTHER`.

## 4. Database (Flyway)

| Migration | Tables |
| --- | --- |
| V1__wallet.sql | `wallet`, `wallet_transaction` |
| V2__transfer_idempotency.sql | `transfer`, `idempotency_record` |
| V3__outbox.sql | `outbox_event`, `processed_event` |
| V4__chat_memory.sql | Spring AI JDBC chat memory table (or let Spring AI initialise it; to be decided) |
| V5__fraud.sql | `suspicious_activity` |
| V6__vector_store.sql | `CREATE EXTENSION vector` + `vector_store` (768 dims, HNSW index) |

Key constraints:

- `wallet.version` for optimistic locking; `CHECK (balance >= 0)` as a last line of defence.
- `idempotency_record(key, operation)` unique; stores the response so a retry returns the same body.
- `wallet_transaction(wallet_id, created_at DESC)` index for history and assistant queries.
- `outbox_event(published_at) WHERE published_at IS NULL` partial index.

## 5. REST API

All errors: `application/problem+json` (RFC 9457). Money as string in JSON: `{"amount": "150.00", "currency": "TRY"}`.

| Method | Path | Notes |
| --- | --- | --- |
| POST | `/api/wallets` | body `{ownerId, currency}` → 201 |
| GET | `/api/wallets/{walletId}` | |
| POST | `/api/wallets/{walletId}/deposits` | header `Idempotency-Key`; body `{amount}` |
| POST | `/api/wallets/{walletId}/payments` | header `Idempotency-Key`; body `{amount, merchantName, category}` |
| POST | `/api/transfers` | header `Idempotency-Key`; body `{sourceWalletId, targetWalletId, amount}` |
| GET | `/api/wallets/{walletId}/transactions` | `from, to, type, category, page, size` |
| POST | `/api/wallets/{walletId}/assistant/chat` | body `{conversationId, message}` → `{answer, conversationId}` |
| POST | `/api/support/chat` | body `{conversationId, question}` → `{answer, sources[]}` |
| POST | `/api/admin/knowledge/reindex` | re-ingest `src/main/resources/knowledge/*.md` |
| GET | `/api/wallets/{walletId}/suspicious-activities` | |

Error codes: `wallet-not-found` 404, `insufficient-balance` 422, `currency-mismatch` 422, `wallet-not-active` 409, `idempotency-key-reused` 409 (same key, different body), `concurrent-update` 409 (after retries), `validation-error` 400, `ai-unavailable` 503.

## 6. Money movement and concurrency

- One transfer = one database transaction: load both wallets, apply domain methods, save both, insert two transactions, insert outbox events, save idempotency record.
- Optimistic locking (`@Version`). On `OptimisticLockingFailureException` the use case retries up to 3 times with small backoff (Spring Framework 7 `@Retryable`, ADR-003).
- Idempotency: check `idempotency_record` first; insert the record in the same transaction as the money movement so both commit or both roll back. A unique constraint violation on a race means "already processed" → return stored response.
- Concurrency test: 50 parallel payments on one wallet whose balance covers only 20 → exactly 20 succeed, balance is 0.00, never negative.

## 7. Events (outbox + Kafka)

```
use case tx ──► wallet + wallet_transaction + outbox_event (same commit)
OutboxPublisher (@Scheduled, every 1 s)
   SELECT ... WHERE published_at IS NULL ORDER BY id LIMIT 100 FOR UPDATE SKIP LOCKED
   send to Kafka, wait for ack, set published_at
```

| Topic | Key | Payload |
| --- | --- | --- |
| `wallet.transaction-created` | walletId | eventId, transactionId, walletId, type, amount, currency, merchantName, category, occurredAt |
| `wallet.transaction-created.DLT` | walletId | failed events after retries |

- Key by walletId keeps per-wallet ordering (one partition per key).
- Consumers store `eventId` in `processed_event` in the same transaction as their work → idempotent.
- Spring Kafka `DefaultErrorHandler` with exponential backoff (3 attempts) and `DeadLetterPublishingRecoverer`.

## 8. AI features

### 8.1 Spending assistant (tool calling)

```
POST /assistant/chat ─► AssistantService
   ChatClient
     .system(spending-assistant prompt: scope, language, refusal rules)
     .advisors(MessageChatMemoryAdvisor(conversationId))
     .tools(new SpendingTools(walletId))   ◄─ walletId fixed by the server
     .call()
SpendingTools (@Tool methods, read-only, use wallet query ports)
   spendingByCategory(from, to)    topMerchants(from, to, limit)
   listTransactions(from, to, category, limit)   currentBalance()
```

Security: tools are created per request with the path `walletId`; the LLM cannot pass another wallet id. Tool outputs are capped (max 50 rows). Today's date is passed in the system prompt so "last month" is resolved correctly.

### 8.2 Support assistant (RAG)

- Knowledge base: 5–6 markdown files in `src/main/resources/knowledge/` (fees, limits, transfers, account freeze, FAQ). Written by us, fictional company "Wallet Copilot".
- Ingestion: read markdown → split into chunks (~500 tokens, overlap) → embed with `nomic-embed-text` → `PgVectorStore`. Metadata: `source` file name, `section` heading.
- Query: `QuestionAnswerAdvisor` (or `RetrievalAugmentationAdvisor`), top-k 4, similarity threshold ~0.6. Return `sources` from retrieved document metadata.
- Empty retrieval → fixed answer "I couldn't find this in our help documents."
- Evaluation: `src/test/resources/rag-eval.json` with 10 questions and expected source; a `@Tag("llm")` test reports hit rate.

### 8.3 Suspicious activity

```
Kafka wallet.transaction-created ─► FraudConsumer
   rules R1–R3 (pure Java, unit tested; R2/R3 read recent history via query port)
   if flagged ─► ExplanationService: ChatClient.entity(RiskAssessment.class)
                 record RiskAssessment(RiskLevel riskLevel, String explanation, String recommendedAction)
              ─► on timeout/error: rule-based explanation, riskLevel from rule
   save suspicious_activity + processed_event in one transaction
```

### 8.4 MCP server

- Spring AI MCP server starter (WebMVC, streamable HTTP / SSE transport as supported by the chosen version).
- Tools: `get_wallet`, `list_transactions`, `spending_summary`, `list_suspicious_activities`. Read-only, reuse query ports.
- Manual test: connect from Claude Desktop; automated test: MCP client in an integration test.

### 8.5 Testing AI code

| Level | How |
| --- | --- |
| Unit | Tools and rules tested as plain Java |
| Slice | `ChatModel` mocked; assert the prompt, tool registration and response mapping |
| LLM tests | `@Tag("llm")`, real Ollama, run manually (`./mvnw verify -Pllm`), not in CI |

## 9. Observability and configuration

- Actuator: `health`, `info`, `metrics`, `prometheus`.
- Spring AI observations give `gen_ai.client.operation` timers and token usage; add custom counters: `assistant.requests`, `fraud.flags{rule}`, `outbox.lag`.
- Profiles: `default` (local, Docker Compose), `test` (Testcontainers), `demo` (seed data generator).
- Config keys under `wallet-copilot.*` with `@ConfigurationProperties` records (fraud thresholds, outbox interval, AI timeouts).

## 10. CI

GitHub Actions on push and PR: JDK 21, `./mvnw -B verify` (Testcontainers runs on the GitHub runner), upload test reports. LLM-tagged tests excluded.
