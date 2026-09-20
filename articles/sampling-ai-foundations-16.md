---
title: "Sampling: How Temperature, Top-k, and Top-p Choose the Next Token"
description: "AI Foundations #16 explains how logits become generated text through greedy decoding, temperature, top-k, and top-p sampling, with simple probability intuition and worked examples."
pubDate: 2026-09-20
updatedDate: 2026-09-20
category: "AI Fundamentals"
tags: ["AI fundamentals", "sampling", "temperature", "top-k", "top-p", "logits", "inference"]
draft: false
featured: false
author: "machaMochaLatte"
editor: "RAMGPT Editorial Team"
sources:
  - https://arxiv.org/abs/1706.03762
  - https://arxiv.org/abs/1904.09751
  - https://arxiv.org/abs/2005.14165
evidenceStatus: "explainer"
evidenceNote: "A cumulative conceptual lesson on autoregressive decoding and sampling, grounded in Transformer language modeling and established nucleus-sampling literature."
---

Last lesson ended at a very specific point:

```text
model runs a forward pass
-> produces logits
-> now we must choose one next token
```

That last arrow bothered me.

If the model already gives scores for every possible next token, why do we need more settings like:

```text
temperature
top-k
top-p
```

Why not just choose the biggest score every time?

The answer is that **the model predicts a distribution, while the decoding algorithm decides how to use that distribution**.

Today I want to understand that decision mathematically, but without turning it into a statistics lecture.

## Start with logits

Suppose the model is trying to continue:

```text
The cat sat on the
```

After the forward pass, imagine it produces these simplified logits:

```text
mat       5.0
floor     4.2
sofa      3.8
roof      1.2
banana   -1.0
```

A logit is just a score.

Higher means the model currently prefers that token more strongly.

The logits are not probabilities yet.

To turn them into probabilities, we usually apply **softmax**.

## Softmax turns scores into probabilities

The softmax formula is:

```text
P(i) = exp(z_i) / sum_j exp(z_j)
```

where:

```text
z_i = logit for token i
```

The exponential makes larger logits dominate more strongly, then division normalizes everything so the probabilities sum to 1.

For intuition, suppose softmax gives us approximately:

```text
mat      0.57
floor    0.26
sofa     0.14
roof     0.02
banana   0.01
```

Now the model is saying something like:

```text
57% mat
26% floor
14% sofa
2% roof
1% banana
```

This is not a statement that "mat is objectively 57% correct."

It is the model's next-token probability distribution under the current context and decoding setup.

## Greedy decoding: always choose the largest probability

The simplest decoding rule is:

```text
choose argmax(probability)
```

In our example:

```text
mat = 0.57
```

so greedy decoding always chooses:

```text
mat
```

Then the context becomes:

```text
The cat sat on the mat
```

The model runs another decode step and produces another distribution.

Greedy decoding repeats this process:

```text
step 1: choose most likely token
step 2: choose most likely token
step 3: choose most likely token
...
```

It is deterministic if everything else is deterministic.

That makes greedy decoding extremely useful for testing.

If I want to compare two runtimes and ask whether they produce the exact same output, greedy decoding removes sampling randomness from the experiment.

But greedy decoding is not always the best way to generate natural text.

## Why always taking the maximum can be too rigid

Imagine the model produces:

```text
The weather today is
```

with probabilities:

```text
sunny     0.31
warm      0.29
pleasant  0.22
clear     0.15
purple    0.03
```

Greedy decoding always chooses `sunny`.

But `warm`, `pleasant`, and `clear` are also plausible.

If the model always takes the single maximum at every step, generation can become repetitive or overly predictable.

Sampling allows the runtime to sometimes choose another high-probability token.

The important word is **sometimes**.

We do not want pure randomness.

We want randomness that is still guided by the model's distribution.

## Sampling is weighted randomness

Suppose the distribution is:

```text
sunny     0.31
warm      0.29
pleasant  0.22
clear     0.15
purple    0.03
```

