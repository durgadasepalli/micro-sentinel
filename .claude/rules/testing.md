---
paths:
  - "**/*Test.java"
  - "src/test/**"
---

# Testing conventions

1. **JUnit 5 only.** Use `org.junit.jupiter.api` (`@Test`, `@ParameterizedTest`, `@Nested`). No JUnit 4 imports or runners.
2. **One behavior per test, named given/when/then.** Each test asserts a single behavior. Name it like `givenBudgetExhausted_whenCallModel_thenRejectsWithBudgetExceeded`.
3. **No real network or model calls in unit tests.** Use fakes for providers, nsai-deterministic-gate and the clock. Unit tests must be deterministic and run offline.
