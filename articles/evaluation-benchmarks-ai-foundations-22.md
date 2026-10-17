---
title: "Evaluation and Benchmarks: How Do We Know an AI Model Is Better?"
description: "AI Foundations #22 explains test sets, metrics, benchmark design, sampling variance, contamination, and why a fair model comparison must control more than one score."
pubDate: 2026-09-26
updatedDate: 2026-09-26
category: "AI Fundamentals"
tags: ["AI fundamentals", "evaluation", "benchmarks", "metrics", "test sets", "reproducibility", "model comparison"]
draft: false
featured: false
author: "machamochalatte"
editor: "RAMGPT Editorial Team"
sources:
  - https://arxiv.org/abs/2211.09110
  - https://arxiv.org/abs/2009.03300
  - https://github.com/EleutherAI/lm-evaluation-harness
evidenceStatus: "explainer"
evidenceNote: "Cumulative AI Foundations lesson on model evaluation and benchmarking. Examples are pedagogical; benchmark principles are grounded in HELM, MMLU, and lm-evaluation-harness."
---

Yesterday's lesson ended the core Foundations sequence with reasoning and reinforcement learning.

We learned that training can push a model toward behaviors that receive higher reward, and that inference-time reasoning can spend more tokens working through a problem.

That immediately creates a new question:

> How do we know whether the model actually became better?

If a new training run gets a higher reward during training, that does not automatically mean it is better for every user.

If one demo looks impressive, that does not automatically mean the model improved.

If a benchmark score rises from 72 to 74, that does not automatically mean the difference is meaningful.

This is the problem of **evaluation**.

I used to think evaluation was the easy part.

Train model, ask questions, count correct answers.

Then I realized that almost every word in that sentence hides a decision.

What questions?

How many?

What counts as correct?

Can the model use tools?

How many attempts does it get?

What sampling settings?

Was any of the test data in the training data?

Was the faster model tested on the same hardware?

Evaluation is not just a scoreboard.

It is an experiment.

## A benchmark is a controlled test

The simplest mental model I use is:

~~~text
model
+ fixed task
+ fixed rules
+ fixed scoring method
= evaluation result
~~~

A **benchmark** packages some of those pieces so different systems can be compared.

For example, imagine a 100-question math test.

Model A answers 73 correctly.

Model B answers 78 correctly.

Then the simplest accuracy numbers are:

~~~text
A: 73 / 100 = 73%
B: 78 / 100 = 78%
~~~

That is useful.

But it is only useful if the comparison is fair.

Suppose Model A was forced to answer immediately while Model B could generate 8,000 reasoning tokens and use a calculator.

Now the question has changed.

We are no longer comparing only model weights.

We are comparing two different systems with different inference budgets.

The number 78% may still be valid, but its meaning is different.

## Training data and test data must play different roles

This idea connects back to our lessons on training, loss, and gradient descent.

During training, the model sees examples and changes its weights.

If we then evaluate it on exactly those same examples, the result can be misleading.

It is like studying from an answer sheet and then being tested on the same sheet.

So machine learning usually separates data into roles such as:

~~~text
training set
-> used to update weights

validation or development set
-> used to make design decisions

test set
-> reserved for final evaluation
~~~

The exact names and workflow vary, but the core idea is separation.

A good test asks:

> Can the system handle examples it was not directly optimized on?

That is a question about **generalization**.

## Why test contamination matters for language models

Large language models are trained on huge collections of text.

That creates a difficult problem.

A benchmark question may already exist somewhere in the training corpus.

If the model has seen the exact question and answer, a high score might partly reflect memorization rather than general problem-solving ability.

This is called **contamination**.

The problem becomes especially tricky when benchmark questions are public and widely copied online.

Imagine a famous reasoning benchmark.

People publish:

~~~text
the questions
the answers
explanations
tutorials
forum discussions
model-generated solutions
~~~

Later training corpora may contain some of that material.

Then a benchmark designed to measure unseen reasoning can gradually become less clean.

This does not make all public benchmarks useless.

It means benchmark age, data provenance, and contamination checks matter when interpreting results.

## One metric cannot describe every kind of quality

Accuracy is easy when there is exactly one correct answer.

For example:

~~~text
37 x 24 = 888
~~~

A system either gives the correct number or it does not.

But consider:

~~~text
Explain why the sky appears blue.
~~~

There are many valid answers.

Exact string matching would be silly.

Now consider code generation.

A program may be written in many different ways but still pass the same tests.

So evaluation metrics depend on the task.

Common ideas include:

~~~text
accuracy
exact match
F1 or overlap measures
unit-test pass rate
human preference
model-based judging
latency
throughput
memory use
cost
~~~

A benchmark score is therefore not a universal unit like meters.

It measures something under a particular definition.

## The scoring rule changes what "better" means

Suppose two answers are:

~~~text
Answer A:
888