A random sampler does not treat all five tokens equally.

It behaves more like a weighted lottery:

```text
sunny gets 31 tickets
warm gets 29
pleasant gets 22
clear gets 15
purple gets 3
```

Then one ticket is drawn.

`sunny` is still the most likely choice, but it is not guaranteed.

This is the basic idea behind stochastic decoding.

## Temperature changes how sharp the distribution is

Temperature is one of the most common sampling controls.

The formula is usually written as:

```text
P(i) = softmax(z_i / T)
```

where:

```text
T = temperature
```

The important part is the division:

```text
logit / temperature
```

Let's see what that does.

## Temperature below 1 makes the model more confident

Suppose two logits are:

```text
A = 5
B = 4
```

At temperature:

```text
T = 1
```

we keep:

```text
5 / 1 = 5
4 / 1 = 4
```

Now lower temperature to:

```text
T = 0.5
```

Then:

```text
5 / 0.5 = 10
4 / 0.5 = 8
```

The difference becomes larger in the softmax scale.

That makes the highest-probability tokens dominate more strongly.

So lower temperature generally means:

```text
more concentrated distribution
less randomness
more predictable choices
```

## Temperature above 1 flattens the distribution

Now try:

```text
T = 2
```

The same logits become:

```text
5 / 2 = 2.5
4 / 2 = 2.0
```

The difference between them is smaller.

After softmax, lower-ranked tokens receive relatively more probability mass.

So higher temperature generally means:

```text
flatter distribution
more randomness
more unusual choices
```

This is why increasing temperature can make writing more varied, but if it is too high the model can start choosing weak candidates too often.

## Temperature does not add knowledge

This is important.

If I increase temperature, I am not making the model smarter or more creative internally.

I am changing how aggressively I sample from the distribution it already produced.

A bad analogy would be:

```text
higher temperature = more intelligent model
```

A better analogy is:

```text
same model scores
+ different selection rule
= different generated path
```

The weights do not change.

The prompt does not change.

The decoding policy changes.

## What does temperature 0 mean?

Mathematically, dividing by exactly zero is not valid.

But inference APIs commonly interpret:

```text
temperature = 0
```

as a request for deterministic or greedy-style decoding.

The exact implementation can vary by runtime.

Conceptually, the intent is:

```text
always choose the strongest candidate
```

That is why temperature 0 is often used for reproducibility tests.

## Temperature alone still leaves a long tail

Modern vocabularies can contain tens or hundreds of thousands of tokens.

Even if most of the probability mass sits on a small group of plausible choices, many weak tokens may still have tiny nonzero probabilities.

Imagine:

```text
top 5 tokens = 94% total probability
remaining 99,995 tokens = 6%
```

Each tail token may be individually unlikely, but the tail as a whole is not necessarily negligible.

This motivates filters such as **top-k** and **top-p**.

## Top-k: only keep the k highest-ranked tokens

Top-k is simple.

If:

```text
k = 3
```

and our probabilities are:

```text
sunny     0.31
warm      0.29
pleasant  0.22
clear     0.15
purple    0.03
```

we keep only:

```text
sunny
warm
pleasant
```

and discard:

```text
clear
purple
```

Then the remaining probabilities are renormalized so they add to 1.

Originally:

```text
0.31 + 0.29 + 0.22 = 0.82
```

After renormalization:

```text
sunny     0.31 / 0.82 ≈ 0.378
warm      0.29 / 0.82 ≈ 0.354
pleasant  0.22 / 0.82 ≈ 0.268
```

Now sampling happens only among those three choices.

## Top-k uses a fixed number of candidates

This makes top-k easy to reason about:

```text
top-k = 1
```

means only the best token survives.

```text
top-k = 10
```

means the ten highest-scoring tokens remain eligible.

But there is a weakness.

A fixed `k` does not care whether the probability distribution is very confident or very uncertain.

## The fixed-k problem

Consider two situations.

### Situation A: the model is very confident

