---
description: Review the current changes against the micro-sentinel checklist
---

Review the current changes against this checklist. Get them with `git status` and `git diff HEAD` (staged and unstaged). If the repo has no commits yet, `git diff HEAD` fails, so read the staged, modified and untracked files directly instead. Never treat an empty diff as "clean" without checking `git status`. Report only real findings, each with `file:line`, what is wrong and a suggested fix. If a category is clean, say so in one line.

## Java 21 conventions
- Use records for immutable data carriers (requests, decisions, audit entries) instead of hand-written classes.
- Use sealed interfaces and pattern-matching `switch` for closed sets such as policy decisions and provider results.
- Prefer `var` only where the type is obvious. Use text blocks and `List.of`/`Map.of` where they fit.
- No raw types, no unchecked casts without justification, no `System.out` (use SLF4J).

## Null safety
- Public APIs must not return `null`. Use `Optional` for absent results, and never use `Optional` as a field or parameter.
- Validate inputs at boundaries with `Objects.requireNonNull` and a message.
- Check any `@Nullable` use for consistency and look for unguarded dereferences.

## No secrets in code
- No API keys, tokens, passwords or provider credentials in source, tests, fixtures or config files.
- Credentials come from environment or Spring configuration properties.
- Prompts, responses and PII must not be logged in plaintext. Logs and audit entries carry redacted content only.
- Redaction fails closed: if the redactor throws or errors, the call is blocked and audited. Raw content must never fall through to the provider, logs or audit entries.

## Tests for new logic
- Every new branch of behaviour (rate limit, token budget, redaction, policy decision) has a test.
- Cover failure paths: "budget exceeded", denied by policy, provider error.
- Policy tests are deterministic. Nothing depends on a live LLM or the network.

## Audit-log fields
- Every model call produces an audit entry with: who, model, tokens, cost, decision, rule hit.
- Blocked and errored calls are audited too, not only successful ones.
- The decision and rule hit are explicit values, not free text. Field names are consistent across code paths.
- No raw PII in audit entries.

## Project rules
- Policy allow/deny decisions come from nsai-deterministic-gate, never from another LLM call.
- Provider-specific code stays behind the provider-neutral interface.
- Gradle build uses the Groovy DSL.
