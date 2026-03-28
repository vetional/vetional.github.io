---
layout: post
title: "Step Distillation vs Guidance Distillation: Two Ways to Make Diffusion 2× Faster"
comments: true
description: "A visual comparison of two distillation techniques that speed up diffusion model inference — one reduces the number of steps, the other makes each step cheaper."
keywords: "diffusion models distillation progressive distillation guidance distillation classifier-free guidance CFG inference optimization"
---

Diffusion models generate images by running a transformer **50 times** — once per denoising step. Each step predicts a direction to remove noise, then moves the image slightly in that direction. The result is stunning quality, but painfully slow inference.

Two distillation techniques attack this problem from different angles. Understanding when to use which is key to deploying diffusion models efficiently.

## The Problem: Why Diffusion Is Slow

At each denoising step, two things happen:

1. **The transformer predicts a direction** — "which way should I move to remove noise?" This is expensive: 24+ layers, billions of FLOPs.
2. **The scheduler moves the image** — apply the predicted direction to take one small step. This is cheap: simple math.

The cost is dominated by the transformer. And if you're using classifier-free guidance (CFG), the transformer runs **twice per step** — once with the prompt, once without — doubling the cost.

**50 steps × 1-2 transformer passes per step = 50-100 transformer passes per image.**

Both distillation techniques reduce this number, but in fundamentally different ways.

<a href="https://vetional.github.io/learning-animations/diffusion-accel.html" style="display:inline-block;padding:8px 16px;background:#4a90d9;color:#fff;border-radius:4px;text-decoration:none;font-size:0.9em;">▶ See the interactive "Why Cache?" animation</a>

## Step Distillation: Fewer Steps

**Core idea:** Train a student model to take one step that equals two steps of the teacher.

The teacher model generates images in 50 steps. We clone it as a student, then train the student so that its single prediction matches the result of two consecutive teacher predictions. After training, the student needs only 25 steps for the same quality.

The clever part: this process is **progressive**. We can repeat it — distilling the 25-step student into a 12-step student, then into 8, then 4, then 2, then 1. Each round halves the step count.

### How It Works

1. Start with a trained teacher model (50 steps)
2. Clone it as a student
3. For each training example:
   - Run the teacher for 2 steps from a noisy image
   - Train the student to match that result in 1 step
4. The student now needs 25 steps
5. Repeat: the 25-step model becomes the new teacher

This is called **progressive distillation**, introduced by Salimans & Ho (2022). The key insight is that each round only requires matching 2 steps, not the entire 50-step trajectory — making it computationally tractable.

### The Tradeoff

Each halving introduces a small quality loss. Going from 50 to 8 steps is nearly lossless. Going to 4 steps shows minor degradation. Going to 1 step produces noticeably blurrier results — the model is trying to do in one shot what originally took 50 careful steps.

The sweet spot for most applications is **4-8 steps**: a 6-12× speedup with minimal quality loss.

<a href="https://vetional.github.io/learning-animations/distillation.html" style="display:inline-block;padding:8px 16px;background:#4a90d9;color:#fff;border-radius:4px;text-decoration:none;font-size:0.9em;">▶ See the interactive Step Distillation animation</a>

## Guidance Distillation: Cheaper Steps

**Core idea:** Train a student model to produce the guided output in one forward pass instead of two.

Classifier-free guidance (CFG) is what makes prompt-following work. At each step, the model runs twice:

1. **With the prompt** → produces a vague, safe image
2. **Without the prompt** → produces generic noise removal

The difference between these two outputs isolates what the prompt contributes. Amplifying that difference (typically 7×) produces vivid, prompt-faithful images.

The problem: both passes take the **current noisy image** as input, so neither can be precomputed. Every step costs 2× the compute.

### How It Works

Guidance distillation trains a student to directly predict the guided output — the result of the subtraction and amplification — in a single forward pass:

