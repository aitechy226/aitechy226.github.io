---
title: "GPU Infrastructure: The Five Calculations That Actually Matter"
date: 2026-04-10
description: "VRAM fit, quantization impact, multi-node thresholds, egress cost, and real TCO — the five calculations that determine whether a GPU deployment actually works."
tags: ["gpu", "infrastructure", "llm", "mlops", "distributed-training", "tco", "quantization"]
author: "Srikanth Samudrla"
showToc: true
TocOpen: false
draft: false
---

I was building a GPU recommendation engine — one that maps workload descriptions to specific configurations, primary recommendations, and cost ranges — and kept hitting the same wall: getting the recommendations right meant going deep on every constraint that determines whether a deployment actually works. Not whether it's affordable. Whether it works at all.

VRAM has to fit the full training state, not just the model weights. Training data has to be where the GPUs are. The interconnect has to support the parallelism strategy. None of that shows up in a $/hr comparison. Here are the five calculations that come before it.

---

## 1. VRAM Fit: The Hard Constraint Before Everything Else

The first thing I had to establish was whether the model fits on the GPU at all. If VRAM is exhausted, training crashes — not degrades, crashes. The full memory envelope is what matters: model weights plus optimizer state plus activations plus the KV cache for inference.

**Model weights** are the baseline. The per-parameter byte cost depends entirely on your training mode:

| Mode | Bytes per parameter |
|------|---------------------|
| Full fine-tuning (mixed precision: BF16 compute + FP32 optimizer states, Adam) | 18 bytes |
| LoRA (BF16 base frozen + adapter trained in BF16 + Adam optimizer) | 2 bytes base + ~6–12 GB adapter overhead |
| QLoRA (NF4 quantized base: 0.5 bytes/param + LoRA adapter + paged Adam overhead) | ~0.5 bytes base + ~5–15 GB adapter overhead |
| BF16 inference (weights only, no optimizer) | 2 bytes |

A 70B parameter model under full fine-tuning requires roughly 1.26 TB of VRAM for the full training state. In practice, this is distributed across multiple GPUs using ZeRO-3 (DeepSpeed) or FSDP, which shard optimizer states and gradients across devices — a single GPU no longer holds the full 1.26 TB, but the aggregate VRAM requirement across the cluster is unchanged. The number still determines how many nodes you need; it's just not the per-GPU ceiling when you use modern sharding.

Under QLoRA, the NF4-quantized base model weighs about 35 GB, with LoRA adapter parameters and paged optimizer adding roughly 5–15 GB. Real-world peak VRAM during training typically lands between 45–65 GB depending on LoRA rank and batch size — comfortably within a single A100 80GB node in most configurations. That's the difference between a multi-node cluster and a single node.

**KV cache for inference** adds memory that scales with sequence length, not model size:

```
KV cache = 2 × num_layers × num_kv_heads × head_dim × seq_len × 2 bytes
```

Modern large models use Grouped Query Attention (GQA), where `num_kv_heads` is much smaller than the total attention head count — LLaMA-70B uses 8 KV heads vs. 64 query heads, reducing KV cache by 8×. For a 70B model at 8K context with 80 layers, 8 KV heads, and 128 head_dim, that's roughly 2.5 GB per sequence in flight. At a batch of 10 concurrent requests, that's 25 GB of KV cache before a single weight is loaded.

**Activations** assume gradient checkpointing. Without it, activation memory can exceed weight memory on long sequences.

The VRAM calculation isn't hard math. It's knowing which formula applies to which mode. Skip it and the $/hr number becomes irrelevant at job launch time.

---

## 2. Quantization Changes the Hardware Requirement Entirely

Quantization isn't a fine-tuning detail. It's a hardware selection variable. The same model at different precision levels requires completely different GPU configurations.

| Precision | Bytes per parameter | 70B model footprint |
|-----------|--------------------|--------------------|
| FP32 | 4 | 280 GB |
| BF16 | 2 | 140 GB |
| INT8 | 1 | 70 GB |
| INT4 | 0.5 | 35 GB |

Going from FP32 to INT4 on a 70B model reduces the weight footprint by 8×. That's the difference between a four-card A100 configuration (4 × 80 GB = 320 GB total VRAM) and a single H100 (80 GB) — for the exact same model.

The catch is precision loss. INT4 (via GPTQ or GGUF) can introduce measurable accuracy loss on tasks that require numerical precision — financial calculations, structured output with tight constraints, legal reasoning where exact phrasing matters. From what I found in my research, INT8 generally holds up well for inference with minor accuracy degradation. For training, BF16 is the standard — it has the same exponent range as FP32 without the memory cost.

Quantization and hardware selection turned out to be the same decision.

---

## 3. Multi-Node Is Not Just More GPUs