Answer B:
The result is 888 because 37 x 24 = 37 x (6 x 4) = 222 x 4 = 888.
~~~

For a final-answer math benchmark, both may receive the same score.

For an explanation-quality benchmark, they may not.

For a latency benchmark, Answer A is cheaper.

For a teaching benchmark, Answer B may be more useful.

The model did not change between those interpretations.

The metric changed.

That taught me an important rule:

> Before comparing scores, ask what the score rewards.

This connects directly to reinforcement learning.

In [AI Foundations #21](/articles/reasoning-reinforcement-learning-ai-foundations-21/), we learned that a reward signal shapes training behavior.

Evaluation metrics do something related but different.

Training reward says:

~~~text
what behavior should optimization encourage?
~~~

Evaluation says:

~~~text
what behavior are we going to measure afterward?
~~~

If those two are identical, a model can become very specialized to the metric.

If they are different, training may improve one property while evaluation reveals another.

## Sampling makes repeated runs important

Earlier, in [AI Foundations #16](/articles/sampling-ai-foundations-16/), we learned that language generation can be stochastic.

The same prompt can produce different token paths.

That means one run may not represent the model reliably.

Imagine a difficult problem where a model succeeds about 60% of the time.

If I test it once, I see either:

~~~text
correct
~~~

or:

~~~text
wrong
~~~

Neither single result tells me that the true success tendency is around 60%.

More trials give a better estimate.

Suppose I run 100 independent examples and get 60 correct.

My measured accuracy is:

~~~text
60 / 100 = 0.60
~~~

If I only run 10 examples and get 6 correct, the measured accuracy is still 0.60.

But the 100-example result is much more stable.

This is why sample size matters.

## A little probability helps

If each benchmark item is roughly a correct-or-wrong trial, a useful rough uncertainty scale for an accuracy estimate is:

~~~text
standard error approximately
sqrt( p * (1 - p) / n )
~~~

where:

~~~text
p = measured accuracy
n = number of independent test items
~~~

Suppose:

~~~text
p = 0.60
n = 100
~~~

Then:

~~~text
sqrt(0.60 * 0.40 / 100)
= sqrt(0.0024)
approximately 0.049
~~~

That is about 4.9 percentage points for one standard error.

Now increase the test to 10,000 items:

~~~text
sqrt(0.60 * 0.40 / 10000)
approximately 0.0049
~~~

Now the rough uncertainty scale is about 0.49 percentage points.

The formula is simplified and real benchmark items are not always independent, but the lesson is powerful:

> A tiny score difference on a tiny test may be noise.

This is one reason I do not want to treat every leaderboard movement as a scientific discovery.

## Multiple attempts change the task

Reasoning and coding benchmarks sometimes allow more than one sample.

Suppose a model has a 40% chance of solving a problem on one independent attempt.

One try gives:

~~~text
success probability = 0.40
~~~

Two independent tries give a chance that at least one succeeds:

~~~text
1 - probability both fail
= 1 - 0.60 x 0.60
= 0.64
~~~

Five tries give:

~~~text
1 - 0.60^5
approximately 0.922
~~~

That is a huge difference.

The model weights did not improve.

The evaluation budget changed.

So when I see something like "pass@k," I ask what k is.

A model allowed 100 attempts is solving a different operational problem from a model allowed one attempt.

This matters for code generation, search, planning, and reasoning.

## Token budget is another hidden variable

Consider two models on the same reasoning benchmark.

Model A receives:

~~~text
maximum output: 1,000 tokens
~~~

Model B receives:

~~~text
maximum output: 16,000 tokens
~~~

If Model B can use long chains of intermediate steps, its higher accuracy may partly come from greater test-time compute.

Again, that may be exactly what we want to measure.

But we should label it correctly.

A fair report might say:

~~~text
quality at equal token budget
~~~

or:

~~~text
best quality under each model's recommended reasoning budget
~~~

Those are both valid experiments.

They answer different questions.

## Benchmarking a runtime requires even more controls

Now connect evaluation to the systems lessons in this course.

Suppose we compare inference speed.

Then quality is not enough.

We must control variables such as:

~~~text
same model weights
same quantization
same prompt length
same output length
same batch size
same context state
same hardware
same runtime settings
same sampling settings
same cache conditions
~~~

Otherwise a throughput number can be impossible to interpret.

For example, a Q4 model may be faster and smaller than an FP16 model, but then the comparison mixes runtime performance with quantization.

A warm prefix cache may make one request faster than a cold request, but then the comparison mixes cache state with model speed.

A 4,000-token prompt and a 100,000-token prompt are not the same workload.

Good benchmarking is controlled comparison.

## Quality and speed are separate axes

One of my favorite ways to think about model evaluation is as a coordinate system instead of a single leaderboard.

Imagine two axes:

~~~text
x-axis: latency
y-axis: task quality
~~~

A model can be:

~~~text
fast but less accurate
slow but more accurate
fast and accurate
slow and inaccurate
~~~

Then add another axis:

~~~text
memory
~~~

Then another:

~~~text
cost
~~~

Then another:

~~~text
context capacity
~~~

Real model selection is multi-dimensional.

There may be no single "best model."

There may only be the best model under a particular set of constraints.

## HELM's lesson: evaluate many dimensions

The HELM project is useful here because it treats language-model evaluation as more than one benchmark number.

Its broader idea is that model behavior should be measured across scenarios and metrics, including properties beyond raw accuracy.

I like this framing because it matches engineering reality.

A model can improve on a reasoning test while becoming:

~~~text
slower
more verbose
more expensive
less robust
worse calibrated
harder to deploy
~~~

Whether that is an improvement depends on the application.

One number cannot answer every question.

## MMLU's lesson: breadth matters too

MMLU became influential partly because it evaluates many subject areas rather than one narrow task.

That illustrates another dimension of evaluation: **coverage**.

A model might be excellent at:

~~~text
math
coding
physics
~~~

and weak at:

~~~text
history
law
medicine
~~~

A single aggregate score can hide that shape.

So I like to inspect both:

~~~text
overall score
per-category scores
~~~

The average tells me the center.

The categories tell me where the model is strong or fragile.

## Reproducibility means recording the whole recipe

If another person cannot reconstruct the evaluation conditions, the score becomes much less useful.

A good evaluation record includes things such as:

~~~text
model identifier
checkpoint revision
runtime version
prompt template
system prompt
sampling parameters
maximum tokens
number of samples
benchmark version
scoring code
hardware, if measuring performance
quantization, if applicable
~~~

This may look like boring metadata.

It is actually what turns a number into evidence.

Without it, "Model X scored 82" is only a claim.

## Leaderboards can create their own overfitting pressure

Once a benchmark becomes famous, developers naturally optimize for it.

That is not necessarily dishonest.

If a benchmark measures something valuable, improving on it can be useful.

But repeated optimization against the same public test gradually weakens its role as an independent measure.

The process can look like:

~~~text
benchmark becomes popular
-> teams inspect failures
-> training data and prompts adapt
-> models improve on benchmark
-> benchmark becomes less surprising
~~~

Eventually we may need new tests.

This is similar to school exams.

If students know the exact questions years in advance, the exam measures something different.

## Model-based judges are useful but not magical

Some answers cannot be scored with exact match, so another language model may be used as a judge.

That can scale evaluation dramatically.

But it creates a new measurement system with its own biases.

A judge model may prefer:

~~~text
longer answers
certain writing styles
answers similar to its own
specific formatting
~~~

So a judge score should be treated as a measurement produced by another model, not as objective truth.

Human evaluation has biases too.

The goal is not to find a perfect evaluator.

The goal is to understand the measurement process.

## My benchmark checklist

Before I believe a model comparison, I now ask:

~~~text
1. What exact task is being measured?
2. What counts as success?
3. Is the test separated from training?
4. Could the benchmark be contaminated?
5. Are prompts and tools the same?
6. Are sampling settings the same?
7. Is the token or attempt budget the same?
8. Is the sample size large enough?
9. Are runtime and hardware controlled for speed claims?
10. Can someone reproduce the setup?
~~~

A comparison does not need every variable to be identical.

Sometimes the whole point is to compare different systems.

But the differences should be explicit.

## The connection to the entire Foundations sequence

Evaluation reaches backward through almost everything we have learned.

Parameters and training determine what behavior is available.

Sampling affects which behavior appears on a particular run.

Context and KV cache affect long-sequence cost.

Quantization can change memory, speed, and sometimes quality.

MoE changes active computation.

Multimodal models require modality-specific tests.

Reasoning changes inference-time token budgets.

Reinforcement learning changes what behaviors training rewards.

Then evaluation asks:

> After all those design choices, what actually improved, under what conditions, and by how much?

That is why I think evaluation belongs immediately after the core sequence.

Without evaluation, every previous lesson gives us mechanisms but no disciplined way to compare outcomes.

## My final mental model

I no longer think:

~~~text
benchmark score = model quality
~~~

I think:

~~~text
benchmark result
= model
+ data
+ prompt
+ inference budget
+ sampling
+ tools
+ runtime
+ scoring rule
+ statistical uncertainty
~~~

Change one term and the result may change.

That does not make benchmarks useless.

It makes them experiments.

And experiments become powerful when we control the important variables and state clearly what was measured.

## Next lesson

Now that we can ask whether a model is actually better, the next step is to give a model information it did not memorize in its weights.

That leads naturally to **retrieval-augmented generation, or RAG**:

~~~text
question
-> retrieve relevant external information
-> place it in context
-> generate an answer
~~~

That next topic will reconnect embeddings, context windows, inference, and evaluation into one practical AI system.
