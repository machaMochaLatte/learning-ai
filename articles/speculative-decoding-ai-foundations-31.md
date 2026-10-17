---
title: "Speculative Decoding: Why a Draft Model Can Speed Up a Larger LLM"
description: "AI Foundations #31 explains draft-and-verify generation, acceptance and rejection, why the target distribution can be preserved, and when speculative decoding actually reduces latency."
pubDate: 2026-10-06
updatedDate: 2026-10-15
category: "AI Fundamentals"
tags: ["AI fundamentals", "speculative decoding", "LLM inference", "draft model", "latency", "vLLM", "serving"]
draft: false
featured: false
author: "machamochalatte"
editor: "RAMGPT Editorial Team"
sources:
  - https://arxiv.org/abs/2211.17192
  - https://docs.vllm.ai/en/latest/features/speculative_decoding/index.html
evidenceStatus: "explainer"
evidenceNote: "Conceptual explainer grounded in the original speculative-decoding paper and current vLLM documentation. Examples are simplified, not benchmark results."
---

In [AI Foundations #30](/articles/paged-attention-ai-foundations-30/), we looked at a memory problem: how an LLM server stores growing KV caches efficiently.

Now consider a latency problem.

Ordinary autoregressive generation produces tokens serially:

~~~text
run target model
-> token 1
run target model
-> token 2
run target model
-> token 3
~~~

Speculative decoding asks:

> What if something cheaper guesses several likely next tokens, and the large target model checks them together?

That is the core idea.

## Draft and target

A basic system has two roles.

**Draft:** a smaller or cheaper predictor proposes several future tokens quickly.

**Target:** the model whose output distribution we actually want.

Suppose the text ends with:

~~~text
The capital of France is
~~~

The draft might propose:

~~~text
Paris . It is
~~~

Instead of asking the target model four times serially, the runtime can evaluate the proposed continuation in a more parallel verification pass.

If several draft tokens are accepted, the target advances multiple positions in one speculative round.

## The target remains in charge

A common oversimplification is:

> the small model writes and the big model approves.

The original speculative-sampling method is more precise.

The draft is only a proposal mechanism. The target probabilities determine acceptance, and a correction distribution is used when a proposal is rejected.

With the proper rule, sampling can preserve the target model's distribution instead of silently switching to the draft model's distribution.

## A simplified round

Suppose the draft proposes:

~~~text
d1 d2 d3 d4
~~~

Verification might produce:

~~~text
d1 accepted
d2 accepted
d3 accepted
d4 rejected
~~~

The runtime samples a corrected token at the rejection point and starts another round.

The useful outcome was three accepted target tokens from one verification round instead of three separate serial target decode iterations.

![Speculative decoding round: four proposed tokens, three accepted by target verification, one rejected and replaced through correction sampling.](/images/diagrams/speculation-draft-verification.svg)

*For sampled decoding, target-dependent acceptance and correction preserve the intended distribution; the drawn accept/reject outcome is an example.*

## Why verification can be cheaper

Transformers are good at processing multiple known token positions in parallel.

Autoregressive decode cannot fully exploit that because token N+1 depends on token N.

The draft temporarily breaks the dependency by guessing future tokens. The target can then score those known proposed positions together.

The bet is profitable when:

~~~text
draft cost
+ verification cost
<
serial target cost for accepted progress
~~~

That makes acceptance rate central.

## Acceptance rate controls the economics

If the draft closely matches the target, many tokens survive each round.

If the draft disagrees early, most speculative work is discarded.

So speculative decoding is not automatically faster. Its value depends on:

- draft speed;
- target speed;
- number of proposed tokens;
- acceptance rate;
- batch size;
- hardware utilization;
- verification implementation;
- memory overhead.

The useful metric is **accepted target progress per verification round**, not simply the number of draft tokens proposed.

## The smallest draft is not always best

A tiny draft can be very fast but disagree frequently with the target.

A larger draft can agree more often but consume too much compute.

So there is a trade-off:

~~~text
tiny draft
-> low proposal cost
-> possibly low acceptance

larger draft
-> higher acceptance
-> higher proposal cost
~~~

The best point depends on the model pair and workload.

## The proposer does not have to be another full LLM

Current vLLM documentation lists several speculative methods, including:

- EAGLE-style proposers;
- multi-token prediction;
- separate draft models;
- parallel draft models;
- n-gram methods;
- suffix decoding.

The proposer changes, but the high-level structure remains:

~~~text
propose
-> verify
-> accept useful prefix
-> correct at first failure
-> repeat
~~~

## Multi-token prediction is closely related

Some models include heads or modules trained to predict several future tokens.

Those predictions can become speculative candidates without loading a completely separate draft model.

That can reduce proposer overhead and improve alignment with the target, although the runtime still needs an efficient verification path.

## What speculation is trying to reduce

The main target is serial inter-token latency.

vLLM documentation frames speculative decoding as particularly useful for reducing inter-token latency in suitable medium-to-low-QPS, memory-bound workloads.

At high server load, the result can change. Normal continuous batching may already keep the accelerator busy, while the draft consumes capacity that could have served additional target requests.

Latency and maximum aggregate throughput are different objectives.

## Why high concurrency can shrink the advantage

At low concurrency, one large model decoding one token can underuse the GPU.

Speculation widens the verification work.

At high concurrency, the server already has token positions from many requests to batch together. Extra draft compute can become overhead rather than useful parallelism.

A method can therefore improve single-user latency while failing to improve saturated-server throughput.

## Memory is part of the trade-off

A separate draft model needs weights and runtime state.

That memory could otherwise support a larger target model, more KV cache, longer contexts, larger batches or more replicas.

So the correct comparison is whole-system performance, not draft-model speed in isolation.

## Speculation length has a sweet spot

Proposing very few tokens leaves little opportunity to skip target iterations.

Proposing many tokens increases theoretical progress, but an early rejection wastes more draft work.

If a proposal had an independent per-token acceptance probability p, the chance that k tokens all survive would roughly shrink like:

~~~text
p raised to k
~~~

Real acceptance is not independent, but the intuition explains why longer speculation is not always better.

## Greedy decoding is the simplest mental model

For deterministic greedy output, the idea is intuitive:

~~~text
draft guesses
target checks
keep matching prefix
~~~

Sampling needs the proper acceptance and correction rule because several tokens can be valid probabilistic outcomes.

That distribution-preservation property is what makes speculative decoding more than a heuristic shortcut.

## Benchmark it correctly

Keep the target model and request set fixed.

Compare:

~~~text
baseline target only
vs
target plus speculation
~~~

Record:

- TTFT;
- inter-token latency;
- output throughput;
- request throughput;
- acceptance rate;
- accepted tokens per speculative round;
- GPU memory;
- concurrency.

A speedup without acceptance data is hard to interpret.

## Main failure modes

Speculation can disappoint when:

**The draft disagrees too often.** Too many proposals are discarded.

**The draft is too expensive.** Saved target iterations are replaced by nearly equal proposal cost.

**Verification is inefficient.** Multi-token verification does not exploit enough parallelism.

**The server is already saturated.** Extra speculative work competes with normal batching.

**Memory pressure reduces batch capacity.** The proposer consumes memory the target needed for KV cache or concurrency.

## Mental model to keep

Speculative decoding does not make the target model smaller.

It tries to reduce the number of **serial target decode iterations** required for the same accepted progress.

The core equation is:

~~~text
cheap accurate guesses
+ efficient parallel verification
= fewer expensive serial target steps
~~~

That is why speculative decoding can be a major latency optimization in one workload and nearly useless in another.
