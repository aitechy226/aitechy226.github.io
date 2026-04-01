---
title: "I Used MCP as a Service-to-Service Protocol. Here's What I Learned."
date: 2026-03-31
description: "MCP was designed as an LLM-to-tool protocol. I used it as a service-to-service layer between a LangGraph orchestrator and independently deployable integration servers. It worked — with real tradeoffs."
tags: ["mcp", "agents", "agentic-ai", "architecture", "langgraph", "kyc"]
author: "Srikanth Samudrla"
showToc: true
TocOpen: false
draft: false
---

When I designed the architecture for my KYC onboarding orchestrator, I made a deliberate choice: use MCP not as an LLM-to-tool protocol — the way it was originally designed — but as a service-to-service protocol between a LangGraph orchestrator and a set of independently deployable integration servers.

It worked. But it came with real tradeoffs I want to document, because I don't think this pattern is well understood yet.

---

## Background: What I Built

The system onboards corporate clients through a fixed sequence of checks — entity profile retrieval, credit rating, sanctions screening, PEP check, CRM update, document generation. Each of those integrations runs as a separate MCP server. A LangGraph graph orchestrates the sequence by calling MCP tools directly from its nodes.

Four MCP servers. One orchestrator. No LLM in the routing path.

---

## The Two Patterns

Before getting into tradeoffs, it's worth being precise about what distinguishes these patterns.

**The original MCP pattern — LLM-to-tool:**

```
User prompt
  → LLM reads tool schemas via list_tools()
  → LLM decides: call get_company_profile("Acme Corp")
  → MCP client executes
  → result returned to LLM
  → LLM decides next call
  → ... and so on
```

The LLM is the orchestrator. It reads available tools at runtime, sequences the calls, and interprets results. This is how Claude Desktop works with MCP servers.

**My pattern — service-to-service:**

```python
# graph/nodes.py
async def fetch_company_data(state: OnboardingState) -> dict:
    profile = await mcp.call_tool(
        MOODYS_SERVER_URL,
        "get_company_profile",
        {"entity_name": state["entity_name"]},
    )
    rating = await mcp.call_tool(
        MOODYS_SERVER_URL,
        "get_credit_rating",
        {"entity_id": profile["entity_id"]},
    )
```

A LangGraph node calls specific tools by hardcoded name in a fixed sequence. The LLM never sees the tool schemas. MCP is purely the transport layer between services.

---

## Tradeoff 1: Predictability vs Flexibility

In my architecture, the call sequence is defined in the graph and cannot deviate:

```python
# graph/graph.py — fixed at build time
builder.add_edge(START, "load_entity")
builder.add_edge("load_entity", "fetch_company_data")
builder.add_edge("fetch_company_data", "run_compliance_checks")
builder.add_edge("run_compliance_checks", "evaluate_risk")
builder.add_conditional_edges(
    "evaluate_risk",
    _route_after_risk,
    {"update_crm": "update_crm", "blocked": "blocked"},
)
```

`run_compliance_checks` always follows `fetch_company_data`. A BLOCKED case always skips CRM. This is testable, traceable, and reproducible — the same entity run twice produces the same sequence of tool calls.

The cost is rigidity. If a new entity type requires a different check sequence, I have to change the graph. In the LLM-to-tool pattern, the LLM could theoretically adapt its tool use based on what it finds at runtime.

In practice I view that flexibility as a risk, not a feature. LLMs do not reliably honor sequencing rules given in system prompts. I made the deliberate choice to move routing logic into the graph precisely because the LLM cannot be trusted to enforce it consistently under all conditions. I traded flexibility for guarantees, and I'd make the same call again.

---

## Tradeoff 2: Auditability vs Discoverability

My pattern produces a complete, deterministic audit trail. Every tool call maps to a specific node, with specific inputs from state, producing specific outputs written back to state. When something fails, I know exactly where.

What I gave up is discoverability. The `list_tools()` mechanism — which lets a client dynamically learn what tools a server exposes — is never called in my system. The nodes call tools by hardcoded name:

```python
# mcp_client/client.py
# No list_tools() call — tools are called by name directly.
async with ClientSession(read, write) as session:
    await session.initialize()
    result = await session.call_tool(tool_name, arguments)
```

I'm carrying the full weight of the MCP discovery mechanism — handshake, capability negotiation, schema generation — and using none of it. Discoverability matters when an LLM is the client and needs to learn what's available at runtime. When the caller is a graph node written by a developer, that mechanism adds overhead without value.

---

## Tradeoff 3: Integration Contracts vs Protocol Overhead

This is where my choice is most defensible.

Each MCP server in my system defines an explicit, schema-enforced contract for its tools. The Moody's server, for example, generates this automatically from Python type hints:

```json
{
  "name": "get_company_profile",
  "inputSchema": {
    "type": "object",
    "properties": {
      "entity_name": {"type": "string"}
    },
    "required": ["entity_name"]
  }
}
```

The mock servers in the current build will eventually be replaced with real production integrations — real Moody's API, real Refinitiv World-Check, real Salesforce. When that happens, the tool names, argument shapes, and response structures must stay identical. My orchestration layer changes nothing. MCP enforces that contract at the boundary.

The cost is protocol overhead I wouldn't otherwise pay. Every tool call requires an HTTP connection, a JSON-RPC handshake, and a `tools/call` envelope — for what is functionally a cross-container function call. In the current implementation I open a fresh connection per call:

```python
# A single onboarding case makes ~7 tool calls = ~7 MCP handshakes.
async with streamablehttp_client(server_url) as (read, write, _):
    async with ClientSession(read, write) as session:
        await session.initialize()
        result = await session.call_tool(tool_name, arguments)
```

Plain HTTP REST between containers would carry less overhead for the same job. MCP earns its keep here only because of what it enables at integration time — the stable, versioned contract that makes the mock-to-production swap possible without touching the orchestration layer.

---

## When This Pattern Makes Sense

**Use MCP as LLM-to-tool when:**
- The LLM needs to dynamically discover and sequence tools at runtime
- The tool set is not fully known at build time
- Flexibility in tool selection matters more than sequencing guarantees

**Use MCP as service-to-service (my approach) when:**
- You need stable contracts across independently replaceable integrations
- Routing and sequencing must be deterministic and auditable
- Services are deployed independently and you want a standardized protocol rather than bespoke REST APIs per integration

**Use neither — just REST or direct function calls when:**
- The integrations are internal and will never be swapped
- You control both sides of every call
- The protocol overhead isn't justified by any real integration boundary

---

## The Honest Summary

Using MCP as a service-to-service protocol is not the natural fit for the protocol. You pay for discovery you don't use and carry handshake overhead on every call. But the contract enforcement at integration boundaries is real, and for a system designed to swap mock integrations for production ones, that contract is exactly what you need.

The pattern works. Just go in clear-eyed about what you're getting and what you're giving up.

---

*Building production-grade agentic AI systems for regulated industries. If any of this connects with work you're doing, reach me on [LinkedIn](https://www.linkedin.com/in/srikanthsamudrla/).*
