---
layout: post
title: "Diffusion Acceleration, Interactive Visual Explainers"
comments: true
description: "12 interactive animations explaining caching, parallelism, and quantization techniques for diffusion models"
keywords: "diffusion models vllm gpu parallelism quantization caching teacache dbcache taylorseer"
---

{% include animations.html %}

## How expensive is generating one image?

A 20B parameter diffusion model (like Qwen-Image) generating a single 1024×1024 image with 50 steps and classifier-free guidance requires roughly **17,200 TFLOPs** of compute.

For comparison, rendering one frame of Call of Duty at 4K with ray tracing takes about **2.8 TFLOPs**. That means generating a single image costs as much compute as rendering **~6,000 frames of Call of Duty**, or about 2 minutes of gameplay at 60fps.

This is why diffusion acceleration matters. The techniques below can cut that cost by up to 31×.

---

I built **12 interactive animated explainers** covering all the diffusion acceleration techniques from vLLM-Omni. Each one walks through the "why" before the "how", with play/pause/replay controls. Here is a summary of what each technique does and how much compute it saves.

## Caching: skip what barely changed

At each denoising step, the transformer predicts a direction to remove noise. In late steps, this direction barely changes from one step to the next. Caching detects this and reuses the previous output instead of recomputing it. This skips roughly half the transformer passes, saving **~8,600 TFLOPs per image (~50 seconds of Call of Duty).**

There are several ways to decide when to cache. **TeaCache** measures the L1 distance between consecutive outputs and compares it to a threshold. If the difference is small enough, it reuses the cached result (1.5-2× speedup). **DBCache** takes a cheaper approach: run only the first 2 transformer layers, measure the residual difference, and decide whether to cache the remaining 22 layers (1.85× speedup). **TaylorSeer** goes further by predicting the next output from the trend using Taylor expansion, at zero GPU cost. **SCM** avoids runtime decisions entirely with a pre-defined schedule of which steps to compute and which to cache.

After caching: **~8,600 TFLOPs per image (~3,000 CoD frames, ~50s of gameplay).**

## Parallelism: split work across GPUs

When a single GPU isn't fast enough, you split the work. The most impactful technique for diffusion models is **CFG-Parallel**: since classifier-free guidance requires two independent transformer passes per step (one with the prompt, one without), you can run them on separate GPUs. This halves the wall-clock time per step.

For high-resolution images, the sequence length grows quadratically, making attention the bottleneck. **Ulysses-SP** splits the sequence across GPUs and uses all-to-all communication for attention heads (4 GPUs: 2.84× speedup). **Ring-Attention** takes a different tradeoff: it circulates K/V blocks in a ring, using less memory but achieving slightly lower speedup (4 GPUs: 1.94×).

When the model itself doesn't fit on one GPU, **HSDP** shards the weights across GPUs and gathers them on-demand during the forward pass. It doesn't speed up inference, but it makes large models possible on smaller hardware. Finally, **VAE Patch Parallel** splits the memory-hungry VAE decode step spatially across GPUs for a 4× faster decode.

## Quantization: shrink the numbers

Every weight in a 20B model takes 2 bytes in BF16. That is 40GB just for weights. FP8 quantization rounds each weight to 1 byte, cutting memory in half. But the real win is speed: fewer bytes to transfer through the memory bus means faster matrix multiplications. FP8 gives roughly a **1.28× speedup, saving ~3,400 TFLOPs (~20s of CoD).**

Why does rounding work? Most weight values cluster near zero. Eight bits captures the important range. The rare outliers get rounded with minimal accuracy loss. Some layers (especially attention) are more sensitive, so per-layer control lets you quantize what is safe and keep sensitive layers in BF16.

Applied alone, FP8 brings the cost to **~13,400 TFLOPs per image.** Combined with other techniques, the savings compound.

## Combining everything

These techniques stack. Applied together:

| Optimization | TFLOPs/image | CoD frames | Gameplay at 60fps |
|---|---|---|---|
| Baseline (20B, 50 steps, CFG) | 17,200 | ~6,100 | ~102s |
| + FP8 quantization (1.28×) | 13,400 | ~4,800 | ~80s |
| + Caching, skip 50% passes (2×) | 6,700 | ~2,400 | ~40s |
| + Step distillation, 8 steps (6×) | 1,100 | ~400 | ~6.5s |
| + Guidance distillation (2×) | 550 | ~200 | ~3.3s |

From 102 seconds of Call of Duty per image down to 3.3 seconds. A **31× reduction**.

**Important:** these optimizations compound quality loss too. Always validate against a golden set of 200-500 representative prompts after each optimization. Measure FID, CLIP score, and do blind human evaluation on faces, text, and fine details. A small drop at each stage can add up to a noticeable degradation.

## The big picture: where GPU memory goes

The animation below shows how GPU memory is split between model weights, activations, KV cache, and VAE decode, and how each technique targets a different piece. For a deeper dive, see [Where Does GPU Memory Actually Go During Inference?](/2026/where-does-gpu-memory-go-during-inference/)

<div class="anim-scroll"><div class="anim-embed" style="height:700px"><diffusion-accel scene="gpu-memory"></diffusion-accel></div></div>

<a href="/learning-animations/diffusion-accel.html" style="display:inline-block;padding:12px 24px;background:#2ecc71;color:#fff;border-radius:6px;text-decoration:none;font-weight:bold;margin:16px 0;">▶ Launch all 12 Interactive Explainers</a>

Also available: [P90 Latency](/learning-animations/p90.html) · [Step Distillation](/learning-animations/distillation.html) · [Guidance Distillation](/learning-animations/guidance.html) · [GPU Parallelism](/learning-animations/parallelism.html)

## References

1. Black Forest Labs. "[FLUX.2: Frontier Visual Intelligence](https://github.com/black-forest-labs/flux2)", 2025.
2. Black Forest Labs. "[FLUX.2 klein 4B](https://huggingface.co/black-forest-labs/FLUX.2-klein-4B)", 2025.
3. Austin et al. "[How to Think About GPUs](https://jax-ml.github.io/scaling-book/gpus/)", Google DeepMind, 2025.
4. Dieleman. "[The paradox of diffusion distillation](https://sander.ai/2024/02/28/paradox.html)", 2024.
5. vLLM-Omni. "[Diffusion Acceleration](https://docs.vllm.ai/projects/vllm-omni/en/latest/user_guide/diffusion_acceleration/)", 2025.