1. Run the teacher with CFG (2 passes) to get the guided prediction
2. Train the student to match that guided prediction in 1 pass
3. The student is conditioned on the guidance scale, so it can produce different strengths of guidance

After distillation, each step costs half as much. The student doesn't need to run the unconditional pass at all — it has internalized the effect of guidance.

### The Tradeoff

Guidance distillation preserves quality well because it doesn't change the number of steps — it just makes each step cheaper. The main risk is that the student may not perfectly capture the guidance effect at all noise levels, leading to subtle differences in prompt adherence.

This technique is especially valuable as a **preprocessing step before step distillation**. Reducing the number of steps weakens the effect of guidance (since guidance relies on repeated small adjustments). Distilling guidance first ensures the effect is preserved when steps are later reduced.

<a href="https://vetional.github.io/learning-animations/guidance.html" style="display:inline-block;padding:8px 16px;background:#4a90d9;color:#fff;border-radius:4px;text-decoration:none;font-size:0.9em;">▶ See the interactive Guidance Distillation animation</a>

## Head-to-Head Comparison

| | Step Distillation | Guidance Distillation |
|---|---|---|
| **What it reduces** | Number of steps (50 → 4-8) | Cost per step (2 passes → 1) |
| **Speedup** | 6-12× | 2× |
| **Quality impact** | Slight blur at low step counts | Minimal |
| **Diversity impact** | Reduced at very low steps | Preserved |
| **Works without CFG?** | Yes | No (only relevant with CFG) |
| **Can be combined?** | Yes — apply guidance distillation first, then step distillation |

## When to Use Which

**Use step distillation when:**
- You need maximum speedup (4-12×)
- You can tolerate minor quality loss
- You're deploying to latency-sensitive applications (real-time generation)

**Use guidance distillation when:**
- You use classifier-free guidance (most text-to-image models do)
- You want a safe 2× speedup with minimal quality risk
- You plan to also apply step distillation afterward

**Use both together when:**
- You want the best of both worlds
- Apply guidance distillation first (preserves guidance effect)
- Then apply step distillation (reduces steps)
- Combined speedup: up to 24× (2× from guidance × 12× from steps)

## Beyond Distillation: Caching

There's a third approach that doesn't require any retraining: **caching**. Instead of distilling a new model, caching techniques like TeaCache, DBCache, and TaylorSeer skip transformer passes at runtime by reusing previous outputs when consecutive steps produce nearly identical results.

Caching is orthogonal to distillation — you can combine all three approaches for maximum speedup.

<a href="https://vetional.github.io/learning-animations/diffusion-accel.html" style="display:inline-block;padding:8px 16px;background:#2ecc71;color:#fff;border-radius:4px;text-decoration:none;font-size:0.9em;">▶ Explore all 12 Diffusion Acceleration animations</a>

## References

1. Salimans, Ho. "[Progressive Distillation for Fast Sampling of Diffusion Models](https://arxiv.org/abs/2202.00512)", ICLR 2022.
2. Meng, Rombach, Gao, Kingma, Ermon, Ho, Salimans. "[On Distillation of Guided Diffusion Models](https://arxiv.org/abs/2210.03142)", CVPR 2023.
3. Ho, Salimans. "[Classifier-Free Diffusion Guidance](https://arxiv.org/abs/2207.12598)", NeurIPS 2021.
4. Dieleman, Sander. "[The paradox of diffusion distillation](https://sander.ai/2024/02/28/paradox.html)", 2024.
5. Song, Dhariwal, Chen, Sutskever. "[Consistency Models](https://arxiv.org/abs/2303.01469)", ICML 2023.
6. Sauer, Lorenz, Blattmann, Rombach. "[Adversarial Diffusion Distillation](https://arxiv.org/abs/2311.17042)", 2023.
7. Liu, Gong, Liu. "[Flow Straight and Fast: Learning to Generate and Transfer Data with Rectified Flow](https://arxiv.org/abs/2209.03003)", ICLR 2023.
