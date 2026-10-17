---
title: "Continuous Batching and Scheduling: How LLM Servers Share a GPU"
description: "AI Foundations #28 explains static batching, continuous batching, request scheduling, chunked prefill, head-of-line blocking, fairness, and why server throughput and user latency can pull in opposite directions."
pubDate: 2026-10-03
updatedDate: 2026-10-15
category: "AI Fundamentals"
tags: ["AI fundamentals", "continuous batching", "LLM serving", "scheduling", "vLLM", "latency", "throughput"]
draft: false
featured: false
author: "machamochalatte"
editor: "RAMGPT Editorial Team"
sources:
  - https://arxiv.org/abs/2206.02658
  - https://arxiv.org/abs/2309.06180
  - https://arxiv.org/abs/2403.02310
  - https://docs.vllm.ai/
evidenceStatus: "explainer"
evidenceNote: "Foundations explainer grounded in LLM serving literature. Queue and scheduling examples are illustrative, not measurements of a specific deployment."
---

In [AI Foundations #27](/articles/latency-throughput-prefill-decode-ai-foundations-27/), we separated LLM speed into prefill, time to first token, decode and total throughput.

Now we can ask the next operational question:

> If many users share one GPU, who gets to run next?

That is a **scheduling** problem.

A modern LLM server is not simply a model waiting for one prompt at a time. It continuously decides which requests should use limited compute and memory.

## Why one request at a time wastes capacity

Imagine three users arrive:

~~~text
A: long prompt, short answer
B: short prompt, long answer
C: short prompt, short answer
~~~

A naive server can process A completely, then B, then C.

That is simple, but C may wait behind much larger jobs and decode can leave the GPU under-utilized.

This is **head-of-line blocking**: a large request at the front delays smaller requests behind it.

## Static batching helps, but synchronizes unlike jobs

Traditional batching groups several inputs together:

~~~text
batch = [A, B, C, D]
~~~

The GPU processes them in parallel.

That works well when jobs have similar shapes and finish at similar times. LLM generation is awkward because sequences have different prompt lengths and different output lengths.

If one request generates 30 tokens and another generates 800, a rigid batch can keep resources tied to the long request after the short request is already done.

## Continuous batching changes the unit of scheduling

Continuous batching lets requests enter and leave the active set over time:

![Continuous batch scheduling replaces finished requests C and A with D and E across successive GPU iterations.](/images/diagrams/continuous-batch-slot-turnover.svg)

*C finishes and D enters; later A finishes and E enters while B remains active. The slots are illustrative, not a fixed hardware limit.*

The batch is not permanent.

The scheduler repeatedly fills available execution slots with work from requests that are ready. This is one reason serving engines can achieve much higher aggregate utilization than a simple request-by-request loop.

## Decode gives frequent scheduling points

Autoregressive decode produces one next-token step at a time.

That gives the server frequent opportunities to choose work:

~~~text
GPU iteration
-> choose active sequences
-> run token positions
-> update state
-> repeat
~~~

The implementation can be more sophisticated, but this captures the basic opportunity.

## Memory is part of the scheduler

The scheduler cannot consider compute alone.

Every active sequence carries state, especially KV cache for transformer attention.

So admitting another request may require more memory.

This connects scheduling to the paged KV-cache ideas popularized by vLLM's PagedAttention work. Managing cache memory in smaller blocks reduces fragmentation and makes it easier to keep many sequences active.

## Throughput and latency can conflict

A larger active batch may improve total GPU utilization:

~~~text
more active sequences
-> more parallel work
-> more total tokens per second
~~~

But an individual request may wait longer or receive less frequent service.

That can increase TTFT, TPOT or end-to-end latency.

A scheduler therefore does not optimize one universal “speed” number. It balances objectives.

## Prefill is a scheduling challenge too

Prefill processes the input prompt.

A very long prompt can require a large burst of compute. If the runtime executes that whole prefill while other users are already decoding, their token streams may stall.

This is one reason serving systems study **chunked prefill**.

Instead of treating a 64K-token prompt as one indivisible job, the server can break prefill into chunks and interleave those chunks with ongoing decode work.

## Chunked prefill is a trade

A simplified schedule might be:

~~~text
decode existing users
-> prefill chunk for new long request
-> decode existing users
-> next prefill chunk
-> repeat
~~~

That can protect ongoing decode latency.

But the new request may take longer to complete its full prefill.

Production serving repeatedly makes this kind of trade.

## Fairness matters

Imagine one user submits an enormous prompt while twenty users submit small chat requests.

A pure throughput optimizer might choose whichever combination keeps the GPU most efficient. That may starve some requests.

A production scheduler can therefore care about:

- arrival time;
- request age;
- service-level targets;
- prompt length;
- output budget;
- current KV-cache occupancy;
- tenant priority;
- fairness.

This starts to resemble operating-system scheduling because conceptually it is another resource-allocation problem.

## Preemption may be necessary

Sometimes the server has admitted more work than it can keep resident.

A runtime may need to pause, evict, swap or recompute some request state. That is **preemption**.

The mechanism depends on the engine.

The important point is that a request can be logically active without receiving GPU time continuously. Client-visible latency can therefore diverge from raw model kernel speed.

## Request rate changes everything

A single-user benchmark asks:

~~~text
How fast can this model run?
~~~

A server benchmark asks:

~~~text
At what arrival rate does queueing become unacceptable?
~~~

At low arrival rates, almost nobody queues. Near capacity, the server may have excellent hardware utilization while p95 and p99 latency rise sharply. Above sustainable capacity, the queue can grow faster than it drains.

That is a capacity boundary, not merely a tokens/s number.

## Tail latency reveals scheduler pain

Average TTFT can stay reasonable while a small group of requests experiences long waits.

That is why production reports often include:

~~~text
p50 TTFT
p95 TTFT
p99 TTFT
~~~

A scheduling change can improve total throughput while making p99 worse.

Whether that trade is acceptable depends on the application.

## A useful mental model

Think of an LLM server as four interacting layers:

~~~text
request queue
-> scheduler
-> memory manager
-> model execution
~~~

The model determines what computation is necessary.

The scheduler determines whose computation runs now.

The memory manager determines which request state can stay resident.

The hardware determines how quickly the selected work executes.

You need all four to explain production performance.

## The cumulative Foundations map

The sequence now extends from model internals into service architecture:

~~~text
parameters
-> tensors
-> forward pass
-> training
-> embeddings and attention
-> Transformer
-> pretraining and fine-tuning
-> inference and sampling
-> context / KV cache
-> quantization and MoE
-> multimodal and reasoning
-> evaluation and retrieval
-> tools and agents
-> alignment
-> monitoring
-> latency / throughput
-> continuous batching / scheduling
~~~

The later lessons do not replace the earlier ones. They show how model behavior meets real resource constraints.

## The main takeaway

Continuous batching is not merely “put more prompts into one batch.”

It is a dynamic scheduling system:

~~~text
requests arrive at different times
-> prompts and outputs have different lengths
-> the server repeatedly chooses active work
-> memory limits how many sequences can stay active
-> batching improves utilization
-> queueing and fairness affect user latency
~~~

Once multiple users share a GPU, model speed becomes only one part of system performance.

The scheduler becomes part of the product.
