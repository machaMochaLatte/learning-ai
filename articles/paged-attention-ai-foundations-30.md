---
title: "PagedAttention: How LLM Servers Stop KV Cache From Wasting Memory"
description: "AI Foundations #30 explains why variable-length KV caches fragment GPU memory, how paging divides cache into blocks, how logical token positions map to physical memory, and why this helps serving throughput."
pubDate: 2026-10-05
updatedDate: 2026-10-15
category: "AI Fundamentals"
tags: ["AI fundamentals", "PagedAttention", "KV cache", "LLM serving", "vLLM", "GPU memory", "memory management"]
draft: false
featured: false
author: "machamochalatte"
editor: "RAMGPT Editorial Team"
sources:
  - https://arxiv.org/abs/2309.06180
  - https://docs.vllm.ai/
evidenceStatus: "explainer"
evidenceNote: "Foundations explainer grounded in the PagedAttention paper and vLLM documentation. Memory examples are simplified illustrations, not measured results from a specific GPU."
---

In [AI Foundations #29](/articles/prefix-caching-kv-reuse-ai-foundations-29/), we saw that a server can reuse KV state when requests share the same token prefix.

But there is a more basic memory problem:

> Where should all of those KV-cache tokens live in GPU memory?

Different requests have different prompt lengths, generate different numbers of tokens and finish at different times.

If the server reserves one large contiguous region for every request, memory can be wasted badly.

PagedAttention was designed to attack that problem.

## First remember what KV cache stores

During generation, a transformer repeatedly attends to earlier tokens.

Instead of recomputing key and value tensors for every previous token on every decode step, the runtime stores them in the **KV cache**.

A simplified request might look like:

~~~text
token 1 -> KV
token 2 -> KV
token 3 -> KV
...
token N -> KV
~~~

As the sequence grows, the cache grows.

With many simultaneous requests, KV cache can consume a large fraction of available accelerator memory.

## The problem with reserving one big continuous region

Suppose a server admits three requests.

It does not know in advance exactly how long each output will become.

A simple allocator could reserve:

~~~text
A: [....................]
B: [....................]
C: [....................]
~~~

Each region has enough room for the maximum possible sequence length.

If A stops after 200 tokens but its reservation allowed 4,000, much of that reservation sits unused.

Another problem appears when free memory exists but is broken into separated gaps.

For example:

~~~text
used | free | used | free | used | free
~~~

The total free memory might be large, yet there may be no single contiguous region large enough for a new request.

That is fragmentation.

## Paging changes the unit of allocation

PagedAttention borrows an idea familiar from operating systems: divide memory into fixed-size blocks instead of requiring one giant continuous region.

A sequence is represented logically as a continuous stream of tokens:

~~~text
0 1 2 3 4 5 6 7 8 9 ...
~~~

But its KV data can be stored physically in blocks scattered through available memory:

~~~text
logical block 0 -> physical block 12
logical block 1 -> physical block 4
logical block 2 -> physical block 31
~~~

The model still needs the correct K and V tensors for each token.

The runtime keeps a mapping from logical blocks to physical blocks so the attention kernel can find them.

![A paged KV cache maps logical token blocks zero, one and two to non-adjacent physical blocks 12, 4 and 31 through a block table.](/images/diagrams/paged-kv-block-mapping.svg)

*The block IDs match the illustrative mapping above. Physical KV blocks need not be adjacent, but logical token order is preserved.*

## Why this reduces waste

Instead of reserving the maximum sequence length up front, the runtime can allocate another block only when the sequence actually grows into it.

Imagine blocks that hold four token positions.

A seven-token sequence needs two blocks:

~~~text
block A: tokens 0 1 2 3
block B: tokens 4 5 6 _
~~~

Only the unused portion of the final block is wasted.

That is much smaller than reserving thousands of unused token slots for every active request.

## Physical blocks do not need to be adjacent

This is the crucial point.

A request might use:

~~~text
physical block 8
physical block 19
physical block 3
~~~

Those blocks are not adjacent in device memory.

The logical block table makes them appear like one continuous sequence to the attention operation.

That means freed blocks can be reused by other requests without first compacting all active KV data into a single continuous region.

## Requests can grow one block at a time

Generation is naturally incremental:

~~~text
prompt
-> next token
-> next token
-> next token
...
~~~

Paged allocation fits that behavior.

As a request grows, the server assigns additional KV blocks.

When the request finishes, those blocks can return to the free pool.

This makes memory management more compatible with [continuous batching](/articles/continuous-batching-scheduling-ai-foundations-28/), where requests enter and leave the active batch at different times.

## Paging and prefix caching are related but different

These ideas are easy to mix up.

**PagedAttention** answers:

> How do we store KV-cache memory efficiently?

**Prefix caching** answers:

> Can another request reuse KV state that was already computed?

A server can use block-based KV memory without sharing prefixes.

It can also use a paged block system as the storage mechanism that makes shared prefix blocks easier to reference.

So the ideas complement each other, but they solve different problems.

## Paging does not make KV cache free

The cache still consumes memory.

A longer context still means more key/value state.

Larger batch concurrency still means more live cache.

Higher-precision KV formats still require more bytes per token than lower-precision formats.

Paging mainly improves **allocation efficiency** and reduces fragmentation. It does not remove the underlying memory cost.

## There is also bookkeeping

The server now needs data structures that track:

- which physical blocks are free;
- which blocks belong to each sequence;
- how logical positions map to physical blocks;
- when a block can be released;
- whether a block is shared;
- how the attention kernel reads non-contiguous storage.

That indirection has implementation cost.

PagedAttention is useful because the memory savings and higher serving utilization can outweigh that cost for real multi-request workloads.

## A simple mental model

Think of a hotel.

The bad strategy is:

> Every guest reserves an entire floor because we do not know how many rooms they might eventually need.

The paged strategy is:

> Give each guest a room now and assign more rooms as needed. The rooms can be on different floors because the front desk tracks the mapping.

The guest's stay is logically continuous even though the physical rooms are not.

## Why this matters to LLM serving

An LLM server often has two scarce resources:

1. GPU compute;
2. GPU memory.

Continuous batching helps use compute more effectively.

Paged KV management helps use memory more effectively.

If memory is wasted, the server may admit fewer concurrent requests even when the GPU has compute capacity left.

That is why PagedAttention became an important serving-system idea: it changes KV cache from a large per-request reservation problem into a block-allocation problem.

## What comes next

Once KV state is divided into manageable blocks, the server can make more sophisticated decisions about:

- prefix sharing;
- eviction;
- preemption;
- swapping or offloading;
- KV-cache precision;
- distributed KV transfer.

The next layers of serving systems are built on top of this basic observation:

> Tokens are logically ordered, but their cache does not have to occupy one continuous physical region.
