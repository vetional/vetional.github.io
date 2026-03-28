---
layout: post
title: "Where Does GPU Memory Actually Go During Inference?"
comments: true
description: "A concrete breakdown of GPU memory during diffusion model inference — using FLUX.2 klein 4B as a worked example."
keywords: "GPU memory VRAM HBM inference diffusion model FLUX weights activations KV cache VAE quantization"
---

{% include animations.html %}

You load a 4B parameter diffusion model. The card says "~13GB VRAM." You have a 24GB GPU. That leaves 11GB free — plenty, right?

Then you try generating a 2048×2048 image and get an OOM error.

Where did all the memory go?

## First: What Is "GPU Memory"?

Before diving into the breakdown, let's clarify what we're actually talking about. A modern GPU has a **memory hierarchy** — not just one pool of memory, but several layers with very different sizes and speeds.

| Memory | Size (H100) | Speed | What it stores |
|--------|-------------|-------|----------------|
| **HBM** (High Bandwidth Memory) | 80 GB | 3.35 TB/s | Weights, activations, KV cache — everything |
| **L2 Cache** | 50 MB | ~5.5 TB/s | Recently accessed data (hardware-managed) |
| **SMEM** (Shared Memory / L1) | 256 KB per SM | ~30 TB/s | Data actively being computed on |
| **Registers** | 256 KB per SM | Instant | Values mid-computation |

When people say "GPU memory" or "VRAM," they mean **HBM** — the big pool. That's what OOMs are about. The caches are managed by hardware and invisible to you.

The key insight: **HBM bandwidth is the bottleneck, not capacity.** An H100 can store 80GB but only read it at 3.35 TB/s. If your model needs to read 14GB of weights for every step, that's 4.2ms just for the memory transfer — before any computation happens.

This is why quantization helps with speed, not just memory: **FP8 weights are half the bytes to transfer**, so the memory bus moves them twice as fast.

