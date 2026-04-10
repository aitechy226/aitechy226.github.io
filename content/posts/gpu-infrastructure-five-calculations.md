---
title: "GPU Infrastructure: The Five Calculations That Actually Matter"
date: 2026-04-10
description: "$/hr is one number. Here are the five calculations that determine whether your GPU deployment actually works — VRAM fit, quantization impact, multi-node thresholds, egress cost, and real TCO."
tags: ["gpu", "infrastructure", "llm", "mlops", "distributed-training", "tco", "quantization"]
author: "Srikanth Samudrla"
showToc: true
TocOpen: false
draft: true
---

Every GPU procurement conversation starts with $/hr. I've seen teams spend weeks on vendor comparisons built entirely around that one number — and then discover after the contract is signed that their model doesn't fit in the node's VRAM, their training data lives in a different cloud, and the InfiniBand they assumed exists doesn't.

$/hr is a real cost. It's also the last thing you should be calculating. Here are the five that come first.

---

## 1. VRAM Fit: The Hard Constraint Before Everything Else

Before you compare GPU vendors, you need to know whether your model fits on the GPU at all. This is not a soft requirement. If VRAM is exhausted, training crashes — not degrades, crashes. The number you need is the full memory envelope: model weights plus optimizer state plus activations plus the KV cache for inference.

**Model weights** are the baseline. The per-parameter byte cost depends entirely on your training mode:

| Mode | Bytes per parameter |
|------|---------------------|
| Full fine-tuning (mixed precision: BF16 compute + FP32 optimizer states, Adam) | 18 bytes |
| LoRA (BF16 weights + low-rank adapter + optimizer) | 8 bytes |
| QLoRA (NF4 quantized base: 0.5 bytes/param + LoRA adapter + paged Adam overhead) | ~0.5 bytes base + ~5–15 GB adapter overhead |
| BF16 inference (weights only, no optimizer) | 2 bytes |

A 70B parameter model under full fine-tuning requires roughly 1.26 TB of VRAM. Under QLoRA, the NF4-quantized base model weighs about 35 GB, with LoRA adapter parameters and paged optimizer adding roughly 5–15 GB — total training state fits on a single A100 80GB node. That's not a rounding error. That's the difference between a multi-node cluster and a single node.

**KV cache for inference** adds memory that scales with sequence length, not model size:

```
KV cache = 2 × num_layers × num_kv_heads × head_dim × seq_len × 2 bytes
```

The `2` at the front is for K and V. The `2 bytes` at the end assumes BF16 storage. Note that modern large models use Grouped Query Attention (GQA), where `num_kv_heads` is much smaller than the total attention head count — LLaMA-70B uses 8 KV heads vs. 64 query heads, which reduces KV cache by 8×. For a 70B model at 8K context with 80 layers, 32 KV heads, and 128 head_dim, that's roughly 10 GB per sequence in flight. In a serving scenario with batched requests, this adds up fast.

**Activations** are estimated with gradient checkpointing assumed. With checkpointing enabled, you're trading GPU memory for recomputation — storing only a subset of intermediate activations and recomputing the rest on the backward pass. Without checkpointing, activation memory can exceed weight memory on long sequences.

The VRAM calculation isn't hard math. It's knowing which formula applies to which mode. Teams skip this step, get a flashy $/hr number, and find out at job launch time.

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

The catch is precision loss. INT8 is generally safe for inference with minor accuracy degradation. INT4 (via GPTQ or GGUF) can introduce measurable accuracy loss on tasks that require numerical precision — financial calculations, structured output with tight constraints, legal reasoning where exact phrasing matters. For most inference workloads, INT8 is the right default. For training, the choice between FP32 and BF16 affects numerical stability — BF16 has the same exponent range as FP32, which is why it's the standard training dtype for large models.

The point is not to pick a quantization level in isolation. The point is that your quantization decision and your hardware selection are the same decision. Make them together or you're optimizing the wrong variable.

---

## 3. Multi-Node Is Not Just More GPUs

Once the model doesn't fit on a single node, the architecture changes — and not just because there are more GPUs.

The threshold is simple: if required VRAM exceeds single-node capacity, you need multi-node. The ceiling division tells you how many nodes:

```
nodes_required = ceil(total_vram_required / vram_per_node)
```

But that calculation only tells you how many nodes. It doesn't tell you whether the nodes can communicate fast enough to make distributed training work.

**Inside a single node**, GPUs communicate over NVLink — NVIDIA's proprietary interconnect running at 900 GB/s on H100 NVLink 4.0 (600 GB/s on A100 NVLink 3.0). Tensor parallelism (splitting a single layer across multiple GPUs) is viable at this bandwidth. You can shard individual weight matrices horizontally across GPUs and the synchronization cost is low enough to be worth it.

**Across nodes**, the story changes. You're on InfiniBand or Ethernet. InfiniBand HDR gives you about 200 Gb/s (25 GB/s) per port — two orders of magnitude slower than NVLink. At this bandwidth, tensor parallelism across nodes is usually a net loss. The synchronization overhead exceeds the compute benefit. You switch to pipeline parallelism instead — each node holds complete model layers, and the forward pass flows through nodes sequentially.

This distinction matters for workload design. If your model requires cross-node tensor parallelism, InfiniBand isn't optional — it's a correctness requirement, not a performance preference.

The mechanism that synchronizes multi-GPU training is **NCCL** — NVIDIA Collective Communications Library. NCCL handles AllReduce operations: after each backward pass, every GPU has computed its local gradients, and NCCL aggregates them across all GPUs so each one updates with the full gradient. The abstraction is clean. The failure modes aren't.

