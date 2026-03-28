---
layout: post
title: "Where Does GPU Memory Actually Go During Inference?"
comments: true
description: "A concrete breakdown of GPU memory during diffusion model inference — using FLUX.2 klein 4B as a worked example."
keywords: "GPU memory VRAM HBM inference diffusion model FLUX weights activations KV cache VAE quantization"
---

{% include animations.html %}

You load a 4B parameter diffusion model. The card says "~13GB VRAM." You have a 24GB GPU. That leaves 11GB free — plenty, right?

Then you try generating a 2048×2048 image and get an OOM error. Where did all the memory go?

## What Is "GPU Memory"?

A GPU isn't one pool of memory — it's a hierarchy with very different sizes and speeds.

| Memory | Size (H100) | Bandwidth | Role |
|--------|-------------|-----------|------|
| **HBM** | 80 GB | 3.35 TB/s | Weights, activations, KV cache — everything |
| **L2 Cache** | 50 MB | ~5.5 TB/s | Recently accessed data (hardware-managed) |
| **SMEM** (per SM) | 256 KB | ~30 TB/s | Data actively being computed on |
| **Registers** (per SM) | 256 KB | Instant | Values mid-computation |

"GPU memory" and "VRAM" mean **HBM**. That's what OOMs are about.

*Architecture details from [How to Think About GPUs](https://jax-ml.github.io/scaling-book/gpus/) by the Google DeepMind scaling team.*

## The Five Components: FLUX.2 klein 4B

Using [FLUX.2 klein 4B](https://huggingface.co/black-forest-labs/FLUX.2-klein-4B) — a 4B parameter rectified flow transformer, 4 inference steps, model card says ~13GB VRAM.

### 1. Model Weights (~8 GB) — Fixed

4B params × 2 bytes (BF16) = 8 GB. Always in memory regardless of resolution or batch size. Includes the DiT transformer, text encoder, and VAE decoder.

**With FP8:** drops to ~4 GB. This is the single biggest win for memory-constrained deployment.

### 2. Activations (~2-8 GB) — Scales with Resolution²

Intermediate tensors from every transformer layer. At 1024×1024 (sequence length ~4096): ~2 GB. At 2048×2048 (sequence length ~16384): ~8 GB.

**Why quadratic:** double the resolution → 4× the tokens → 4× the activations.

### 3. KV Cache (~1-6 GB) — The Hidden Hog

Every attention layer stores Key and Value tensors for the current sequence. Modest at 1024×1024, but at 2048×2048 it can hit 6+ GB.

**This is why high-res OOMs.** Weights don't change with resolution. KV cache does — quadratically.

### 4. VAE Decode (~1-4 GB) — The Final Spike

Converting latent space to pixels creates large temporary tensors. Often the thing that pushes you over the edge at high resolution.

### 5. Framework Overhead (~0.5 GB) — The Tax

CUDA context, PyTorch allocator, memory fragmentation. The "where did my last gigabyte go?" tax.

## The Numbers

| Component | 1024×1024 | 2048×2048 | What scales it |
|-----------|-----------|-----------|----------------|
| Weights | ~8 GB | ~8 GB | Parameter count (fixed) |
| Activations | ~2 GB | ~8 GB | Resolution² × batch |
| KV Cache | ~1.5 GB | ~6 GB | Resolution² × layers |
| VAE Decode | ~1 GB | ~4 GB | Output resolution |
| Overhead | ~0.5 GB | ~0.5 GB | Fixed |
| **Total** | **~13 GB** ✅ | **~26.5 GB** ❌ | |

<div class="anim-embed" style="height:700px"><diffusion-accel scene="gpu-memory"></diffusion-accel></div>

## Bandwidth: Why FP8 Helps Speed, Not Just Size

Even when the model fits, **bandwidth determines speed.** Reading 8 GB of weights on an H100 (3.35 TB/s) takes 2.4ms. With 4 steps, that's 9.6ms just moving weights.

FP8 halves the bytes → halves the transfer time → faster inference. Caching skips the transfer entirely.

The ratio of compute to bandwidth is the **arithmetic intensity**. On an H100, you need ~295 FLOPs per byte to keep the GPU busy. Below that, the GPU waits for data.

## What To Do About It

**Reduce weights:** FP8 quantization (8→4 GB). Always do this first.

**Reduce dynamic memory:** CPU offloading moves the text encoder to RAM when unused. Caching (TeaCache, DBCache) skips redundant transformer passes.

**Distribute across GPUs:** Weight sharding (HSDP), sequence splitting (Ulysses-SP, Ring-Attention), spatial VAE decode (VAE Patch Parallel).

**Reduce step count:** [Step distillation and guidance distillation](/2026/step-distillation-vs-guidance-distillation/) cut the number of transformer passes from 50-100 down to 4-8.

## Key Takeaways

1. **Weights are fixed.** Everything else scales with resolution² × batch size.
2. **The model card number is for minimum resolution.** Real usage can 2-3× it.
3. **Bandwidth matters as much as capacity.** Fitting in memory ≠ fast inference.
4. **No single technique solves everything.** Quantize + cache + parallelize.

## References

1. Black Forest Labs. "[FLUX.2 klein 4B](https://huggingface.co/black-forest-labs/FLUX.2-klein-4B)", 2025.
2. Austin et al. "[How to Think About GPUs](https://jax-ml.github.io/scaling-book/gpus/)", Google DeepMind, 2025.
3. NVIDIA. "[H100 Tensor Core GPU](https://www.nvidia.com/en-us/data-center/h100/)", 2023.
