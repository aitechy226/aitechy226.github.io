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

[Part 1](/posts/mcp-production-part-1/) covered the transport layer — keeping sessions alive, recovering from failures, and handling the anyio/asyncio cancellation edge cases that only surface under a real pool. This part covers everything else that separates a working prototype from something you can actually deploy: authentication, traceability across servers, and the operational design decisions that make the system self-contained.

---

## Authentication: Close the Network Gap Before It Reaches Code Review

Tool call payloads in a KYC system carry `legal_name`, `beneficial_owners`, `sanctions_clear`, and `pep_role`. In a Docker Compose setup, every container in the network can call every MCP server directly with no authentication. That fails a security review before it gets to one.

The fix is bearer token authentication injected at the transport layer. On the client side, `_open_session` passes the token as a header when establishing each session — so every session in the pool carries auth automatically, with no per-call overhead:

```python
headers = {"Authorization": f"Bearer {MCP_BEARER_TOKEN}"} if MCP_BEARER_TOKEN else None
read, write, _ = await stack.enter_async_context(
    streamablehttp_client(server_url, headers=headers)
)
```

On each MCP server, the token is validated in a pure ASGI middleware class before any request reaches the MCP layer. The key design decision here is *pure ASGI* over Starlette's `BaseHTTPMiddleware`. FastMCP uses Server-Sent Events for streaming. `BaseHTTPMiddleware` buffers responses, which breaks SSE. A pure ASGI `__call__(scope, receive, send)` intercepts at the connection level and never touches the response body.

One `MCP_BEARER_TOKEN` env var is shared across all containers. Empty means auth is off (dev/test pass-through). Set means the gate is live on all servers.

### What This Doesn't Cover

Bearer tokens over plain HTTP protect against unauthorized callers within the Docker network. They do not encrypt payloads in transit — anything that can observe the Docker bridge network can read the PII. They also do not verify caller identity: any container with the token is trusted equally.

For a compliance system, **mTLS** is the right next step. It encrypts all inter-service traffic and requires each container to present a certificate, so the sanctions server can verify it is talking to the orchestrator specifically — not just any process that obtained the shared secret.

---

## Correlation IDs: Make Multi-Server Traces Readable

A single onboarding case generates 8–10 tool calls across 4 servers. Without a shared identifier, debugging means correlating timestamps across four log streams and guessing which entries belong together.

The fix is a `case_id` threaded through `call_tool` as a keyword-only parameter:

```python
async def call_tool(
    server_url: str,
    tool_name: str,
    arguments: dict,
    *,
    case_id: str = "",
) -> dict | str:
    if case_id:
        arguments = {**arguments, "case_id": case_id}
```

`{**arguments, "case_id": case_id}` creates a new dict — the caller's original is never mutated. Each MCP server logs `case_id` at every tool entry. The result: every log line for a given case, across all four servers, shares one identifier. `grep case_id=<id>` gives you the complete trace.

The parameter is keyword-only (`*`) and defaults to empty. Callers that don't need tracing pass nothing. Empty is treated as absent — the field is not injected and no extra log line is emitted.

---

## Operational Design: Three Decisions That Affect Deployability

### Open Sessions Lazily

In Docker Compose, containers start in parallel. Opening sessions at startup couples the orchestrator's readiness to every MCP server being up. Lazy initialization inverts this: the orchestrator starts immediately, the first tool call triggers pool filling, and `call_tool`'s retry logic handles a server that isn't ready yet. Adding a new MCP server to the system requires no changes to startup order or health-check dependencies.

### All Thresholds in Config, Not Code

`MCP_CALL_TIMEOUT_SECONDS`, `MCP_PING_TIMEOUT_SECONDS`, `MCP_HEARTBEAT_INTERVAL_SECONDS`, `MCP_POOL_SIZE`, `RETRY_BASE_DELAY`, `RETRY_MAX_DELAY` — all env vars. A client deploying this system can tune timeouts for their network without touching code. `MCP_HEARTBEAT_INTERVAL_SECONDS=0` disables the heartbeat entirely, which simplifies test setups.

### Shut Down Cleanly

The heartbeat is started at application startup and stopped at shutdown via the FastAPI `lifespan` context manager. `close_all_sessions()` cancels the heartbeat task, waits for it to finish, then evicts every session in every pool. This releases server-side session state cleanly and avoids spurious error log entries from sessions closed mid-request.

---

## Tradeoffs Worth Knowing

**False positive evictions.** If a healthy server is momentarily overloaded and the `ping` tool takes longer than `MCP_PING_TIMEOUT_SECONDS`, the session gets evicted unnecessarily. It recovers on the next call — but at the cost of a fresh session open. This is why the `ping` tool has no I/O: a slow response from a no-I/O tool is an unambiguous signal of event-loop saturation, not normal variance.

**Pool size vs. open connections.** Each session holds an open SSE connection. `MCP_POOL_SIZE=2` means two idle SSE connections per server for the process lifetime. Too high a value multiplies idle connections across all servers. Tune based on observed concurrency — the default of 2 is conservative and correct for most deployments.

**Heartbeat doesn't protect in-flight calls.** If a server dies while a tool call is in flight, the call fails and reactive eviction handles it. The heartbeat only improves the *next* call that would have hit a stale session. The in-flight failure is bounded by `MCP_CALL_TIMEOUT_SECONDS` — it will fail fast, not hang — but it will still fail.

---

## Conclusion

Across both parts, every design decision has the same origin: something broke during testing.

- Per-call connections → per-call overhead → session pool
- Single session → single point of failure → pool with round-robin
- Dead sessions not evicted → anyio `CancelledError` escaping `except Exception` → explicit catch
- Cancel scope leakage across pool sessions → isolated child task per call
- In-flight hangs → per-call timeout
- Thundering herd on retry → exponential backoff with jitter
- Reactive eviction too late → heartbeat on real code path
- Unauthenticated PII → bearer tokens at transport layer
- 4-server log correlation impossible → `case_id` injected at client, logged at every server

None of these were added speculatively. The right approach to resilience is not to read a checklist and implement everything upfront — it is to build a minimal working system, break it deliberately, understand the root cause of each failure, and fix it at that root. Resilience designed around real failures is reliable. Resilience designed around hypothetical failures is overhead.

---

*← [Part 1: Persistent Sessions, Pooling, and Fault Tolerance](/posts/mcp-production-part-1/)*
