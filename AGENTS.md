# AGENTS.md — rules for every AI agent in this repo

Wallet Copilot is an AI-assisted digital wallet backend. Requirements: `docs/PRD.md`. Design: `docs/TECHNICAL_DESIGN.md`. Decisions: `docs/adr/`.

## 1. Commands

| Task | Command |
| --- | --- |
| Run locally (starts Docker Compose) | `./mvnw spring-boot:run` |
| Run with demo data | `./mvnw spring-boot:run -Dspring-boot.run.profiles=demo` |
| All tests (no LLM) | `./mvnw verify` |
| Tests that call Ollama | `./mvnw verify -Pllm` |
| One test class | `./mvnw test -Dtest=WalletTest` |

## 2. Architecture rules (do not break)

1. `wallet.domain` has no Spring, JPA, Jackson or Lombok imports. Pure Java 21 (records, sealed types welcome).
2. Controllers call inbound ports (use cases), never repositories.
3. Other modules (`assistant`, `support`, `fraud`, `mcp`) use `wallet` only through its ports.
4. `@Transactional` lives on application services only.
5. Money is `Money` (BigDecimal, scale 2). Never `double` or `float` for amounts.
6. Every schema change is a new Flyway migration. Never edit a migration that is already on `main`.
7. The LLM never generates SQL and never chooses a wallet id. Tools are bound to the wallet id from the request.
8. No secrets in the repo. Config comes from `application.yml` and environment variables.

## 3. Conventions

- Package by feature, then by layer (see TECHNICAL_DESIGN §2).
- Errors: domain exceptions → `ProblemDetail` in `shared.web`. Problem `type` values from TECHNICAL_DESIGN §5.
- Tests: JUnit 5 + AssertJ. Test names: `methodName_condition_expectedResult` or `@DisplayName` in plain English.
- Integration tests use Testcontainers with `@ServiceConnection`. Never depend on the developer's local Docker Compose.
- LLM-calling tests are tagged `@Tag("llm")` and excluded from the default build.
- Commits: Conventional Commits (`feat(wallet): ...`, `test(...)`, `fix(...)`, `docs: ...`, `build: ...`).
- Code, comments, docs and commit messages in English.

## 4. Roles

Development uses four AI agents with separate responsibilities. The maintainer is the tech lead and architect: makes every final decision, writes critical domain code and merges.

| Role | Owns | Must not |
| --- | --- | --- |
| Maintainer (human) | Final requirement and architecture decisions, critical domain code, code review, merge | — |
| Product Owner agent | `docs/PRD.md`, tasks and their acceptance criteria (Given / When / Then), priorities | Write code or tests; make architecture decisions |
| Architect agent | Design options with trade-offs, ADR drafts (`Proposed`), architecture review of pull requests | Write production code; mark an ADR `Accepted` |
| Test agent | `src/test/**`; writes tests from the acceptance criteria before the implementation | Change files under `src/main/**` |
| Developer agent | `src/main/**`, `pom.xml`, migrations, configuration | Delete, weaken, skip or `@Disabled` tests written by the Test agent |

Role details and prompts: `docs/agents/`.

## 5. Workflow

```
1. Product Owner → task with acceptance criteria; maintainer approves
2. Architect     → if the task needs a design decision: options + ADR draft; maintainer decides
3. Feature branch is created
4. Test agent    → writes failing tests for every acceptance criterion
5. Developer     → implements until ./mvnw verify is green
6. Test agent    → full run, adds edge cases, summarises results for the pull request
7. Architect     → reviews the pull request against section 2
8. Maintainer    → reviews the diff, CI green, merges to main
```

If the Developer agent thinks a test is wrong, it stops and explains why. A test is changed only with the maintainer's approval.

## 6. Definition of done (every task)

- [ ] All acceptance criteria have automated tests
- [ ] `./mvnw verify` green locally and in CI
- [ ] Architecture rules in §2 respected (Architect agent review)
- [ ] No TODOs without a linked issue
- [ ] Pull request describes what changed and how it was tested
- [ ] The change is understood and reviewed by the maintainer, not just generated