Once the model doesn't fit on a single node, the architecture changes — and not just because there are more GPUs.

The threshold is simple: if required VRAM exceeds single-node capacity, you need multi-node. The ceiling division tells you how many nodes:

```
nodes_required = ceil(total_vram_required / vram_per_node)
```

But that calculation only tells you how many nodes — not whether they can communicate fast enough to make distributed training work.

**Inside a single node**, GPUs communicate over NVLink — NVIDIA's proprietary interconnect running at 900 GB/s on H100 NVLink 4.0 (600 GB/s on A100 NVLink 3.0). Tensor parallelism (splitting a single layer across multiple GPUs) is viable at this bandwidth. You can shard individual weight matrices horizontally across GPUs and the synchronization cost is low enough to be worth it.

**Across nodes**, the story changes. You're on InfiniBand or Ethernet. InfiniBand HDR gives you about 200 Gb/s (25 GB/s) per port — more than an order of magnitude slower than NVLink. At this bandwidth, tensor parallelism across nodes is usually a net loss. The synchronization overhead exceeds the compute benefit. You switch to pipeline parallelism instead — each node holds complete model layers, and the forward pass flows through nodes sequentially.

If your model requires cross-node tensor parallelism, InfiniBand isn't optional — it's a correctness requirement, not a performance preference.

The mechanism that synchronizes multi-GPU training is **NCCL** — NVIDIA Collective Communications Library. NCCL handles AllReduce operations: after each backward pass, every GPU has computed its local gradients, and NCCL aggregates them across all GPUs so each one updates with the full gradient. The abstraction is clean. The failure modes aren't.

NCCL misconfiguration doesn't crash training. It silently degrades throughput — sometimes to 20% of expected performance — because the collective operations serialize where they should be parallel. The symptom looks like slow hardware. From what I found in my research, the usual cause is a wrong `NCCL_SOCKET_IFNAME` environment variable pointing at the management network instead of the high-speed fabric, or a topology the auto-detection logic didn't handle correctly. If multi-node training is slower than single-node extrapolation would predict, NCCL environment variables are the first place to look.

**What I'd want answered before signing a multi-node contract:**
- GPUs per node, by GPU type
- Interconnect type and generation (InfiniBand HDR/NDR, or Ethernet)
- Whether InfiniBand is enabled per-node or cluster-wide
- NVLink topology within a node

---

## 4. The Egress Line Item You Didn't Quote

Moving to a new GPU provider doesn't just mean moving your workload. It means moving your data. If your training dataset lives in AWS S3 and you're training on a bare metal GPU provider, you're paying egress on every training run that reads from S3.

The formula is mechanical:

```
monthly_egress_cost = dataset_size_GB × training_runs_per_month × egress_rate_per_GB
```

AWS egress is tiered: first 100 GB/month free, then $0.09/GB up to 10 TB, dropping to $0.085/GB and lower at higher volumes. GCP and Azure are similar. In practice, $0.09/GB is the operative rate for iterative training experiments. A 500 GB dataset running 20 training experiments per month is $900/month in egress — before a single GPU-hour. Caching training data locally on the GPU provider after the first pull, or using a same-cloud provider, eliminates this cost entirely. The formula tells you whether it's worth solving.

The egress number is exact arithmetic. It was the last thing I added to the recommendation tool — and nearly the last thing I would have thought to include.

Beyond egress cost, migration friction has a qualitative dimension. Not all cloud dependencies detach cleanly:

**High friction** — services with logic baked in, not just data stored. SageMaker endpoints embed training pipelines that aren't portable. EKS/GKE workloads carry IAM policies, cluster autoscalers, and networking rules that reference cloud-specific primitives. Moving these isn't a copy operation — it's a rewrite.

**Medium friction** — data dependencies where the data is portable but moving it takes time and money. S3 training data with a fixed egress cost. CloudWatch dashboards that need to be rebuilt elsewhere. These have a price you can calculate.

**Low friction** — ECR container images, exported model weights, raw datasets in open formats. These move freely.

Understanding which category each dependency falls into tells you the real migration cost — both the one-time move and the ongoing egress that doesn't go away after you've moved.

---

## 5. TCO Has Four Rows, Not One

True total cost of ownership for GPU infrastructure is four numbers:

| Component | Precision |
|-----------|-----------|
| Compute | Exact — GPU-hours × hourly rate |
| Storage | Exact — GB-months × storage rate |
| Egress | Exact — dataset size × runs × egress rate |
| Managed services premium | Estimated — SageMaker carries roughly 30% overhead vs self-managed |

**Compute** is the number everyone starts with. GPU-hours is the right unit — calculated from the training compute formula:

```
gpu_hours = (6 × parameters × dataset_tokens × epochs) / gpu_flops
```

