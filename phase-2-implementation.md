## Goal
Ship the local-first differentiators: scheduled monitors, the contract-drift engine, and lightweight load testing.

## Build
- Scheduler — runs collections automatically on a timer (in-app) or on demand from CI, so monitoring happens without vendor cloud
- Latency budgets (p50/p95/p99) with alerts — flags when an API gets too slow so performance regressions are caught early
- Contract-drift validator — compares every response against the API's OpenAPI spec, maps tested endpoints, and blocks a merge when the API drifts without a spec update
  - *Where the OpenAPI spec comes from — the user supplies it.* It is the API's own contract, authored by the API's team. It reaches us one of three ways: (1) the user points at a local `openapi.yaml` in their repo (most common), (2) a URL where the API serves its spec, or (3) it arrived during import (FR-104). We never generate it. No spec supplied → nothing to validate against (drift detection is opt-in per the spec they give). Note: this is a *different* schema from our own collection-file schema in Phase 0.
- Load-test runner — ramps up QPS from any folder to measure p50/p95/p99 performance locally
  - *High-level overview:* replay a folder of requests in a loop at a controlled, ramping rate (e.g. 10 → 100 → 500 req/s), fire them concurrently, record every single request's latency, then compute percentiles from that distribution. One click, fully local — no cloud. Orchestration can start in TypeScript; the actual load generation moves to a Rust worker for trustworthy numbers (see the AC note below).
- Response snapshots with visual diff — shows exactly what changed between runs and environments
  - *Why it's necessary / the pain point:* "the API changed and I didn't notice until it broke something." Re-running a request normally discards the old response — so a renamed field, a type change, or a now-sometimes-null field slips by silently. Snapshots keep every historical response; the diff shows field-by-field what changed between two runs or two environments (staging vs prod). It is `git diff` for API responses — humans can't eyeball JSON blobs and catch a one-field change; the diff does.
- One-click promotion of findings — turns any drift, perf, or assertion finding into a permanent collection test or monitor check
  - *Baseline story:* you run `/users`; the drift engine flags "`email` is missing but your spec says it's required." That's a one-time **finding**. Click **"Promote to test"** and it becomes a *permanent assertion* baked into the collection ("`/users` must always return `email`"), saved as plain text in Git so teammates get it too. Every future run and CI pipeline now checks it automatically. Value: turn a fleeting observation into a permanent regression guard with one click, instead of hand-writing the assertion. This same rail promotes drift, perf, vuln, and jailbreak findings alike (FR-204) — "you found it once; you never find it by hand again."

## Acceptance Criteria
- A scheduled run fires an alert when a latency budget is breached
- A drift report blocks a merge when an endpoint was changed without updating its spec
- A load test produces real p50/p95/p99 numbers
  - *How we guarantee the numbers are real, vs how others do it:* established tools (k6, wrk, Vegeta, JMeter) measure latency at the client into an accurate histogram (HDR-style) and report true percentiles, not averages — and critically avoid **coordinated omission** (a naive loop that waits for a slow response before sending the next under-counts slow requests and fakes a good p99). We match that bar by: (1) timestamping with a monotonic clock right at send/receive; (2) open-model load — send at the target rate regardless of response timing; (3) an HDR-style histogram for accurate tails; (4) running the generator **out-of-process in Rust** so Node's event loop / GC pauses don't pollute the measurement (you'd otherwise be measuring our tool, not the API); (5) storing raw per-request timings in the snapshot so the user can re-derive the percentiles themselves — no black box.
- A snapshot diff highlights a changed field

## Gate
A live demo of the headline story - change the API without changing its spec, and watch CI block the merge.
