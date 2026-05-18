---
author: Ivaylo Bahtchevanov
pubDatetime: 2026-04-20T10:00:00Z
title: The shape of your reward function matters more than you think
slug: reward-shape-matters-alphapo
featured: true
draft: false
tags: []
description: When we set out to improve direct alignment algorithms like DPO and SimPO, we expected better data, smarter regularization, more careful sweeps. What we found instead was the shape of the reward function itself — a single parameter that controls how probability mass moves during training, and outperforms the strongest existing DAA baselines.
---

When we set out to improve direct alignment algorithms (DAAs) like DPO and SimPO, we expected to find the usual suspects — better data, smarter regularization, more careful hyperparameter sweeps. What we found instead was a more fundamental lever: the shape of the reward function itself.

## The problem we kept running into

DAAs work by collapsing reward modeling into the policy objective, sidestepping one of the most fragile stages of RLHF. But anyone who has trained DPO or SimPO at scale has watched the same uncomfortable pattern: the probability of the preferred response — the one you're explicitly training the model to favor — often drops as training proceeds. This is *likelihood displacement*, and its close cousin *reward over-optimization* is just as common. Both quietly degrade the very models you think you're improving.

The standard response is early stopping. Pick a good checkpoint, ship it, move on. But this is a workaround, not an answer. It tells us the optimization is misbehaving and we've learned to live with it.

## Our claim: it's the shape, not the slope

The standard DAA reward is a log-probability ratio. We asked a simple question — what if `log` isn't the right shape?

AlphaPO introduces a single parameter, α, that reshapes the reward beyond the log form (formally, via an α-divergence parameterization). Push α one way and you get aggressive likelihood updates — fast learning, higher displacement risk. Push it the other way and the updates stay conservative — slower margin growth, but preferred-response probabilities hold up.

Our gradient analysis shows that shape governs how probability mass moves during training, not just how much it moves. That's the design axis the existing DAA work hasn't separated cleanly.

## What we found

Training Mistral-7B-Instruct and Llama3-8B-Instruct on UltraFeedback:

- **7–10% relative improvement over SimPO**, the strongest DAA baseline.
- **15–50% relative improvement over DPO** on the same models.
- **No length-hacking** — gains aren't coming from longer or more verbose outputs, which is the usual confound in this space.

Tuning α also gives finer-grained control over the displacement / over-optimization trade-off than any existing DAA exposes.

## Why this matters

**Most of the field has been searching in the wrong design space.** New DAAs keep proposing new losses. We're showing that even within a single loss family, reward shape is an almost-free design axis that hasn't been seriously explored.

**It's a drop-in change.** AlphaPO adds one hyperparameter to SimPO. No new data, no new model, no new infra. If you're already running SimPO in production, this is a few lines of code.

**It gives you a dial, not a dogma.** Different alignment goals — safety vs. helpfulness, conservative vs. exploratory generation — likely call for different displacement dynamics. α gives you something to actually tune.

We don't think reward shape is the last word on alignment. But it's the most underexplored knob in the current DAA toolkit, and the one most likely to repay attention.