*GPU architecture details from [How to Think About GPUs](https://jax-ml.github.io/scaling-book/gpus/) — an excellent deep dive from the Google DeepMind scaling team.*

## The Breakdown: FLUX.2 klein 4B at 1024×1024

Let's use a concrete model: [FLUX.2 klein 4B](https://huggingface.co/black-forest-labs/FLUX.2-klein-4B) from Black Forest Labs. It's a 4 billion parameter rectified flow transformer that generates images in 4 steps. The model card says ~13GB VRAM.

Here's where that memory actually goes:

### 1. Model Weights (~8 GB) — The Fixed Cost

4 billion parameters × 2 bytes (BF16) = **8 GB**.

This is the floor. No matter what resolution you generate, no matter the batch size, the weights are always in memory. They include:
- **DiT transformer** (~3.5B params): the denoising backbone
- **Text encoder** (T5-XXL or CLIP): encodes your prompt into embeddings
- **VAE decoder** (~100M params): converts latent space back to pixels

With FP8 quantization, this drops to **~4 GB** — half the bytes, same model. That's why the model card says "as little as 13GB" — they're assuming some offloading or quantization.

### 2. Activations (~2-3 GB) — Scales with Resolution

At each denoising step, the transformer produces intermediate tensors — the outputs of every attention layer, every feedforward block, every normalization. These are the **activations**.

For a 1024×1024 image with a typical patch size of 2, you get a sequence length of ~4096 tokens. Each transformer layer produces activations proportional to `sequence_length × hidden_dim`. With 24+ layers, this adds up.

**The killer: activations scale with resolution².** Double the resolution (1024→2048) and the sequence length quadruples (4096→16384), so activations grow ~4×.

### 3. KV Cache (~1-2 GB) — The Hidden Memory Hog

Every attention layer stores Key and Value tensors for the current sequence. For a 4B model with ~24 layers, each storing K and V of shape `[seq_len, head_dim × num_heads]` in BF16:

At 1024×1024 (seq_len ≈ 4096): modest, maybe 1-2 GB.
At 2048×2048 (seq_len ≈ 16384): **4-8 GB** — this is where OOMs come from.

The KV cache is why high-resolution generation is so memory-hungry. The weights don't change, but the KV cache grows quadratically with resolution.

### 4. VAE Decode (~1-2 GB) — The Final Spike

After all denoising steps, the VAE decoder converts the latent representation to pixels. This involves upsampling from latent space (e.g., 128×128) to full resolution (1024×1024), creating large intermediate tensors.

At 2048×2048, the VAE decode can spike to **4+ GB** of temporary memory — often the thing that pushes you over the edge.

### 5. Framework Overhead (~0.5-1 GB) — The Tax

CUDA context, PyTorch allocator, memory fragmentation, gradient graphs (even in inference mode if you're not careful). This is the "where did my last gigabyte go?" tax.

## The Full Picture

<div class="anim-embed" style="height:700px"><diffusion-accel scene="gpu-memory"></diffusion-accel></div>

For FLUX.2 klein 4B at 1024×1024 on a 24GB GPU:

| Component | 1024×1024 | 2048×2048 | What scales it |
|-----------|-----------|-----------|----------------|
| **Weights** | ~8 GB | ~8 GB | Fixed (parameter count) |
| **Activations** | ~2 GB | ~8 GB | Resolution² × batch size |
| **KV Cache** | ~1.5 GB | ~6 GB | Resolution² × num_layers |
| **VAE Decode** | ~1 GB | ~4 GB | Output resolution |
| **Overhead** | ~0.5 GB | ~0.5 GB | Fixed |
| **Total** | **~13 GB** | **~26.5 GB** ❌ | |

At 1024×1024, it fits on a 24GB GPU with room to spare. At 2048×2048, it doesn't fit at all — even though the model is the same.

## What Each Acceleration Technique Reduces

Now the acceleration techniques from the [diffusion acceleration explainers](https://vetional.github.io/learning-animations/diffusion-accel.html) map directly to these memory components:

| Technique | What it reduces | How |
|-----------|----------------|-----|
| **FP8/INT8 Quantization** | Weights (8→4 GB) | Half the bytes per parameter |
| **HSDP (Weight Sharding)** | Weights per GPU | Split across multiple GPUs |
| **TeaCache / DBCache / SCM** | Activations | Skip redundant transformer passes |
| **Ulysses-SP** | KV Cache + Activations | Split sequence across GPUs |
| **Ring-Attention** | KV Cache | Circulate K/V, less memory per GPU |
| **CFG-Parallel** | Activations (2× from guidance) | Split the two CFG passes |
| **VAE Patch Parallel** | VAE Decode | Split spatial decode across GPUs |
| **CPU Offloading** | Everything | Move unused components to RAM |

The practical recipe for fitting high-res generation on limited hardware:

1. **Quantize weights** (FP8) — saves ~4 GB, always do this first
2. **Enable CPU offloading** — moves text encoder to RAM when not needed
3. **Use caching** (TeaCache) — reduces peak activation memory by skipping passes
4. **If still OOM** — reduce resolution, or add GPUs with parallelism

## The Bandwidth Bottleneck

Here's the thing most people miss: even when the model fits in memory, **bandwidth determines speed**.

An H100 has 3.35 TB/s of HBM bandwidth. Reading 8 GB of weights takes 2.4ms. With 4 denoising steps, that's at least 9.6ms just moving weights — before any computation.

This is why:
- **FP8 quantization makes inference faster** (not just smaller): half the bytes = half the transfer time
- **Caching skips entire weight reads**: if you reuse the previous output, you don't read the weights at all
- **Batch size matters**: reading weights once for 2 images costs the same bandwidth as 1 image

The ratio of compute FLOPs to memory bandwidth is called the **arithmetic intensity**. On an H100, you need about 295 FLOPs per byte transferred to keep the GPU busy. Below that, you're **memory-bound** — the GPU is waiting for data, not computing.

*For a deeper treatment of arithmetic intensity and roofline analysis, see [How to Think About GPUs](https://jax-ml.github.io/scaling-book/gpus/) from the Google DeepMind scaling team.*

## Key Takeaways

1. **Weights are the fixed cost** — same regardless of resolution or batch size. Quantization cuts this in half.
2. **Everything else scales with resolution²** — double the resolution, quadruple the dynamic memory.
3. **The model card VRAM number is for the minimum resolution** — real-world usage at higher resolutions can easily 2-3× that number.
4. **Bandwidth matters as much as capacity** — fitting in memory is necessary but not sufficient for fast inference.
5. **No single technique solves everything** — combine quantization (weights) + caching (activations) + parallelism (everything) for maximum effect.

## References

1. Black Forest Labs. "[FLUX.2 klein 4B](https://huggingface.co/black-forest-labs/FLUX.2-klein-4B)", Hugging Face, 2025.
2. Austin, Douglas, Frostig, Levskaya, et al. "[How to Think About GPUs](https://jax-ml.github.io/scaling-book/gpus/)", Google DeepMind Scaling Book, 2025.
3. NVIDIA. "[H100 Tensor Core GPU Architecture](https://www.nvidia.com/en-us/data-center/h100/)", 2023.
4. Orbit2x. "[Ultimate VRAM Calculator Guide](https://orbit2x.com/blog/ultimate-vram-calculator-guide-gpu-memory-ai-models)", 2025.
5. Lyceum Technology. "[GPU Memory Requirements for Transformer Models](https://lyceum.technology/magazine/gpu-memory-requirements-transformer)", 2025.
