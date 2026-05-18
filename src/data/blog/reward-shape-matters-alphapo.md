---
author: Ivaylo Bahtchevanov
pubDatetime: 2026-04-20T10:00:00Z
title: The shape of your reward function matters more than you think
slug: reward-shape-matters-alphapo
featured: true
draft: false
tags: []
description: When we set out to improve direct alignment algorithms like DPO and SimPO, we expected better data, smarter regularization, more careful sweeps. What we found instead was the shape of the reward function itself — a single parameter that controls how probability mass moves during training, and outperforms strong DAA baselines on Mistral-7B and Llama3-8B.
---

When we set out to improve direct alignment algorithms (DAAs) like DPO and SimPO, we expected to find the usual suspects — better data, smarter regularization, more careful hyperparameter sweeps. What we ended up changing was the shape of the reward function itself.

## The problem we kept running into

DAAs simplify alignment by characterizing the reward directly as a function of the policy being learned, bypassing the reward modeling stage of RLHF. But anyone who has trained DPO or SimPO at scale has watched the same pattern: the probability of the preferred response — the one you're explicitly training the model to favor — often drops as training proceeds. This is *likelihood displacement*. Its close cousin, *reward over-optimization*, is just as common: the margin between preferred and dispreferred responses widens, but generalization gets worse.

The standard workaround is early stopping. SimPO uses one epoch of training and a careful sweep over learning rate, β (reward scaling), and γ (margin shift) to pick a strong checkpoint. It works, but it's a workaround. The optimization is misbehaving along the trajectory and we've learned to live with it.

## Our claim: it's the shape, not the slope

Most DAAs, including SimPO, use a log-probability reward. We asked what happens if `log` isn't the right shape.

The approach introduces a fourth hyperparameter — α — that reshapes the reward beyond log via an α-divergence parameterization, with length normalization applied:

```
r_α(y; x) = β · [1 − π_θ(y|x)^(−α/|y|)] / α
```

At α → 0, this reduces to SimPO's log reward. At α = 1 and α = −1, the reward becomes inverse-linear and linear respectively. The length normalization is load-bearing: f-DPO previously explored α-divergence rewards without length normalization and found no value beyond α = 0 (standard DPO). With length normalization, non-zero α produces real generalization gains.

The resulting loss, under the margin-based Bradley-Terry model, is:

```
L_AlphaPO = −E_(x,y_w,y_l) [ log σ(  −(β/α)·π_θ(y_w|x)^(−α/|y_w|)
                                    + (β/α)·π_θ(y_l|x)^(−α/|y_l|)
                                    − γ ) ]
```

Same Bradley-Terry structure as SimPO; only the reward shape changes.

### Why the shape changes training dynamics

Our gradient analysis decomposes the magnitude of the per-sample gradient `|∂ℓ/∂v|` into two factors `T₁(α) · T₂(α)`, which behave very differently as functions of α. The product is **non-monotonic**: there is no single direction "more α = more aggressive" or "less α = more conservative."

Specifically (Theorem 3.1 in the paper):

- As **α → −∞**, the gradient vanishes for all samples — strong regularization that slows everything down.
- As **α → +∞** with a positive margin (preferred-likelihood already ahead), the gradient vanishes — selectively regularizes samples that are already in the right order.
- As **α → +∞** with a negative margin (wrong-order pair), the gradient blows up — selectively aggressive on the samples that need correcting.

That asymmetry on positive α is the interesting part. It's not "α controls a single dial labeled aggressiveness." It's "α reshapes how the optimizer allocates effort across samples depending on whether they're already going the right direction."

A companion analysis (Corollary 3.3) of how the preferred-response probability π_w evolves under gradient flow makes this concrete: for samples with a negative margin, choosing α ≥ α₀ guarantees π_w increases over training. For samples with a positive margin, the opposite — α ≤ α₀ keeps π_w going up rather than getting dragged down by the dispreferred-response gradient. Tuning α is tuning which regime dominates.

## What we found

Training on UltraFeedback with PairRM-based on-policy data:

- On **AlpacaEval 2.0 length-controlled win rate**, AlphaPO outperforms SimPO by **7–10% relatively** on Mistral-7B-Instruct and Llama3-8B-Instruct.
- On the same benchmark, AlphaPO outperforms DPO by **15% on Llama3-8B and 50% on Mistral-7B**.
- **A slightly positive α gives the best AlpacaEval 2.0 performance**, with smooth dropoff in either direction (steeper on the negative side, consistent with the more aggressive likelihood displacement at negative α).
- **No length-hacking** — generation lengths don't blow up versus SimPO or DPO. Length is the usual confound in this space, and we ruled it out.
- **KL divergence to the SFT model is comparable to SimPO's**, while LC is higher. The gains aren't coming from drifting further from the reference policy — they're coming from how the trajectory gets there.
- **Gains transfer to non-alignment benchmarks.** On HellaSwag (commonsense reasoning) and TruthfulQA (truthfulness), AlphaPO outperforms SimPO on both Mistral-7B and Llama3-8B. Alignment improvements aren't trading off general capability.
- **Competitive but not strictly better on Gemma2-9B-Instruct** — slight LC improvement, slightly lower or comparable WR. We name this honestly.
- **Composes cleanly with SPPO**: AlphaPO + SPPO reaches **47.42% LC on AlpacaEval 2.0**, improving on SimPO + SPPO (45.06%) without extensive tuning.

Tuning α also gives finer-grained control over the displacement / over-optimization trade-off than any other DAA hyperparameter we tried.

## Why this matters

**Most of the field has been searching in a narrow design space.** New DAAs keep proposing new losses. We're showing that even within a single loss family, reward shape is a separate design axis from learning rate, β, and γ. Adding it gives you a four-dimensional sweep instead of a three-dimensional one.

**It's a drop-in change.** AlphaPO adds one hyperparameter to SimPO. No new data, no new model, no new infra. If you're already running SimPO, this is a few lines of code.

**It separates the lever from the dogma.** Different alignment goals — safety vs. helpfulness, conservative vs. exploratory generation — likely call for different displacement dynamics. The α parameter gives you something to tune for that explicitly, rather than relying on early stopping to land on a checkpoint that happened to balance them.

We don't think reward shape is the last word on alignment. But it's a real fourth dimension in the DAA design space, and the gradient analysis suggests it's controlling something the existing three hyperparameters can't get at on their own.