This is derived from the standard estimate that training a transformer requires approximately 6 multiply-accumulate operations per parameter per token (forward pass + backward pass). GPU FLOPS are published in the vendor spec sheet — H100 SXM5 delivers 989 TFLOPS of BF16 tensor core throughput (dense; 1,979 TFLOPS with structured sparsity — vendors sometimes quote the sparsity figure, so verify which number you're comparing against). The math is exact — verifiable against your engineering team's own estimates.

**Storage** is exact given a provider's storage rate. It's easy to compare compute rates from one provider against storage rates from another, or skip storage entirely. At 500 GB of training data plus checkpoints plus output artifacts, storage is not a rounding error.

**Egress** is exactly the formula from the previous section. It's often the surprise line item — paid on every training run for the life of the contract if your data stays in the source cloud.

**Managed services** is the one honest estimate. SageMaker, Azure ML, and Vertex AI abstract away cluster management, autoscaling, and experiment tracking — at roughly 30% over bare metal compute. The point is to see it explicitly, not buried in a blended rate.

A bare metal provider at $2.50/hr looks cheap until you add $900/month in egress and the cost of managing your own cluster. SageMaker at $3.50/hr looks expensive until you account for what you're not building. Neither is universally correct. Both are visible when you run all four rows.

---

## Example: All Five Calculations on a Real Scenario

**Setup**: Fine-tuning LLaMA-3 70B on a 500 GB proprietary corpus. Training mode: QLoRA. Two candidates — a bare metal H100 80GB provider at $2.80/hr and AWS SageMaker. Current data location: AWS S3.

**Calculation 1 — VRAM fit**

NF4-quantized base: 70B × 0.5 bytes = 35 GB. LoRA adapter + paged Adam: ~12 GB. Peak: ~47 GB. One H100 80GB clears it with 33 GB to spare. No further VRAM analysis needed.

For reference: full fine-tuning on the same model (18 bytes/param × 70B = 1,260 GB aggregate) would require two eight-GPU H100 nodes under ZeRO-3 sharding — a completely different procurement.

**Calculation 2 — Quantization decision**

QLoRA (NF4) locks in a single-node configuration. If you later serve the trained adapter merged into the base model at BF16 for inference: 140 GB → two 80GB GPUs. That's a separate hardware decision for the serving layer.

**Calculation 3 — Multi-node**

47 GB peak < 80 GB node capacity. No multi-node. No InfiniBand questions to ask. No NCCL configuration. If the mode were full fine-tuning: ceil(1,260 / 80) = 16 GPUs minimum — two nodes, InfiniBand required, NCCL setup non-trivial. The quantization decision from step 2 made this a non-issue.

**Calculation 4 — Egress**

Training data lives in S3; bare metal provider is not AWS.

- One-time data pull: 500 GB × $0.09 = **$45**
- Per-run if re-reading from S3: $45/run
- 20 training experiments, each pulling fresh: **$900 total**
- Cache the dataset on the provider after the first pull: **$45 total**

Whether you cache is a workflow decision that changes the egress line by $855. Write it down.

**Calculation 5 — TCO**

GPU-hours: (6 × 70B params × 1B training tokens × 3 epochs) ÷ 989 TFLOPS = **354 GPU-hours**

| Component | Bare metal H100 | SageMaker |
|-----------|----------------|-----------|
| Compute | $991 (354 hr × $2.80) | $1,288 (+30%) |
| Storage (corpus + checkpoints + artifacts) | $28/month | $28/month |
| Egress — data cached after first pull | $45 | $0 (same-cloud) |
| Egress — 20 runs, pulling from S3 each time | $900 | $0 (same-cloud) |
| **Total — data cached** | **$1,064** | **$1,316** |
| **Total — 20 runs uncached** | **$1,919** | **$1,316** |

Bare metal wins by $252 if you pull data once and cache it on the provider. SageMaker wins by $603 if you run 20 experiments pulling fresh from S3 each time.

Neither answer is universally correct. Both are calculable before you sign anything.

---

## What This Changes

Working through these to build the recommendation engine made clear why $/hr comparisons fall short — the number is real, but it doesn't carry information about whether the configuration works, what the data movement costs, or whether InfiniBand is available for the multi-node case.

Before I could compare an H100 at $2.80/hr against an A100 at $1.60/hr, I needed to know whether my model fit in a single A100 node, what egress looked like, and whether InfiniBand was available. That's what these five calculations gave me.

**These calculations tell you whether a configuration works. They don't tell you which configurations to put on the table in the first place — that's the problem I built the tool to solve.**

---

*I've been building GTM intelligence tools — GPU Navigator™ is one of them. It runs these calculations against your actual workload profile and generates a report with actionable recommendations. Happy to show you a live demo. Reach out on [LinkedIn](https://www.linkedin.com/in/srikanthsamudrla/).*
