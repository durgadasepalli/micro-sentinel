# micro-sentinel

An AI/LLM gateway for Java applications: rate limits, token budgets, PII redaction and an audit log for every model call, with policy decisions delegated to [nsai-deterministic-gate](https://github.com/durgadasepalli/nsai-deterministic-gate).

> Status: idea / scaffolding. Nothing implemented yet.

## Why

Traditional API gateways meter requests and bytes. LLM traffic needs controls that understand prompts, tokens and model output:

- **Token budgets**, not just request counts (per user, team, app, model)
- **PII redaction** on prompts going out and on responses coming back
- **Policy checks** on model output before it reaches users or tools
- **An audit trail** of who asked what, which model answered, what was blocked and why

## Positioning

Hosted AI gateways (LiteLLM proxy, Portkey, Kong AI Gateway, Envoy AI Gateway) run as separate services. micro-sentinel is meant to be:

- **Java-native and embeddable**: a Spring Boot starter that runs in-process, with an optional standalone mode
- **Deterministic about policy**: allow/deny decisions come from nsai-deterministic-gate rules, not another LLM call
- **Provider-neutral**: Claude, Gemini, OpenAI and local models (Ollama) behind one interface

## Planned capabilities (draft)

1. Rate limiting and token budgets with clear "budget exceeded" errors
2. PII detection and redaction (prompt and response)
3. Policy enforcement via nsai-deterministic-gate (output and tool-call checks)
4. Structured audit log (who, model, tokens, cost, decision, rule hit)
5. Metrics: tokens, cost, latency, block rate per app and model

## Decision: output policy checks

Output and tool-call policy checks are deterministic only: regex, schema, allowlist and nsai-deterministic-gate rules. LLM-judged checks are out of scope for v1. If added later, they will be opt-in, advisory and never block on their own.

## Non-goals (for now)

- A hosted SaaS or UI console
- Prompt management or prompt versioning
- Model fine-tuning or evaluation tooling

## License

Apache 2.0 (to match nsai-deterministic-gate).
