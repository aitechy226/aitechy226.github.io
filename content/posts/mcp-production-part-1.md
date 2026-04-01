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

This is Part 1 of a two-part series on what it takes to run an MCP client reliably. I'll cover the transport layer: sessions, pooling, dead connection recovery, timeouts, and the heartbeat. [Part 2](/posts/mcp-production-part-2/) covers the system layer: authentication, observability, and operational design.

The context is a KYC Onboarding Orchestrator I built that calls four MCP servers — Moody's entity data, sanctions/PEP screening, CRM, and document generation — for every onboarding case. Each case makes 8–10 tool calls. Every decision below was made in response to something that actually broke.

---

## Per-Call Connections Don't Scale

I started with the naive design: open a fresh session per tool call, run the tool, close the session. Simple, stateless, nothing to manage.

The problem: every call pays full session establishment cost — TCP connect, MCP `initialize` handshake, then the actual tool call. Inside a Docker network that's a few milliseconds. Against a real Moody's or Refinitiv endpoint over the internet, it's 3–4 RTTs of latency on every single call, multiplied across 8–10 calls per case. There's also a server-side cost — every `initialize` allocates session state that is immediately discarded. I wasn't prototyping anymore, so I needed to fix this.

The fix is obvious: keep sessions open and reuse them.

---

## Decision 1: A Pool, Not a Single Session

My first instinct was one persistent session per server. That fixes the overhead problem but creates a new one: the session becomes a single point of failure. If it dies while a tool call is in flight, everything blocks until the dead session is detected and replaced.

A pool tolerates one dead session without impacting callers on other sessions. I went with a pool of two per server — enough to survive one failure without blocking, without flooding a small server with idle connections.

One design detail I'm glad I got right upfront: the pool fill uses a partial-failure policy. If the server is reachable but only opens one session successfully, the caller gets that one session and proceeds. A pool of one is still a pool. Failing hard when you could degrade gracefully is the wrong call.

Round-robin selection across the pool also means a dead session gets discovered within at most two calls — not after some arbitrary delay.

---

## Decision 2: Evict the Session, Not the Server

When a server restarts, the pool holds stale session objects pointing at dead TCP connections. The next call fails. My first implementation evicted the entire pool entry for that server — which worked fine with one session, but caused a problem under a pool.

If two sessions to the same server die simultaneously, two concurrent callers both try to evict. If eviction removes the whole server entry, the second caller finds nothing and may open duplicate sessions. The correct fix is to evict the specific dead session and guard against the second caller — if the session is already gone by the time the second caller tries, just return cleanly.

One thing that bit me here: the transport layer fires its own cancellation when a connection dies, and `asyncio.CancelledError` is a `BaseException`, not an `Exception`. My original eviction handler caught `Exception` — which meant the cancellation escaped the handler and the session was never removed from the pool. Adding `asyncio.CancelledError` to the catch fixed it. Obvious in hindsight, not obvious when you're looking at a session that should be evicted but isn't.

---

## Decision 3: Timeout + Exponential Backoff

Two more failure modes I hit that needed explicit fixes:

**In-flight hang.** When `docker stop` kills a container, the network namespace stays alive but nothing is listening. A tool call in flight at that moment waits for a TCP response that will never arrive. Without a timeout, the asyncio task blocks indefinitely and the case stays stuck. A per-call timeout bounds every tool call — when it fires, the session is evicted and the retry cycle begins.

**Thundering herd on restart.** I originally used a fixed retry delay. Under concurrency, every failed caller wakes up at exactly the same moment and hammers the recovering server simultaneously. The fix is exponential backoff with jitter:

```
delay = min(BASE * 2^attempt + random(0, 1), MAX)
```

The jitter — that `random(0, 1)` — is the part that actually matters. It desynchronizes callers so the recovering server sees a trickle rather than a burst. I expose the base and max as environment variables so they can be tuned per deployment without touching code.

---

## Decision 4: The Heartbeat Probe Must Test the Real Code Path

The reactive eviction handles failures after they hit. I added a background heartbeat to catch them proactively — a task that pings every session in the pool on a fixed interval and evicts any that fail before real traffic reaches them.

The probe design mattered more than I expected. My first version used `list_tools()` as the ping. It seemed reasonable — if the server can respond to a discovery call, it's alive. But `list_tools()` is never called in production. My graph nodes call tools by hardcoded name. I was testing an operation that didn't exist in the real flow.

I replaced it with a dedicated `ping` tool on each server that returns `"pong"` immediately with no I/O. The heartbeat calls it through the exact same function every production tool call uses. If that path is broken, the session gets evicted. Testing the wrong path gives you false confidence.

The ping also needs its own timeout. When a container is gracefully stopped, Docker keeps the network namespace alive but stops accepting connections — without a timeout, the ping hangs forever. Five seconds is enough because the `ping` tool has no I/O. If a no-I/O tool takes more than 5 seconds to respond, the server has a real problem.

### The Prerequisite I Almost Missed

A heartbeat only works if the server can respond to a ping while another tool is in progress. FastMCP runs on uvicorn — a single async event loop. If any tool function makes a synchronous I/O call, it blocks the entire loop. My document generation server originally used the synchronous Anthropic client for LLM calls. A 15-second LLM call meant the heartbeat ping waited 15 seconds and timed out, falsely evicting a healthy session.

Switching to the async Anthropic client fixed it. This isn't a performance optimization — it's a correctness requirement for any async server making I/O calls.

---

## Summary

Each decision came from something that broke:

| Problem | Fix |
|---------|-----|
| Per-call handshake overhead | Persistent session pool |
| Single session = single point of failure | Pool with round-robin selection |
| Dead sessions block callers | Per-session eviction with `CancelledError` catch |
| In-flight hang on server stop | Per-call timeout |
| Thundering herd on restart | Exponential backoff with jitter |
| Dead sessions hit before detected | Background heartbeat |
| Heartbeat tests wrong code path | Dedicated `ping` tool on real call stack |
| Blocking I/O breaks heartbeat | Async I/O in all server tool functions |

[Part 2](/posts/mcp-production-part-2/) covers the system-level concerns: protecting PII-carrying tool calls in transit, making 8–10 calls across 4 servers traceable to a single case, and the operational decisions that make this deployable on a client system.

---

*Part 2: [Authentication, Observability, and Operational Design →](/posts/mcp-production-part-2/)*