```text
Paris      0.96
London     0.01
Berlin     0.01
Rome       0.01
Tokyo      0.01
```

If top-k = 5, all five survive even though one answer dominates.

### Situation B: the model is uncertain

```text
red        0.12
blue       0.11
green      0.10
yellow     0.09
orange     0.08
purple     0.08
white      0.07
black      0.07
...
```

If top-k = 5, we throw away many candidates even though probability is spread broadly.

So the same fixed `k` can be too permissive in one case and too restrictive in another.

That leads us to top-p.

## Top-p: keep enough tokens to cover a probability mass

Top-p is also called **nucleus sampling**.

Instead of saying:

```text
keep exactly 10 tokens
```

we say something like:

```text
keep the smallest set of top-ranked tokens whose cumulative probability reaches 0.9
```

Suppose:

```text
A = 0.40
B = 0.25
C = 0.15
D = 0.10
E = 0.06
F = 0.04
```

With:

```text
top-p = 0.90
```

we accumulate from the top:

```text
A                 = 0.40
A + B             = 0.65
A + B + C         = 0.80
A + B + C + D     = 0.90
```

So the candidate set is:

```text
A, B, C, D
```

E and F are removed.

Then A-D are renormalized and sampled.

## Why top-p adapts to confidence

Now reconsider the highly confident case:

```text
Paris  = 0.96
London = 0.01
Berlin = 0.01
Rome   = 0.01
Tokyo  = 0.01
```

With:

```text
top-p = 0.9
```

`Paris` alone already exceeds 0.9.

So the nucleus may contain only one token.

But if the model is uncertain and probability is spread out, the nucleus becomes larger.

That adaptive behavior is the main intuition behind top-p:

```text
confident model -> small candidate set
uncertain model -> larger candidate set
```

## Top-k and top-p can be combined

Many runtimes apply several filters in sequence.

For example:

```text
logits
-> temperature
-> top-k filter
-> top-p filter
-> renormalize
-> sample
```

The exact order can vary by implementation, so two runtimes with nominally identical settings are not guaranteed to behave identically unless their sampler pipelines match.

This matters when comparing llama.cpp, vLLM, Transformers, Ollama, or another runtime.

The model file may be identical while the sampler implementation differs.

## Worked example

Suppose the model produces these logits:

```text
A = 4.0
B = 3.5
C = 3.0
D = 2.0
E = 1.0
```

Imagine softmax at temperature 1 gives approximately:

```text
A = 0.46
B = 0.28
C = 0.17
D = 0.06
E = 0.02
```

### Greedy

Choose:

```text
A
```

every time.

### Top-k = 2

Keep:

```text
A, B
```

Renormalize:

```text
A = 0.46 / 0.74 ≈ 0.62
B = 0.28 / 0.74 ≈ 0.38
```

Now B has a meaningful chance to be chosen.

### Top-p = 0.8

Accumulate:

```text
A       = 0.46
A + B   = 0.74
A+B+C   = 0.91
```

We need C to cross 0.8, so the nucleus becomes:

```text
A, B, C
```

### Lower temperature

If we lower temperature, A becomes more dominant before filtering.

The same top-p threshold might then need only A and B, or perhaps only A.

That shows why these controls interact.

## Sampling makes generation branch

This is the part I find most useful mentally.

Suppose the prompt is fixed.

At one decode step:

```text
A = 45%
B = 35%
C = 20%
```

Run 1 samples A.

Run 2 samples B.

Now the contexts are different:

```text
run 1 context = prompt + A
run 2 context = prompt + B
```

The next forward pass therefore produces two different distributions.

One random choice early in generation can send the rest of the response down a different path.

So sampling randomness compounds over time.

This is why two answers from the same model can diverge dramatically even if they differ at only one early token.

## A seed controls the random-number generator, not the model

Many runtimes expose a seed.

If sampling uses pseudorandom numbers, a fixed seed can often make the random choices reproducible under the same environment and sampler behavior.

So conceptually:

```text
same model
same prompt
same sampling settings
same seed
```

