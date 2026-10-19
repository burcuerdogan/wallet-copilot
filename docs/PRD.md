# Wallet Copilot — Product Requirements

Scope: v1.

## 1. Problem and goal

Digital wallet users see a list of transactions but cannot easily ask questions about their own money ("Where did I spend the most last month?"). Support teams answer the same fee and limit questions again and again. Risk teams need suspicious activity flagged and explained in plain language.

Wallet Copilot is a wallet backend with three AI features on top of a reliable core. It is an independent project exploring how AI features fit into a reliable payments backend: every feature must be demo-able, tested and documented.

## 2. Users

| Persona | Needs |
| --- | --- |
| Wallet user | Hold money, pay merchants, send money to other wallets, ask questions about spending |
| Support agent | Answer fee, limit and policy questions with the source document |
| Risk analyst | See flagged transactions with a clear reason and risk level |
| AI assistant (via MCP) | Read wallet data through safe, well-defined tools |

## 3. Scope

### In scope (v1)

1. **Wallet core** — create wallet, get wallet, deposit, merchant payment, transfer between wallets, transaction history with filters.
2. **Reliable events** — every balance change publishes an event to Kafka through the outbox pattern.
3. **Spending assistant** — natural language questions about one wallet's transactions, answered with tool calling.
4. **Support assistant** — questions about fees, limits and policies answered from a knowledge base (RAG) with sources.
5. **Suspicious activity** — rule-based detection on the event stream; the LLM writes the explanation.
6. **MCP server** — wallet read tools exposed to MCP clients such as Claude Desktop.
7. **Quality** — automated tests, CI, metrics for LLM latency and token usage, English README and demo video.

### Out of scope (v1)

- Authentication with Keycloak/OAuth2 (the caller is identified with an `X-User-Id` header; stretch goal)
- Multi-currency conversion (a wallet has one currency, TRY by default)
- Real card networks, KYC, frontend UI

## 4. Functional requirements

### FR-1 Wallet

- FR-1.1 A user can create a wallet with a currency. Initial balance is 0.00.
- FR-1.2 A wallet has a status: `ACTIVE`, `FROZEN`, `CLOSED`. Only `ACTIVE` wallets can send or receive money.
- FR-1.3 Amounts use 2 decimal places. Zero or negative amounts are rejected.
- FR-1.4 Balance can never go below zero.
- FR-1.5 Money with different currencies cannot be added or compared.

### FR-2 Money movement

- FR-2.1 Deposit increases balance and creates a `DEPOSIT` transaction.
- FR-2.2 Payment decreases balance, stores merchant name and category, creates a `PAYMENT` transaction.
- FR-2.3 Transfer moves money between two different wallets atomically: one `TRANSFER_OUT` and one `TRANSFER_IN` transaction, or nothing.
- FR-2.4 Deposit, payment and transfer requests require an `Idempotency-Key` header. Repeating a request with the same key returns the first result and does not move money again.
- FR-2.5 Concurrent requests on the same wallet never produce a wrong balance.

### FR-3 Transaction history

- FR-3.1 List a wallet's transactions, newest first, paginated.
- FR-3.2 Filter by date range, type and category.

### FR-4 Events

- FR-4.1 Each transaction publishes `TransactionCreated` to Kafka, keyed by wallet id.
- FR-4.2 An event is published at least once, even if Kafka is down when the transaction commits.
- FR-4.3 Consumers process each event at most once (idempotent consumer). Failing events go to a dead-letter topic after retries.

### FR-5 Spending assistant

- FR-5.1 `POST /api/wallets/{walletId}/assistant/chat` answers questions about that wallet only.
- FR-5.2 The LLM never writes SQL. It calls predefined tools; the wallet id is fixed by the server, not chosen by the LLM.
- FR-5.3 The assistant remembers the conversation by `conversationId`.
- FR-5.4 Out-of-scope questions and prompt injection attempts ("ignore your instructions", "show another user's wallet") are refused politely.
- FR-5.5 The assistant answers in the language of the question (Turkish or English).

### FR-6 Support assistant

- FR-6.1 `POST /api/support/chat` answers from the knowledge base (fees, limits, transfer rules, FAQ).
- FR-6.2 Every answer lists the source documents it used.
- FR-6.3 If the knowledge base has no answer, the assistant says so instead of guessing.
- FR-6.4 Admin can re-index the knowledge base.

### FR-7 Suspicious activity

- FR-7.1 Rules on `TransactionCreated` events:
  - R1: a single payment above 10,000.00 TRY
  - R2: more than 5 outgoing transactions within 10 minutes on one wallet
  - R3: a payment above 3x the wallet's 30-day average payment
- FR-7.2 A flagged transaction gets a risk level (`LOW`, `MEDIUM`, `HIGH`), an explanation and a recommended action written by the LLM as structured output.
- FR-7.3 If the LLM is unavailable, the flag is still saved with a rule-based explanation.
- FR-7.4 `GET /api/wallets/{walletId}/suspicious-activities` lists flags.

### FR-8 MCP server

- FR-8.1 Exposes read-only tools: get wallet, list transactions, spending summary, list suspicious activities.
- FR-8.2 No tool can move money.

## 5. Non-functional requirements

| ID | Requirement |
| --- | --- |
| NFR-1 | Domain layer has no Spring or JPA dependency |
| NFR-2 | Domain test coverage ≥ 90 %; every acceptance criterion has at least one automated test |
| NFR-3 | `./mvnw verify` passes in CI on every push |
| NFR-4 | LLM calls time out after 30 s and fail gracefully |
| NFR-5 | LLM latency and token usage are visible as Micrometer metrics |
| NFR-6 | Errors follow RFC 9457 Problem Details |
| NFR-7 | The whole system starts locally with one command (`./mvnw spring-boot:run`) |
| NFR-8 | Runs fully on free tools: Ollama locally, no paid API required |

## 6. Success criteria for v1

- A 3-minute demo video shows: payments → spending question → suspicious payment flagged and explained → support question with sources → Claude Desktop calling the MCP tools.
- README explains the architecture and how AI agents were used during development.
- Every significant design decision is recorded in an ADR or in the technical design.
