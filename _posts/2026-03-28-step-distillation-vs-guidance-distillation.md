---
layout: post
title: "Step Distillation vs Guidance Distillation: Making Diffusion Inference Up to 24× Faster"
comments: true
description: "When you can't afford 50 transformer passes per image. A practical guide to choosing between step distillation (6-12x) and guidance distillation (2x), no special dataset needed."
keywords: "diffusion models distillation progressive distillation guidance distillation classifier-free guidance CFG inference optimization latency"
---

{% include animations.html %}

Diffusion models run the transformer **50-100 times** per image. Two distillation techniques cut this down, but they work differently, cost differently to train, and break differently. Here's how to choose.

Neither technique requires a special dataset. The teacher model generates the training signal. You can use the original training data, or even random noise as input.

A good example is [FLUX.2 klein 4B](https://huggingface.co/black-forest-labs/FLUX.2-klein-4B) from Black Forest Labs. It is a 4B parameter rectified flow transformer that already ships as a distilled model: 4 inference steps, sub-second generation, ~13GB VRAM. It uses both step distillation (reduced from 50 to 4 steps) and guidance-free generation (trained with guidance scale 1.0, so no CFG overhead at all). This is what the end result of applying both techniques looks like in practice.

*For background on where GPU memory goes during inference, see [Where Does GPU Memory Actually Go?](/2026/where-does-gpu-memory-go-during-inference/)*

## Step Distillation: Fewer Steps

**What:** Train a student where 1 step = 2 teacher steps. Repeat: 50 → 25 → 12 → 8 → 4.

**Why it works:** Each round only matches 2 consecutive steps, not the full 50-step trajectory. The student is initialized from the teacher's weights, so it converges fast.

**The sweet spot:** 4-8 steps. Below 4, faces get soft and textures lose crispness. At 1 step, you need adversarial losses (like SDXL Turbo) to recover sharpness. This reduces output diversity.

**Training cost:** Multiple rounds, each requiring a dataset + GPU hours. But you pay once; the distilled model serves forever.

<div class="anim-scroll"><div class="anim-embed"><step-distillation></step-distillation></div></div>

**Pitfalls:**
- Each halving round accumulates error, so quality degrades progressively
- v-prediction parameterization is needed (standard ε-prediction breaks at high noise levels with few steps)
- Reducing steps weakens classifier-free guidance because the effect relies on repeated small adjustments

## Guidance Distillation: Cheaper Steps

**What:** Train a student to produce the guided output in 1 pass instead of 2.

**Why 2 passes exist:** CFG subtracts the unconditional output from the conditional output to isolate the prompt's contribution, then amplifies it 7×. Both passes take the *current* noisy image as input, so neither can be precomputed.

<div class="anim-scroll"><div class="anim-embed"><diffusion-accel scene="cfg-parallel"></diffusion-accel></div></div>

**Why it works:** The student learns to directly predict the amplified result, conditioned on the guidance scale. One forward pass replaces two.

**Training cost:** Single round, simpler task than step distillation. The student can still vary guidance strength at inference time.

**Pitfalls:**
- Only helps if you use CFG (most text-to-image models do)
- Must be applied **before** step distillation. Otherwise, the guidance effect is already weakened by fewer steps
- May not perfectly capture guidance at all noise levels, causing subtle prompt adherence issues

## Decision Guide

| Scenario | Recommendation | Speedup |
|----------|---------------|---------|
| Ship fast, minimal training budget | Guidance distillation | 2× |
| Maximum throughput on limited hardware | Step distillation to 4-8 steps | 6-12× |
| Real-time generation (< 100ms) | Both (guidance first, then steps) + FP8 | ~24× |
| Can't retrain at all | [Caching](/2026/where-does-gpu-memory-go-during-inference/#what-each-acceleration-technique-reduces) (TeaCache, DBCache) | ~2× |

## The Third Option: No Training Required

If retraining isn't an option, **caching** skips transformer passes at runtime by detecting when consecutive steps produce nearly identical outputs. Zero training cost, ~2× speedup.

<div class="anim-scroll"><div class="anim-embed"><diffusion-accel scene="teacache"></diffusion-accel></div></div>

Caching is orthogonal to distillation. You can stack all three.

## References

1. Salimans, Ho. "[Progressive Distillation for Fast Sampling of Diffusion Models](https://arxiv.org/abs/2202.00512)", ICLR 2022.
2. Meng et al. "[On Distillation of Guided Diffusion Models](https://arxiv.org/abs/2210.03142)", CVPR 2023.
3. Ho, Salimans. "[Classifier-Free Diffusion Guidance](https://arxiv.org/abs/2207.12598)", NeurIPS 2021.
4. Dieleman. "[The paradox of diffusion distillation](https://sander.ai/2024/02/28/paradox.html)", 2024.
5. Sauer et al. "[Adversarial Diffusion Distillation](https://arxiv.org/abs/2311.17042)", 2023.
