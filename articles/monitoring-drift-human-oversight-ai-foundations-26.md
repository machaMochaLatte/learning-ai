---
title: "Monitoring and Drift: How to Know an AI System Is Still Working"
description: "AI Foundations #26 explains production monitoring, data and concept drift, feedback loops, human review, and why a model that passed evaluation can still fail after deployment."
pubDate: 2026-09-30
updatedDate: 2026-09-30
category: "AI Fundamentals"
tags: ["AI fundamentals", "monitoring", "drift", "evaluation", "human oversight", "production AI"]
draft: false
featured: false
author: "machamochalatte"
editor: "RAMGPT Editorial Team"
sources:
  - https://www.nist.gov/itl/ai-risk-management-framework
  - https://proceedings.neurips.cc/paper/2015/hash/86df7dcfd896fcaf2674f757a2463eba-Abstract.html
  - https://arxiv.org/abs/1810.03993
evidenceStatus: "explainer"
evidenceNote: "Cumulative AI Foundations lesson grounded in NIST AI RMF, model-card reporting, and production ML systems literature. Examples are conceptual, not a specific vendor deployment."
---

In [AI Foundations #25](/articles/alignment-guardrails-ai-foundations-25/), we ended with a simple idea:

> Do not make critical boundaries depend on a model behaving perfectly.

That raises the next question.

Suppose we trained the model, evaluated it, added guardrails, and deployed it.

How do we know it is still working next week?

That question sounds operational, but it is also mathematical.

Before deployment, we evaluate a system on a set of examples.

After deployment, the system meets a changing stream of real inputs, users, tools, documents, policies, and software versions.

Those two distributions are not guaranteed to stay the same.

Monitoring is how we compare what the system is doing now with what we expected it to do.

## Passing a benchmark is not the end

Imagine a model scores 92% on a test set.

It is tempting to think:

~~~text
92% accuracy
-> system quality is known
-> deploy
~~~

But the 92% describes performance on a particular dataset under particular conditions.

Deployment adds new variables:

~~~text
different users
different prompt lengths
new topics
new tools
new data
new software versions
new failure modes
changing costs
changing latency
~~~

So the real question becomes:

~~~text
Is the production system still operating inside the region where our earlier evaluation is informative?
~~~

That is what monitoring tries to answer.

## A useful baseline comes first

You cannot detect change unless you know what normal looked like before the change.

Suppose a support assistant is evaluated before launch.

We might record:

~~~text
answer quality
citation accuracy
refusal rate
tool-call success rate
latency
token usage
human escalation rate
~~~

Those numbers become a baseline.

After deployment, we compute the same measurements over rolling windows.

For example:

~~~text
launch baseline:
tool-call success = 98%

this week:
tool-call success = 91%
~~~

The important part is not that 91% is automatically bad.

The important part is that the system changed.

Now we have something to investigate.

## Monitoring is not one metric

A model can improve on one metric while getting worse on another.

For example:

~~~text
shorter answers
-> lower latency
-> lower cost
-> maybe less complete answers
~~~

Or:

~~~text
stricter safety filter
-> fewer unsafe outputs
-> more unnecessary refusals
~~~

So a production dashboard should not reduce the system to one score.

A useful set of dimensions might include:

| Dimension | Example signal |
| --- | --- |
| Quality | task success, human rating, factual error rate |
| Safety | unsafe compliance, over-refusal |
| Reliability | request failures, tool errors, retries |
| Latency | p50, p95, p99 response time |
| Cost | tokens, GPU time, API or tool cost |
| Retrieval | hit rate, citation coverage, stale documents |
| Agents | tool-call success, loops, aborted workflows |
| Human review | escalation rate, override rate |

The exact metrics depend on the application.

A calculator assistant and a medical-document summarizer should not use the same monitoring policy.

## What is drift?

Drift means that something about the production environment changes relative to the assumptions behind training or evaluation.

There are several useful kinds.

### Data drift

The inputs change.

Suppose an email classifier was trained mostly on short English emails.

Six months later, users send much longer messages with more attachments and more multilingual text.

The input distribution has shifted.

The relationship can be written conceptually as:

~~~text
P_now(X) != P_baseline(X)
~~~

where X represents the inputs.

We do not need advanced probability to understand the idea.

The model is seeing a different population of examples.

### Concept drift

The relationship between input and correct answer changes.

Suppose a fraud system learned patterns from last year's attacks.

Attackers adapt.

The same pattern that once meant safe may now be associated with fraud.

Conceptually:

~~~text
P_now(Y | X) != P_baseline(Y | X)
~~~

The inputs may look similar, but the meaning of those inputs has changed.

### System drift

The model may not change, but the surrounding software does.

Examples:

~~~text
new tokenizer
new inference runtime
new quantization
new prompt template
new retrieval index
new tool schema
new GPU kernel
new API behavior
~~~

This matters because an AI application is a system, not just a checkpoint.

A model can be identical while the deployed behavior changes.

## Drift does not always mean failure

Change is not automatically bad.

Suppose users begin asking more advanced questions.

The input distribution changes, but the system may still answer them well.

So drift detection is not the same as error detection.

A useful flow is:

~~~text
detect change
-> measure impact
-> investigate cause
-> decide whether action is needed
~~~

This prevents a common mistake:

~~~text
distribution changed
-> panic
~~~

The goal is to know when earlier evidence may no longer be enough.

## Rolling windows make change visible

Imagine we track tool-call success each day.

A single day can be noisy.

So instead of comparing individual requests, we can compare windows.

For example:

~~~text
baseline window:
10,000 requests

current window:
last 10,000 requests
~~~

Then compare:

~~~text
success rate
error types
latency distribution
tool distribution
input length
~~~

The same idea applies to model quality if we have labels or human review.

A rolling window trades speed for stability:

~~~text
small window
-> reacts quickly
-> noisier

large window
-> more stable
-> slower to notice change
~~~

There is no universal best size.

It depends on traffic volume and how quickly failures matter.

## Average latency can hide a production problem

Suppose response time is:

~~~text
p50 = 1.2 s
p95 = 2.1 s
p99 = 15.0 s
~~~

The average may look acceptable.

But one percent of users experience a very slow request.

That is why production systems often inspect percentiles rather than one mean.

The same principle appears in model behavior.

An average accuracy number can hide a subgroup where performance is poor.

Monitoring should ask not only:

~~~text
How is the system doing on average?
~~~

but also:

~~~text
Where is it failing?
For whom?
Under which input types?
At what context lengths?
With which tools?
~~~

This connects directly back to [AI Foundations #22](/articles/evaluation-benchmarks-ai-foundations-22/): aggregate metrics are useful, but they can hide important structure.

## Error categories are more useful than one failure count

Suppose an agent has a 5% failure rate.

That number alone does not tell us how to fix it.

We should decompose failures:

~~~text
2.0% wrong tool choice
1.0% invalid arguments
0.8% permission denied
0.6% tool timeout
0.4% repeated loop
0.2% bad final synthesis
~~~

Now the engineering action becomes clearer.

A parser bug does not need more model training.

A permission failure does not need a larger context window.

A bad tool choice may need better tool descriptions, routing, or model training.

Monitoring is most valuable when it maps observed failure to the layer that can actually fix it.

## Human review creates labels from the real world

Before deployment, labels come from a curated evaluation set.

After deployment, we can learn from sampled production cases.

One approach is to send a small fraction of interactions to human review.

Reviewers might answer:

~~~text
Was the answer correct?
Were citations supported?
Was the refusal appropriate?
Was the tool action justified?
Did the system complete the task?
~~~

This creates fresh evidence about the environment the system is actually serving.

It also helps detect silent failures that automatic metrics cannot see.

A tool call can return HTTP 200 and still produce the wrong business outcome.

## Human oversight is not a person reading everything

For a high-volume system, reviewing every interaction may be impossible.

Oversight can be risk-based.

For example:

~~~text
ordinary low-risk requests
-> automatic monitoring

unusual confidence or high-risk action
-> sampled review

irreversible write action
-> explicit approval

detected anomaly
-> temporary stricter gate
~~~

The important design question is where human judgment provides the most value.

This mirrors the tool-use lesson from [AI Foundations #24](/articles/tool-use-agents-ai-foundations-24/).

An LLM can propose an action.

The surrounding system decides how much autonomy is acceptable.

## Feedback loops can make monitoring tricky

Suppose a recommendation model chooses what users see.

Then user behavior depends partly on the model's previous recommendations.

Now the data is not independent of the system.

We get a loop:

~~~text
model recommendation
-> user sees item
-> user clicks or ignores it
-> that behavior becomes new training data
-> future model changes
~~~

The system is helping create the data used to evaluate and train itself.

This can amplify biases or make a metric look better simply because the model controls exposure.

For example, if a recommendation system never shows a category, it may record few negative clicks for that category.

That does not prove users dislike it.

They may never have had the opportunity to see it.

Monitoring needs to distinguish:

~~~text
what users chose
from
what the system allowed users to see
~~~

## Monitoring an LLM has special difficulties

For a conventional classifier, the output may be one label.

For an LLM, the output can be an open-ended sequence.

That makes quality harder to summarize.

We may need a combination of:

~~~text
automatic checks
task-specific tests
human review
model-based evaluation
tool traces
retrieval evidence
safety checks
~~~

Each has limitations.

An automatic evaluator can be wrong.

A human reviewer can disagree with another reviewer.

A model judge may share the same blind spots as the model being evaluated.

So monitoring should not assume one observer is perfect.

## Ground truth can arrive late

Some tasks have immediate answers.

For example:

~~~text
Did the API call succeed?
~~~

Others have delayed ground truth:

~~~text
Did this recommendation reduce churn?
Did this generated code fail next week?
Was this risk prediction actually correct?
~~~

That changes the monitoring architecture.

We may have:

~~~text
fast proxy signals
+ slower confirmed outcomes
~~~

Fast signals help detect incidents quickly.

Slow labels tell us whether the proxies were actually meaningful.

Both matter.

## Deployment changes should be treated like experiments

Suppose we change:

~~~text
model version
prompt
quantization
runtime
retrieval index
tool parser
guardrail policy
~~~

and production quality moves.

If all of those changed at once, root cause is difficult to establish.

A better deployment discipline is:

~~~text
record exact version
change one controlled component when practical
measure before
measure after
preserve rollback
~~~

This is the same logic used in scientific experiments.

Control variables make causal reasoning possible.

It is also why model cards and system documentation matter: they preserve the context needed to interpret a result later.

## Canary deployments reduce blast radius

Instead of sending every user to a new model immediately, we can expose a small fraction of traffic first.

Conceptually:

~~~text
95% -> known version
5%  -> candidate version
~~~

Then compare:

~~~text
quality
errors
latency
cost
safety
~~~

If the candidate behaves badly, the system can roll back before every user is affected.

The exact percentages are not important here.

The principle is:

> Test a change on a limited surface before making it universal.

## Monitoring should trigger actions

A dashboard that everyone ignores is not a control.

Useful monitoring connects signals to responses.

For example:

~~~text
tool failure rate > threshold
-> alert

unsafe-output incident
-> preserve trace + human review

latency spike
-> inspect runtime or dependency change

retrieval freshness failure
-> stop citing stale index

severe regression
-> rollback
~~~

The thresholds depend on the application and risk.

The important part is defining the response before an incident happens.

## The model is only one component to watch

Production AI observability should include the full chain.

For a RAG agent:

~~~text
user input
-> prompt construction
-> retrieval
-> reranking
-> model inference
-> tool call
-> tool response
-> final answer
~~~

A bad answer can come from any step.

If retrieval returned the wrong document, retraining the language model may not help.

If the tool API timed out, changing the prompt may not help.

If a runtime upgrade silently changes tokenization, the checkpoint is not the main cause.

This is why production AI debugging needs traces, not only final outputs.

## The cumulative Foundations map

We can now extend the sequence again:

~~~text
parameters and weights
-> tensors
-> forward pass
-> training
-> loss
-> gradient descent
-> backpropagation
-> embeddings
-> tokenization
-> attention
-> Transformer
-> pretraining
-> fine-tuning
-> inference
-> sampling
-> context and KV cache
-> quantization
-> mixture of experts
-> multimodal models
-> reasoning and reinforcement learning
-> evaluation
-> retrieval
-> tools and agents
-> alignment and guardrails
-> monitoring and drift
~~~

The early lessons explain how a model computes.

The later lessons explain how a model becomes part of a real system.

Those are different levels of the same stack.

## The main takeaway

A model does not become permanently good because it passed evaluation once.

Production changes the evidence.

Users change.

Data changes.

Tools change.

Software changes.

The environment can change even when the checkpoint does not.

So the final mental model is:

~~~text
evaluate before deployment
-> establish a baseline
-> monitor real behavior
-> detect change
-> investigate impact
-> use human review where it matters
-> fix or roll back
-> evaluate again
~~~

Monitoring is not an admission that evaluation failed.

It is recognition that deployment is a continuing experiment.

The question is no longer only:

> Did this AI system work in our test?

It becomes:

> Is it still working now, for the people and conditions it is actually serving?
