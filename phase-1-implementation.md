## Goal
Deliver the full set of table-stakes features so the client is on par with tools like Bruno and Postman, with each protocol shipped as an adapter.

## Build
- GraphQL adapter — exposes GraphQL request types on the Phase 0 SDK
  - *How it adapts / how we achieve it:* an adapter does **not** stretch to cover new protocols — one adapter = one protocol. The GraphQL adapter is a thin wrapper over an existing library. GraphQL over HTTP is just a POST with `{query, variables}`, so the adapter builds that request, reuses HTTP plumbing to send it, and parses the response; mature libraries handle introspection/validation. If a *brand-new* protocol appears later, it gets its **own** sibling adapter built the same way — GraphQL/WS/SSE are the templates, not special magic. GraphQL is HTTP-shaped, so it is largely manifest-authorable (Type A, FR-407).
- WebSocket (WS) adapter — wraps a WS client, emits stream chunks as kernel events; adapter, not core kernel code
- Server-Sent Events (SSE) adapter — wraps an EventSource-style reader; adapter, keeping the kernel protocol-neutral
- **Manifest-only authoring UX (FR-407)** — a guided editor so a user can self-create a config-only adapter for any HTTP-shaped protocol with zero code, never waiting on us
- Authentication suite (Bearer, Basic, API key, OAuth 2.0 with PKCE and auto-refresh, AWS SigV4) — covers the auth methods users need without hard-coding them into request execution
- Scripting system with a `bru`-style API — lets users run pre-request and post-response JavaScript snippets for dynamic behavior
- Importers for Postman and OpenAPI (OAS) — these convert the most users, so migration friction is reduced early
- Command-line runner (`leo run`) — executes collections headlessly in CI with JSON, JUnit, and TAP output and proper exit codes

## Acceptance Criteria
- Import 3 Postman collections with nothing lost - assertions, variables, and scripts all survive
- The CLI runner (`leo run`) exits with a failure code when an assertion fails inside CI
- GraphQL/WS/SSE each work purely as adapters - removing one removes only that protocol, with no effect on the kernel or other protocols
- A user authors a working adapter for a new HTTP-shaped protocol from a manifest alone, zero code, no vendor involvement (FR-407)

## Gate
A CI pipeline running against a sample collection passes end to end.
