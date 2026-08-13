---
title: "Beyond GPUDirect: Moving Storage Control to the GPU"
date: 2026-08-10
description: "NVIDIA is open-sourcing cuFile and formalizing SCADA — GPU-initiated storage where the GPU builds NVMe commands and polls completions in VRAM. What GPUDirect already fixed, what SCADA actually moves, and why the drives have to change too."
tags: ["gpu", "storage", "nvme", "gpudirect", "scada", "kv-cache", "rag", "infrastructure"]
author: "Srikanth Samudrla"
showToc: true
TocOpen: false
draft: false
---

At FMS 2026 — the memory and storage industry's annual conference, held in early August — NVIDIA announced it is open-sourcing the `cuFile` API and the storage stack beneath it into a neutral GitHub organization, **XIO-SIG**, with **Google, Intel, and Meta** as founding co-maintainers. It also formalized **Storage-Next**, a 40-plus-vendor coalition (Micron, KIOXIA, DDN among them), and designated **SCADA** — Scaled Accelerated Data Access, no relation to the industrial-control acronym — as the framework to build against.

My first reaction was that it sounded incremental. `cuFile` is the API behind GPUDirect Storage, which was introduced in 2019 and has been shipping since 2021, with vendor certifications going back years — so what was left to announce? The distinction that changed my read is one NVIDIA's materials lean on — in my words:

**GPUDirect Storage removed the CPU from the data path. SCADA goes after the control path.**

**The short version:**

- GDS (2021) let drives DMA payload straight into GPU memory. The *bytes* stopped touching the CPU — the *requests* didn't.
- When reads get small and numerous, the per-request work is the whole cost. Host cores saturate before the drives do.
- SCADA puts a runtime on the GPU that builds NVMe commands, rings the doorbell, and polls completions in VRAM — the host does one-time setup, then it's out of the loop.
- Micron showed 230M 512-byte IOPS on one server with the host CPU essentially idle. That's only ~118 GB/s of bandwidth — a handful of drives' worth. The operation count is the news.

---

## Why request *count* became the problem, not request size

Every I/O has two halves. The **data path** is the payload in motion, and its cost scales with transfer size. The **control path** is request construction, submission, and completion handling, and its cost is roughly constant per request. GDS attacked the first: through `cuFile`, an NVMe device DMAs payload straight into GPU HBM over PCIe peer-to-peer, no bounce buffer in host DRAM. For a 100 GB checkpoint load that's all that matters.

Inference doesn't look like a checkpoint load. But when I tried to pin down *which* inference patterns are actually small, the story turned out less uniform than the headline:

- **Vector, graph, and embedding access really is fine-grained.** Quantized vector codes run tens to a few hundred bytes, and index traversal touches scattered nodes with no locality.
- **KV-cache paging isn't sub-kilobyte.** Working the numbers for a 70B-class model with GQA — the same KV-cache arithmetic I'd been doing for my GPU sizing tools — one token's K and V for a single layer is a few kilobytes, and across all layers it's into the hundreds. Serving systems page blocks of 16–256 tokens and move them layer by layer, so a transfer lands in tens of kilobytes to a megabyte. What makes KV offload hard seems to be latency and concurrency, not payload size.

So I read 512 bytes as a **forward-looking target** — the granularity Storage-Next is targeting — rather than today's average request. DiskANN-style indexes, for instance, are typically built around 4 KB nodes.

The structural argument survives that. Once request *count* is the scaling variable, the constant term dominates and host cores saturate before a Gen6 array does. It's an operations-per-second problem, not a bandwidth problem.

---

## How GDS works — and where the CPU sits

I'd been collapsing three architectures into two. Pre-GDS, the drive DMAs into a host DRAM bounce buffer and a second copy crosses PCIe into GPU memory. GDS drops the bounce buffer — data path bypassed, control path still on the host — and that's what's deployed today. SCADA moves the control path too. This section is the middle one.

Make it concrete. Say a chatbot sits on a few terabytes of embeddings and document chunks on NVMe — too big for HBM, past what you'd hold resident in DRAM. A question comes in, and the retrieval isn't one read: an embedding search narrows to candidates, reranking pulls more, and a long session pages KV blocks back from flash. Every one of those round trips looks like this:

```
                ┌────────────────────────────────────────────┐
                │                Host x86 CPU                │
                │                                            │
                │  (1) Kernel completes; control returns     │
                │  (2) cuFileRead() → syscall entry          │
                │  (3) Filesystem: file offset → LBA         │
                │  (4) Write SQE to host-DRAM SQ, ring bell  │
                │  (6) MSI-X IRQ → update I/O state          │
                │  (7) Signal stream; launch next kernel     │
                └──────────────────────┬─────────────────────┘
                                       │
                    CONTROL PATH — traversed per request
                                       │
   ┌──────────────────┐          ┌─────────────┐          ┌──────────────┐
   │ NVMe Controller  │─────────►│ PCIe Switch │─────────►│   GPU HBM    │
   └──────────────────┘          └─────────────┘          └──────────────┘
                        (5) P2P DMA — the part GDS fixed
```

