---
layout: post
title: "Step Distillation vs Guidance Distillation: Two Ways to Make Diffusion 2× Faster"
comments: true
description: "When you can't afford 50 transformer passes per image — a practical guide to choosing between step distillation and guidance distillation."
keywords: "diffusion models distillation progressive distillation guidance distillation classifier-free guidance CFG inference optimization latency"
---

You've got a diffusion model that generates beautiful images. There's just one problem: it takes **50 transformer passes** to produce a single image. On your hardware, that's seconds — not milliseconds. Your users won't wait, and your GPU bill is already too high.

Two distillation techniques can help. They attack the problem from different angles, and choosing the wrong one wastes your limited training budget. Here's how to decide.

## Why It's So Expensive

Each denoising step runs the full transformer (24+ layers, billions of FLOPs) to predict a direction, then takes a tiny step in that direction. Cheap math, expensive prediction.

If you're using classifier-free guidance (CFG) — and you almost certainly are, since it's what makes prompts actually work — the transformer runs **twice per step**: once with the prompt, once without. The difference between the two outputs is what makes images vivid instead of bland.

**50 steps × 2 passes = 100 transformer passes per image.**

On a single GPU, that's the difference between serving 10 users and serving 100.

<a href="https://vetional.github.io/learning-animations/diffusion-accel.html" style="display:inline-block;padding:8px 16px;background:#4a90d9;color:#fff;border-radius:4px;text-decoration:none;font-size:0.9em;">▶ See the interactive "Why Cache?" animation</a>

## Option 1: Step Distillation — Cut the Number of Steps

**What it does:** Trains a student model where 1 step = 2 teacher steps. Apply repeatedly: 50 → 25 → 12 → 8 → 4.

**Why you'd pick this:** You need the biggest possible speedup. Going from 50 steps to 8 gives you a **6× speedup** with barely noticeable quality loss. Push to 4 steps for **12×** if you can tolerate slight softness in fine details.

**The cost:** Each halving round requires a training run. You need the original teacher model's weights, a training dataset, and GPU hours for distillation. For a large model, each round might take days on a multi-GPU setup. But you only pay this cost once — the distilled model serves forever.

**The catch at low step counts:** Below 4 steps, quality drops noticeably. The model is trying to do in one shot what originally took 50 careful corrections. Faces get soft, textures lose crispness. At 1 step, you'll likely need an adversarial loss (like SDXL Turbo uses) to recover sharpness — which adds training complexity and can reduce output diversity.

**Sweet spot for resource-constrained deployment: 4-8 steps.** This is where you get the most speedup per unit of quality loss.

<a href="https://vetional.github.io/learning-animations/distillation.html" style="display:inline-block;padding:8px 16px;background:#4a90d9;color:#fff;border-radius:4px;text-decoration:none;font-size:0.9em;">▶ See the interactive Step Distillation animation</a>

## Option 2: Guidance Distillation — Make Each Step Cheaper

**What it does:** Trains a student to produce the guided output (the result of the 2-pass CFG subtraction) in a single forward pass.

**Why you'd pick this:** You want a **safe, predictable 2× speedup** without touching the step count. Quality stays almost identical because you're not skipping any steps — you're just making each one half as expensive. If your latency budget is tight but not desperate, this is the lower-risk option.

**The cost:** One training run, typically shorter than step distillation since the task is simpler (matching one combined output, not collapsing two sequential steps). The student is conditioned on the guidance scale, so you can still tune prompt strength at inference time.

**The catch:** Only helps if you're using CFG. If you're not (rare for text-to-image, but possible for unconditional generation), this technique doesn't apply. Also, reducing steps later will weaken the guidance effect — so if you plan to combine with step distillation, do guidance distillation **first**.

**Why "first" matters:** CFG works by making small adjustments to the predicted direction at every step. Fewer steps = fewer adjustments = weaker guidance. If you distill guidance into the model before reducing steps, the effect is baked in and survives the step reduction.

<a href="https://vetional.github.io/learning-animations/guidance.html" style="display:inline-block;padding:8px 16px;background:#4a90d9;color:#fff;border-radius:4px;text-decoration:none;font-size:0.9em;">▶ See the interactive Guidance Distillation animation</a>

## Decision Guide

**"I need to ship fast with minimal training budget"**
→ Guidance distillation. One training run, 2× speedup, low risk.

**"I need maximum throughput on limited hardware"**
→ Step distillation to 4-8 steps. Bigger speedup, but requires more training rounds.

**"I need real-time generation (< 100ms)"**
→ Both. Guidance distillation first, then step distillation to 4 steps. Combined: up to **24× speedup** (2× from guidance × 12× from steps). You'll also want FP8 quantization for another 1.3× on top.

**"I can't afford any retraining"**
→ Neither distillation technique works without training. Look at **caching** instead (TeaCache, DBCache) — these skip redundant transformer passes at runtime with zero training cost. Roughly 2× speedup for free.

## Head-to-Head

| | Step Distillation | Guidance Distillation |
|---|---|---|
| **Speedup** | 6-12× | 2× |
| **Training cost** | Multiple rounds | Single round |
| **Quality risk** | Noticeable below 4 steps | Minimal |
| **Works without CFG?** | Yes | No |
| **Combine with other?** | Yes (apply after guidance distillation) | Yes (apply before step distillation) |
| **Best for** | Max throughput | Safe, predictable speedup |

## The Third Option: Caching (No Training Required)

If retraining isn't an option, caching techniques skip transformer passes at runtime by detecting when consecutive steps produce nearly identical outputs. The transformer predicts almost the same direction at step 45 and step 46 — so why recompute it?

Techniques like TeaCache measure the L1 distance between consecutive outputs and reuse the cached result when the difference is below a threshold. DBCache goes further: run only the first 2 transformer layers to decide whether to cache the remaining 22. TaylorSeer predicts the next output from the trend using Taylor expansion — zero GPU cost.

These are orthogonal to distillation. You can stack all three: guidance distillation + step distillation + caching.

<a href="https://vetional.github.io/learning-animations/diffusion-accel.html" style="display:inline-block;padding:8px 16px;background:#2ecc71;color:#fff;border-radius:4px;text-decoration:none;font-size:0.9em;">▶ Explore all 12 Diffusion Acceleration animations</a>

## References

1. Salimans, Ho. "[Progressive Distillation for Fast Sampling of Diffusion Models](https://arxiv.org/abs/2202.00512)", ICLR 2022.
2. Meng, Rombach, Gao, Kingma, Ermon, Ho, Salimans. "[On Distillation of Guided Diffusion Models](https://arxiv.org/abs/2210.03142)", CVPR 2023.
3. Ho, Salimans. "[Classifier-Free Diffusion Guidance](https://arxiv.org/abs/2207.12598)", NeurIPS 2021.
4. Dieleman, Sander. "[The paradox of diffusion distillation](https://sander.ai/2024/02/28/paradox.html)", 2024.
5. Song, Dhariwal, Chen, Sutskever. "[Consistency Models](https://arxiv.org/abs/2303.01469)", ICML 2023.
6. Sauer, Lorenz, Blattmann, Rombach. "[Adversarial Diffusion Distillation](https://arxiv.org/abs/2311.17042)", 2023.
7. Liu, Gong, Liu. "[Flow Straight and Fast: Learning to Generate and Transfer Data with Rectified Flow](https://arxiv.org/abs/2209.03003)", ICLR 2023.
