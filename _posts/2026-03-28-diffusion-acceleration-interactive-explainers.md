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
- **Why Cache?**, GPS navigation analogy: the transformer predicts a direction, the scheduler moves. Late steps predict almost the same direction, so reuse it.
- **TeaCache**, Measure L1 distance between outputs, compare to threshold, decide to cache or recompute.
- **DBCache**, Run only the first 2 layers to decide if the rest should be cached.
- **TaylorSeer**, Predict the next output from the trend using Taylor expansion. Zero GPU cost.
- **SCM**, Pre-defined compute/cache schedule. No runtime overhead.

### Parallelism (5 animations)
- **CFG-Parallel**, Visual subtraction: (with prompt) − (without prompt) = prompt direction. Split across GPUs.
- **Ulysses-SP**, Split sequence across GPUs, all-to-all for attention heads. Near-linear scaling.
- **Ring-Attention**, K/V blocks circulate in a ring. Less memory than all-to-all.
- **HSDP**, 28GB model vs 24GB GPU. Shard weights, gather on-demand.
- **VAE Patch Parallel**, Split latent spatially, decode per GPU, stitch.

### Quantization (1 animation)
- **FP8/Int8**, Visual bars showing 2× memory reduction. Bell-curve histogram showing why rounding works.

### Big Picture (1 animation)
- **GPU Memory Map**, Stacked bars showing memory breakdown at different resolutions and batch sizes. Table mapping each technique to its memory and latency impact.

<a href="/learning-animations/diffusion-accel.html" style="display:inline-block;padding:12px 24px;background:#2ecc71;color:#fff;border-radius:6px;text-decoration:none;font-weight:bold;margin:16px 0;">▶ Launch Interactive Explainers</a>

Also available: [P90 Latency](/learning-animations/p90.html) · [Step Distillation](/learning-animations/distillation.html) · [Guidance Distillation](/learning-animations/guidance.html) · [GPU Parallelism](/learning-animations/parallelism.html)
