---
title: "Cost-Per-Token: The Physics Behind the Number"
date: 2026-11-01
description: "I built a cost-per-token (CPT) engine for an inference GPU recommender and rewrote it twice — first because the formula ignored the KV cache (the conversation history the GPU re-reads for every token), then because the UI described the same number three different ways. Here is how that changed the way I read a CPT number."
tags: ["gpu", "inference", "llm", "cost-per-token", "physics", "mlops"]
author: "Srikanth Samudrla"
showToc: true
TocOpen: false
draft: false
---

I was building an inference GPU recommender — a tool that takes a workload description and ranks GPUs by cost per million output tokens (CPT). I thought I understood the formula, and then I had to rewrite it twice. The first rewrite was a physics gap: the formula ignored the KV cache, the stored attention keys and values the GPU re-reads for every token it generates. The second was a label problem. What I ended up with: a CPT number means very little until you know the context length, batch size and attention type behind it.

---

## 1. The first formula I wrote had no KV cache in it

Here's the formula I started with for single-stream decode throughput:

```
tokens/sec = (memory_bandwidth × efficiency) / (paramsB × bytes_per_param)
```

The denominator is how much memory the GPU has to read from VRAM per output token. For a 70B dense model in FP8 that's about 70 GB. (In BF16 it's about 140 GB, which doesn't fit on a single H100 and leaves essentially no room on a single H200 — so for single-GPU 70B serving, FP8 is the realistic baseline, and it's the one I'll use throughout.) On an H200 with 4.8 TB/s of HBM3e and a 0.80 efficiency factor, you get `(4800 × 0.80) / 70 ≈ 55 tokens/sec`.

This formula is a decent approximation at short context (2K–4K), where the weights dominate the per-token read. It gets progressively more wrong as context grows. How wrong depends heavily on the attention architecture: for a modern GQA 70B model at 32K context and batch=1, it overstates throughput by roughly 15%. For an MHA model, a longer context, or batched serving, the error is far larger.

The missing term is the KV cache. On every output token, the GPU doesn't just stream the model's weights — it also streams the key-value cache for every token already in the conversation. For Llama-3-70B (80 layers, 8 KV heads, head dimension 128, BF16 KV), that's:

```
2 (K and V) × 80 layers × 8 KV heads × 128 dims × 2 bytes ≈ 0.33 MB per token
```

At 32K context, that's about 10.7 GB. Add it to 70 GB of weights and the per-token read becomes ~81 GB, not 70 — and tokens/sec drops from ~55 to ~48.

The corrected formula:

```
tokens/sec = (memory_bandwidth × efficiency) / (weight_GB + kv_cache_GB)
```

Where `kv_cache_GB` follows directly from the model's config — layers, KV heads, head dimension, KV precision — multiplied by context length. It doesn't depend on parameter count directly; it depends on the attention architecture. GQA shares each KV head across a group of query heads; MHA gives every query head its own, so an MHA model with 64 heads would carry 8× the KV of the same model with 8 KV heads.

This was the first rewrite. The check that would have caught the old formula: sweep context from 2K to 32K and assert tokens/sec strictly decreases, since longer context never speeds up single-stream decode. The old formula fails it, because it ignored context entirely and returned the same number at every length. The new one passes by construction.

A cost-per-token formula that ignores context length is wrong exactly where context is longest — summarization, RAG, code assistants with long files. And as section 3 shows, the error compounds once requests are batched, because every in-flight request brings its own KV cache.

---

## 2. The second thing I had to fix was the label

After the formula was right, I reviewed the UI and found the tool was saying different things in different places about what the number meant. One tooltip said "single stream throughput." Another section said "per-user rate at idle." A third said "single-query cost." These aren't synonyms. They're three different framings of the same underlying measurement, and each one pushes a reader toward a different mental model.

The fix was a single canonical phrase: **"one request at a time."** Not "single stream," not "single user," not "single query."

"Request" is the right abstraction from the GPU's perspective. The GPU doesn't see users or queries; it sees requests in a scheduling queue. One request in flight means batch=1: the GPU reads all the model weights and that request's KV cache from VRAM, computes the next token, and moves on. Yours plus 63 other requests in flight at once is batch=64.

I settled on that phrase because it holds up across the physics, the scheduling and the user-facing wording. Every cost-per-token number the tool surfaces carries it as an inline qualifier, and a banner above the recommendation explains that the number is a conservative cost ceiling relative to batched serving — on paper, batched serving is several times cheaper at long context and more than an order of magnitude cheaper at short context (section 3).

---

## 3. Batched serving is where the number diverges — by how much depends on context

The reason the batch=1 disclaimer matters is that batched serving changes the per-token arithmetic fundamentally. It's not a small adjustment. It's a regime change.

Here's the physics. At batch=1, for each output token the GPU reads:

- All of the model weights (streamed once)
- The KV cache for the one request in flight

At batch=64 (an illustrative batch size for a continuous-batching server like vLLM), for each decode step the GPU reads:

