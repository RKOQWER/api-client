## Goal
Treat AI agents as first-class users of the client: they can list and run collections, and when the API drifts, an agent can propose the fix for a human to approve.

## Build
- Built-in MCP server — exposes tools agents can call (list, run, generate, repair) plus hooks into the security tools, so coding agents operate on collections
- In-app agent chat via Agent Client Protocol (ACP) — gives agents a conversational surface to onboard collections, execute runs, and read findings
- Drift-triage agent — reviews contract drift and suggests collection patches that a human approves
- A2A adapter — imports agent cards as a request type so agent-era APIs can be tested directly

## Acceptance Criteria
- Claude Code can discover our MCP tools and run a collection on its own, without being hand-held
- A drift-fix patch suggested through ACP is accepted into a pull request

## Gate
A full end-to-end flow where a commit adds a drift check that an agent authored via ACP.
