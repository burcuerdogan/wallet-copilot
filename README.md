# Wallet Copilot

An AI-assisted digital wallet backend built with Java 21, Spring Boot and Spring AI.

## Goals

- A clean wallet core: accounts, balances and transfers with idempotency and optimistic locking
- Reliable event publishing with the outbox pattern and Kafka
- AI features: a spending assistant (tool calling), a support assistant (RAG with pgvector) and an MCP server

## Architecture

Hexagonal (ports and adapters):

```
adapter/in/web  ->  application/port/in  ->  application/service  ->  domain
                                               |
                          application/port/out  <-  adapter/out/persistence
```

## Documentation

- [Product requirements](docs/PRD.md)
- [Technical design](docs/TECHNICAL_DESIGN.md) and [ADRs](docs/adr/)
- [Rules for AI agents](AGENTS.md) and [agent roles](docs/agents/)

## Running locally

Requirements: Java 21, Docker, Ollama.

```
./mvnw spring-boot:run   (or run WalletCopilotApplication from the IDE)
```

Spring Boot starts the containers in `compose.yaml` automatically.
Health check: http://localhost:8080/actuator/health

## How I used AI in development

_To be written as the project grows._