- All of the model weights **once, for the whole batch** — the same read is used to compute the next token for every request
- The KV cache for each of the 64 in-flight requests

The weight bytes amortize across the batch. The KV bytes scale with batch size. That second fact is what makes the batching multiple so sensitive to context length. Working through the same FP8 70B model on an H200:

- **At 2K context**, each request's KV cache is ~0.67 GB. A batch of 64 adds ~43 GB of KV to 70 GB of weights — it fits, and each decode step reads ~113 GB to produce 64 tokens. That's ~2,200 tokens/sec aggregate versus ~54 at batch=1 (KV included): roughly 40× on paper; I'd expect real runs to land lower once scheduling overhead and attention kernels take their cut. Per-request speed drops to ~34 tokens/sec, which is still comfortably faster than anyone reads.
- **At 32K context**, each request's KV cache is ~10.7 GB. After the weights and roughly 6 GB of runtime overhead (my assumption), there's room for only about six requests. Each step reads ~134 GB to produce 6 tokens — ~170 tokens/sec aggregate versus ~48 at batch=1. About 3.5×. Treat request counts like "six" as roughly ±1: marketed VRAM is quoted in binary units, which adds capacity, while real FP8 checkpoints weigh closer to 72 GB and serving stacks reserve headroom, which takes it away.

The aggregate gain can never exceed the batch size: a step that produces 64 tokens takes at least as long as a step that produces one.

Cost-per-token is `$/hr ÷ tokens/sec`. The $/hr is fixed. So CPT drops by whatever factor tokens/sec rises — anywhere from low single digits at long context to tens of times at short context, depending on model size, context length, and how aggressively the serving stack batches.

Here is that on an H200 at an illustrative $4/hr (an assumption, not a quote). At 2K context, batch=1 is ~54 tokens/sec, about $20.60 per million output tokens; batch=64 is ~2,200 tokens/sec, about $0.50. At 32K, batch=1 is ~48 tokens/sec, about $23; six requests at once is ~170 tokens/sec, about $6.50.

So when my tool quotes a CPT number, it's quoting the batch=1 number, for a buyer who hasn't built a serving stack yet. The banner says so. A batched number for the same model on the same GPU comes out several times to an order of magnitude lower. To compare two quotes, I'd need to pin which regime each one is in.

---

## 4. Context length changes what fits, not the bandwidth ranking

A side effect of the corrected formula for the recommender: GPUs don't rank the same way for decode as they do for training, where raw FLOPS matter and H100 and H200 share the same compute die.

Decode at batch=1 is bandwidth-bound, and FLOPS barely matter. What matters is memory bandwidth per dollar *and* enough VRAM to hold the model plus its KV cache. For a 70B FP8 model at long context:

| GPU | VRAM | Memory bandwidth | Notes |
|-----|------|------------------|-------|
| H100 | 80 GB HBM3 | 3.35 TB/s | Fits 70B FP8 weights with only ~4 GB left for KV after overhead; one 32K request doesn't fit |
| H200 | 141 GB HBM3e | 4.8 TB/s | Fits weights + 32K KV with room to batch; ~43% bandwidth edge over H100 |
| B200 | 192 GB HBM3e (about 180 GB usable on HGX B200) | 8 TB/s | Most bandwidth and the most KV headroom |

One thing the KV term does *not* change is the bandwidth ranking. Two GPUs serving the same model at the same context read the same bytes per token, so their throughput ratio is just their bandwidth ratio — H200 is ~1.43× faster than H100 wherever both can hold the workload, 2K for example. Whether H200 wins on cost-per-token comes down to whether its price premium over H100 is less than that 1.43×, and at batch=1 that doesn't depend on context length.

What context length *does* change is whether the workload fits — and that's where the rankings move:

- **At batch=1**, H100 runs out of room first. A 32K-context request doesn't fit alongside 70 GB of FP8 weights on one H100 under these assumptions, and fits only at the margin even with FP8 KV. The buyer has to cap context or split the model across two GPUs with tensor parallelism. That roughly doubles $/hr, but at batch=1 it also roughly doubles the memory bandwidth available, so tokens/sec nearly doubles and CPT stays about flat, plus communication overhead. What the buyer really pays for is almost half the combined VRAM going unused at batch=1.
- **In batched serving**, the gap widens sharply, and free VRAM drives it at every context length. An H100 has about 4 GB left, roughly six 2K requests against about 97 on an H200 (a count this close to the limit swings a lot with the overhead assumption, but the gap stays large). At 32K, an H100 can't hold even one request, while an H200 holds about six and a B200 about nine to ten (same ~6 GB overhead assumption). Since batching is where CPT falls, the GPU with more KV headroom gets cheaper per token much faster.

This is the part of the recommender that most changed with the physics fix. Before the KV term was in the formula, the tool had no idea that a 32K workload wouldn't fit on a single H100. After the fix, the ranking explains itself: at the context length you're running, here's the per-token memory read, here's whether it fits, here's the tokens/sec, and here's the cost-per-token.

---

## 5. What I haven't modeled yet

