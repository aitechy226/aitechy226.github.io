---
title: "The Trust Layer: What Separates Good RAG from Enterprise RAG"
date: 2026-04-08
description: "Three bugs found while hardening a RAG system for FSI and life sciences — and the architectural principles behind fixing them."
tags: ["rag", "enterprise-ai", "retrieval", "grounding", "fsi", "langgraph", "production"]
author: "Srikanth Samudrla"
showToc: true
TocOpen: false
draft: true
---

I've been building a RAG system for regulated industries — financial services and life sciences specifically. Not a prototype. Something a compliance officer or clinical analyst would actually rely on.

At some point I had a working system. It retrieved passages, called an LLM, returned grounded answers with citations. It passed demos. And then I started stress-testing it the way a real deployment would stress-test it, and I found things I'm not proud of.

This post is about those things. Four bugs, the design gaps that caused them, and the three properties that any RAG system in a regulated environment must satisfy — or it will eventually lose the analyst's trust, even if it never technically fails.

*This is Part 1. Part 2 will cover the autonomous quality loop I built on top — scenario-based retrieval benchmarks that run on a schedule and write back config changes when they find regressions.*

---

## Three Properties That Actually Matter

Most RAG discussions focus on one thing: grounding. Does the answer come from the corpus or from the LLM's training knowledge? That matters. But it's the easiest of the three properties to reason about, and the one most builders focus on exclusively.

The two properties that get ignored are **consistency** and **performance**.

**Consistency** means: same query, same corpus, same source passages, every run. Not approximately the same. The same.

**Performance** means: fast enough that the analyst doesn't context-switch to a different tool while waiting.

Here's why consistency matters more than most people realize. In a regulated environment, an analyst doesn't just read the answer — they review the source passages. They verify the citations. They may need to explain to a regulator exactly which section of which policy document the answer came from. If they run the same query tomorrow and get different passages — even if both answers are technically correct — they will stop trusting the system. The AI becomes a liability rather than a tool. "The AI is inconsistent" is a career risk for anyone who signs off on work done with it.

My system had a consistency problem. It took me longer than I'd like to admit to fully understand why.

---

## Bug 1: The Embedding Variance Problem

My retrieval pipeline is hybrid: BM25 + TF-IDF + dense embeddings fused via Reciprocal Rank Fusion (RRF). The BM25 and TF-IDF lanes are deterministic by construction — exact string matching, frequency counts. The dense lane isn't.

I'm running Ollama locally with `nomic-embed-text`. I discovered that calling the embedding API twice with identical input text returns slightly different vectors. Not wildly different — the cosine similarity is very high — but different enough that the floating-point ordering of candidates shifts. When you're taking the top-5 from a ranked list, small ordering differences become visible differences in which passages get returned.

The cascade is subtle but real: same query → slightly different dense vector → different dense lane rankings → different RRF fusion candidates → different cross-encoder candidates → different final passages. The analyst gets different citations on the same question without understanding why.

The fix was two caches working together.

For corpus chunks, I added a disk-persisted cache keyed by a SHA-256 fingerprint of all chunk texts plus the model name. On index build, if the cache file exists and the fingerprint matches the current corpus, the saved vectors are loaded directly — no API call. If the corpus changes or the model changes, the fingerprint doesn't match and a fresh embed is triggered. This means a corpus is embedded exactly once, and every restart uses the same vectors.

For query embeddings, I added an in-memory dict on the retriever instance. Same query string within a session returns the same vector from memory. No re-embed, no variance.

Then I found one more: even with identical vectors, the RRF fusion sort was non-deterministic for tied scores. A float comparison between two `0.016667` values has no guaranteed stable order. I added a secondary sort key — chunk ID — so ties always break the same way.

The important thing to understand about these changes: they're not just dev scaffolding. When I flip to OpenAI or Anthropic embeddings in production, those models are already deterministic. The caches make Ollama's behavior match production semantics during development. The bug you find in dev with consistent retrieval is a bug that won't hide until prod.

---

## Bug 2: The Tautological Gate

I had a function called `is_sufficient()` in my search agent. Its job was to evaluate whether retrieved evidence was good enough to warrant synthesis — a gate between retrieval and LLM call.

The implementation:

```python
def is_sufficient(self, evidence: Evidence) -> bool:
    return evidence.total_retrieved >= 1 and evidence.passages[0].score > self.min_score
```

The problem: `retrieve()` had already filtered out every passage below `min_score`. By the time `is_sufficient()` ran, every passage in the evidence object had a score above `min_score` by definition. The check always returned `True` whenever any passage was retrieved. The system reported "evidence sufficient" even when the top passage was marginal.

