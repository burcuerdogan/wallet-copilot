# ADR-001: Spring Boot 4.1 and Spring AI 2.0

Status: Proposed

## Context
The project started on Spring Boot 3.5. Spring AI 2.0.x (current stable) supports Spring Boot 4.0.x and 4.1.x. Boot 3.5 is near or past the end of its open-source support.

## Decision
Upgrade to the latest Spring Boot 4.1.x and Spring AI 2.0.x before any business code exists.

## Alternatives
- Stay on Boot 3.5 with Spring AI 1.1.x: less migration work, but the project would start on an older generation of the stack.

## Consequences
- Boot 4 is modular: some features need their own starter (for example Flyway). We generate a reference `pom.xml` from start.spring.io and compare.
- Jackson 3 is the default JSON library in Boot 4; keep this in mind for custom serializers.
- Notable Boot 3 → 4 changes we run into are recorded in this ADR.
