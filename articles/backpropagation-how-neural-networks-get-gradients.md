---
title: "Backpropagation: How a Neural Network Gets a Gradient for Every Weight"
description: "AI Foundations #8 explains backpropagation as repeated chain-rule bookkeeping that carries loss information backward through a neural network to every trainable parameter."
pubDate: 2026-09-12
updatedDate: 2026-09-12
category: "AI Fundamentals"
tags: ["AI fundamentals", "backpropagation", "neural networks", "gradients", "chain rule", "training"]
draft: false
featured: false
author: "machaMochaLatte"
editor: "RAMGPT Editorial Team"
sources:
  - https://docs.pytorch.org/tutorials/beginner/basics/autogradqs_tutorial.html
  - https://www.deeplearningbook.org/contents/mlp.html
---

In the last lesson, William showed how gradient descent uses a gradient to update a weight:

```text
new weight = old weight - learning rate × gradient
```

That leaves a much harder question.

A tiny model with one weight can have its derivative written down by hand. A neural network may contain millions or billions of trainable parameters, with some weights many layers away from the final loss.

**How do we get a gradient for every one of them without solving the whole network again for each weight?**

The answer is **backpropagation**.

I used to think backpropagation was another optimization algorithm. It is not. Gradient descent tells us how to use gradients. Backpropagation is the efficient procedure that computes those gradients through a chain of operations.

## Start with a chain, not a giant neural network

Suppose we build a tiny computation:

```text
x -> multiply by w -> y -> square -> loss
```

Let:

```text
x = 3
w = 2
y = w × x = 6
loss = y² = 36
```

The forward pass goes left to right. We start with `x` and `w`, calculate `y`, and then calculate the loss.

But training needs the opposite question:

```text
If w changed a tiny amount, how would the final loss change?
```

There are two links between `w` and the loss:

```text
w -> y

y -> loss
```

Backpropagation walks those dependencies backward.

## Each operation knows a local derivative

For:

```text
y = w × x
```

the sensitivity of `y` to `w` is:

```text
dy/dw = x
```

Since `x = 3`:

```text
dy/dw = 3
```

For:

```text
loss = y²
```

the sensitivity of the loss to `y` is:

```text
dloss/dy = 2y
```

Since `y = 6`:

```text
dloss/dy = 12
```

Neither derivative alone tells us how the loss changes with `w`. One describes `w -> y`; the other describes `y -> loss`.

The useful quantity is:

```text
dloss/dw
```

This is where the chain rule connects the pieces.

## The chain rule is the bridge

Because changing `w` changes `y`, and changing `y` changes the loss:

```text
dloss/dw = dloss/dy × dy/dw
```

Substitute our numbers:

```text
dloss/dw = 12 × 3
          = 36
```

So the gradient for `w` is `36`.

The arithmetic is not the main point. The structure is:

```text
upstream sensitivity × local sensitivity
```

At each step backward, we take information arriving from later in the computation and combine it with the derivative of the local operation.

That repeated multiplication is the core idea behind backpropagation.

## Why going backward is efficient

Imagine a network with one final loss and a million parameters.

A wasteful strategy would be:

```text
change weight 1 slightly -> run model -> measure loss
change weight 2 slightly -> run model -> measure loss
change weight 3 slightly -> run model -> measure loss
...
```

That would require an enormous number of extra forward evaluations.

Backpropagation instead reuses intermediate results from the computation graph.

During the forward pass, the network calculates activations such as:

```text
input -> layer 1 -> layer 2 -> layer 3 -> prediction -> loss
```

During the backward pass, gradient information flows in the reverse dependency direction:

```text
loss -> prediction -> layer 3 -> layer 2 -> layer 1
```

Each operation contributes its local derivative. Once a later gradient has been calculated, earlier operations can reuse it rather than recomputing the entire effect from scratch.

This reuse is what makes gradients for large networks computationally practical.

## A two-weight example makes the reuse clearer

Now consider:

```text
a = w1 × x
b = w2 × a
loss = b²
```

Let:

```text
x = 2
w1 = 3
w2 = 4
```

The forward pass gives:

```text
a = 3 × 2 = 6
b = 4 × 6 = 24
loss = 24² = 576
```

Start backward from the loss:

```text
dloss/db = 2b = 48
```