Gaps in my cost-per-token engine:

**Prefill is a separate physics regime.** Everything above models decode — generating output tokens one at a time, bandwidth-bound. Prefill is the phase before decode, where the GPU processes all the prompt tokens in parallel to build the initial KV cache. Prefill is compute-bound, not bandwidth-bound. It uses a different cost model (FLOPS per token), and it can rival or exceed decode cost for long-prompt / short-output workloads — summarization of a long document, classification over a long context, RAG where the context-stuffed prompt is much longer than the answer. My engine models decode only, so prefill is not in its CPT. A full prefill model is on my backlog.

**KV cache coverage is narrow.** The KV-per-token footprint depends on attention architecture and the specific model's layer count, head dimension, and KV-head grouping. I have configs on file for Llama-3, Mistral, and Llama-2-70B (all GQA) and the smaller Llama-2 models (MHA), derived from their published architectures. Any other family needs its config added before the numbers mean anything. Two families need more than a config:

- **Mixture-of-experts models** (Mixtral, DeepSeek, and others) break the weight term itself. At batch=1, only the active experts' weights are read per token, so `paramsB × bytes_per_param` overstates the read badly — Mixtral 8x7B has ~47B total parameters but only ~13B active per token. Batching muddies this further, since different requests route to different experts.
- **DeepSeek V2/V3 use Multi-head Latent Attention (MLA)**, which compresses the KV cache far below what GQA achieves. It isn't MHA or GQA, and neither formula fits.

Closed models like Claude or Gemini are out of scope entirely — you can't self-host them, so there's no GPU to recommend.

**Efficiency is engineering judgment, not physics.** My engine uses 0.80 as the bandwidth efficiency factor — the fraction of theoretical peak HBM bandwidth a GPU actually delivers during decode. That number depends on the decode attention kernel (flash-decoding, for example), whether KV is paged or contiguous, whether speculative decoding is on, and the specific serving stack. From what I've read, measured utilization in real deployments often lands lower, in the 50–75% range, so 0.80 sits at the optimistic end. I chose it; it's not a natural constant. What that implies: the formula is a roofline, so the tokens/sec it produces is an *upper* bound on batch=1 throughput. The cost number is conservative relative to batched serving, but the throughput number is optimistic relative to a real batch=1 deployment. A different picked value produces a different CPT.

**Nothing here is validated against a measured run.** I haven't checked the engine's tokens/sec against a measured vLLM or TensorRT-LLM deployment, so every number in this post is roofline math.

**Batching has a ceiling, and for a 70B model it's VRAM.** In the roofline, aggregate tokens/sec climbs with batch size but levels off at bandwidth × efficiency ÷ per-request KV bytes — about 357 tokens/sec on an H200 at 32K, about 5,700 at 2K. A 70B model on these GPUs runs out of memory before reaching either: six requests at 32K gives ~170 tokens/sec, and roughly 97 requests at 2K gives ~2,750. Past that point the serving stack has to stop admitting requests or preempt them (vLLM, for example, evicts and later recomputes them), and throughput stops scaling. The KV-dominated flattening shows up first with smaller models or more VRAM. My engine models batch=1, not this ceiling, because it assumes less about the deployment.

I chose to build the tool around the batch=1 number, with the disclosure as part of the design.

---

## 6. How I read a CPT number now

After this build, three questions I ask of any CPT quote, including my own.

**At what batch size was the number measured?** At batch=1, the number is the conservative end relative to batched serving (section 3). In my roofline it falls by a few times at long context and an order of magnitude or more at short context once the batch is sustained; I haven't measured how much of that survives in a real deployment. A low-QPS workload with bursty traffic may never hit the batch.

**At what context length, on what model family?** CPT at 2K context and CPT at 32K context on the same model on the same GPU aren't the same number. GQA, MHA, and MLA at the same context aren't the same number, and MoE models don't follow the dense-model formula at all. At batch=1 on a GQA model, the context-length swing is modest — around 15% from 2K to 32K on a 70B. On an MHA model, or once requests are batched, it can be several times that. When a quote doesn't pin these, I can't tell how much of the gap between two numbers is physics and how much is pricing.

**Is prefill in the number or not?** For long-prompt / short-output workloads, prefill can dominate total cost. "Cost per 1M output tokens" doesn't settle this on its own: the figure could be decode-only cost divided by output tokens, or total request cost divided by output tokens. So I look for whether the quote says which. The per-request cost that includes both phases is the one I can actually compare.

If I can answer all three for a given number, I can compare it to another number. If I can't, I don't treat the two numbers as comparable — they may be quoting different regimes with the same word.

---

I went into this build thinking cost-per-token was a well-defined quantity. I came out of it thinking it's a family of quantities that share a name, and the value of the number depends on which member of the family is being quoted. The physics behind the name is tractable — bandwidth, weights, KV cache, memory capacity, batch size, context length, attention family. Before I trust a CPT number now, I pin which regime it's measuring, and I want my tool to make that pin visible on every number it shows.