Step one is the one worth dwelling on. CUDA threads have no syscall interface, no OS context, no file descriptors — a running kernel has no way to issue `read()`. So in the standard `cuFile` path a kernel that needs data it didn't have at launch has to *finish* first, and control returns to host code: a kernel-launch boundary, not a mid-kernel trap. (I had this wrong initially — my mental model was the GPU trapping mid-kernel into the OS, which isn't a thing it can do. Device code *can* spin on mapped memory and signal a host-side proxy, and newer GPU-initiated networking skips the proxy — GPU threads write the NIC's queues and ring its doorbell directly. Either way there's no syscall on the GPU side.)

From there the CPU calls `cuFileRead()`, resolves the file offset into block ranges through the filesystem, writes the 64-byte command into a submission queue in host DRAM, and rings the doorbell — the one MMIO write in the sequence. The controller fetches the command from host memory and DMAs the payload into VRAM — the one step GDS optimized — then fires an MSI-X interrupt back at the CPU, which services it and signals the stream so the next kernel can launch.

One caveat: that's the synchronous path. GDS also has stream-ordered async calls and batch submission, which pipeline requests instead of serializing one per launch boundary. As best I can tell that amortizes the hand-off without changing its shape — the host still builds and submits every request, and completions still retire through host software.

This felt familiar to me, and I think I know why. It's the SPDK argument moved one processor over: get NVMe queues into userspace, poll instead of taking interrupts, stop paying syscall cost per I/O. Networking got there first with DPDK and RDMA. SCADA applies that logic to GPU threads, which are the ones generating the requests now.

---

## What SCADA actually moves