NCCL misconfiguration doesn't crash training. It silently degrades throughput — sometimes to 20% of expected performance — because the collective operations serialize where they should be parallel. The symptom looks like slow hardware. The cause is usually a wrong `NCCL_SOCKET_IFNAME` environment variable pointing at the management network instead of the high-speed fabric, or a topology that the auto-detection logic didn't handle correctly. If multi-node training is slower than single-node extrapolation would predict, check NCCL environment variables before blaming the hardware.

**What you need from your provider before signing a multi-node contract:**
- GPUs per node, by GPU type
- Interconnect type and generation (InfiniBand HDR/NDR, or Ethernet)
- Whether InfiniBand is enabled per-node or cluster-wide
- NVLink topology within a node

These aren't questions you should be asking after procurement. They determine whether your distributed training architecture is even possible on that infrastructure.

---

## 4. The Egress Line Item You Didn't Quote

Moving to a new GPU provider doesn't just mean moving your workload. It means moving your data. If your training dataset lives in AWS S3 and you're training on a bare metal GPU provider, you're paying egress on every training run that reads from S3.

The formula is mechanical:

```
monthly_egress_cost = dataset_size_GB × training_runs_per_month × egress_rate_per_GB
```

AWS egress runs $0.09/GB (as of writing, first 10 TB/month). GCP and Azure are both around $0.08/GB. A 500 GB dataset running 20 training experiments per month at $0.09/GB is $900/month in egress alone — before you've paid for a single GPU-hour.

The egress number is exact arithmetic. Most infrastructure comparisons ignore it entirely.

Beyond egress cost, migration friction has a qualitative dimension. Not all cloud dependencies detach cleanly:

**High friction** — services with logic baked in, not just data stored. SageMaker endpoints embed training pipelines that aren't portable. EKS/GKE workloads carry IAM policies, cluster autoscalers, and networking rules that reference cloud-specific primitives. Moving these isn't a copy operation — it's a rewrite.

**Medium friction** — data dependencies where the data is portable but moving it takes time and money. S3 training data with a fixed egress cost. CloudWatch dashboards that need to be rebuilt elsewhere. These have a price you can calculate.

**Low friction** — ECR container images, exported model weights, raw datasets in open formats. These move freely.

Understanding which category each dependency falls into tells you what the real migration cost is — both the one-time move cost and the ongoing egress cost that doesn't go away even after you've moved. Teams that only calculate one-time costs are surprised by the recurring line item.

---

## 5. TCO Has Four Rows, Not One

True total cost of ownership for GPU infrastructure is four numbers:

| Component | Precision |
|-----------|-----------|
| Compute | Exact — GPU-hours × hourly rate |
| Storage | Exact — GB-months × storage rate |
| Egress | Exact — dataset size × runs × egress rate |
| Managed services premium | Estimated — SageMaker carries roughly 30% overhead vs self-managed |

**Compute** is the number everyone starts with. The right unit is GPU-hours, calculated from the training compute formula:

```
gpu_hours = (6 × parameters × dataset_tokens × epochs) / gpu_flops
```

This is derived from the standard estimate that training a transformer requires approximately 6 multiply-accumulate operations per parameter per token (forward pass + backward pass). GPU FLOPS are published in the vendor spec sheet — H100 SXM5 delivers 989 TFLOPS of BF16 tensor core throughput (dense; 1,979 TFLOPS with structured sparsity — vendors sometimes quote the sparsity figure, so verify which number you're comparing against). The math is exact. You hand it to your engineering team and they verify it against their own estimates.

**Storage** is exact given a provider's storage rate. The gotcha is that many GPU infrastructure comparisons use compute rates from one provider and storage rates from another, or omit storage entirely. At 500 GB of training data plus checkpoints plus output artifacts, storage is not a rounding error.

**Egress** is exactly the formula from the previous section. It's often the surprise line item — paid on every training run for the life of the contract if your data stays in the source cloud.

**Managed services** is the one honest estimate. SageMaker, Azure ML, and Vertex AI all abstract away cluster management, autoscaling, and experiment tracking — and they charge for it. The premium is roughly 30% over equivalent GPU compute on a bare metal provider. That premium may be worth it for teams without MLOps capacity. It may be expensive overhead for teams that can manage their own infrastructure. The point is to see it explicitly, not buried in a blended rate.

What the four-row table forces is a side-by-side comparison that includes all the terms. A bare metal GPU provider at $2.50/hr looks cheap until you add $900/month in egress and the engineering time to manage your own cluster. SageMaker at $3.50/hr looks expensive until you account for what you're not building. Neither answer is universally correct. Both answers are visible when you calculate all four rows.

---

## What This Changes

None of these calculations require proprietary data. VRAM fit math uses published model specs and your training mode. Quantization requirements follow from the precision table. Multi-node thresholds follow from VRAM fit. Egress is dataset size times a published rate. TCO is four line items.

The reason most infrastructure decisions skip them isn't complexity — it's that vendor comparison tools are built around the one number vendors compete on: $/hr. The other four numbers are things you calculate yourself, and most buyers don't know they need to.

An H100 at $2.80/hr and an A100 at $1.60/hr are not comparable numbers until you know whether your model fits in a single A100 node, what your training data egress looks like, and whether InfiniBand is available for the multi-node case. Once you know those things, the comparison is straightforward arithmetic.

Do the five calculations first. The $/hr comparison is the easy part.