A function that always returns `True` isn't a gate. It's a comment.

The fix was to introduce a second threshold: `EVIDENCE_QUALITY_THRESHOLD`. It defaults to `MIN_SCORE * 3` (so 0.003 when MIN_SCORE is 0.001), is configurable in `.env`, and is explicitly higher than the noise filter. Now `is_sufficient()` checks against the quality threshold, not the noise floor. The two thresholds have different jobs — one filters garbage, one certifies quality — and confusing them was the root of the bug.

---

## Bug 3: The Authoritative Low-Confidence Answer

Even after fixing the gate, there was a deeper problem. The system could retrieve passages that cleared the noise filter (MIN_SCORE) but weren't meaningfully on-topic. Synthesis would proceed, the LLM would dutifully generate an answer from whatever it had, and the result would be a low-confidence answer that sounded authoritative.

In a compliance context, a confident-sounding answer with a `confidence: low` flag buried in the metadata is as dangerous as a hallucination. A compliance officer under deadline pressure may not check the confidence field before acting on the answer.

The right fix isn't a softer answer. It's no answer.

I added a hard gate at the top of `synthesize()`. Before any LLM call, check the top passage score against `EVIDENCE_QUALITY_THRESHOLD`. If it doesn't clear, return a structured decline immediately:

```python
if top_score < EVIDENCE_QUALITY_THRESHOLD:
    return SynthesisResult(
        answer=f"Evidence quality too low to synthesize (top score {top_score:.4f} < threshold {EVIDENCE_QUALITY_THRESHOLD:.4f}). "
               "Try a more specific query or check corpus coverage.",
        grounded=False,
        confidence="insufficient",
        ...
    )
```

No tokens spent. No LLM call. No misleading answer. The analyst gets a diagnostic message that tells them exactly what happened and what to try next.

"I found something" is not the same as "I found something relevant." The gate has to be at synthesis entry, not just at retrieval.

---

## Bug 4: The Score Scale Mismatch

This one is architectural. My hybrid retriever uses RRF fusion scores in the range of roughly 0.001 to 0.05. When I enable the cross-encoder reranker (`ms-marco-MiniLM-L-6-v2`), scores become raw logits — unbounded values roughly between -10 and +10.

Thresholds calibrated for RRF scores (EVIDENCE_QUALITY_THRESHOLD = 0.003) are meaningless against cross-encoder logits. A logit of 0.003 is essentially zero on that scale.

I didn't change a threshold to fix this. I changed how the threshold is documented and configured. The `.env.example` now has explicit calibration guidance:

    MIN_SCORE=0.001                    # Hard minimum retrieval score (filters noise)
    EVIDENCE_QUALITY_THRESHOLD=0.003  # Min top-passage score for synthesis to proceed
                                       # BM25/TF-IDF: ~0.003; cross-encoder: 0.0 to 2.0

The mismatch would otherwise stay invisible until I flipped to the reranker in a demo and got puzzling behavior — either no answers (threshold too high for logits) or answers on garbage (threshold too low).

The lesson isn't to hardcode the right value. It's that any threshold in your system that will be interpreted across multiple score scales needs to have its calibration context documented explicitly, not inferred from the variable name.

---

## The Architectural Insight

These four bugs have a common theme. In each case, the system was technically working — it retrieved passages, it called synthesis, it returned answers — but the internal invariants it depended on were subtly violated. The violations weren't visible in demos because demos are short, queries are cherry-picked, and nobody runs the same query twice while watching the citations.

Production is different. Analysts run the same queries repeatedly. They compare answers across sessions. They spot inconsistencies. And when they do, they stop trusting the system, and that trust doesn't come back.

The retrieval engine is your contract with the analyst. The LLM is just the renderer.

Once the retrieval layer is deterministic and the trust layer enforces grounding, switching from local Ollama to Claude in production is literally a `.env` change. The entire architecture was designed so the LLM backend is pluggable — and that design decision only pays off if retrieval is rock-solid underneath it.

---

## What "Enterprise RAG" Actually Means

Good RAG: retrieves something, synthesizes an answer, passes a demo.

Enterprise RAG: same query → same passages → grounded answer → audit trail the regulator can inspect.

The gap between the two isn't model quality. It's the trust layer: deterministic retrieval, evidence gating, citation traceability, governance scoring. None of those things show up in benchmark leaderboards. All of them determine whether a compliance officer in a regulated industry will stake their name on the output.

That's what I've been building toward. Part 2 will cover how I built a continuous quality loop on top of it — scenario-based benchmarks that run on a schedule and find retrieval regressions before analysts do.
