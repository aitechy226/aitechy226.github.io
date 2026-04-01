---
title: "Start Here"
layout: "single"
url: "/start/"
summary: "A map of the writing on this site — where to start and how the pieces connect."
---

This site is about building AI systems that actually work in production — not demos. Here's a map of the writing and how the pieces connect.

---

## On Agentic Architecture

The foundation. What agentic systems actually look like when you move past the demo and into something deployable.

- [Why Your AI Agent Demo Looks Great and Your Production System Doesn't](/posts/agentic-hype-vs-reality/) — The gap between demos and production, the four buckets most "crushing it" claims fall into, and the architecture pattern I keep returning to.

- [Designing a Professional Digital Twin: The Architecture](/posts/professional-digital-twin/) — What it looks like when you model professional expertise as an agent specification — personas, tools, skills, rules, and memory. Includes the full [Sri System](/sri-system/).

---

## On MCP in Production

A ground-up account of using the Model Context Protocol as a service-to-service layer in a regulated KYC system — the tradeoffs of the pattern and what it takes to run it reliably.

- [I Used MCP as a Service-to-Service Protocol. Here's What I Learned.](/posts/mcp-service-to-service/) — Why I used MCP as a transport layer between a LangGraph orchestrator and four integration servers, and the tradeoffs that come with it.

- [MCP in Production, Part 1: Persistent Sessions, Pooling, and Fault Tolerance](/posts/mcp-production-part-1/) — Five transport-layer decisions, each driven by a real failure: session pooling, dead connection eviction, cancel scope isolation, timeouts, and heartbeat design.

- [MCP in Production, Part 2: Authentication, Observability, and Operational Design](/posts/mcp-production-part-2/) — Bearer token auth at the transport layer, correlation IDs across four servers, lazy session init, and clean shutdown.

---

*New posts go to [/posts/](/posts/). Organized by topic as the archive grows.*
