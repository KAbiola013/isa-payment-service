# ISA Payment Service

Spring Boot service for recording and validating ISA maturity instructions.

## What it does

When a member's ISA matures, they choose one of four options:

1. Reinvest the whole balance into a new ISA product
2. Add more money and reinvest
3. Withdraw some funds and reinvest the rest
4. Withdraw everything

## Phases

- **Phase 1:** validate and record instructions. No money moves. Spring Boot, Spring Data JPA, PostgreSQL, Flyway, Testcontainers, GitHub Actions CI.
- **Phase 2:** carry instructions out: Open Banking payment for "add money" (mocked provider, WireMock), payouts for withdrawals, payment states, idempotency.
- **Phase 3:** deploy to AWS.

The phase 1 spec lives in `docs/phase-1-spec.md`. Read it before changing behaviour.

## Decisions

- ISA allowance limits are configuration per tax year, never hardcoded (they change with Budgets).
- Validation lives on request DTOs, not JPA entities.
- Unexpected errors are logged in full but return a generic 500 to the client.

## Conventions

- Test first: write a failing test, confirm it fails for the expected reason, then fix.
- One concern per branch and PR. Branch from an up-to-date `main`.
- UK English in docs and messages.