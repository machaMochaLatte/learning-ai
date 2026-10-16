---
title: "Fine-Tuning: How a Pretrained Model Learns a New Job Without Starting Over"
description: "AI Foundations #14 explains how fine-tuning continues gradient-based training from pretrained weights, using narrower data to change model behavior without relearning language from scratch."
pubDate: 2026-09-18
updatedDate: 2026-10-16
category: "AI Fundamentals"
tags: ["AI fundamentals", "fine-tuning", "supervised fine-tuning", "pretraining", "gradient descent", "instruction tuning"]
draft: false
featured: false
author: "machaMochaLatte"
editor: "RAMGPT Editorial Team"
sources:
  - https://arxiv.org/abs/1810.04805
  - https://arxiv.org/abs/2203.02155
---

Fine-tuning continues training from a pretrained model on more targeted examples. The starting weights already encode useful patterns; additional gradient updates specialize their behavior for a task or format.

## Start from useful weights, not random weights

The easiest way for me to understand fine-tuning is to compare the starting points.

Pretraining starts roughly like this:

```text
random or newly initialized parameters
-> enormous general training corpus
-> many optimization steps
-> pretrained model
```

Fine-tuning starts here instead:

```text
pretrained model
-> smaller, more targeted dataset
-> more optimization steps
-> fine-tuned model
```

The model does not forget how tensors, attention, or language work and begin again.

We take the parameter values learned during pretraining and continue training them toward a more specific objective.

That starting point is the whole advantage.

## What does the fine-tuning dataset look like?

It depends on the goal.

For a classifier, the data might look like:

```text
text -> label
```

For a language model being taught instruction following, examples might look like:

```text
User: Explain compound interest to a teenager.
Assistant: Compound interest means...
```

or:

```text
User: Convert this JSON into CSV.
Assistant: ...
```

or:

```text
User: Summarize the following article in three bullets.
Assistant: ...
```

The training set demonstrates the behavior we want.

Instead of learning from arbitrary text continuation alone, the model sees curated examples of prompts and desirable responses.

## Supervised fine-tuning is still token prediction

This confused me at first.

If we are "teaching instructions," does the model suddenly use a completely different learning algorithm?

For a causal language model, often not.

The desired assistant response is still tokenized, and the model still learns by predicting tokens.

A simplified example is:

```text
prompt:
What is 7 × 8?

response:
56
```

During supervised fine-tuning, the loss can be computed on the target response tokens so that the model is pushed toward assigning high probability to the demonstrated answer.

Conceptually:

```text
prompt + target response
-> forward pass
-> token probabilities
-> supervised loss
-> backpropagation
-> update weights
```

Implementations differ in exactly which tokens contribute to the loss. Many instruction-tuning pipelines mask prompt tokens and train primarily on response tokens; others can use different formatting or objectives.

The key point is that the Transformer is still doing token prediction. We have changed the **training examples and which predictions matter**.

## Why not just put instructions in the prompt?

Prompting and fine-tuning solve different problems.

A prompt changes the model's **input**. Fine-tuning changes its **parameters**.

![Prompting leaves pretrained weights unchanged while fine-tuning uses targeted examples and gradients to produce updated weights for future inference.](/images/diagrams/prompting-vs-finetuning-parameters.svg)

*The input change is temporary; the parameter update persists in the resulting fine-tuned model.*

If I tell a model in one prompt:

```text
Always answer in JSON.
```

that instruction exists only in the current context.

If I fine-tune on many examples where valid JSON is the desired output, I am trying to make that behavior more likely because of changes in the parameters themselves.

That does not guarantee perfect JSON. Fine-tuning changes probabilities; it does not install an infallible rule engine.

## Fine-tuning can teach format, style, task behavior, or domain patterns

The target behavior does not have to be one thing.

Fine-tuning can specialize a model toward:

```text
instruction following
conversation style
classification
code generation
medical or legal terminology
customer-support formats
structured outputs
translation conventions
domain-specific language
```

But I should be precise about the word "teach."

Sometimes fine-tuning teaches genuinely new task behavior. Sometimes it mostly makes knowledge already latent in the pretrained model easier to elicit. Sometimes it adds domain patterns from new data.

Those cases can look similar from the outside.

## Fine-tuning is not the same as adding a database

Suppose I fine-tune on company policy documents.

It is tempting to imagine this:

```text
model weights now contain a searchable copy of the policy manual
```

That is not a safe mental model.

Fine-tuning changes distributed parameters through optimization. It can improve behavior on patterns represented in the training data, but it is not a reliable replacement for a database or retrieval system when exact, current facts matter.

