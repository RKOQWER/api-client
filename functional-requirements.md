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

- **FR-100 Request Engine**: a protocol-neutral request/response surface driven entirely by adapters (see FR-400/FR-405), not by hard-coded protocols. HTTP/REST ships as the pre-bundled adapter; GraphQL, WebSocket, SSE, gRPC, and MCP are imported adapters that light up the same surface once installed. Common request concerns the kernel understands generically - methods, headers, query, body, cookies, TLS client certs, proxy support - are passed through to whichever adapter handles the protocol. Latency/timeline/event view works for all of them because it lives in the kernel, not the adapter.
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

- **FR-400 Neutral Session Kernel**: core primitives = sessions, envelopes, timeline, snapshot store, secrets, out-of-process execution. **No protocol assumptions leak into the core** - not even HTTP (LSP's big lesson: a neutral document model won). The kernel exposes only the ~5-7 generic adapter verbs (FR-401); every protocol, HTTP included, is an imported adapter.
  - **Bootstrap exception (the one built-in):** the kernel contains a minimal HTTP *fetch-plumbing* used solely to reach the registry and download/update adapters and run health checks. This plumbing is **not** a testable protocol surface - HTTP/REST as a protocol you send requests with is itself a pre-bundled adapter (see FR-405). This resolves the chicken-and-egg problem (you need HTTP to fetch the HTTP adapter) without welding a protocol into the core.
- **FR-401 Adapter SDK + Manifest**: a mini public API (5-7 operations). Manifest carries: protocol ID + version, capability flags, JSON-Schema form descriptor (auto-UI), permissions scope (network hosts/ports, FS paths, memory/fuel), and MCP tool descriptions.
  - *The rule the v0 registry ships on*: **freeze the SDK before any public registry launch.** Once the registry exists, the contract is permanent public API.
  - **Two adapter types** (determines who can author an adapter and how):
    - **Type A - config-only (manifest, zero code):** for HTTP-shaped protocols (GraphQL, REST variants, OpenAI-compatible LLM endpoints, webhooks, most JSON-over-HTTP). Everything the adapter needs is declared in the manifest - fields, how to build the request, how to read the response. No code. See FR-407.
    - **Type B - logic (manifest + thin code wrapper):** for binary/stateful protocols (gRPC protobuf, MQTT broker sessions, raw TCP) where decoding/state cannot be expressed declaratively. Needs a small code wrapper around an existing library.
  - **Anyone can author either type** - us (seed adapters), the community, or the end user. Because adapters are just files in the `adapters/` folder validated against the frozen SDK schema, the vendor is never the required author. The user is never blocked waiting for us.
  - **Isolation guarantee:** each adapter is a separate folder, separately validated, run in its own out-of-process sandbox. Adding a new manifest physically cannot modify or break an existing adapter; a malformed manifest is rejected at load and the others keep working. This is why the frozen SDK contract (above) is the only thing that must stay stable.
- **FR-402 Runtimes**:
  - **What "out-of-process sandbox" means (pin this down - two independent protections):**
    1. **Out-of-process = memory isolation.** The adapter runs as its *own OS process* with private memory, talking to the kernel over a stdio/pipe boundary. It cannot read or corrupt the app's memory, and if it crashes it dies alone (how VS Code isolates language servers, how browsers isolate tabs). This buys isolation *from our app* - but a plain separate process can still, by default, touch files and the network.
    2. **Sandbox = action permissions.** On top of that, the OS restricts what the process may *do to the outside world* (files, network, system calls), deny-by-default, driven by the adapter's manifest. This is what stops a stranger's adapter from reading your disk or phoning home.
    - The two are different protections: out-of-process is about *memory*, sandbox is about *actions*. "Out-of-process sandbox" = both at once.
  1. **Out-of-process** (any language, stdio/HTTP boundary, LSP-style) - the **default and only runtime for v1.0**. All seed adapters (HTTP, gRPC, MQTT, MCP, etc.) and all heavy/slow protocols live here. This runtime alone covers every adapter on the 1.0 roadmap.
     - **Day-zero sandboxing (non-negotiable):** every out-of-process adapter runs under **deny-by-default, manifest-declared permissions** (network hosts/ports, FS paths from FR-401). An adapter can only reach what its manifest explicitly grants.
     - **How it is enforced - OS primitives, no Docker:** the sandbox uses the operating system's own mechanisms - **Linux** `seccomp` + namespaces + `landlock`; **macOS** Seatbelt; **Windows** AppContainer / restricted tokens - not a container runtime. When the kernel launches an adapter it applies a sandbox profile derived from the manifest ("allow only `api.example.com:443`, deny all else"). A normal protocol adapter therefore needs **no Docker and no extra install for the user**. (Cost to us: three OS-specific sandbox profiles to maintain - the one real downside of this approach vs a single cross-platform mechanism.)
     - **Container (Docker/OCI) is required only for the heavy offensive tooling** (Nuclei FR-501, exploit loop FR-502): they run genuinely dangerous untrusted input and bundle large toolchains, so an OS sandbox profile is not enough - full container (or microVM: Firecracker/gVisor) isolation is used there instead. "Container," not "Docker" specifically.
  2. **In-process WASM** (WASI 0.2/0.3) - **deferred to post-1.0, as a performance optimization only.** Out-of-process (above) already provides the *safety* for untrusted adapters; WASM adds *speed* (lower startup/call overhead for hot, lightweight adapters) and in-process execution. WASM is an *alternative to the OS sandbox* (one cross-platform sandbox instead of three OS-specific profiles), not a stronger one. No seed adapter needs it, so it is not on the 1.0 critical path. When built, the rules hold: capability-based sandbox ([wasi.dev security model](https://wasi.dev/security)), deny-by-default, capability manifest per adapter, Wasmtime-class runtime (never Node's `node:wasi` - explicitly not a security boundary, per Node docs).
- **FR-403 Git-Native Adapter Registry**: adapters live in `adapters/` inside the user's repo (reviewable in PRs - **DBeaver-driver-style** click-to-add, upgradeable without app updates) plus a signed central registry (sigstore provenance, SBOM per release) with tier levels: maintained/core, certified, community.
  - **How search works (app-store model, plain terms):** the client does not carry a built-in list of adapters. When a user searches (e.g. "gRPC"), the client asks the live registry over the FR-400 fetch-plumbing and shows whatever the registry holds *right now*. So a newly published adapter appears in search **without any app update** - exactly like a new app showing up in the App Store without updating your phone's OS.
  - **Versioning:** adapters are semver-versioned like npm/VS Code extensions. A user installs latest, pins an exact version, or accepts a range; the resolved version is committed in `adapters/`, so a teammate who clones the repo gets the identical adapter - reproducible, reviewable in a PR.
  - **v0 scope (honest):** the Phase 3 registry is a *static signed index* the client re-fetches on each search (publish → appears on next search). "Signed" = cryptographically verified so no attacker can swap in a malicious adapter (trust, not freshness). A push/live-update index is a later refinement; re-fetch-on-search already feels live to users.
- **FR-404 Conformance Suite (core engineering, not community polish)**: two conformance profiles, because adapters come in two shapes.
  1. **Request/response protocol adapters** (HTTP, gRPC, MQTT, MCP...): lifecycle tests - open/send/onEvent/close, streaming, snapshot fidelity, form-rendering, manifest-schema validation.
  2. **Long-running job adapters** (security scans FR-501, jailbreak red-team runs FR-602): a *job* lifecycle - start/progress/finding-emit/cancel/complete, finding-schema validation, proof-capsule replay fidelity, and clean kill-switch behavior. These do not fit the request/response lifecycle and must not be forced into it.
  - Both profiles gate tier badges. Every adapter declares which profile it conforms to in its manifest (FR-401).
- **FR-405 Seed Adapters authored by us**: **HTTP/REST** (pre-bundled, ships in the box so the app works on first launch), MCP, A2A, gRPC, MQTT, WebSocket-JSON, SMTP - we are the first publisher. Only the HTTP/REST adapter is pre-installed; the rest are registry downloads the user imports on demand. Target 6-10 official adapters before any registry announcement.
  - **Bundling rule (consequence of Option A):** exactly one protocol adapter ships pre-installed - HTTP/REST - because it is the most common surface and the registry itself is reached over the FR-400 fetch-plumbing. Every other protocol, including GraphQL/WS/SSE, is searched for and imported from the registry; none is welded into the core.
- **FR-406 Auto-UI from Manifest Schema**: form renderer generates request editors from JSON Schema per adapter, so third parties don't ship UI code. Complements FR-106 (Bruno Apps-style custom UI for power users).
- **FR-407 User-Authored Manifest-Only Adapters (anti-bottleneck, day-zero)**: the user can create a working **Type A (config-only)** adapter themselves - fill in / upload a manifest template describing an HTTP-shaped protocol (fields, request shape, response parsing), and it works immediately with no code and no vendor involvement. The form renderer (FR-406) draws its UI automatically.
  - *Why this exists:* the user must never be helpless waiting for us (or anyone) to publish a wrapper. For the majority of protocols - which are HTTP-shaped - a manifest alone suffices, so self-service covers most real needs on day one. Only Type B (binary/stateful) protocols need code, and even those can be authored by the user or community, not just us.
  - *Scope boundary:* Type A is declarative-only. If a protocol needs real decoding/state logic (gRPC, MQTT), the client clearly tells the user "this needs a Type B code wrapper" rather than silently failing.

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
- **FR-501 Nuclei Engine Adapter**: Nuclei runs as the flagship out-of-process adapter, conforming to the **long-running job profile** (FR-404.2) with a manifest declaring its scope permissions (FR-401), in a **container** (not the user's host). API-focused template packs + OWASP API Security Top 10 mapping. Enforce template signatures (Nuclei has 2026 advisories: arbitrary command execution via DAST signature bypass [GO-2026-6570], local file read via require() loader bypass [GO-2026-4965]).
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

The product is built in seven phases (Phase 0 through Phase 6), each roughly a few weeks long. Every phase describes four things:

- **Goal** - what this phase is trying to achieve.
- **Build** - the concrete pieces we construct.
- **Acceptance Criteria (AC)** - the tests that prove the phase actually works.
- **Gate** - the single bar we must clear before moving to the next phase.

> **Overriding principle (adapter-first):** the kernel implements *no* protocol - not even HTTP (per Option A, FR-400). The protocol-neutral kernel and adapter SDK are built in Phase 0, and every protocol ships as an adapter from the first commit. This avoids building protocols as hard-coded core features and then tearing them out later - the single biggest rewrite risk the original sequencing carried.

### Phase 0 - Bootstrap + adapter spine (Week 1)

*In plain terms: stand up the skeleton AND prove the core architecture on day one - the kernel implements no protocol itself, and even the first HTTP request goes through an adapter. We de-risk the biggest bet (does a protocol-neutral kernel actually work?) immediately, not in month 3.*

> **Sequencing note (adapter-first):** the original plan hard-coded protocols in Phases 0-2 and extracted the neutral kernel later in Phase 3 - which, under the Option A "HTTP is also an adapter" decision, would force a painful rewrite (ripping welded-in protocols back out). We invert that: the adapter SDK and the kernel's generic verbs exist from Phase 0, and every protocol - including HTTP/REST - is an adapter from the first commit.

- **Goal**: Get the project structure in place and prove the protocol-neutral kernel + adapter model work on the easiest protocol (HTTP) before any hard ones.
- **Build**:
  - A single TypeScript repository (a "monorepo") holding all our code packages.
  - `kernel-core`: the protocol-neutral engine exposing only the ~5-7 generic adapter verbs (open/send/events/close + secrets/snapshots/timeline) - **no HTTP knowledge inside it** (FR-400).
  - A minimal **HTTP fetch-plumbing** inside the kernel, used *only* to reach the registry and download adapters (the Option A bootstrap exception) - not a testable protocol surface.
  - The first draft of the **adapter SDK + manifest** (FR-401): TypeScript types + JSON Schema manifest, out-of-process runtime only.
  - **Day-zero adapter sandboxing** (FR-402, Non-Negotiable #7): the out-of-process runtime enforces deny-by-default, manifest-declared permissions, so even the first adapters run isolated.
  - The **HTTP/REST adapter** (FR-405) built against that SDK and pre-bundled - so the very first request is proof the adapter model holds.
  - **Manifest-only adapter authoring v0** (FR-407): a user can define a basic HTTP-shaped adapter from a manifest alone, no code - proven on day one so the product is never vendor-captive.
  - A local file store that saves everything to plain files on disk and checks each file against a JSON Schema so bad data can't sneak in.
  - A response inspector to view what comes back.
  - Secret values (passwords, tokens) stored in the operating system's secure keychain, not in project files.
- **AC**:
  - We can send an HTTP request *through the HTTP adapter* with no user interface involved (headless) - and the kernel contains zero HTTP-specific code.
  - A file is correctly validated against its JSON Schema.
  - No secret ever ends up written into a collection file.
- **Gate**: A manual once-over passes, a basic end-to-end browser test (Playwright) is green, **the kernel has zero protocol assumptions** (the HTTP adapter can be removed and the kernel still builds), and a sample manifest-only adapter authored by hand (no code) works.

### Phase 1 - P0 completion (Weeks 2-4)

*In plain terms: build everything a normal API client is expected to have, so we're at least on par with tools like Bruno and Postman - but build each protocol as an adapter, not as core code. This is the "catch up to the basics, the right way" phase.*

- **Goal**: Deliver the full set of table-stakes features (FR-100 through FR-107) - the things no API client can ship without.
- **Build**:
  - More protocols, each shipped **as an adapter** on the Phase 0 SDK: GraphQL (a Type A / manifest-friendly, HTTP-shaped protocol), WebSocket (WS), and Server-Sent Events (SSE). (These are imported adapters, not core features - FR-100/FR-405.)
  - **Mature the manifest-only authoring UX** (FR-407): a guided editor so a user can create a config-only adapter for any HTTP-shaped protocol without code.
  - A complete authentication suite (Bearer tokens, Basic auth, API keys, OAuth, AWS signing).
  - A scripting system with a `bru`-style API, so users can run small JavaScript snippets before and after a request.
  - Importers that bring in existing collections from other tools, starting with Postman and OpenAPI (OAS) because they convert the most users.
  - A command-line runner that can execute collections in CI and output results as JUnit, JSON, or TAP.
- **AC**:
  - Import 3 Postman collections with nothing lost - assertions, variables, and scripts all survive.
  - The CLI runner (`leo run`) exits with a failure code when an assertion fails inside CI.
  - GraphQL/WS/SSE each work purely as adapters - removing one removes only that protocol, with no effect on the kernel or other protocols.
  - A user authors a working adapter for a new HTTP-shaped protocol from a manifest alone, with zero code and no vendor involvement (FR-407).
- **Gate**: A CI pipeline running against a sample collection passes end to end.

### Phase 2 - P1 differentiators (Weeks 5-8)

*In plain terms: build the features that actually make us different - automatic checks that catch when an API quietly changes or slows down, run on a schedule and able to block a bad change from merging. This is our real competitive edge.*

- **Goal**: Ship the local-first differentiators (FR-200 through FR-205): scheduled monitors, the contract-drift engine, and lightweight load testing.
- **Build**:
  - A scheduler that can run collections automatically on a timer (in-app) or on demand from CI.
  - Latency budgets (p50/p95/p99 speed targets) with alerts when an API gets too slow.
  - A validator that compares every response against the API's OpenAPI spec, maps which endpoints were tested, and - crucially - can **block a code merge** when the API drifts away from its spec.
  - A load-test runner that ramps up request volume (QPS) to measure performance.
  - Response snapshots plus a visual diff view, so you can see exactly what changed between runs.
  - One-click promotion of any "finding" into a permanent test.
- **AC**:
  - A scheduled run fires an alert when a latency budget is breached.
  - A drift report blocks a merge when an endpoint was changed without updating its spec.
  - A load test produces real p50/p95/p99 numbers.
  - A snapshot diff highlights a changed field.
- **Gate**: A live demo of the headline story - *change the API without changing its spec, and watch CI block the merge.*

### Phase 3 - Adapter registry + more adapters (Weeks 9-14)

*In plain terms: the kernel is already protocol-neutral (proven in Phase 0), so this phase is NOT a rewrite. It's about making adapters searchable, installable, and removable on demand - plus shipping more of them and the conformance suite that keeps their quality honest.*

> **Sequencing note:** the neutral kernel and adapter SDK were built in Phase 0 and exercised by HTTP/GraphQL/WS/SSE adapters in Phases 0-1. This phase hardens the SDK toward freeze, adds the registry, and broadens protocol coverage. It is additive, not a retrofit.

- **Goal**: Deliver the searchable registry and broaden adapter coverage (FR-401, FR-403, FR-404, FR-405, FR-406); harden the SDK toward freeze.
- **Build**:
  - Harden the adapter SDK (FR-401) and manifest toward its pre-registry freeze - the contract becomes permanent public API once the registry launches.
  - The **out-of-process runtime only** - this covers every seed adapter. The in-process WASM/WASI runtime (FR-402.2) is **deferred to post-1.0**; no adapter on the roadmap needs it (see FR-402), so building it now is wasted effort.
  - A conformance suite with **both profiles** (FR-404): request/response and long-running-job.
  - Three more seed adapters we author ourselves: MCP, gRPC, and MQTT.
  - A form renderer that auto-builds the request editor UI straight from an adapter's manifest, so adapter authors don't write UI code.
  - A version-0 registry (a static, signed list) with **search, install, and remove on demand**, writing to the `adapters/` folder inside the user's collection (reviewable in PRs).
- **AC**:
  - A user can search the registry for a protocol (e.g. "gRPC"), install it, use it, and remove it - entirely on demand, with no app release.
  - The MCP adapter drives a real MCP server from start to finish.
  - Adapter conformance tests (both profiles) run automatically in CI on every pull request.
  - A request form renders automatically from a manifest schema.
- **Gate**: Search→install→remove works end to end; the full set of seed adapters passes conformance; the kernel still contains zero protocol-specific assumptions (re-verified, not re-established).

### Phase 4 - Agent-native surface (Weeks 15-18)

*In plain terms: make AI coding agents (like Claude Code or Cursor) first-class users of the tool. They can list and run collections, and when the API drifts, an agent can propose the fix for a human to approve.*

- **Goal**: Treat AI agents as first-class users of the client (FR-300 through FR-303).
- **Build**:
  - A built-in MCP server exposing tools agents can call (list, run, generate, repair) plus hooks into the security tools.
  - An in-app agent chat built on the Agent Client Protocol (ACP).
  - A drift-triage agent that reviews contract drift and suggests patches.
  - An A2A adapter that imports agent cards as a request type.
- **AC**:
  - Claude Code can discover our MCP tools and run a collection on its own, without being hand-held.
  - A drift-fix patch suggested through ACP is accepted into a pull request.
- **Gate**: A full end-to-end flow where a commit adds a drift check that an agent authored via ACP.

### Phase 5 - Security adapter (Weeks 19-24)

*In plain terms: build in penetration testing - automated security scanning of APIs - but safely. Scans run inside a locked-down container, only with explicit permission, and only report vulnerabilities that were actually proven, not guessed.*

- **Goal**: Add built-in penetration testing (FR-500 through FR-504).
- **Build**:
  - An authorization gate plus scope manifests that force the user to explicitly declare what may be scanned.
  - The Nuclei scanner running as an out-of-process adapter inside a container (never on the user's own machine).
  - Template packs focused on APIs and the OWASP API Security Top 10.
  - A verified-exploit agent loop (Strix-style) that runs only inside the container: recon, exploit, then verify.
  - One-click conversion of a vulnerability finding into a regression test.
  - A `run_security_scan` tool exposed to coding agents, and a generator that drafts fix pull requests.
- **AC**:
  - A deterministic Nuclei scan of a deliberately vulnerable sample API finds the planted vulnerabilities with zero false positives across repeated runs (deterministic templates make this achievable; it does **not** apply to the LLM exploit loop).
  - Every finding from the LLM-driven exploit loop (FR-502) ships a replayable proof capsule, and **only PoC-verified findings surface** - unverifiable/hallucinated findings are dropped, never reported (per Risk #7).
  - The authorization gate refuses any target that wasn't explicitly attested.
  - A finding converts cleanly into a regression test.
- **Gate**: The false-finding rate stays under the agreed threshold on a bug-bounty-style sample suite, every surfaced LLM-loop finding replays successfully from its capsule, and a container-escape audit passes.

### Phase 6 - LLM jailbreak adapter (Weeks 25-30)

*In plain terms: test AI-powered APIs for safety - trying to trick the model into misbehaving (jailbreaks, prompt injection, leaking data). When the app resists an attack, we lock that in as a check so a future change can't quietly make it vulnerable again.*

- **Goal**: Add adversarial robustness testing for LLM-backed APIs (FR-600 through FR-608).
- **Build**:
  - Target configuration for LLM endpoints: OpenAI-compatible, Responses API, or a custom adapter template.
  - A version-1 attack library: prompt injection, jailbreak strategies, PII extraction, system-prompt leakage, bad output handling, excessive agency, and reputation probes.
  - A composition framework shaped after PyRIT - converters, probes, orchestrators, and scorers - for building attacks.
  - Local-first scoring and plain-text replay of every attack.
  - Each successful attack saved as a replayable "proof capsule."
  - A safety-regression rail: when the app successfully resists an attack, that success becomes a monitored check that must keep passing.
  - A first version of evil-MCP scanning, plus reporting mapped to the OWASP LLM Top 10 (2025/2026).
- **AC**:
  - A red-team run against a deliberately weak sample LLM app produces verified findings with replayable capsules.
  - Humans approve findings through ACP.
  - The safety-regression rail triggers when a guardrail is deliberately weakened.
  - All probes run within budget and rate caps, making zero external network calls by default.
- **Gate**: Reports generate and can be diffed between two red-team cycles, the probes themselves stay within rate limits, and no prompt text leaks into the session timeline logs.

### Post-1.0 (explicitly deferred - do not build now)

These are intentionally out of scope for version 1.0:

- The **in-process WASM/WASI runtime** (FR-402.2) - deferred until a concrete use case for untrusted in-process adapters exists; out-of-process covers all 1.0 adapters.
- A2A/x402 implementations beyond request types.
- Our own mock-server cloud, our own LLM, a Bruno-style proprietary file format, a Postman-style collaboration cloud, IDE plugins, and mobile clients.

---

## 5. Non-Negotiables

1. **Freeze the adapter SDK before any registry launch.** Public API permanence is real.
2. **Local-first is a policy, not a setting.** No telemetry-or-cloud paths, not even for red-team/jailbreak runs.
3. **Humans approve every security/patch action.** Semi-autonomy, per CAI/RedAmon evidence.
4. **Authorization attest before any offensive scan.** No exceptions.
5. **Verified findings only.** Whether vuln, drift, or jailbreak - if it can't be replayed, it isn't reported.
6. **MIT/Apache-2.0-compatible dependencies only.** No AGPL paths (PentAGI's VXControl Cloud SDK is a case in point).
7. **Every adapter runs sandboxed with deny-by-default, manifest-declared permissions - from day zero.** An adapter reaches only what its manifest grants. Safe isolation of untrusted adapters is a launch property, not a later add-on (out-of-process delivers it; WASM is only a later speed optimization, FR-402).
8. **The user is never blocked on the vendor to use a protocol.** HTTP-shaped protocols are self-service via manifest-only adapters (FR-407); code adapters can be authored by user or community, not only by us.

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
| Kernel (`kernel-core`), HTTP/REST adapter, collections, drift engine (FR-201), monitors (FR-200), MCP server (FR-300), UI (FR-106/FR-406), ACP chat (FR-301) | **TypeScript** | Entire competitor ecosystem (Bruno, Hoppscotch, Insomnia, Yaak) and the importer formats (FR-104) are JS/TS-native; scripting sandbox (FR-103) is JS by definition; MCP/A2A reference tooling is TS-first. |
| CLI runner (FR-107) | **TypeScript on Bun** (Node fallback) | Startup speed for CI-native headless runs; Node remains the safe fallback if a Bun gap appears. |
| Desktop shell | **Tauri (Rust + webview)** | We already need Rust for the WASI sandbox (FR-402.2); Tauri keeps that Rust native to the shell instead of behind a subprocess bridge. Yaak (our closest architectural sibling) validates this. Electron is the fallback if the all-TS kernel friction outweighs the Rust benefit. |
| WASI adapter sandbox (FR-402.2) | **Rust (embed Wasmtime)** - deferred to post-1.0 | FR-402 forbids `node:wasi` as "not a security boundary," so when this is built it requires a Wasmtime-class runtime (Rust). But no 1.0 adapter needs it (all run out-of-process), so it is deferred - the first *mandatory* Rust is now the desktop shell / load engine, not the sandbox. |
| Load-test engine (FR-202) | **Rust** (TS to prototype) | Accurate p50/p95/p99 under high concurrency is polluted by Node's single-threaded event loop and GC jitter. A Rust worker driven by the TS orchestrator yields trustworthy percentiles. Prototype in TS, migrate when numbers must be trusted. |
| Nuclei (FR-501), PyRIT/garak (FR-601/602), third-party gRPC/MQTT/other adapters (FR-405) | **Their own language, behind the out-of-process/container boundary** | Not our decision. The FR-402.1 boundary exists precisely so adapters are language-agnostic. We call Nuclei as a binary and port only the *shapes* (converters/probes/orchestrators/scorers) of PyRIT into TS - we do not make Go or Python first-class in the core. |

### 7.3 Sequencing rule

**Start 100% TypeScript. Introduce Rust at the desktop-shell / load-engine layer, not the sandbox.** Under the adapter-first plan, the WASM sandbox (FR-402.2) is deferred to post-1.0, so the first Rust you need is the Tauri shell (and, when measurement fidelity demands it, the load engine FR-202) - not the sandbox. Do not pre-emptively go polyglot in the core; the adapter design (FR-400) exists so the core never has to.

### 7.4 Constraint check

All choices satisfy Non-Negotiable #6 (MIT/Apache-2.0-compatible): TypeScript (Apache-2.0), Wasmtime (Apache-2.0), Tauri (MIT/Apache-2.0), Bun (MIT). No AGPL paths introduced.

### 7.5 WASM decision record (why deferred, and the language reality)

**What WASM is:** two things in one name - (a) a compile *target* (you compile a language *to* `.wasm`), and (b) a *sandbox* to run that `.wasm` in (deny-by-default; the code can only do what the host grants). WASI is the standard defining those granted capabilities.

**The language reality (the deciding factor):** WASM matters for running *untrusted third-party* adapters, not our own code - we trust our own code and don't need to sandbox it. But the most likely community adapter author writes **TypeScript/JavaScript, which does not compile to WASM** (JS runs *on* an engine; it is not a WASM guest - AssemblyScript is a TS-*looking* language, not real TS). Of the three languages this project uses: **Rust and C compile to WASM cleanly; TypeScript does not.** So a WASM-only adapter path would lock out exactly the authors most likely to write adapters.

**Pros of WASM (in-process adapter sandbox):** safe execution of untrusted adapters *in-process*; lower startup/call overhead for hot lightweight adapters; one portable artifact across OSes; one cross-platform sandbox instead of three OS-specific profiles; precise CPU/memory ("fuel") limits.

**Cons of WASM:** large, months-long subsystem to build; excludes TS/JS adapter authors; its *safety* benefit is already covered by the day-zero out-of-process OS sandbox (FR-402), so it adds mainly *speed*; heavy native protocol libs (gRPC, native MQTT) fit WASM poorly and want out-of-process anyway; WASI 0.2/0.3 still maturing.

**Decision:** defer WASM to post-1.0 as a *performance* path. Its most natural future home is **our own Rust/C adapters that need speed** (trusted by us, compile to WASM cleanly, called hot) - not untrusted community code, which the OS sandbox already handles safely. The one thing that could pull it earlier: if maintaining three OS-specific sandbox profiles (seccomp/landlock, Seatbelt, AppContainer) becomes a tarpit, WASM's single cross-platform sandbox becomes the cheaper option.

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