For `w2`, because `b = w2 × a`:

```text
db/dw2 = a = 6
```

Therefore:

```text
dloss/dw2 = dloss/db × db/dw2
           = 48 × 6
           = 288
```

But we also need to continue backward through `a`.

Since:

```text
b = w2 × a
```

we have:

```text
db/da = w2 = 4
```

So:

```text
dloss/da = dloss/db × db/da
          = 48 × 4
          = 192
```

Now `a = w1 × x`, so:

```text
da/dw1 = x = 2
```

and:

```text
dloss/dw1 = dloss/da × da/dw1
           = 192 × 2
           = 384
```

Notice what happened: the gradient `dloss/db = 48` was useful for more than one backward path. We did not start from the loss again for each parameter.

## A computation graph is bookkeeping for dependencies

Real neural networks are not simple straight lines. Values can branch, combine, be reused, or feed multiple later operations.

It helps to think of the forward pass as building a graph of dependencies.

For example:

```text
        -> branch A ->
input                  combine -> loss
        -> branch B ->
```

During backpropagation, a value that influences the loss through multiple paths receives gradient contributions from those paths. Those contributions must be accumulated.

This is why automatic differentiation systems track which operations produced which tensors. They are not merely remembering numerical values. They are retaining enough structure to apply the chain rule backward through the computation.

## Backpropagation is not the same as gradient descent

This distinction is important enough to repeat.

| Concept | Job |
| --- | --- |
| Loss function | Defines what training is trying to reduce |
| Backpropagation | Computes gradients of that loss |
| Optimizer | Uses gradients to update parameters |
| Learning rate | Controls update size |

A typical training step therefore looks like:

```text
1. forward pass
2. calculate loss
3. backpropagate gradients
4. optimizer updates parameters
5. repeat
```

If you use PyTorch, calls such as `loss.backward()` correspond to step 3. An optimizer's `step()` corresponds to step 4.

The library automates the derivative bookkeeping, but the mathematical relationship is still the chain rule applied through the recorded computation.

## Why activations from the forward pass matter

Look again at one of our derivatives:

```text
dloss/dy = 2y
```

To evaluate it, we needed the forward value of `y`.

Many neural-network operations have backward calculations that depend on values produced during the forward pass. Frameworks therefore often save tensors or other context needed for backward computation.

This creates an important systems consequence: **training needs memory not only for model parameters but also for information required by the backward pass.**

That is one reason training a model generally requires substantially more memory than simply running it for inference.

Later techniques such as activation checkpointing trade additional computation for lower activation memory by recomputing some forward values instead of storing all of them.

You do not need that optimization yet, but it follows directly from understanding what backpropagation needs.

## Gradients can become too small or too large

Repeated chain-rule multiplication also explains two famous training problems.

If many local derivatives are smaller than 1, multiplying them through many layers can make an early gradient extremely small. This is related to the **vanishing gradient** problem.

If the products become very large, gradients can instead explode.

Modern architectures, initialization methods, normalization, residual connections, and optimization techniques help manage these effects. But the reason they can happen is visible in the basic equation:

```text
upstream gradient × local derivative
```

Do that many times and the scale of the result matters.

## What backpropagation does not mean

Backpropagation does not mean the model is literally sending its prediction backward through the network.

It also does not mean neurons are reversing their forward computation to reconstruct the input.

What travels backward is **derivative information**: how sensitive the final loss is to intermediate values and parameters.

The forward pass answers:

```text
Given these parameters and this input, what output do we get?
```

The backward pass answers:

```text
Given this loss, how sensitive is it to each trainable parameter?
```

Those are different calculations over the same dependency structure.

## The training loop now has all of its major pieces

Across the last few AI Foundations lessons, we have assembled a complete simplified training step:

```text
parameters
   ↓
forward pass
   ↓
prediction
   ↓
loss
   ↓
backpropagation
   ↓
gradients
   ↓
gradient-descent update
   ↓
new parameters
```

That closes the first major loop in our curriculum.

The next question is about what modern language models actually do with their inputs. Words are not directly fed into a transformer as English strings. The model needs numerical representations it can operate on.

Next we will move to **embeddings**: how discrete items such as tokens can be represented as learned vectors, and why nearby directions in a high-dimensional space can encode useful relationships.