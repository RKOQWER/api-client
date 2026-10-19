## Goal
Add adversarial robustness testing for LLM-backed APIs so jailbreaks, prompt injection, and data leakage are caught and locked in as regression checks.

## Build
- LLM target configuration — supports OpenAI-compatible endpoints, Responses API endpoints, and custom adapter templates; system-prompt storage stays in the secrets vault
- Version-1 attack library — prompt injection (direct, indirect, payload splitting, multimodal, multilingual), jailbreak strategies (single-shot encoded and multi-turn Crescendo/TAP/Skeleton Key), PII extraction, system-prompt leakage, bad output handling, excessive agency, and reputation probes
- Composition framework shaped after PyRIT — converters, probes, orchestrators, and scorers for building composable attacks
- Local-first scoring and plain-text replay — every attack runs and is scored locally by default, with zero external network calls
- Replayable proof capsules — each successful attack saves request template, payload sequence, expected violation class, and seed so it can be re-run to prove the finding
- Safety-regression rail — when the app resists an attack, that success becomes a permanent monitored check that must keep passing
- Evil-MCP scanning (first version) — detects tool-poisoning in MCP servers exposed by the client
- Reporting mapped to OWASP LLM Top 10 (2025/2026) with severity, attack class, evidence capsule, and remediation hint; diffable between red-team cycles

## Acceptance Criteria
- A red-team run against a deliberately weak sample LLM app produces verified findings with replayable capsules
- Humans approve findings through ACP
- The safety-regression rail triggers when a guardrail is deliberately weakened
- All probes run within budget and rate caps, making zero external network calls by default

## Gate
Reports generate and can be diffed between two red-team cycles, the probes stay within rate limits, and no prompt text leaks into the session timeline logs.
