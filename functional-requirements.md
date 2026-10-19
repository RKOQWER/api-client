# Agent-Native API Client - Functional Requirements

**Status:** Draft v0.1 (October 2026)
**Working name:** TBD (tentative: "Leo")
**Document owner:** Product spec - single source of truth for what we build and in what order.

---

## 1. Product Thesis

**The client for the era where agents write, review, and consume APIs.**

Agents now write 30% more code than humans, with no matching rise in review capacity or spec updates ("silent spec-code drift", arXiv:2606.27045). APIs drift, contracts rot, and every incumbent API client (Postman, Bruno, Requestly, Hoppscotch, Thunder Client, Yaak) still assumes a human clicking Send.

Our client treats collections as **source code**, agents as **first-class users**, and contract/safety violations as **build-breaking signals**. Three pillars:

1. **Local-first, Git-native, agent-exposed** - everything lives in plain-text files reviewable in PRs; a built-in MCP server + ACP agent chat lets Claude Code / Cursor / Gemini CLI discover, run, and extend collections.
2. **Assurance rails nobody ships today** - contract drift detection vs OpenAPI, local scheduled monitors, micro load testing; findings become permanent, versioned regression checks.
3. **A neutral protocol-adapter kernel** - the client implements HTTP as its thin core, everything else (gRPC, MQTT, MCP, A2A, WebSocket-shaped protocols) arrives as *imported, reviewed, signed adapters*, reviewed via the same PR flow as API changes.

Plus two security extensions built on the same kernel (Sections 7 and 8):

4. **Inbuilt penetration testing** - Nuclei-powered deterministic scanning plus a Strix-style verified exploit loop, gated by explicit authorization.
5. **LLM jailbreak / prompt-injection red teaming** - adversarial robustness testing for LLM-backed APIs.

### Why now