The research behind SCADA is **BaM (Big Accelerator Memory)**, [published at ASPLOS 2023](https://arxiv.org/abs/2203.05572) by NVIDIA, IBM, the University of Illinois, and the University at Buffalo, which had a GPU managing NVMe directly — coalescing and caching the scattered requests its threads generate to hold drive-saturating queue depths with no host in the loop.

SCADA is the productized version. The detail I nearly missed: **there's a runtime layer on the GPU between kernel threads and the device.** Threads don't hand-craft NVMe commands out of attention math. They request logical addresses; the runtime batches them, absorbs hits against a software cache in GPU memory, and issues the misses.

```
┌──────────────────────────── GPU ─────────────────────────────┐
│                                                              │
│   [ CUDA threads ]                                           │
│          │ (1) request LBAs        (6) poll CQ, consume      │
│          ▼                              ▲                    │
│   [ SCADA runtime ] ── coalesce ── on-GPU cache lookup       │
│          │ (2) miss → write 64-byte SQE                      │
│          ▼                                                   │
│   VRAM:  [ SQ ]        [ CQ ]        [ data buffers ]        │
│            │             ▲                ▲                  │
└────────────┼─────────────┼────────────────┼──────────────────┘
             │             │                │
   (3) doorbell write   (5) CQE       (4) payload DMA
       PCIe BAR0 MMIO       posted        into VRAM
             │             │                │
             ▼             │                │
      ┌──────────────────────────────────────┐
      │           NVMe Controller            │
      └──────────────────────────────────────┘
```

On a cache miss the runtime writes a 64-byte submission-queue entry into a queue living in VRAM, and a GPU thread rings the controller's doorbell over PCIe BAR0 MMIO.

The controller then fetches the command from GPU memory, reads the blocks, P2P-DMAs the payload into VRAM, and posts the completion entry into a CQ in VRAM — no host interrupt. The originating threads poll and continue. The published demos run against local NVMe; NVIDIA says the model extends to remote storage as well.

Back to the chatbot: the same query issues the same scattered reads, but the host is no longer in the loop. Threads ask for addresses, the runtime batches them, and the cache absorbs the repeats — top-of-index nodes get hit constantly — so only the misses reach a drive.

### What stays on the CPU (it's device passthrough)

My first read of the announcement was "the CPU is eliminated," and that's not quite right. Privileged setup remains: a raw queue pair in unprivileged hands could read or write any block on the device, so a privileged component configures protected access between each application and only its approved storage *once*, before the fast path runs — built on standard Linux security mechanisms.

Structurally it reads as device passthrough — the VFIO / SR-IOV pattern: a privileged step establishes the IOMMU mappings and device scope, then the unprivileged consumer drives the hardware directly. Two separate fences do the containment — the IOMMU bounds where the device may DMA in memory, and namespace or virtual-function scoping bounds which blocks the application can touch. One-time setup buying per-operation freedom.

**SCADA complements `cuFile` rather than replacing it** — bulk sequential transfers through `cuFile`, high volumes of small random reads through SCADA.

| | GDS (today) | SCADA |
| --- | --- | --- |
| **Who builds the request** | Host, between kernel launches | Runtime on the GPU |
| **Per-request CPU work** | Syscall, filesystem mapping, SQE write, doorbell, IRQ | None — setup only, once |
| **Completion** | MSI-X interrupt to host | CQE into VRAM, threads poll |
| **Saturates when** | Host cores run out | Drives or PCIe run out — minus whatever the runtime costs |

---

## Why the drives have to change too

Moving control to the GPU exposes the next constraint, in the drive.

From what I've read, the reason controllers do roughly the same work for a 512-byte read as for 4 KB comes down to three things:

- **FTL indirection unit.** Enterprise SSDs typically map at 4 KB granularity, so a sub-4K read still costs a full mapping lookup.
- **ECC codeword size.** LDPC codewords are kilobyte-scale — you decode a whole one to return 512 bytes.
- **NAND page granularity.** A read senses an entire page, 16 KB or larger on modern TLC, regardless of what was asked for.

If that's right, it explains why the answer isn't firmware tuning — and it fits KIOXIA's announced Storage-Next entry using XL-FLASH, where smaller page structure and lower read latency change the economics rather than the bookkeeping around them. One wrinkle I hadn't considered: many enterprise SSDs ship 4Kn-formatted, where a 512-byte read isn't expressible at all, so this presumes a 512-byte LBA format on the namespace.

Storage-Next is the effort to retool indirection, error correction, and media around the smaller unit. The published roadmap talks about on the order of 100 million 512-byte IOPS per GPU under power and tail-latency constraints, with coverage naming Marvell among the controller vendors designing toward it. GPU-initiated I/O and small-block flash look like they only pay off together.

---

## The 230M IOPS number

[Micron's SC25 demo](https://www.micron.com/about/blog/storage/ssd/sc25-performance-breakthrough-230m-iops-in-a-single-server): **230 million 512-byte random-read IOPS from a single server** — 44 Micron 9650 Gen6 SSDs behind Broadcom PEX90000 Gen6 switches, driven by three H100s. [The coverage](https://www.blocksandfiles.com/ai-ml/2025/11/25/nvidia-scada-offloads-storage-control-path-to-the-gpu/1711995) puts each 7.68 TB drive at 5.4M random-read IOPS, so 44 cap at 237.6M; the demo hit roughly 97% of that, scaling linearly from 1 to 44 drives, with the host CPU essentially idle.

The arithmetic is worth doing: 230 million × 512 bytes is about **118 GB/s** — and with the 9650 specced at 28 GB/s sequential reads, four or five drives could deliver that. So the bandwidth isn't the impressive part — the operation count is, and that's the part host software would have to burn real CPU on if it were still in the loop. For scale: SPDK's published record is [10.39M IOPS from one polling thread](https://spdk.io/news/2019/05/06/nvme/) with the kernel fully bypassed — against that reference, 230M IOPS is over twenty cores doing nothing but submitting and completing I/O.

One qualifier: it's a synthetic benchmark, so it measures the mechanism, not an end-to-end pipeline.

The pitch behind all of it is that serving KV cache from flash rather than memory — the chatbot's long-session paging — raises the context length and concurrent-user count each GPU supports, which sets per-user cost. Going by the arithmetic earlier, what KV paging stands to gain is the host-free completion path and the concurrency, not the small-block granularity — the sub-kilobyte case is the vector and graph side. [The same FMS announcement](https://blogs.nvidia.com/blog/ai-storage-fms/) also listed NVIDIA CMX Context Memory Storage for agentic AI inference.

---

## Where I've landed

The mental model I ended up with is simpler than the one I started with: the I/O path we have was built to broker requests for a handful of CPU threads, and the consumer now is a hundred thousand GPU threads retiring millions of fine-grained requests per second. Peer-to-peer DMA fixed the bytes; the request *about* the bytes stayed on the CPU. SCADA moves the request too.

If you work on any of this and I've got something wrong, I'd like to hear it.

---

*I've been building GTM intelligence tools for AI infrastructure. Meridian maps an inference workload — model, precision, context, concurrency, latency SLO — to a specific NVIDIA GPU count, topology, and server configuration; Tokenomics Navigator turns the same workload into an on-prem fleet plan with power draw, capex, and cost per million tokens. The KV-cache arithmetic in this post is the same physics both tools run. Happy to show you a live demo. Reach out on [LinkedIn](https://www.linkedin.com/in/srikanthsamudrla/).*
