## Goal
Get the project structure in place and prove the protocol-neutral kernel + adapter model works on HTTP before any harder protocol.

## Build
- A single TypeScript monorepo — holds all code packages in one place so adapters and the kernel evolve together
- `kernel-core` package — the protocol-neutral engine exposing only generic adapter verbs (open/send/events/close + secrets/snapshots/timeline) with no HTTP knowledge inside it
- Minimal HTTP fetch-plumbing inside the kernel — used only to reach the registry and download adapters (bootstrap exception), not as a testable protocol surface
- First draft of the adapter SDK + manifest — TypeScript types plus JSON Schema manifest for the out-of-process runtime, so every protocol plugs in through the same contract
- **Day-zero adapter sandboxing** — the out-of-process runtime enforces deny-by-default, manifest-declared permissions (network hosts/ports, FS paths). Even the first adapters run isolated; a stranger's adapter cannot touch anything its manifest doesn't grant. (This is what makes untrusted adapters *safe* — WASM, deferred, would only make them *faster*.)
  - *What "out-of-process sandbox" means (two separate protections — don't confuse them):* **(1) out-of-process = memory isolation** — the adapter runs as its own OS process with private memory, messaged over a pipe; it can't read or corrupt our app, and a crash kills only itself. **(2) sandbox = action permissions** — the OS restricts what that process may do to files/network/system, deny-by-default, from the manifest. Out-of-process is about *memory*; sandbox is about *actions*; we use both.
  - *How it's enforced — no Docker:* OS primitives drive it — Linux `seccomp`/namespaces/`landlock`, macOS Seatbelt, Windows AppContainer — with a profile built from the manifest. Normal adapters need no container and no extra install. (Container/Docker is used only later for the heavy offensive tooling in Phase 5.)
- HTTP/REST adapter built against that SDK and pre-bundled — proves the very first request works through the adapter model
- **Manifest-only adapter authoring v0 (FR-407)** — a user can define a basic HTTP-shaped adapter from a manifest alone, no code, so the product is never vendor-captive from day one
- Local file store with JSON Schema validation — saves plain files to disk and rejects malformed data before it is written
  - *Why we need it + who supplies the schema:* the schema here is **our own**, authored and shipped by us — it defines what a valid collection/request/manifest file looks like. Collections are Git-native and hand-editable, so a user, an agent, or a bad merge can corrupt a file; validation rejects a malformed file before it loads so the app never crashes on half-broken input. This is a *different* schema from the OpenAPI spec in Phase 2 (which validates API **responses** and is supplied by the user). Do not confuse the two.
- Response inspector — views what comes back from a request so the adapter flow is observable end to end
- OS keychain secret storage — keeps passwords and tokens out of project files and version control
  - *How, at a high level:* we don't build a vault; we use the OS's own (macOS Keychain, Windows Credential Manager/DPAPI, Linux Secret Service) via a thin cross-platform library. The collection file stores only a **reference** like `{{secret:prod-key}}`; at send time the kernel resolves it from the OS keychain, injects the real value, sends, and masks it in logs/UI. Plaintext lives only in OS-protected storage and briefly in memory.

## Acceptance Criteria
- We can send an HTTP request through the HTTP adapter with no user interface involved (headless) and the kernel contains zero HTTP-specific code
- A file is correctly validated against its JSON Schema (the schema is ours, shipped with the product — see Build note above)
- No secret ever ends up written into a collection file
- A sample manifest-only adapter authored by hand (no code) works end to end

## Gate
A manual once-over passes, a basic end-to-end browser test (Playwright) is green, the kernel has zero protocol assumptions (the HTTP adapter can be removed and the kernel still builds), and the hand-authored manifest-only adapter works.