- **Market credibility vacuum**: Postman is cloud-locked, account-forced, expensive per seat, and pivoted to AI credits over core client quality (developer exodus: [HN, $5.6B raise discussion](https://news.ycombinator.com/item?id=39702610); [Requestly comparison](https://requestly.com/alternatives/postman-alternative/)). Bruno is beloved but deliberately minimal: no monitors ([open issue #9553](https://github.com/usebruno/bruno/issues)), no mock server ([#3071](https://github.com/usebruno/bruno/issues/3071)), no load testing ([#1569](https://github.com/usebruno/bruno/issues/1569), "coming soon" on their own site), Git-only collaboration ([third-party gap analysis](https://apidog.com/blog/bruno-limitations)). Requestly owns interception but is proprietary and account-walled.
- **Agentic demand exists.** MCP is the de facto tool protocol ([official spec revisions through 2026](https://modelcontextprotocol.io/docs/2026-07-28/learn/architecture)); A2A reached v1.0 under the Linux Foundation in April 2026 ([announcement](https://a2a-protocol.org/)). Postman already validates appetite with its MCP server ([mcp.postman.com](https://www.postman.com/product/mcp-server)).
- **Security + safety tooling is hot.** AI pentest OSS exploded in 2026 (Strix ~69k stars, PentAGI ~28k stars), and LLM red teaming has mature OSS engines (promptfoo ~26k stars, PyRIT, garak) - Section 7-8 build directly on them.

---

## 2. Terminology

| Term | Meaning |
|---|---|
| **Collection** | A directory tree of plain-text files: requests, environments, scripts, docs, monitor configs. Diffable in Git. |
| **Session** | One execution of a request: envelopes (request, response headers/body/chunks, errors), timeline events, duration, and provenance. Persisted as snapshots. |
| **Adapter** | An installable module that implements one protocol on top of the session kernel. Declares capabilities + permission scopes in a manifest. |
| **Finding** | A verified violation/observation with replayable evidence (drift, perf regression, vuln, jailbreak success). |
| **Proof capsule** | A Finding bundled with everything needed to replay it: request template, payload sequence, environment, assertions. Versioned in Git. |
| **Monitor** | A scheduled run of a collection/folder that evaluates assertions, latency budgets, and findings - locally or in the user's own CI cluster. |
| **Agent rail** | MCP server and/or ACP chat surface allowing coding agents to operate on collections and findings. |

---

## 3. Requirement Catalog

Priority scale: **P0** = table stakes, ship-gap; **P1** = differentiators; **P2** = agent-native; **P3** = adapter kernel; **P4** = security; **P5** = LLM jailbreak. Within each P, "In order" = recommended implementation sequence.

### 3.1 P0 - Table Stakes (cannot ship without)

**In order:**

- **FR-100 Request Engine**: REST, GraphQL, WebSocket, SSE; gRPC + MCP request types. Methods, headers, query, body, cookies, TLS client certs, proxy support. Latency/timeline/event view for all of them.
- **FR-101 Collections + Environments as plain-text files**: Bruno-style Git-native storage; JSON Schema validated; secrets flagged and excluded from VCS; per-request/folder/collection inheritance for vars, auth, scripts.
- **FR-102 Auth Suite**: Bearer, Basic, API key, OAuth 2.0 (incl. PKCE + auto-refresh), AWS SigV4; one-click flows; per-request override.
- **FR-103 Scripting + Assertions**: pre-request/post-response JavaScript (sandboxed), `bru`-like API, request chaining, data-driven test runs; Chai-style assertions.
- **FR-104 Import/Export**: Postman, Bruno (.bru + OpenCollection YAML), Insomnia, OpenAPI/Swagger, cURL, HAR, WSDL. Migration friction is the #1 adoption blocker - invest here early.
- **FR-105 Response Inspector**: formatted JSON/YAML/XML/HTML, diff view, pretty-print, headers/timeline/timing waterfall, save-as-example.
- **FR-106 Client Apps**: Bruno proved demand with v4 Apps - custom HTML/JS/React UIs inside a request response pane, sandboxed webview, code stored in the collection file ([docs](https://docs.usebruno.com/apps/overview)). Ship the same primitive: `apps/` folder + sandboxed bridge, code travels with the collection via Git.
- **FR-107 CLI Runner**: headless execution of collections/folders; JSON + JUnit + TAP output; CI-native exit codes.

### 3.2 P1 - Differentiators (the local-first wedge)

**In order:**

- **FR-200 Local Scheduled Monitors**: run any collection on a cron while the app is open or via CLI in the user's own CI/CD (no vendor cloud). Latency p50/p95/p99 budgets; assertion diffs between runs; Slack/email/webhook alerting; results shareable by URL or exportable.
  - *Why*: Bruno has none (usebruno.com/performance page says "coming soon"; schedules issue #9553 open). Postman monitors are cloud-only and per-seat paid.
- **FR-201 Contract Assurance / Drift Engine**: auto-validate every response against an OpenAPI/OAS schema on send/run; produce a coverage map (endpoints hit/never-hit) and a drift report; treat out-of-contract behavior as a **blocking merge condition** in CI (spec-code divergence = silent drift, the arXiv:2606.27045 finding).
  - *Why*: deterministic (no fragile LLM parsing); nobody in the client market - including Hoppscotch, Kong, Thunder Client, Yaak, Apidog - ships drift-as-block automatically.
- **FR-202 Micro Load Testing**: ramping QPS with latency percentiles from any folder; one click, local. Bruno #1569 (open since 2024); Postman hides equivalents behind paid cloud tiers.
- **FR-203 Snapshots w/ Visual Diff**: each run produces a response snapshot; diffs between envs/versions are first-class UI; "time travel" browse of any historical session.
- **FR-204 Findings-as-Tests**: any finding (drift, perf, vuln, jailbreak, assertion) can be promoted into a collection test or a monitor check with one click; versioned in Git; skips nothing.
- **FR-205 Secrets Vault**: OS keychain-backed secret storage with masking in UI/logs; never plaintext in collections.

### 3.3 P2 - Agent-Native (the 2026+ user surface)

**In order:**

- **FR-300 First-class MCP support (both directions)**:
  1. *MCP requests* - test any MCP server: tools/list, tools/call, resources, notifications, OAuth flows.
  2. *Built-in MCP server* - exposes your collections (list/run/generate/repair) as MCP tools to Claude Code / Cursor / Gemini CLI / VS Code Copilot. Postman ships this ([mcp.postman.com](https://www.postman.com/product/mcp-server)); Bruno only has community wrappers ([bruno-mcp](https://github.com/macarthy/bruno-mcp)). We ship it.
- **FR-301 Agent Chat via ACP**: Agent Client Protocol chat inside the client; agents can onboard a collection (schema walkthrough, auth setup, scaffold requests), execute runs, and read findings. Driven by the drift engine so humans no longer triage by eye.
- **FR-302 Agent-Native Drift Triage**: every contract drift produces a suggested patch (collection edit) the agent can apply and the human approves; telemetry strips response bodies before they reach any model.
- **FR-303 Agentic-Era Request Types as Test Targets**: MCP request type (1) plus direct support for common agent-era transport surfaces: A2A agent cards with x402 hooks, agent session endpoints, SSE streams, etc. The client *tests* agentic APIs even if it doesn't itself implement them all (Postman does MCP requests only; nobody ships the agent-era *test* type cleanly).

### 3.4 P3 - Adapter Kernel ("Protocol LSP")

**In order:**

- **FR-400 Neutral Session Kernel**: core primitives = sessions, envelopes, timeline, snapshot store, secrets, out-of-process execution. **No HTTP assumptions leak into the core** (LSP's big lesson: a neutral document model won).
  - HTTP remains the only built-in transport (it is the distribution channel for adapters and health checks). Everything else is imported.
- **FR-401 Adapter SDK + Manifest**: a mini public API (5-7 operations). Manifest carries: protocol ID + version, capability flags, JSON-Schema form descriptor (auto-UI), permissions scope (network hosts/ports, FS paths, memory/fuel), and MCP tool descriptions.
  - *The rule the v0 registry ships on*: **freeze the SDK before any public registry launch.** Once the registry exists, the contract is permanent public API.
- **FR-402 Two Runtimes**:
  1. **Out-of-process** (any language, stdio/HTTP boundary, LSP-style) - first-class; heavy/slow protocols live here.
  2. **In-process WASM** (WASI 0.2/0.3) - capability-based sandbox ([wasi.dev security model](https://wasi.dev/security)); rules: deny-by-default, capability manifest per adapter, Wasmtime-class runtime (never Node's `node:wasi` - explicitly not a security boundary, per Node docs).
- **FR-403 Git-Native Adapter Registry**: adapters live in `adapters/` inside the user's repo (reviewable in PRs - **DBeaver-driver-style** click-to-add, upgradeable without app updates) plus a signed central registry (sigstore provenance, SBOM per release) with tier levels: maintained/core, certified, community.
- **FR-404 Conformance Suite (core engineering, not community polish)**: lifecycle tests every adapter must pass (open/send/onEvent/close, streaming, snapshot fidelity, form-rendering, manifest-schema validation). Gating for tier badges.
- **FR-405 Seed Adapters authored by us**: MCP, A2A, gRPC, MQTT, WebSocket-JSON, SMTP - we are the first publisher. Target 6-10 official adapters before any registry announcement.
- **FR-406 Auto-UI from Manifest Schema**: form renderer generates request editors from JSON Schema per adapter, so third parties don't ship UI code. Complements FR-106 (Bruno Apps-style custom UI for power users).

### 3.5 P4 - Security Adapter (Inbuilt Penetration Testing)

**Inspiration sources verified on GitHub (Oct 2026):**

| Repo | Stars | License | Why we care |
|---|---|---|---|
| [usestrix/strix](https://github.com/usestrix/strix) | ~69k | Apache-2.0 | Agent loop: recon → exploit → verify → **fix PRs**. Verified findings only. |
| [vxcontrol/pentagi](https://github.com/vxcontrol/pentagi) | ~28k | MIT (core)* | 13+ agent architecture, Kali 200+ tools in Docker, observability, human escalation. |
| [projectdiscovery/nuclei](https://github.com/projectdiscovery/nuclei) | ~30k | MIT | The **embed-as-engine** pick: YAML DSL, 10k+ templates, API-specific packs, CISA KEV coverage, CI-ready. |
| [KeygraphHQ/shannon](https://github.com/topics/offensive-security) | top OSS in topic | OSS | Source-code aware AI pentester; executes real exploits. |
| [samugit83/redamon](https://github.com/topics/ai-pentesting) | newer | OSS | Attack-surface graph + human approval gates + MCP both ways. |
| [aliasrobotics/cai](https://github.com/aliasrobotics/CAI) | ~9.8k | MIT (archived) | Benchmark king (HTB #1); lesson: full autonomy is premature - semi-autonomous + evidence wins. |

**Requirements (in order):**

- **FR-500 Authorization Gate (non-negotiable)**: explicit target authorization attest signed per engagement; scope manifest (which hosts/ports are in-scope, bound to adapter capability model); safe-by-default scan profiles; hard rate limits; kill switch.
- **FR-501 Nuclei Engine Adapter**: Nuclei runs as the flagship out-of-process adapter, in a **container** (not the user's host). API-focused template packs + OWASP API Security Top 10 mapping. Enforce template signatures (Nuclei has 2026 advisories: arbitrary command execution via DAST signature bypass [GO-2026-6570], local file read via require() loader bypass [GO-2026-4965]).
- **FR-502 Verified Exploit Loop (Strix-style)**: LLM agent layer runs *only* in the container; recon → exploit → verify cycle. **Only PoC-verified findings surface**; each finding ships a replayable proof capsule in the client's own runner.
- **FR-503 Findings → Regression + Agent**: one click converts a vuln finding to a collection test (regression); MCP server exposes `run_security_scan` to coding agents; drift rails welcome a new regression check as a monitor.
- **FR-504 Remediation**: fix suggestions; PR-generation against the user's repo (strix/redamon precedent) - always human-approved.

### 3.6 P5 - LLM Jailbreak / Prompt-Injection Red Teaming

**Position**: a semantic engine on the same session kernel. **No new transport layer** - an LLM app is just an HTTP endpoint (OpenAI-compatible `/chat/completions`, Responses API, or custom). Attack payload sequences are replayed through the adapter request templating; every exchange lands in the universal session timeline, so **drift, monitor, and report rails treat a safety regression exactly like any other API drift**. This is the layer where existing LLM security Overflow-OSS tools sit *outside* the API client; we are the first client that treats model-app attacks as first-class API findings.

**Inspiration sources (verified Oct 2026):**

| Repo | Maintain | License | Why we care |
|---|---|---|---|
| [promptfoo](http://github.com/promptfoo/promptfoo) | OpenAI (since Mar 2026) | MIT | 100+ attack plugins; YAML config; runs 100% locally; used by OpenAI + Anthropic; dominant CLI shape. Ships red-teaming for MCP too via [evil-mcp-server](https://github.com/promptfoo/evil-mcp-server). |
| [PyRIT](https://microsoft.github.io/PyRIT/) | Microsoft | OSS | Framework architecture worth copying: **targets, converters, probes, orchestrators, scorers, memory**; multi-turn strategies (Crescendo, TAP, Skeleton Key); Scanner CLI + human-led GUI (CoPyRIT). |
| [garak](https://github.com/NVIDIA/garak) | NVIDIA | Apache-2.0 | "The LLM vulnerability scanner": probes for hallucination, data leakage, prompt injection, jailbreaks; clean generator/buff/detector taxonomy. |
| [aglio-lab/ai-red-teaming-tools](https://github.com/aglio-lab/ai-red-teaming-tools/blob/main/README.md) | curated | reference | Helpful catalog incl. FuzzyAI (CyberArk: adversarial suffixes, genetic search), promptmap, Giskard, DeepTeam; also benchmarks JailbreakBench/HarmBench/StrongREJECT. |

**Requirements (in order):**

- **FR-600 LLM Target Adapter Config**: target config supports (a) OpenAI-compatible endpoints, (b) Responses-API endpoints, (c) custom adapter request templates (e.g., a UI LLM endpoint via the user's existing collection request). System-prompt storage stays in the secrets vault.
- **FR-601 Attack Vector Library (taxonomy first)**: curated probe plugins mapping to **OWASP Top 10 for LLM Applications 2025/2026** and MITRE ATLAS:
  - LLM01 Prompt Injection - direct, indirect (via RAG content, embedded documents), payload splitting, multimodal injection, multilingual/obfuscation (ATLAS AML.T0051.000/.001, AML.T0054 Hack-Style Direct Jailbreak).
  - Jailbreak strategies - single-shot (encoded/obfuscated prompts) and **multi-turn adversaries** (Crescendo / TAP / Skeleton Key) ported from PyRIT.
  - LLM02 Sensitive Info Disclosure - PII extraction, system-prompt leakage (LLM07), RAG-context leakage, API-key/secret guessing probes.
  - Improper Output Handling (LLM05) - SQL/shell injection in LLM output, XSS in rendered output, ASCII-smuggling (invisible characters).
  - Excessive Agency (LLM06) - tool misuse, unintended writes, self-escalation against user's existing MCP tool collections.
  - Hallucination/Overreliance (LLM09 2023 / Misinformation 2025), Unbounded Consumption (LLM10 - budget-attack patterns), Vector/Embedding Weaknesses where schemas exist (LLM08).
  - Reputation/brand probes: political statements, competitor endorsement, imitation/brand-impersonation, unintended contracts (promptfoo precedent).
- **FR-602 Strategy Composition Framework (copy PyRIT's shape)**: composable **converters** (transform prompts: encoding, base64, role-play wrappers, ASCII smuggling, infinitely many multi-turn wraps), **probes** (attack payloads), **orchestrators** (single-turn / multi-turn / red-team-cycle drivers), **scorers** (deterministic string match, regex policy, LLM-as-judge with true/false, Likert, or classification), **memory** (session snapshots as first-class citizen, local-first).
- **FR-603 Local-First Rule**: attack generation, replay, and scoring all run locally by default. No vendor cloud round-trips. Users BYO their model keys for LLM-as-judge scorers. ([promptfoo precedent: evals run 100% locally](https://www.promptfoo.dev/docs/intro/).)
- **FR-604 Safe Mode by Default**: probes are read-only by default; token budget caps and rate limiting on the target are mandatory; kill switch; explicit target acknowledge per run (extending FR-500's authorization gate).
- **FR-605 Finding = Proof Capsule**: a successful jailbreak or injection exports as a **replayable capsule**: request template + payload sequence + expected violation class + seed. Load the capsule into any session; re-run to prove the finding is still valid (no hallucinated exploits).
- **FR-606 Safety Regression Rails (the killer tie-in)**: every probe that **fails** (i.e., the target resisted attack, the good case) is one click to save as a permanent monitor/CI check: *"this probe must keep failing."* If a future prompt/system change makes it succeed, the monitor fails - model-behavior regression caught independently of prompt-authoring taste.
- **FR-607 Agent-Tiered MCP Attacks**: since our MCP server exposes collections to coding agents (FR-300), we treat MCP servers as first-class attack targets too: **evil-MCP scanning / tool-poisoning detection** (promptfoo [evil-mcp-server](https://github.com/promptfoo/evil-mcp-server) precedent; MCPScan-style tool-poisoning detector already exists in OSS).
- **FR-608 Reporting + Compliance**: one-page summary mapped to OWASP LLM Top 10 2025/2026 with severity, attack class, evidence capsule, remediation hint, plus diffing between two red-team cycles ("did we get safer after the guardrail change?").

---

## 4. Part-by-Part Implementation Roadmap

Each phase has: **Goal → Build order → Acceptance Criteria (AC) → Gate**.

### Phase 0 - Bootstrap (Week 1)
- **Goal**: monorepo up, core symmetry proven.
- **Build**: TS monorepo; packages `kernel-core`, `kernel-http`; local-first file store (JSON Schema validated); plain REST request + response inspector; secrets in OS keychain.
- **AC**: send a request headlessly; validate a JSON schema; secrets never land in a collection file.
- **Gate**: manual smoke pass + basic Playwright e2e green.

### Phase 1 - P0 completion (Weeks 2-4)
- **Goal**: full table-stakes (FR-100..FR-107).
- **Build**: BrowserRouter GraphQL/WS/SSE; auth suite; scripting + `bru`-style API; importers (Postman + OAS first - highest conversion); CLI runner with JUnit/JSON/TAP.
- **AC**: import 3 Postman collections losslessly (assertions, variables, scripts); `forge run` exits non-zero on assertion failure in CI.
- **Gate**: CI pipeline on a sample collection passes.

### Phase 2 - P1 differentiators (Weeks 5-8)
- **Goal**: local monitors + drift engine + micro load testing (FR-200..FR-205).
- **Build**: scheduler (in-app cron + CI trigger); latency percentile budgets + alerting; OAS response validator + coverage map + drift-as-block gate in CI; ramping-QPS runner; snapshots + visual diff UI; findings-as-tests promotion.
- **AC**: scheduled run produces alert on latency budget breach; drift report blocks merge on modified endpoint; load test produces p50/p95/p99; snapshot diff shows field-level drift.
- **Gate**: demo of "change the API without changing the spec → CI blocks merge".

### Phase 3 - Adapter kernel v0 (Weeks 9-14)
- **Goal**: session kernel is protocol-agnostic; first adapters ship (FR-400..FR-406).
- **Build**: extract adapter SDK (TS types + manifest JSON Schema, 5-7 ops); out-of-process runtime (stdio/HTTP); WASI runtime v0; conformance suite; 3 seed adapters (MCP, gRPC, MQTT); manifest-driven auto-UI; registry v0 (static signed list); `adapters/` folder in collections.
- **AC**: MCP adapter drives real MCP server end-to-end; adapter conformance runs in CI per PR; form auto-renders from manifest schema; adapters install without app release.
- **Gate**: 3 conformance-passing adapters + zero HTTP assumptions in kernel types.

### Phase 4 - Agent-native surface (Weeks 15-18)
- **Goal**: agents are first-class users (FR-300..FR-303).
- **Build**: built-in MCP server (list/run/generate/repair tools + security tool hooks); ACP agent chat; drift triage agent; A2A adapter (agent-card import) as a request type.
- **AC**: Claude Code lists our MCP tools and runs a collection unprompted; drift patch suggested via ACP accepted in a PR.
- **Gate**: end-to-end "the commit adds a drift check that an agent authored via ACP".

### Phase 5 - Security adapter (Weeks 19-24)
- **Goal**: inbuilt pentest (FR-500..FR-504).
- **Build**: authorization gate + scope manifests; Nuclei out-of-process container adapter; template packs (API-focused, OWASP API Top 10); verified-exploit agent loop (Strix-style) in container; finding → collection test conversion; `run_security_scan` MCP tool; fix-PR generator.
- **AC**: nuclei scan of a deliberately-vulnerable sample API finds seeded vulns with zero false positives in reproducible re-runs; authorization gate refuses unattested targets; finding converts to a regression test.
- **Gate**: false-finding rate < X on bug-bounty-style sample suite; container escape audit passes.

### Phase 6 - LLM jailbreak adapter (Weeks 25-30)
- **Goal**: adversarial robustness on the client (FR-600..FR-608).
- **Build**: target adapter config (OpenAI-compatible / Responses / custom); vector library v1 (prompt injection, jailbreak strategies, PII, system-prompt leakage, output handling, excessive Agency, reputation probes); strategy composition framework (converters/probes/orchestrators/scorers, PyRIT-shaped); local-first scored + plain-text replay; finding = proof capsule; safety-regression rail (a passing defense record becomes a monitor check); evil-MCP scanning v0; reporting mapped to OWASP LLM Top 10 2025/2026.
- **AC**: red-team run against a deliberately-weak sample LLM app produces verified findings with replayable capsules; humans approve via ACP; safety-regression rail triggers on a deliberately-degraded guardrail config; totality of probes runs under budget and rate caps with zero external network calls by default.
- **Gate**: report generation + diffing between two consecutive red-team cycles; probes themselves rate-limit-safe; zero prompt leakage in session timeline logs.

### Post-1.0 (explicitly deferred, do not build now)
- A2A/x402 implementations beyond request/types.
- Own mock-server cloud. Own LLM. Bruno-style proprietary file format. Postman-style collab cloud. IDE plugins. Mobile clients.

---

## 5. Non-Negotiables

1. **Freeze the adapter SDK before any registry launch.** Public API permanence is real.
2. **Local-first is a policy, not a setting.** No telemetry-or-cloud paths, not even for red-team/jailbreak runs.
3. **Humans approve every security/patch action.** Semi-autonomy, per CAI/RedAmon evidence.
4. **Authorization attest before any offensive scan.** No exceptions.
5. **Verified findings only.** Whether vuln, drift, or jailbreak - if it can't be replayed, it isn't reported.
6. **MIT/Apache-2.0-compatible dependencies only.** No AGPL paths (PentAGI's VXControl Cloud SDK is a case in point).

---

## 6. Risk Register (top of mind)

1. **Adapter API is a permanent hostage** - freeze minimal v0 before registry; semver policy from day one.
2. **Registry cold-start** - be the first publisher (FR-405); fund bounties; open the spec before competing.
3. **Sandbox escapes** - real WASI host, capability manifests, fuel limits, signatures (nuclei 2026 CVEs precedent).
4. **Conformance failure = brand damage** - conformance suite is core engineering, not community polish.
5. **EDR/AV flagging** on Nuclei/agents - container isolation keeps the host clean.
6. **Legal/store bans** on offensive tooling - strongest reason for attestation gate, scope manifests, safe profiles.
7. **LLM findings hallucinate** - PyRIT scorers/promptfoo plugin reality: gate on deterministic + verified replay only.
8. **Copy risk (Postman/Hoppscotch)** - moat is the bundle: local-first files + Git-reviewable adapters + agent-exposed tools + drift rails; move fast.

---

## 7. Language & Runtime Decisions

The spec implies a stack in scattered places (Phase 0 "TS monorepo"; FR-402 Wasmtime/WASI; FR-402.1 "any language" adapters). This section consolidates those into one decision record so the stack is a deliberate choice, not an accident of the roadmap.

### 7.1 Core principle

**The core is one language - TypeScript. Every other language enters through the adapter boundary (out-of-process or WASM), never into the core.** This is a direct consequence of FR-400 ("no HTTP assumptions leak into the core") and FR-402. The kernel, drift engine, monitors, UI, MCP server, and CLI are all TS. Go (Nuclei), Python (PyRIT/garak), and any third-party protocol adapter live behind the FR-402.1 stdio/HTTP boundary or in a container - we never absorb them into the core language.

### 7.2 Decision table

| Layer | Language / runtime | Rationale |
|---|---|---|
| Kernel (`kernel-core`, `kernel-http`), collections, drift engine (FR-201), monitors (FR-200), MCP server (FR-300), UI (FR-106/FR-406), ACP chat (FR-301) | **TypeScript** | Entire competitor ecosystem (Bruno, Hoppscotch, Insomnia, Yaak) and the importer formats (FR-104) are JS/TS-native; scripting sandbox (FR-103) is JS by definition; MCP/A2A reference tooling is TS-first. |
| CLI runner (FR-107) | **TypeScript on Bun** (Node fallback) | Startup speed for CI-native headless runs; Node remains the safe fallback if a Bun gap appears. |
| Desktop shell | **Tauri (Rust + webview)** | We already need Rust for the WASI sandbox (FR-402.2); Tauri keeps that Rust native to the shell instead of behind a subprocess bridge. Yaak (our closest architectural sibling) validates this. Electron is the fallback if the all-TS kernel friction outweighs the Rust benefit. |
| WASI adapter sandbox (FR-402.2) | **Rust (embed Wasmtime)** - mandatory | FR-402 already forbids `node:wasi` as "not a security boundary." A capability-based, fuel-limited, deny-by-default sandbox requires a Wasmtime-class runtime, which is Rust. Non-optional. |
| Load-test engine (FR-202) | **Rust** (TS to prototype) | Accurate p50/p95/p99 under high concurrency is polluted by Node's single-threaded event loop and GC jitter. A Rust worker driven by the TS orchestrator yields trustworthy percentiles. Prototype in TS, migrate when numbers must be trusted. |
| Nuclei (FR-501), PyRIT/garak (FR-601/602), third-party gRPC/MQTT/other adapters (FR-405) | **Their own language, behind the out-of-process/container boundary** | Not our decision. The FR-402.1 boundary exists precisely so adapters are language-agnostic. We call Nuclei as a binary and port only the *shapes* (converters/probes/orchestrators/scorers) of PyRIT into TS - we do not make Go or Python first-class in the core. |

### 7.3 Sequencing rule

**Start 100% TypeScript. Introduce Rust only at Phase 3 (FR-402.2), not before.** The WASM sandbox is the first and only mandatory Rust dependency; the load engine follows when measurement fidelity demands it. Do not pre-emptively go polyglot - the adapter design (FR-400) exists so the core never has to.

### 7.4 Constraint check

All choices satisfy Non-Negotiable #6 (MIT/Apache-2.0-compatible): TypeScript (Apache-2.0), Wasmtime (Apache-2.0), Tauri (MIT/Apache-2.0), Bun (MIT). No AGPL paths introduced.

---

## 8. References

**Competitor / market:**
- https://news.ycombinator.com/item?id=39702610 - Postman pricing exodus
- https://requestly.com/alternatives/postman-alternative/ - Postman gaps, 2026 landscape
- https://requestly.com/bruno-vs-postman-why-you-dont-have-to-choose - competitor defaults
- https://apidog.com/blog/bruno-limitations - Bruno gap analysis (no mocks/docs/monitors)
- https://www.usebruno.com/use-cases/performance-testing - "coming soon"
- https://github.com/usebruno/bruno/issues/1569 - stress-testing request
- https://docs.usebruno.com/apps/overview - Bruno Apps precedent
- https://www.postman.com/product/mcp-server - Postman MCP server
- https://a2a-protocol.org/ - A2A v1.0 (Linux Foundation, 2026)
- https://modelcontextprotocol.io/docs/2026-07-28/learn/architecture - MCP current spec baseline

**Adapter prior art:**
- https://microsoft.github.io/debug-adapter-protocol/ - neutral adapter pattern
- https://wasi.dev/security - capability-based sandboxing
- https://nodejs.org/docs/latest/api/wasi.html - node:wasi "not a security model" caveat
- https://pkg.go.dev/github.com/wippyai/wasm-runtime/wasi/preview2/sockets - sockets capability impl

**Security adapter inspiration:**
- https://github.com/usestrix/strix - ~69k stars, verified findings, fix PRs
- https://github.com/vxcontrol/pentagi - 13+ agent arch, container isolation
- https://github.com/projectdiscovery/nuclei - the embed-as-engine choice
- https://github.com/aliasrobotics/CAI - benchmark record; lesson: semi-autonomy
- https://pkg.go.dev/github.com/projectdiscovery/nuclei/v3@v3.4.7 - 2026 advisories incl. GO-2026-6570, GO-2026-4965

**LLM red teaming:**
- http://github.com/promptfoo/promptfoo - CLI/library, 100+ plugins, MIT, used by OpenAI/Anthropic
- https://microsoft.github.io/PyRIT/ - framework shapes: targets/converters/probes/scorers/memory; Crescendo/TAP/Skeleton Key
- https://github.com/NVIDIA/garak - probes for hallucination/data leakage/prompt injection/jailbreaks
- https://github.com/aglio-lab/ai-red-teaming-tools/blob/main/README.md - curated landscape incl. FuzzyAI, promptmap, benchmarks
- https://www.promptfoo.dev/docs/guides/llm-redteaming/ - attack vector taxonomy precedent
- https://github.com/promptfoo/evil-mcp-server - MCP server red-teaming
- https://owasp.org/www-project-top-10-for-large-language-model-applications/assets/PDF/OWASP-Top-10-for-LLMs-v2025.pdf - OWASP LLM Top 10 v2025 (LLM01..LLM10)
- https://github.com/OWASP/www-project-top-10-for-large-language-model-applications - 2026 release

---

*Reviewer note (product): competitors frequently revise. Grounded-true data in this doc was verified October 2026; refresh quarterly.*
