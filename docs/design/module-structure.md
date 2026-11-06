# micro-sentinel module structure

## Context

micro-sentinel is an embeddable Java LLM gateway: rate limits, token budgets, PII redaction and an audit log around every model call, with policy decisions delegated to nsai-deterministic-gate. The repo has docs only (no code). This plan fixes the module layout, the main interfaces and the request flow before any code is written. Stack: Java 21, Spring Boot 3, Gradle Groovy DSL.

Decisions already made:
- `PolicyClient` is an interface from day one. v1 ships only the in-process implementation, backed by `io.github.durgadasepalli:nsai-gate-spring-boot-starter` from Maven Central. An HTTP implementation comes later.
- The policy client fails closed: if the gate cannot decide (error, timeout, missing rule), the call is blocked.
- Output policy checks are deterministic only (regex, schema, allowlist, nsai-gate rules). No LLM-judged checks in v1.
- Redaction fails closed.

## Modules (Gradle multi-project)

```
micro-sentinel/
  settings.gradle              include 'sentinel-core', 'sentinel-policy', 'sentinel-spring-boot-starter'
  build.gradle                 shared Java 21 toolchain, JUnit 5, repositories
  sentinel-core/               no Spring dependency
  sentinel-policy/             PolicyClient + in-process gate adapter
  sentinel-spring-boot-starter/ autoconfiguration, properties, Micrometer wiring
```

Dependency direction: `starter -> policy -> core`, and `starter -> core`. Core depends on nothing from Spring or the gate, so it can be tested with fakes.

### sentinel-core (package `io.github.durgadasepalli.sentinel.core`)

Owns the pipeline and everything except policy transport.

- `ModelCall` (record): caller identity (user, team, app), model id, provider id, prompt messages, estimated input tokens.
- `ModelResult` (record): response content, input/output tokens, cost.
- `Decision` (sealed interface): `Allow`, `Deny(reason, ruleHit)`, `Error(cause)`. `Error` is treated as a block.
- `ModelProvider`: `ModelResult invoke(ModelCall call)`. Provider-neutral. Claude, Gemini, OpenAI and Ollama adapters live outside core (later modules).
- `RateLimiter`: `Decision check(CallerKey key)`. In-memory default; the interface allows a distributed store later.
- `TokenBudget`: `Decision reserve(CallerKey key, String model, long estimatedTokens)`, `void settle(Reservation r, long actualTokens)`. Denial message must say "budget exceeded".
- `Redactor`: `Redacted redact(String text)` returns the redacted text plus the types of PII found. Throwing means fail closed.
- `AuditLog`: `void record(AuditEntry entry)`.
- `AuditEntry` (record): who, model, tokens, cost, decision, ruleHit. `decision` is an enum value, not free text. Contains redacted content only.
- `SentinelGateway`: orchestrates the flow below; single entry point `ModelResult call(ModelCall call)`.

### sentinel-policy (package `...sentinel.policy`)

- `PolicyClient`: `Decision checkInput(PolicyRequest r)`, `Decision checkOutput(PolicyRequest r)`. The only seam to the gate.
- `InProcessPolicyClient`: calls the gate through the API exposed by `nsai-gate-spring-boot-starter`. Wraps every call in try/catch; any exception, timeout or undecided result becomes `Decision.Error`, which blocks (fail closed).
- `PolicyRequest` (record): phase (input/output), redacted content, caller, model, tool-call payload if any.
- Later, not in v1: `HttpPolicyClient` implementing the same interface, selected by property.

### sentinel-spring-boot-starter

- `SentinelAutoConfiguration`: wires default beans (`InMemoryRateLimiter`, `InMemoryTokenBudget`, regex `Redactor`, `Slf4jAuditLog`, `InProcessPolicyClient`, `SentinelGateway`); each `@ConditionalOnMissingBean`.
- `SentinelProperties` (`sentinel.*`): limits, budgets per user/team/app/model, redaction patterns, policy timeout.
- Micrometer metrics: tokens, cost, latency and block rate per app and model.
- `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
- Standalone mode is deferred; the starter is the v1 deliverable.

## Request flow (`SentinelGateway.call`)

1. **Rate limit**: `RateLimiter.check`. Deny -> audit, return error.
2. **Redact input**: `Redactor.redact(prompt)`. Failure -> block, audit.
3. **Token budget**: `TokenBudget.reserve` using estimated tokens. Deny ("budget exceeded") -> audit, return error.
4. **Input policy**: `PolicyClient.checkInput(redacted)`. Deny or Error -> release reservation, audit, block.
5. **Provider call**: `ModelProvider.invoke` with the redacted prompt (or original, if config says only the audit trail is redacted; default: redacted).
6. **Redact output**: `Redactor.redact(response)`. Failure -> block, audit.
7. **Output policy**: `PolicyClient.checkOutput(redacted response)`. Deny or Error -> block, audit.
8. **Settle**: `TokenBudget.settle` with actual tokens; compute cost.
9. **Audit**: `AuditLog.record` runs in a `finally` so every path (allowed, denied, errored) is audited with decision and rule hit.
10. Return `ModelResult` (redacted content) or a typed rejection.

Rules enforced across the flow: every non-Allow decision blocks; the reservation is always released or settled; nothing raw reaches logs or audit entries.

## Critical files to create (first PRs)

- `settings.gradle`, `build.gradle` (root, Groovy DSL, Java 21 toolchain)
- `sentinel-core/build.gradle` and the interfaces/records above
- `sentinel-policy/build.gradle` (depends on `nsai-gate-spring-boot-starter`)
- `sentinel-spring-boot-starter/build.gradle` and autoconfiguration
- Update `CLAUDE.md` with real Gradle commands once the build exists

## Suggested build order

1. Gradle skeleton plus `sentinel-core` records and interfaces.
2. In-memory `RateLimiter`, `TokenBudget`, regex `Redactor`, `AuditLog`; unit tests with fakes.
3. `SentinelGateway` with a fake `PolicyClient`; test every branch, especially fail-closed paths.
4. `InProcessPolicyClient` against the gate starter; test that gate errors block.
5. Starter autoconfiguration and metrics; Spring Boot context test.

## Verification

- `./gradlew test` passes (JUnit 5, given/when/then names, fakes only, no network).
- Gateway tests cover: rate-limited, budget exceeded, redactor throws, policy deny, policy error/timeout (must block), output deny, provider error. Each asserts an audit entry with the right decision and rule hit and no raw PII.
- Starter test: `ApplicationContextRunner` shows default beans load and each can be overridden.
- Verify the gate artifact's actual API before step 4; the adapter signature is unconfirmed.

## Open item

Confirm the exact API exposed by `nsai-gate-spring-boot-starter` (class names, sync vs async, how "no decision" is signaled) when implementing `InProcessPolicyClient`. I haven't inspected it.