If a policy changes tomorrow, a retrieval system can point to the new document immediately. A fine-tuned model may need new training and can still reproduce outdated information.

So:

```text
fine-tuning = change model behavior/parameters
retrieval    = provide external information at inference time
```

They can also be combined.

## Why the learning rate often becomes smaller

Pretraining may involve enormous optimization schedules designed to learn the model from scratch.

Fine-tuning begins with weights that are already valuable.

So making giant parameter jumps can destroy useful behavior.

A simplified gradient update is still:

```text
new weight = old weight - learning_rate × gradient
```

If the learning rate is smaller, each update moves the parameters less aggressively.

That gives me a useful intuition:

```text
pretraining: build the general structure
fine-tuning: nudge that structure toward a target
```

Real training recipes are more complicated, but "nudge rather than rebuild" is a good first picture.

## Too much specialization can hurt

Suppose I fine-tune a broadly capable model on a tiny dataset where every answer looks like:

```text
YES
```

If I train too aggressively, I should not be surprised if the model becomes excessively biased toward that narrow pattern.

Two related risks are useful to know.

### Overfitting

The model can become very good at reproducing the fine-tuning examples without generalizing well to new examples.

```text
training performance improves
validation performance stops improving
```

That is the same basic overfitting problem we see elsewhere in machine learning.

### Catastrophic forgetting

Specialized training can also degrade capabilities learned earlier.

If fine-tuning pushes the parameters too far toward one narrow distribution, performance on unrelated tasks can suffer.

This is one reason practitioners care about:

```text
data quality
learning rate
number of steps
data mixture
regularization
validation sets
```

Fine-tuning is not automatically harmless because the dataset is smaller.

## Full fine-tuning means many parameters can move

In **full fine-tuning**, the optimizer can update the model's original trainable parameters.

For a very large model, that can require a lot of memory because training needs more than the weights themselves.

We may need memory for:

```text
model parameters
gradients
optimizer state
activations
```

That is much more expensive than simply running inference.

This connects back to William's point from pretraining: training and inference have very different hardware requirements.

## But sometimes we do not update every original parameter

There are also parameter-efficient fine-tuning methods.

A famous example is LoRA, where we keep the large pretrained weight matrices fixed and train much smaller low-rank updates.

We do not need to derive LoRA today, but the distinction is useful:

```text
full fine-tuning:
update many/all original model parameters

parameter-efficient fine-tuning:
keep most original parameters fixed
train a smaller set of additional/adapted parameters
```

Both approaches are trying to specialize a pretrained model without paying the cost of pretraining from scratch.

## Fine-tuning does not automatically make a chat model aligned

Supervised fine-tuning can teach a model to imitate high-quality demonstrations.

But modern assistant training can include additional stages such as preference optimization or reinforcement learning.

For example, the InstructGPT work used supervised fine-tuning as one stage before training a reward model and applying reinforcement learning from human feedback.

So this pipeline:

```text
pretraining
-> supervised fine-tuning
-> preference / RL stages
```

is possible, but fine-tuning itself should not be confused with the whole alignment process.

We will reach reasoning and reinforcement-learning ideas much later in this Foundations sequence.

## A tiny worked example

Suppose a pretrained model already knows arithmetic and English reasonably well.

I want it to answer elementary arithmetic in one strict format:

```text
Answer: <number>
```

My fine-tuning data might contain:

```text
Question: 3 + 4
Answer: 7

Question: 8 × 5
Answer: 40

Question: 18 - 6
Answer: 12
```

The model does not need to relearn what numbers or multiplication are from random weights.

Instead, the gradients push it toward:

```text
recognize this task pattern
+ produce the desired answer
+ use the desired output format
```

That is a much smaller learning problem than pretraining a language model from scratch.

## Pretraining versus fine-tuning

The comparison I want to keep is:

| Question | Pretraining | Fine-tuning |
| --- | --- | --- |
| Starting point | Newly initialized model | Pretrained model |
| Typical data | Very broad and massive | Narrower and targeted |
| Goal | Learn broad language/model capabilities | Specialize behavior or task performance |
| Optimization | Gradient-based training | Gradient-based training |
| Cost | Extremely large at frontier scale | Usually much smaller, though still expensive for large models |
| Main risk | Under/overtraining, data and scaling problems | Overfitting, forgetting, narrow behavior shifts |

The algorithms are related. The context and scale are different.

Once training ends, the next question is how the resulting model produces an answer. [Inference](/articles/inference-ai-foundations-15/) follows a prompt through that process.
