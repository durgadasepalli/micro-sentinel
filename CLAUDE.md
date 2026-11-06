# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Current state

The repo is at the idea/scaffolding stage. It contains docs and Claude config only (`README.md`, `CLAUDE.md`, `.claude/commands/review.md`). There is no source code, build files, tests or lint config yet, so there are no build, test or lint commands to document. Add them here once the Gradle build is set up.

## Tech stack

Matches nsai-deterministic-gate:

- Java 21
- Spring Boot 3
- Gradle with the Groovy DSL (`build.gradle`, `settings.gradle`, not Kotlin DSL)

## What this project is

micro-sentinel is a planned AI/LLM gateway for Java applications. It provides rate limits, token budgets, PII redaction (on prompts and responses) and a structured audit log for every model call. Policy decisions are delegated to [nsai-deterministic-gate](https://github.com/durgadasepalli/nsai-deterministic-gate).

## Design intent (from README)

These constraints should guide the architecture when code is added:

- **Java-native and embeddable**: a Spring Boot starter that runs in-process, with an optional standalone mode. It is not a separate hosted service like LiteLLM, Portkey or Kong AI Gateway.
- **Deterministic policy**: allow/deny decisions come from nsai-deterministic-gate rules, never from another LLM call.
- **Output policy checks (decision)**: deterministic only (regex, schema, allowlist, nsai-gate rules). LLM-judged checks are out of scope for v1. If added later, they are opt-in, advisory and never block on their own.
- **Provider-neutral**: Claude, Gemini, OpenAI and local models (Ollama) sit behind one interface.
- **Token-aware controls**: budgets are per user, team, app and model, measured in tokens rather than request counts. Errors must clearly say "budget exceeded".
- **Audit record fields**: who, model, tokens, cost, decision, rule hit.
- **Metrics**: tokens, cost, latency and block rate per app and model.

Non-goals for now: hosted SaaS or UI console, prompt management or versioning, fine-tuning or evaluation tooling.

## Repo notes

- License: Apache 2.0, to match nsai-deterministic-gate.