can often reproduce the same generation.

But a seed is not a universal guarantee across different runtimes, versions, hardware paths, or sampler implementations.

If the probability distribution changes even slightly, the same random number can map to a different token.

That connects directly to runtime debugging.

## Why tiny numerical changes can alter sampled output

Suppose two runtimes produce:

```text
runtime A:
A = 0.5001
B = 0.4999

runtime B:
A = 0.4999
B = 0.5001
```

The distributions are extremely close.

But greedy decoding flips from A to B.

Under sampling, even smaller changes can shift cumulative probability boundaries and change which token corresponds to a random draw.

So when engineers compare runtime correctness, they often use greedy decoding first.

It removes one entire source of variability.

## Sampling is not the same as reasoning

Another confusion I had:

```text
higher temperature = more reasoning
```

No.

Temperature only changes token selection.

The model's internal forward computation still happens before the sampling rule.

A lower or higher temperature may change the generated chain of text, which can indirectly change later context and therefore later computation, but the sampler itself is not a reasoning module.

It is a decoding policy.

## Why coding models often use low temperature

For tasks with narrow acceptable outputs, randomness can be undesirable.

For example:

```text
complete this exact JSON schema
write a deterministic SQL query
return one classification label
```

A low temperature or greedy-like configuration can reduce unnecessary variation.

But for open-ended writing:

```text
brainstorm names
write dialogue
produce creative alternatives
```

some randomness may be useful.

There is no single temperature that is universally correct.

The setting depends on the task and model.

## What happens after a token is sampled?

Suppose the sampler chooses token ID:

```text
7312
```

The runtime appends it to the generated sequence.

Then inference continues:

```text
existing context
+ token 7312
-> next forward/decode step
-> new logits
-> sampler
-> another token
```

So the full loop from the previous lesson becomes:

```text
context
-> model
-> logits
-> sampling policy
-> next token
-> append to context
-> repeat
```

The model and sampler are separate pieces of the loop.

## The sampler does not normally modify model weights

This connects back to our earlier lessons.

During sampling:

```text
weights stay fixed
gradients are not computed
optimizer does not update parameters
```

We are still in inference.

The only thing changing step by step is the current token sequence and the runtime state associated with it.

## Where repetition penalties fit

You may also see settings such as:

```text
repetition penalty
frequency penalty
presence penalty
min-p
typical-p
```

These are additional ways to modify the candidate distribution before choosing a token.

I am not going to treat them as separate foundational concepts today.

The important pattern is the same:

```text
model produces logits
-> decoding logic transforms/filter them
-> sampler chooses token
```

Temperature, top-k, and top-p are enough to understand the core idea.

## A compact comparison

| Method | Main idea | Effect |
| --- | --- | --- |
| Greedy | choose highest probability | deterministic, rigid |
| Temperature < 1 | sharpen distribution | more conservative |
| Temperature > 1 | flatten distribution | more varied |
| Top-k | keep fixed number of best tokens | removes long tail |
| Top-p | keep enough tokens to cover probability p | adaptive candidate set |

These methods are not different models.

They are different ways of selecting from the same model's next-token scores.

## The mental model I am keeping

My short version is:

```text
1. The model produces logits.
2. Softmax turns logits into a probability distribution.
3. Temperature changes how sharp or flat that distribution is.
4. Top-k can restrict choices to the k strongest tokens.
5. Top-p can restrict choices to a probability-mass nucleus.
6. The sampler draws one token from the remaining distribution.
7. That token is appended to context.
8. The entire process repeats.
```

The key distinction is:

```text
model = creates the distribution
sampler = chooses from the distribution
```

That finally completes the missing step from the inference lesson.

But it creates the next question.

Every generated token gets appended to the context. If a conversation grows from 100 tokens to 10,000 or 100,000 tokens, does the model recompute every old token from scratch on every step?

Efficient runtimes avoid that with saved attention state.

That takes us to the next lesson: **context and the KV cache**.