---
title: "Why Your AI Agent Demo Looks Great and Your Production System Doesn't"
date: 2026-03-31
description: "A practitioner's take on the gap between agentic hype and agentic reality."
tags: ["agents", "agentic-ai", "production", "architecture", "llm", "langgraph"]
author: "Srikanth Samudrla"
showToc: true
TocOpen: false
draft: false
---

I've spent the last several months building agentic AI systems — not demoing them, building them. And I want to share something that took me longer than I'd like to admit to fully internalize.

The hype is real. The gap is also real. And the gap is closing — but not in the way most people think.

*This reflects where I am in March 2026, building on roughly 18 months of hands-on agentic work. The field is moving fast and I expect some of this to age.*

---

## The Demo Always Works. Here's Why.

Every impressive agent demo you've seen — the one where the AI autonomously calls five tools, synthesizes the results, and delivers a perfect output — was run until it worked, then recorded. The failures happened before the camera started. That's not dishonest. That's just how demos work. But it creates a perception of reliability that doesn't survive contact with production.

In my own work, I built a multi-agent system using a ReAct-style architecture. The agent would reason about which tool to call next, execute it, observe the result, and decide the next step. In demos it looked extraordinary. In testing — running llama3.1:8b locally — it didn't reliably honor system prompt directives for retry logic. It would skip steps, call tools out of order, hallucinate arguments. The same workflow that worked perfectly at 2pm would fail at 10am for reasons I couldn't reproduce.

To be precise about this: with smaller local models, this failure is consistent and reproducible. With frontier models like Claude Sonnet the reliability picture is meaningfully better — tool calling is one of the areas where the capability gap between an 8B local model and a frontier API model is largest, not smallest. If I had been running Sonnet, the workflow would very likely have executed correctly.

But here's the distinction that matters: **reliability and auditability are different problems. Only one of them improves with a better model.** More on that below.

That system got rebuilt regardless. The routing logic moved out of the LLM and into a deterministic orchestration layer. The LLM kept the one job it's actually reliable at: generating human-readable narrative from structured inputs. The system became predictable, testable, and deployable.

---

## The Field Has Already Validated This Pattern — Quietly

Here's something worth noting: I'm not describing a novel insight. The fact that frameworks like LangGraph, Temporal, and Prefect have become the default infrastructure for serious agentic deployments is the field converging on exactly this architecture. Deterministic orchestration with LLM-powered steps is increasingly how production systems get built — not because engineers are being conservative, but because the teams that tried the alternative learned the same lessons and moved on.

The tooling ecosystem didn't build these frameworks for fun. They built them because LLM-controlled flow at production scale has a known failure profile and practitioners needed infrastructure to route around it.

---

## The Four Buckets Most "Killing It" Claims Fall Into

When I hear that someone is "crushing it" with AI agents, I've started mentally categorizing the claim:

**1. The task is forgiving.** Blog posts, email drafts, document summaries. If the agent skips a step the output is still usable. The agent looks impressive because low-stakes failures are invisible.

**2. The demo is cherry-picked.** See above.

**3. The definition of "agentic" is genuinely contested.** The line between "function with an LLM inside" and "agent" is blurry and the field hasn't settled on a definition — and that's partly a legitimate conceptual ambiguity, not just a rhetorical sleight of hand. But the looseness of the term does create expectations that a lot of deployed systems can't meet. Worth being honest about that gap even when the ambiguity is real.

**4. The guardrails are invisible.** The production systems that are genuinely generating ROI — coding assistants, triage workflows, document extraction pipelines — almost all have a human in the loop, a constrained action space, or deterministic orchestration controlling the sequence. The "agent" is operating inside a box. The box is doing the heavy lifting.

---

## The Mental Model I've Found Most Useful

LLMs are reliable at *generation*. They are unreliable at *decision-making under constraint*.

Ask an LLM to synthesize a narrative from structured evidence — it does this beautifully. Ask it to decide whether to escalate a KYC flag, route a transaction for review, or determine which of five tools to call next — and you've moved into territory where non-determinism is a liability, not a feature.

I want to be precise here because this is where the nuance matters: the *reliability* gap is narrowing. Frontier models today are meaningfully better at tool-calling and sequencing than models from 18 months ago, and that improvement is real and continuing. The argument for deterministic orchestration is not that LLMs will never be reliable enough — it's that reliability alone doesn't solve the problem in regulated environments.

---

## The Argument That Survives Model Improvements

Even if models become perfectly reliable at tool-calling, the auditability argument doesn't go away.

A deterministic orchestration graph is inspectable. You can write a unit test that asserts step 3 always follows step 2. You can show a regulator exactly what sequence ran on a specific transaction and why. You cannot do that with an LLM decision, regardless of how reliable the model gets.

In regulated workflows — KYC, lending decisions, compliance screening — auditability is not an engineering preference. It's a legal requirement. If an agent skips a sanctions check because the LLM decided the entity name was ambiguous, that decision is buried in a reasoning trace nobody reviewed, and you cannot guarantee it won't happen again. That failure mode is unacceptable at any model tier.

So there are actually two separate arguments here worth keeping distinct:

- **Reliability argument** — model-dependent, shrinking, probably less severe in two years than today. Using a frontier model materially reduces this risk today.
- **Auditability argument** — compliance-driven, durable, does not improve as models improve. This one doesn't care what model you're running.

The first argument is a reason to be cautious about model choice. The second is a reason to build this way permanently, at least in regulated domains.

---

## Where the Hype Is Genuinely Justified

I don't want this to read as pure skepticism — that would be equally dishonest.

Agents are genuinely transformative for tasks that are high volume, have well-defined success criteria, and are either reversible or human-reviewable before action. Code generation, first-pass document triage, data extraction, customer-facing Q&A with retrieval — these are real productivity wins, and the productivity numbers cited for these categories are plausible. I've seen them.

The gap isn't that agents don't work. The gap is between what works in a demo and what works reliably in a regulated, auditable, production workflow where a skipped step has consequences.

---

## A Distinction Worth Making: Bounded vs. Unbounded Agents

A useful distinction the field hasn't settled on yet: bounded vs. unbounded agentic systems.

An unbounded agent has an open-ended action space — it decides what to do next at every step, from an unrestricted set of options. That's what ReAct loops implement. It's also what makes them hard to test, audit, and trust in production.

A bounded agentic system takes a high-level goal and autonomously executes a constrained, deterministic workflow to achieve it — coordinating tools and systems the user never touches directly. The sequence is fixed. The decisions within the sequence are rule-based. The LLM synthesizes the output.

The second definition still qualifies as agency. It just qualifies as the kind that ships.

---

## Where I've Landed — For Now

The architecture I keep returning to: deterministic orchestration layer owns the flow, LLM owns the prose. Graph controls decisions, model generates interpretation. That's not less agentic — it's production-grade agentic. And in my experience, it's a far better conversation to have with a risk or compliance team than "trust the model."

I'll also say directly: this is a point-in-time view. The field is moving fast, the models are improving, and I'm genuinely open to being wrong about where the equilibrium lands. But the auditability argument feels durable to me regardless of where model capability goes.

This has been my experience. I'm curious whether it matches yours — or whether you're seeing something in production that pushes back on this. What are you building, and what's actually working?

---

*Building AI infrastructure tooling and agentic systems. Always interested in the gap between what's demoed and what ships.*
