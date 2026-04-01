---
title: "MCP in Production, Part 1: Persistent Sessions, Pooling, and Fault Tolerance"
date: 2026-04-01
description: "Five transport-layer decisions — session pooling, eviction, cancel scope isolation, timeouts, and heartbeat design — each driven by a real failure in a KYC onboarding system."
tags: ["mcp", "production", "architecture", "langgraph", "kyc", "agentic-ai"]
author: "Srikanth Samudrla"
showToc: true
TocOpen: false
draft: false
---

Most MCP client examples open a session, call a tool, and close the session. That pattern is fine for demos. It breaks in production in ways that aren't obvious until you're staring at a hung process or a spike in latency.

This is Part 1 of a two-part series on what it takes to run an MCP client reliably. Part 1 covers the transport layer: sessions, pooling, dead connection recovery, timeouts, and the heartbeat. [Part 2](/posts/mcp-production-part-2/) covers the system layer: authentication, observability, and operational design.

The context is a KYC Onboarding Orchestrator that calls four MCP servers — Moody's entity data, sanctions/PEP screening, CRM, and document generation — for every onboarding case. Each case makes 8–10 tool calls. Every design decision below was made in response to an observed failure, not speculation.

---

## Per-Call Connections Don't Scale

The naive design opens a fresh session per tool call: TCP connect → MCP `initialize` → `tools/call` → close. Simple, stateless, no shared state to manage.

The problem: every call pays full session establishment cost. Inside a Docker network that's a few milliseconds. Against a real Moody's or Refinitiv endpoint over the internet, it's 3–4 RTTs of latency on every single call, multiplied across 8–10 calls per case. There is also a server-side cost — every `initialize` allocates session state that is immediately discarded. This is not a demo-to-production scaling issue. It's just wasteful.

The fix is obvious: keep sessions open and reuse them.

---

## Design Decision 1: A Pool, Not a Single Session

A single persistent session per server fixes the overhead problem but creates a new one: that one session becomes a single point of failure. If it dies, every in-flight caller blocks until the dead session is detected and replaced.

A pool tolerates one dead session without impacting callers on other sessions. The registry reflects this:

```python
_sessions: dict[str, list[ClientSession]] = {}
_stacks:   dict[str, list[AsyncExitStack]] = {}
_rr_index: dict[str, int] = {}   # round-robin counter per server
```

The pool is filled on first use, up to `MCP_POOL_SIZE` (default 2). One design decision worth calling out: the fill loop uses a partial-failure policy. If the server is reachable but only opens one session successfully, `_get_session` returns that one session rather than failing hard. A pool of one is still a pool — the caller gets a session and can proceed.

Round-robin selection spreads load across sessions and means a dead session will be discovered within at most `pool_size` calls, not after an arbitrary delay.

---

## Design Decision 2: Evict the Session, Not the Server

When a server restarts, the pool holds stale `ClientSession` objects pointing at dead TCP connections. The next call fails. The correct response is to evict the specific dead session, not the entire pool entry.

This matters because two sessions to the same server can die simultaneously. If eviction removed the whole server entry, the second eviction would find nothing to remove and potentially open duplicate sessions. Instead, `_evict_session` takes the specific session as an argument and uses a `ValueError` guard — the second caller sees the session already removed and returns cleanly.

One non-obvious requirement: `stack.aclose()` must catch both `Exception` and `asyncio.CancelledError`:

```python
try:
    await stack.aclose()
except (Exception, asyncio.CancelledError) as exc:
    log.warning("Error closing stale session for %s: %s", server_url, exc)
```

`asyncio.CancelledError` is a `BaseException`, not an `Exception`. The `streamablehttp_client` transport runs an anyio SSE receiver as a background task. When the server dies, anyio calls `task.cancel()` on the task that entered the transport context. Without the explicit `CancelledError` catch, that cancel escapes `_evict_session` and the session is never removed from the pool.

---

## Design Decision 3: Isolate anyio Cancel Scopes

There is a subtler issue with anyio and asyncio that only surfaces under a pool.

anyio cancel scopes are task-local — but when a transport dies, anyio's cancellation propagates via `asyncio.Task.cancel()` on the task that entered the transport context. If that task awaits the tool call directly, the cancel lands in its own cancel scope state and re-fires at every subsequent anyio checkpoint in the same task — even after the session has been evicted and the except block has run.

With a pool of two dead sessions, this means two `cancel()` calls on one task. The second fires *inside* `_evict_session`'s exception handler, which is exactly where you don't want it.

