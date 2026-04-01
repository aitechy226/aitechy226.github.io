---
title: "MCP in Production, Part 2: Authentication, Observability, and Operational Design"
date: 2026-04-01
description: "Bearer token auth at the transport layer, correlation IDs across four servers, lazy session init, and clean shutdown — the system-level decisions that make an MCP client deployable."
tags: ["mcp", "production", "architecture", "langgraph", "kyc", "agentic-ai"]
author: "Srikanth Samudrla"
showToc: true
TocOpen: false
draft: false
---

<div class="series-banner">
  <span class="series-label">MCP in Production &middot; Part 2 of 2</span>
  <span class="series-links"><a href="/posts/mcp-production-part-1/">← Part 1: Persistent Sessions, Pooling, and Fault Tolerance</a></span>
</div>

[Part 1](/posts/mcp-production-part-1/) covered the transport layer — keeping sessions alive, recovering from failures, and a few edge cases that only surface when you're running a real pool under real failure conditions. This part covers what I'd call system readiness: the things that separate a working prototype from something I could hand to a client and say "deploy this."

---

## Authentication: I Almost Shipped PII Over Unauthenticated HTTP

Tool call payloads in my KYC system carry `legal_name`, `beneficial_owners`, `sanctions_clear`, and `pep_role`. In a Docker Compose setup, every container in the network can call every MCP server directly — no authentication, no gate. I caught this before shipping, but it was closer than I'd like to admit.

The fix I landed on was bearer token authentication injected at the transport layer. When a session is opened, the token is passed as an HTTP header — so every session in the pool carries auth automatically, with no per-call overhead. On each MCP server, the token is validated before any request reaches the MCP layer.

One implementation detail worth sharing: I used pure ASGI middleware on the server side rather than Starlette's `BaseHTTPMiddleware`. The reason is FastMCP uses Server-Sent Events for streaming. `BaseHTTPMiddleware` buffers responses, which breaks SSE. The pure ASGI approach intercepts at the connection level and never touches the response body. I learned that the hard way on my first attempt.

One `MCP_BEARER_TOKEN` env var is shared across all five containers. Empty means auth is off for local development. Set means the gate is live on every server.

### What This Doesn't Cover

I want to be honest about the limits. Bearer tokens over plain HTTP protect against unauthorized callers within the Docker network. They don't encrypt payloads in transit — anything that can observe the Docker bridge can read the data. And they don't verify caller identity: any container with the token is trusted equally.

For a real compliance deployment, mTLS is the right answer. It encrypts inter-service traffic and lets each server verify the caller's identity specifically — not just that they have the shared secret. That's the next step; bearer tokens are the floor, not the ceiling.

---

## Correlation IDs: Debugging Across Four Servers Was a Nightmare

Early on, debugging a failed case meant opening four log files and trying to correlate entries by timestamp. With 8–10 tool calls across 4 servers, that was genuinely painful.

The fix was straightforward once I decided to do it: thread a `case_id` through `call_tool` as a keyword-only parameter and inject it into every tool call's arguments before it hits the server. Each server logs `case_id` at tool entry. Every log line for a given case, across all four servers, now shares one identifier. One grep gives you the complete trace.

Two design details I'm glad I got right: making `case_id` keyword-only means existing callers don't break when you add it. And creating a new dict rather than mutating the caller's arguments avoids subtle bugs when the same arguments dict gets reused. Small things, but worth getting right upfront.

---

## Three Operational Decisions That Made Deployment Easier

### Open sessions lazily, not at startup

In Docker Compose, containers start in parallel. If I opened sessions at startup, the orchestrator's readiness would depend on all four MCP servers being up first. Lazy initialization inverts that: the orchestrator starts immediately, the first tool call triggers pool filling, and the retry logic handles a server that isn't ready yet. This also means adding a new server requires no changes to startup order or health-check configuration.

### All thresholds in config, not code

Every timeout, retry parameter, pool size, and heartbeat interval is an environment variable. A client deploying this system can tune for their network without touching code. I can also disable the heartbeat entirely with `MCP_HEARTBEAT_INTERVAL_SECONDS=0`, which simplifies test setups considerably.

### Shut down cleanly

I wire the heartbeat start and stop into the FastAPI lifespan context manager. On shutdown, `close_all_sessions()` cancels the heartbeat, waits for it to finish, then evicts every session in every pool. Without this, the server logs are full of errors from sessions closed mid-request. It's a small thing that makes production logs much easier to read.

---

## Tradeoffs Worth Knowing

**False positive evictions.** If a healthy server is momentarily overloaded and the `ping` tool responds slowly, the session gets evicted unnecessarily. It recovers on the next call, but at the cost of a fresh session open. This is why the `ping` tool has no I/O — a slow response from a no-I/O tool is an unambiguous signal that something is wrong with the server, not normal variance.

**Pool size vs. idle connections.** Each session holds an open SSE connection for the lifetime of the process. A pool size of two means two idle connections per server, always open. Setting it too high multiplies that across all servers. I stayed at two — enough to tolerate one dead session without blocking callers.

**Heartbeat doesn't protect in-flight calls.** If a server dies while a tool call is in flight, that call fails and reactive eviction handles it. The heartbeat only helps the *next* call that would have hit a stale session. The in-flight failure won't hang forever — it's bounded by the per-call timeout — but it will still fail.

---

## Conclusion

Looking back across both posts, every decision traces back to something that broke during testing. I didn't design any of this speculatively.

- Per-call overhead → session pool
- Single session fragility → pool with round-robin
- In-flight hangs → per-call timeouts
- Thundering herd → exponential backoff with jitter
- Late failure detection → heartbeat on the real code path
- Unauthenticated PII → bearer tokens at the transport layer
- Four-server log chaos → correlation IDs threaded through every call

The systems I trust most are the ones built by breaking things deliberately and fixing the actual root cause — not the ones built by reading a resilience checklist upfront. Build it, break it, understand why, fix it. That's the loop.

---

*← [Part 1: Persistent Sessions, Pooling, and Fault Tolerance](/posts/mcp-production-part-1/)*
