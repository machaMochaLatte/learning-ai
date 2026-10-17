---
title: "Disaggregated Prefill and Decode: Two Jobs Inside One LLM Request"
description: "AI Foundations #33 explains why LLM servers may separate prompt processing from token generation, how KV state moves, and the trade-offs between latency, throughput and operational cost."
pubDate: 2026-10-08
updatedDate: 2026-10-15
category: "AI Fundamentals"
tags: ["AI fundamentals", "LLM inference", "prefill", "decode", "KV cache", "latency", "vLLM"]
draft: false
featured: false
author: "machamochalatte"
editor: "RAMGPT Editorial Team"
sources:
  - https://docs.vllm.ai/en/latest/features/disagg_prefill/
  - https://arxiv.org/abs/2401.09670
  - https://github.com/LLMServe/DistServe
evidenceStatus: "explainer"
evidenceNote: "Conceptual serving explanation using vLLM's current experimental documentation and the DistServe paper. Illustrative request diagrams are not measurements or claims about RAMGPT's hardware."
---

In [AI Foundations #32](/articles/llm-inference-parallelism-ai-foundations-32/), we asked how a model can be distributed across GPUs. Tensor, pipeline, data and expert parallelism answer different versions of that question.

Now ask a different question:

**What if one user request contains two very different computing jobs?**

A language-model response usually has two phases: **prefill**, which processes the prompt, and **decode**, which produces the answer token by token.

They use the same model, but they create different demands on the machine. Some serving systems therefore place them on separate workers. This is called *disaggregated prefill and decode*.

## First, separate the two phases mentally

Imagine asking a model to summarize a long report.

During **prefill**, the model reads the prompt tokens and constructs internal attention state. Transformer layers can evaluate many positions in a prompt in parallel, subject to the model's attention pattern and implementation.

During **decode**, it uses that prompt state to generate the first output token, then the next, then the next. Each new token depends on the preceding continuation.

A simplified timeline looks like this:

~~~text
long prompt
    |
    v
PREFILL: process existing prompt positions
    |
    v
DECODE: generate token 1 -> token 2 -> token 3 -> ...
~~~

Prefill is not automatically "just CPU work", and decode is not automatically "just GPU work". Both involve model computation. The difference is how much parallel work is available and how their memory-access and latency behaviour differ.

## Two clocks users actually feel

We can describe the experience with two measurements.

**Time to first token (TTFT):** how long the user waits before the first generated token arrives.

**Inter-token latency (ITL), or time per output token (TPOT):** how much time typically passes between later tokens.

A long prompt can increase prefill work and delay TTFT. A heavily loaded decode worker can make the answer stream feel sluggish even after it starts.

The two clocks matter differently:

- An autocomplete interaction may be extremely sensitive to the first token.
- A long streamed answer may suffer most from uneven later-token delivery.
- An agent issuing many short requests may care about both latency and queueing.

A single tokens-per-second figure hides these distinctions.

## Why putting everything together causes interference

A conventional server may schedule new prefill work alongside ongoing decode work on the same GPUs.

This is efficient in many cases because GPUs should not sit idle. But the jobs can compete for compute time and memory bandwidth.

Imagine two requests:

~~~text
Request A: already generating a response
Request B: just arrived with a very long prompt

shared GPU scheduling:
A decode -> B prefill chunk -> A decode -> ...
~~~

If B's prefill occupies significant resources, A may experience a large gap between output tokens. A has not become more complex; it is sharing a machine with a different kind of task.

This is a **tail-latency** problem. Average latency can look acceptable while a smaller share of requests experiences noticeable stalls.

## What disaggregation changes

Instead of having both phases on the same worker, separate them:

![Incoming tokens reach a prefill worker, which transfers KV state to a separate decode worker before streamed generation.](/images/diagrams/disaggregated-prefill-kv-handoff.svg)

*The split allows independent phase scheduling but requires a compatible KV-state transfer. It is not a promise of higher total throughput.*

The decode worker need not recompute the full prompt if it receives the correct state. That transfer is the engineering link between the two workers.

The architecture can assign different GPU counts, tensor-parallel strategies and queue policies to prefill and decode. Its purpose is to control interference and tune the two clocks separately.

The current [vLLM documentation](https://docs.vllm.ai/en/latest/features/disagg_prefill/) explicitly describes this feature as **experimental**. It says the principal goals are independent TTFT/ITL tuning and better control of tail ITL—not a general guarantee of higher throughput.

## What exactly has to move?

Recall the [KV-cache lesson](/articles/what-is-kv-cache/): attention layers reuse key/value state for prompt positions that the model has already processed.

In a split system, the decoder needs relevant state from the prefiller to continue generation without repeating all that work.

This is not the same as passing a short string containing the prompt. The transfer may involve many tensors, across model layers, with positions, request identity and lifecycle bookkeeping.

That creates real costs:

- the state consumes network or interconnect bandwidth;
- the producing and consuming workers must agree on model and cache format;
- buffers need to be allocated, transferred, synchronized and eventually released;
- a failed transfer must not cause the decoder to use the wrong request's state.

Different models, attention layouts and KV-cache formats complicate these contracts. You should not assume that any cache blob can be handed to any worker.

The vLLM implementation describes a family of *KV connectors* to coordinate this boundary. Examples include NIXL- and LMCache-based paths; their exact compatibility is version-specific.

## Does this make the server faster?

There are two different meanings of "faster".

**Raw throughput:** the total work completed per second under a specified benchmark.

**Goodput under latency constraints:** the amount of useful work completed while still meeting a required TTFT and output-token-latency target.

Those are not interchangeable.

vLLM's feature documentation explicitly warns that disaggregated prefilling **does not by itself improve throughput**. Moving state between workers adds overhead, and dedicated worker groups can leave resources underused for the wrong workload mix.

The [DistServe research paper](https://arxiv.org/abs/2401.09670), on the other hand, reports improvements in latency-constrained serving *goodput* under its evaluated configurations. That is not a contradiction: reducing interference can help more requests meet strict latency objectives without creating extra compute capacity for free.

It also does not mean another cluster will reproduce the paper's results. Hardware, interconnect bandwidth, prompt/output length distributions, routing, parallelism and SLO definitions must match before numerical comparisons are meaningful.

## Why not simply use chunked prefill?

There is a simpler alternative: divide long prefill work into chunks so decode steps can run between them on the same worker.

Chunked prefill can improve scheduling fairness without transferring KV state between separate services.

~~~text
co-located:
prefill chunk -> decode -> prefill chunk -> decode

disaggregated:
prefill worker: long prefill work
decode worker: output tokens, isolated from that prefill queue
~~~

The trade-off is not "old versus new".

Chunked prefill avoids transfer and extra-service complexity, but finding a chunk size that preserves good decode latency and good GPU utilization is workload-dependent. Disaggregation creates a clearer resource boundary but requires transport, coordination and capacity planning.

A sensible evaluation compares both under the same request trace and hardware budget.

## A classroom-sized thought experiment

Suppose an online class uses one LLM service.

Most students send short questions. One student submits a long document for summarization. The server begins processing that document while several short responses are already streaming.

Which measurements tell you whether an architecture helps?

1. Record TTFT for each request, not only the longest document request.
2. Record the typical and worst observed gaps between output tokens.
3. Record completed requests and GPU utilization over the same interval.
4. Include the cost of transferring KV state if using separate workers.
5. Repeat at different prompt/output length mixes rather than one convenient scenario.

The table below is an investigation plan, not an experimental result:

| Question | Metric |
| --- | --- |
| Do new long prompts delay existing streams? | Tail ITL / TPOT |
| Do users receive their first token sooner? | TTFT distribution |
| Did total capacity change? | Requests or tokens per second at matched load |
| Is there an added transport bottleneck? | KV transfer time and bytes |
| Does one worker group wait while another queues? | Per-stage utilization and queue length |

## Operational failure modes worth understanding

Splitting phases also splits responsibility.

If a decode worker cannot locate the correct KV state, the system needs a defined failure path. If it restarts, another worker may need to recompute the prompt. If the two phases use mismatched model revisions, seemingly valid tensor shapes do not prove semantic compatibility.

A production implementation therefore needs good request IDs, trace propagation, readiness checks, clear cache ownership and explicit timeouts.

These are ordinary distributed-system concerns, but now they sit directly on the generation path. A serving architecture is only useful if its failure behaviour is understandable.

## The mental model to keep

Parallelism from #32 answers **how many GPUs participate in each phase**.

Disaggregation from #33 answers **which phase each group of GPUs performs**.

You can combine them:

~~~text
prefill pool: one parallelism strategy
       |
       | KV state transfer
       v
decode pool: a different parallelism strategy
~~~

More components do not guarantee better performance. The point is to make independent tuning possible when the workload and latency objectives justify the extra complexity.

## Check your understanding

1. Why can a new long prompt delay a different request that is already streaming?
2. Which metric should you watch if the first token is fast but later output arrives unevenly?
3. What state must pass between prefill and decode workers?
4. Why might disaggregation improve SLO-constrained goodput without increasing raw throughput?
5. When would chunked prefill be the simpler experiment to try first?

## Primary sources

- [vLLM: Disaggregated Prefilling (experimental)](https://docs.vllm.ai/en/latest/features/disagg_prefill/)
- [DistServe: Disaggregating Prefill and Decoding](https://arxiv.org/abs/2401.09670)
- [DistServe reference implementation](https://github.com/LLMServe/DistServe)