The fix: run every tool call in an isolated child task. Cancel scopes are task-local, so the dead transport's anyio state is contained in the child and cannot leak into the parent:

```python
inner = asyncio.get_event_loop().create_task(
    _call_on_session(session, tool_name, arguments)
)
try:
    return await asyncio.wait_for(asyncio.shield(inner), timeout=timeout)
except (asyncio.TimeoutError, asyncio.CancelledError, Exception):
    inner.cancel()
    try:
        await inner
    except (asyncio.CancelledError, Exception):
        pass
    raise
```

`asyncio.shield(inner)` prevents `wait_for` from cancelling the inner task directly on timeout — cancellation goes through the explicit `except` block so the inner task is always awaited and never left as an orphaned coroutine. Every production tool call and every heartbeat ping goes through this wrapper.

---

## Design Decision 4: Timeout + Exponential Backoff

Two failure modes that need explicit handling:

**In-flight hang.** When `docker stop` kills a container, the network namespace stays alive but nothing is listening. An in-flight tool call waits for a TCP response that will never arrive. Without a timeout, that asyncio task blocks indefinitely. `MCP_CALL_TIMEOUT_SECONDS` bounds every call. When it fires, the session is evicted and the retry cycle begins.

**Thundering herd on restart.** A fixed retry delay means every failed caller wakes up at the same moment and hammers a server that just recovered. The fix is exponential backoff with jitter:

```
delay = min(BASE * 2^attempt + random(0, 1), MAX)
```

The `random.uniform(0, 1)` component is the critical part — it desynchronizes callers so the recovering server sees a trickle, not a burst. Expose `RETRY_BASE_DELAY` and `RETRY_MAX_DELAY` as environment variables; never hardcode them.

---

## Design Decision 5: The Heartbeat Probe Must Exercise the Real Code Path

The reactive eviction handles failures after they hit. The heartbeat detects them proactively: a background task pings every session in every pool on a fixed interval and evicts any that fail before real traffic reaches them.

The probe design matters more than it seems. This orchestrator does not use LLM-driven tool discovery — tools are called by name, hardcoded in the graph. `list_tools()` is never called in production. Using it as a heartbeat probe would test an operation that does not exist in the real flow and could pass even when `tools/call` is broken.

Instead, each MCP server exposes a dedicated `ping` tool that returns `"pong"` immediately with no I/O. The heartbeat calls it via the exact same function (`_call_on_session_isolated`) that production tool calls use — exercising the real `tools/call` code path over the real session. If that path is broken, the session gets evicted.

The ping must also have its own timeout (`MCP_PING_TIMEOUT_SECONDS`, default 5s). When a container is gracefully stopped, Docker keeps its network namespace alive but stops accepting connections — without a timeout, the ping hangs forever.

### A Prerequisite: Non-Blocking Server Code

A heartbeat ping only works if the server can respond while another tool is in progress. FastMCP runs on uvicorn — a single-process async server. If any tool function makes a synchronous I/O call, it blocks the entire event loop. A 15-second LLM call with `anthropic.Anthropic().messages.create(...)` means the heartbeat `ping` waits 15 seconds and times out, falsely evicting a healthy session.

The fix is to use the async client (`AsyncAnthropic`). This is not a performance optimization — it is a correctness requirement. Any I/O in a tool function that blocks the event loop makes the heartbeat unreliable and every other concurrent request invisible to the server.

---

## Summary

Each decision addressed a real failure mode:

| Problem | Solution |
|---------|----------|
| Per-call handshake overhead | Persistent session pool |
| Single session = single point of failure | Pool with round-robin selection |
| Dead sessions block callers | Per-session eviction, `CancelledError` catch |
| anyio cancel scope leakage | Isolated child task per call |
| In-flight hang on server stop | Per-call timeout |
| Thundering herd on restart | Exponential backoff with jitter |
| Dead sessions hit before detected | Background heartbeat |
| Heartbeat tests wrong code path | Dedicated `ping` tool via real call stack |
| Blocking I/O breaks heartbeat | Async I/O in all server tool functions |

[Part 2](/posts/mcp-production-part-2/) covers the system-level concerns: how PII-carrying tool calls get protected in transit, how 8–10 calls across 4 servers become traceable to a single case, and the operational design decisions that make this deployable on a client system.

---

*Part 2: [Authentication, Observability, and Operational Design →](/posts/mcp-production-part-2/)*
