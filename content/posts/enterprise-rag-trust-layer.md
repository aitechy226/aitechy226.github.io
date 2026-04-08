---
title: "The Trust Layer: What Separates Good RAG from Enterprise RAG"
date: 2026-04-08
description: "Four bugs found while hardening a RAG system for FSI and life sciences — and what they reveal about the gap between a working system and a trustworthy one."
tags: ["rag", "enterprise-ai", "retrieval", "grounding", "fsi", "langgraph", "production"]
author: "Srikanth Samudrla"
showToc: true
TocOpen: false
draft: false
---

I was stress-testing a RAG system built for regulated industries — financial services and life sciences. The grounding was fine. No hallucinations. What I found were subtler failures — the kind that only surface when analysts run the same query twice, compare citations across sessions, and need to explain to a regulator exactly which document an answer came from.

In regulated environments, that's the standard. And the system wasn't meeting it.

---

## Bug 1: The Embedding Variance Problem

My retrieval pipeline is hybrid: BM25 + TF-IDF + dense embeddings fused via Reciprocal Rank Fusion (RRF). The BM25 and TF-IDF lanes are deterministic by construction — exact string matching, frequency counts. The dense lane isn't.

I'm running Ollama locally with `nomic-embed-text`. I discovered that calling the embedding API twice with identical input text returns slightly different vectors — different enough that the floating-point ordering of candidates shifts.

The cascade is subtle but real: same query → slightly different dense vector → different dense lane rankings → different RRF fusion candidates → different cross-encoder candidates → different final passages. The analyst gets different citations on the same question without understanding why.

The fix required two cooperating caches. For corpus chunks, I added a disk-persisted cache keyed by a SHA-256 fingerprint of all chunk texts plus the model name — vectors load from disk on match, re-embed on mismatch. For query embeddings, I added an in-memory dict on the retriever instance — same query string returns the same vector within a session.

Then I found one more: even with identical vectors, the RRF fusion sort was non-deterministic for tied scores. A float comparison between two `0.016667` values has no guaranteed stable order. I added a secondary sort key — chunk ID — so ties always break the same way.

These changes are not just dev scaffolding. The cache enforces a contract my application controls — the embedding for a given text is a constant, immutable fact within this system, regardless of what the provider does underneath. Even cloud providers can shift behavior across model updates or backend changes. The consistency bug you can reproduce in dev is the one you can actually fix. The one that only appears in prod is the one that erodes analyst trust before you understand what's happening.

---

## Bug 2: The Gate That Always Said Yes

I had a function called `is_sufficient()` in my search agent. Its job was to evaluate whether retrieved evidence was good enough to warrant synthesis — a gate between retrieval and LLM call.

```python
def is_sufficient(self, evidence: Evidence) -> bool:
    return evidence.total_retrieved >= 1 and evidence.passages[0].score > self.min_score
```

The problem: `retrieve()` had already filtered out every passage below `min_score`. By the time `is_sufficient()` ran, every passage in the evidence object already had a score above `min_score` by definition. The check always returned `True` whenever any passage was retrieved at all.

A function that always returns `True` isn't a gate. It's a comment.

The fix: introduce a second threshold — `EVIDENCE_QUALITY_THRESHOLD` — set higher than the noise filter (`MIN_SCORE * 3` by default). The two thresholds have distinct jobs: one filters garbage out of the index, the other certifies that what survived is actually worth synthesizing. Conflating them was the root of the bug.

That fixed the signal. But the gate was in the wrong place.

---

## Bug 3: The Synthesis Layer Had No Gate at All

Even with `is_sufficient()` fixed, synthesis had no pre-check of its own. The agent could call `synthesize()` directly — bypassing the gate entirely — and synthesis would proceed on whatever evidence it received, no matter how weak.

This matters because a confident-sounding answer with a `confidence: low` flag buried in the metadata is as dangerous as a hallucination in a compliance context. A compliance officer under deadline pressure may not check the metadata before acting on the answer.

The right fix isn't a softer answer. It's no answer.

I added a hard gate at the top of `synthesize()`. Before any LLM call, check the top passage score:

```python
if top_score < EVIDENCE_QUALITY_THRESHOLD:
    return SynthesisResult(
        answer=f"Evidence quality too low to synthesize (top score {top_score:.4f} < "
               f"threshold {EVIDENCE_QUALITY_THRESHOLD:.4f}). "
               "Try a more specific query or check corpus coverage.",
        grounded=False,         # audit signal: this answer is not grounded in retrieved evidence
        confidence="insufficient",
    )
```

No tokens spent. No LLM call. The analyst gets a diagnostic message — exact scores, what to try next — rather than a misleading answer they might act on.

Bug 2 was a failure of validation logic — checking a value that had already been pre-filtered. Bug 3 was a failure of architectural enforcement — the synthesis layer had no gate of its own, so the validation fix in Bug 2 could be bypassed entirely. Same theme, two different layers of failure.

---

## Bug 4: The Score Scale Mismatch

My hybrid retriever uses RRF fusion scores in the range of roughly 0.001 to 0.05. When I enable the cross-encoder reranker (`ms-marco-MiniLM-L-6-v2`), scores become raw logits — unbounded values roughly between -10 and +10.

Thresholds calibrated for RRF scores (EVIDENCE_QUALITY_THRESHOLD = 0.003) are meaningless against cross-encoder logits. A logit of 0.003 is essentially zero on that scale.

I didn't normalize scores to a common scale — this is configuration as documentation, and that choice is worth explaining. A normalization layer would have to know which scoring mode is active at threshold-check time, and update correctly every time a new reranker is added. The retrieval backend in this system is configurable — normalizing across modes adds maintenance burden for a problem that only surfaces when you switch backends, and in production you don't. The right fix is to make the calibration requirement visible at configuration time, not hide it behind a normalization that could silently be wrong for the next backend you try.

So the `.env.example` now has this:

    EVIDENCE_QUALITY_THRESHOLD=0.003  # Min top-passage score for synthesis to proceed
                                       # BM25/TF-IDF: ~0.003; cross-encoder: 0.0 to 2.0

That comment is load-bearing. Whoever deploys this system sees the calibration requirement before they configure it, not after they've been confused by the behavior.

---

## Four Gaps in the Trust Layer

These four bugs have a pattern. None of them broke the system in a way that would show up in a demo. The system retrieved passages, called synthesis, returned answers with citations. From the outside, it looked correct.

What was broken was the contract with the analyst.

In a regulated environment, an analyst may need to explain to a regulator exactly which passage an answer came from, on a specific date. When the system returns different passages for the same query on different days, the analyst can't tell if the answer changed because the corpus changed or because the system is unreliable. They stop trusting it.

The retrieval engine is your contract with the analyst. The LLM is just the renderer.

Once the retrieval layer is deterministic and the evidence gate is doing real work, switching from local Ollama to Claude in production is literally a `.env` change. That design only pays off if what's underneath it is solid. These bugs were the work of making it solid.

