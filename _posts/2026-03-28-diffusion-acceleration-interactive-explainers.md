---
layout: post
title: "Diffusion Acceleration, Interactive Visual Explainers"
comments: true
description: "12 interactive animations explaining caching, parallelism, and quantization techniques for diffusion models"
keywords: "diffusion models vllm gpu parallelism quantization caching teacache dbcache taylorseer"
---

## How expensive is generating one image?

A 20B parameter diffusion model (like Qwen-Image) generating a single 1024×1024 image with 50 steps and classifier-free guidance requires roughly **17,200 TFLOPs** of compute.

For comparison, rendering one frame of Call of Duty at 4K with ray tracing takes about **2.8 TFLOPs**. That means generating a single image costs as much compute as rendering **~6,000 frames of Call of Duty**, or about 2 minutes of gameplay at 60fps.

This is why diffusion acceleration matters. The techniques below can cut that cost by 6-24×.

---

A set of **12 interactive animated explainers** (samwho.dev-style) covering all the diffusion acceleration techniques from vLLM-Omni. Each concept gets its own PIXI.js + GSAP animation with play/pause/replay controls.

### Caching (5 animations)

Caching skips ~50% of transformer passes by reusing previous outputs when consecutive steps barely change. **Saves: ~8,600 TFLOPs (~50s of CoD at 60fps).**

- **Why Cache?** GPS navigation analogy: the transformer predicts a direction, the scheduler moves. Late steps predict almost the same direction, so reuse it.
- **TeaCache.** Measure L1 distance between outputs, compare to threshold, decide to cache or recompute. ~1.5-2× speedup.
- **DBCache.** Run only the first 2 layers to decide if the rest should be cached. ~1.85× speedup.
- **TaylorSeer.** Predict the next output from the trend using Taylor expansion. Zero GPU cost.
- **SCM.** Pre-defined compute/cache schedule. No runtime overhead.

After caching: **~8,600 TFLOPs per image (~3,000 CoD frames, ~50s of gameplay).**

### Parallelism (5 animations)

Parallelism splits work across GPUs. CFG-Parallel alone halves the per-step cost. **Saves: ~8,600 TFLOPs on a single GPU (or cuts wall-clock time proportionally across GPUs).**

- **CFG-Parallel.** Split the 2 CFG passes across 2 GPUs. 2× faster per step. **Saves ~8,600 TFLOPs of wall-clock time (~50s of CoD).**
- **Ulysses-SP.** Split sequence across GPUs, all-to-all for attention heads. 4 GPUs: 2.84× speedup.
- **Ring-Attention.** K/V blocks circulate in a ring. Less memory than all-to-all. 4 GPUs: 1.94× speedup.
- **HSDP.** 28GB model vs 24GB GPU. Shard weights, gather on-demand. Enables large models, no speedup.
- **VAE Patch Parallel.** Split latent spatially, decode per GPU, stitch. 4× faster decode.

### Quantization (1 animation)

FP8 halves the bytes per weight. Fewer bytes to transfer = faster matmuls. **Saves: ~20% compute time (~3,400 TFLOPs, ~20s of CoD).**

- **FP8/Int8.** Visual bars showing 2× memory reduction. Bell-curve histogram showing why rounding works. ~1.28× faster inference.

After FP8: **~13,800 TFLOPs per image (~4,900 CoD frames, ~82s of gameplay).**

### Combining everything

| Optimization | TFLOPs/image | CoD frames equivalent | Gameplay at 60fps |
|---|---|---|---|
| Baseline (20B, 50 steps, CFG) | 17,200 | ~6,100 | ~102s |
| + FP8 quantization (1.28×) | 13,400 | ~4,800 | ~80s |
| + Caching, skip 50% passes (2×) | 6,700 | ~2,400 | ~40s |
| + Step distillation, 8 steps (6×) | 1,100 | ~400 | ~6.5s |
| + Guidance distillation (2×) | 550 | ~200 | ~3.3s |

From 102 seconds of Call of Duty per image down to 3.3 seconds. That is a **31× reduction**.

### Big Picture (1 animation)
- **GPU Memory Map.** Stacked bars showing memory breakdown at different resolutions and batch sizes. Table mapping each technique to its memory and latency impact.

<a href="/learning-animations/diffusion-accel.html" style="display:inline-block;padding:12px 24px;background:#2ecc71;color:#fff;border-radius:6px;text-decoration:none;font-weight:bold;margin:16px 0;">▶ Launch Interactive Explainers</a>

Also available: [P90 Latency](/learning-animations/p90.html) · [Step Distillation](/learning-animations/distillation.html) · [Guidance Distillation](/learning-animations/guidance.html) · [GPU Parallelism](/learning-animations/parallelism.html)
