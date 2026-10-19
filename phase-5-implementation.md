## Goal
Add built-in penetration testing: automated security scanning of APIs, running safely inside containers with explicit permission, reporting only proven vulnerabilities.

## Build
- Authorization gate plus scope manifests — forces the user to explicitly declare what hosts and ports may be scanned, with no exceptions
- Nuclei scanner as an out-of-process adapter inside a container — never runs on the user's own machine, keeping the host clean
  - *Why a container here, when normal adapters don't need one:* ordinary protocol adapters (Phase 0-3) are isolated with the OS sandbox alone (seccomp/landlock, Seatbelt, AppContainer) — no Docker. The offensive tooling is different: it runs genuinely dangerous untrusted input (Nuclei has had RCE CVEs) and bundles large toolchains, so an OS profile isn't enough. These get full **container (OCI) or microVM (Firecracker/gVisor)** isolation. "Container," not "Docker" specifically — any OCI runtime works.
- API-focused template packs mapped to OWASP API Security Top 10 — targets real API vulnerabilities with deterministic templates
- Verified-exploit agent loop (Strix-style) inside the container — recon, exploit, then verify; only PoC-verified findings surface
- One-click conversion of a vulnerability finding into a regression test — turns a security finding into a permanent collection test
- `run_security_scan` tool exposed to coding agents, plus a generator that drafts fix pull requests — always human-approved before anything lands

## Acceptance Criteria
- A deterministic Nuclei scan of a deliberately vulnerable sample API finds the planted vulnerabilities with zero false positives across repeated runs
- Every finding from the LLM-driven exploit loop ships a replayable proof capsule, and only PoC-verified findings surface - unverifiable or hallucinated findings are dropped
- The authorization gate refuses any target that wasn't explicitly attested
- A finding converts cleanly into a regression test

## Gate
The false-finding rate stays under the agreed threshold on a bug-bounty-style sample suite, every surfaced LLM-loop finding replays successfully from its capsule, and a container-escape audit passes.
